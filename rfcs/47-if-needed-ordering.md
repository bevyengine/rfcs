# Feature Name: `if-needed-ordering`

## Summary

If-needed ordering constraints allow for safer, non-blocking constraints betweeen groups of systems.
If they would have no effect, they're completely ignored.
If they have an effect, they behave identically to strict ordering constraints.

## Motivation

The stageless rework ([RFC #45](https://github.com/bevyengine/rfcs/pull/45)) allows users to configure entire labels, creating ordering dependencies between entire subgraphs.
However, in many cases, pairs of systems ordered in this way have no logical connection between them (and do not even access the same data).

The current ordering, called strict ordering, will still force these pairs of systems to run according to the specified ordering, pointlessly restricting scheduling flexibility and thus performance.

## User-facing explanation

There are two basic forms of system ordering constraints:

1. **Strict ordering constraints:** `.strictly_before` and `.strictly_after`
   1. A system cannot be scheduled until "strictly before" systems have been completed during this iteration of the schedule.
   2. Simple and explicit.
   3. Can cause unnecessary blocking, particularly when systems are configured at a high-level.
2. **If-needed ordering constraints:** `.before` and `.after`
This conversation was marked as resolved by alice-i-cecile
   1. A system cannot be scheduled until any "before" systems that it is incompatible with have completed during this iteration of the schedule.
   2. In the vast majority of cases, this is the desired behavior. Unless you are using interior mutability (or accessing something outside of the ECS), systems that are compatible will always be **commutative**: their ordering doesn't matter, and these constraints behave "as-if" they were strict.
   3. Note that if-needed ordering constraints are transitive. `a.before(b)` and `b.before(c)` implies `a.before(c)`. This behavior, while intuitive, can have some unexpected effects: it occurs even if the chain is broken by a **spurious** constraint (in which the two systems are compatible).

### Spurious ordering constraints

While if-needed ordering dependencies can be useful for configuring the relative behavior of large groups of systems in an unrestrictive way,
their transitive behavior can lead to some silly results in cases where there's no need for ordering.

Suppose we have two very large, complex groups of systems: `Systems::Early` and `Systems::Late`.
They are well-mixed, and no ordering dependencies exist between the groups.
All is right with the world, and the systems live together in parallel-executing harmony.

Then one day, Bavy adds the following system to our app:

```rust
fn null_system(){}
app.add_system(null_system.after(Systems::Early).before(Systems::Late));
```

Seems innocuous, right?
The `null_system` doesn't access any data; it's trivially compatible with every other system and so the if-needed dependencies should never have any effect.
Except that's not true under Model 1.

Instead, attempting to maintain transitivity, we've induce an if-needed ordering dependency between *every* system in `Systems::Early` to *every* system in `Systems::Late`.
The entire schedule is bifurcated into two blocks, attempting to flow through a entirely pointless bottleneck.
To add insult to injury,`null_system` doesn't even *run* between these two blocks, instead executing at a completely arbitrary time as it has no observable effect.

While this example is deliberately absurd, such situations can naturally arise in real, complex code bases as they grow and are refactored.
Pointless ordering constraints hang around, as we cannot warn that they are useless, since they have non-local effects.
Eventually they become *load-bearing* spurious ordering constraints, and removing these if-needed ordering constraints which seemingly have no effect causes the high-level structure of your schedule to radically change, creating a massive collection of new system ordering ambiguities (and the corresponding non-local, non-deterministic bugs)!

As a result, the schedule will warn you each time you have a **spurious** ordering constraint: an if-needed ordering constraint that has no effect for any system that it is connecting.
Don't be like Bavy: try to clean these up ASAP, and then resolve any resulting execution order ambiguities with carefully thought-out constraints.

Like with execution order ambiguities, this behavior can be configured using the `SpuriousOrderingConstraints` resource.

```rust
enum SpuriousOrderingConstraints{
   /// The schedule will not be checked for spurious ordering constraints
   Allow,
   /// The details of each spurious ordering cosntraint is reported to the console
   Warn,
   /// The schedule will panic if an spurious ordering constraint is found
   Forbid,
}
```

## Implementation strategy

During schedule initialization, if-needed ordering constraints are either converted to strict ordering constraints, or removed.
They can be removed if and only if the schedule is *unobservably* different (ignoring interior mutability), regardless of the relative order of the two systems.

In the case where we only have two systems with an if-needed ordering, this is relatively simple.
If neither system can write to the data the other reads, we *cannot tell* which order they ran in, and so any ordering constraint between them is pointless.
This, of course, is equivalent to the two systems being compatible.

Subtly, this **hypothetical compatibility** (also used to determined if system parameters conflict or if schedules are ambiguous) must be determined statically: on the basis of the filtered access to component and resource types (via `FilteredAccessSet<ComponentId>`).
By contrast, **factual incompatibility** at the time of schedule execution is done based on the actual archetype-components that the systems access (via `Access<ArchetypeComponentId>`).
Just like strict ordering constraints though, ordering constraints inferred on the basis of if-needed ordering constraints are respected at the time of schedule execution, even if there is no data conflict at the time the systems are being run.

If-needed ordering constraints are intended to behave exactly like strict ordering constraints (unless interior mutability or other strangeness is involved), and so they are transitive.
In order to achieve this, the following algorithm is used:

1. For each system that is at the start of an if-needed constraint:
   1. Walk down the tree of if-needed ordering constraints and identify all downstream systems.
   2. Add an if-needed ordering constraint between the root system and each downstream system.
2. For each if-needed ordering constraint:
   1. If the two systems are compatible, remove the constraint.
   2. If they are incompatible, promote it to a strict ordering constraint.
3. Deduplicate all strict ordering constraints.
   1. Each pair of systems can only have one ordering constraint between them.
      1. At this stage, all ordering constraints have been converted to strict ordering constraints or removed.
   2. Any edges that are implied by the transitive property can be removed.

## Drawbacks

- If-needed ordering constraints will be more expensive to initialize, increasing the cost of any schedule intiialization or modification.

## Rationale and alternatives

### Why is if-needed the correct default ordering strategy?

Strict ordering is simple and explicit, and will never result in strange logic errors.
On the other hand, it *will* result in pointless and surprising blocking behavior, possibly leading to unsatisfiable schedules.

If-needed ordering is the correct strategy in virtually all cases: in Bevy, interior mutability at the component or resource level is rare, almost never needed and results in other serious and subtle bugs.

As we move towards specifying system ordering dependencies at scale, it is critical to avoid spuriously breaking users schedules, and silent, pointless performance hits are never good.

### ## Should if-needed ordering constraints be transitive?

If `A` is before `B`, and `B` is before `C`, must `A` be before `C`?

Naturally, this seems like it must be the case: strict ordering constraints are transitive, and time is well-ordered after all!
But if either the `A-B` or `B-C` edges are dissolved due to compatibility (and the `A-C` edge matters due to incompatibility), then `C` could be executed before `A`!

There are two possible models that we could handle this important edge case:

- **Model 1:** Infer if-needed-ordering constraints over each subgraph.
  - Transitivity holds, as expected.
  - Non-transitive if-needed ordering constraints cannot be represented.
  - Some computational overhead, as we must create, evaluate and then deduplicate if-needed ordering constraints over each subgraph.
- **Model 2:** Do nothing, report execution order ambiguities and allow the user to resolve this by adding additional constraints.
  - Reduces risk of accidentally creating unnecessary constraints.
  - More explicit, and thus more verbose.
  - Relies on users checking and resolving execution order ambiguities.

But which behavior is correct?
Consider the following concrete case:

We have five relevant systems:

1. `determine_player_movement`: reads `Input`, writes `PlayerIntent`
2. `collision_detection`: reads `Position` and `Velocity`, writes `Events<Collisions>`
3. `collision_handling`: reads `Events<Collision>`, writes `Velocity`
4. `apply_player_movement:` reads `PlayerIntent`, writes `Velocity`
5. `apply_velocity:` reads `Velocity`, writes `Position`

Suppose our user specifies the following if-needed ordering constraints:

1. `determine_player_movement` is before `collision_detection`
   1. Spurious: these systems are compatible.
2. `collision_detection` is before `collision_handling`
   1. Real: these systems conflict on `Events<Collisions>`
3. `collision_handling` is before `apply_velocity`
   1. Real: these systems conflict on `Velocity`
4. `apply_player_movement` is after `collision_handling`
   1. Spurious: these systems are compatible.
5. `apply_player_movement` is before `apply_velocity`
   1. Real: these systems conflict on `Velocity`

The problem here is that there's no direct link between `determine_player_movement` and `apply_player_movement`: they are ambiguous under Model 2, and the player may move according to stale input!

Under the transitive Model 1, additional if-needed constraints are created between `determine_player_movement` and `apply_player_movement` via the constraint chain (1, 2, 4).
Then, the inferred constraint between `determine_player_movement` and `apply_player_movement` is converted into a strict ordering constraint, as the systems are incompatible.
Finally, all redundant strict ordering constraints (such as between `collision_detection` and `apply_velocity`) are discarded for performance reasons.

Under Model 2, the behavior is much simpler: `determine_player_movement` and `apply_player_movement` are ambiguous.

Model 1 preserves the "logical" behavior, but in a very implicit fashion that is hard to reason about.
Model 2 simply causes the user's logic to break.

By adding powerful (and prominent) tools for detecting spurious ordering constraints and execution order ambiguities, we can follow the expected (and convenient) transitive property while helping users quash strange, fragile bugs as soon as they're introduced.

## Unresolved questions

None.

# Feature Name: `stageless`

## Summary

The existing stage boundaries result in a large number of complications: preventing us from operating across stages.
Moreover, several related features (run criteria, states and fixed timesteps) are rather fragile and overly complex.
This tangled mess needs to be holistically refactored: simplifying the scheduling model down into a single `Schedule`, moving towards an intent-based configuration style, taking full advantage of labels and leveraging exclusive systems to handle complex logic.

## Motivation

The fundamental challenges with the current stage-driven scheduling abstraction are [numerous](https://github.com/bevyengine/bevy/discussions/2801).
Of particular note:

- Dependencies do not work across stages.
- States do not work across stages.
- Plugins can add new, standalone stages.
- Run criteria (including states!) cannot be composed.
- The stack-driven state model is overly elaborate and does not enable enough use cases to warrant its complexity.
- Fixed timestep run criteria do not behave in a fashion that supports.
- The architecture for turn-based games and other even slightly unusual patterns is not obvious.
- We cannot easily implement a builder-style strategy for configuring systems.

Unfortunately, all of these problems are deeply interwoven. Despite our best efforts, fixing them incrementally at either the design or implementation stage is impossible, as it results in myopic architecture choices and terrible technical debt.

## User-facing explanation

This explanation is, in effect, user-facing documentation for the new design.
In addition to a few new concepts, it throws out much of the current system scheduling design that you may be familiar with.
The following elements are radically reworked:

- schedules (flattened)
- run criteria (can no longer loop, are now systems)
- states (simplified, no longer run-criteria powered)
- fixed time steps (no longer a run criteria)
- exclusive systems (no longer special-cased)
- command processing (now performed in a `flush_commands` exclusive system)
- labels (can now be directly configured)
- stages (yeet)
- system sets (yeet)

### Scheduling overview and intent-based configuration

Systems in Bevy are stored in a `Schedule`: a collection of **configured systems**.
Each frame, the `App`'s `runner` function will run the schedule,
causing the **scheduler** to run the systems in parallel using a strategy that respects all of the configured constraints (or panic, if it is **unsatisfiable**).

In the beginning, each `Schedule` is entirely unordered: systems will be selected in an arbitrary order and run if and only if all of the data that it must access is free.
Just like with standard borrow checking, multiple systems can read from the same data at once, but writing to the data requires an exclusive lock.
Systems which cannot be run in parallel are said to be **incompatible**.

While this is a good, simple strategy for maximizing parallelism, it comes with some serious drawbacks for reproducibility and correctness.
The relative execution order of systems tends to matter!
In simple projects and toy examples, the solution to this looks simple: let the user specify a single global ordering and use this to determine the order in the case of conflicts!

Unfortunately, this strategy falls apart at scale, particularly when attempting to integrate third-party dependencies (in the form of plugins), who have complex, interacting requirements that must be respected.
While users may be able to find a single working strategy through trial-and-error (and a robust test suite!), this strategy becomes ossified: impossible to safely change to incorporate new features or optimize performance.

The fundamental problem is a mismatch between *how* the schedule is configured and *why* it is configured that way.
The fact that "physics must run after input but before rendering" is implicitly encoded into the order of our systems, and changes which violate these constraints will silently fail in far-reaching ways without giving any indication as to what the user did wrong.

The solution is **intent-based configuration**: record the exact, minimal set of constraints needed for a scheduling strategy to be valid, and then let the scheduler figure it out!
As the number of systems in your app grows, the number of possible scheduling strategies grows exponentially, and the set of *valid* scheduling strategies grows almost as quickly.
However, each system will continue to work hand-in-hand with a small number of closely-related systems, interlocking in a straightforward and comprehensible way.

By configuring our systems with straightforward, local rules, we can allow the scheduler to pick one of the possible valid paths, optimizing as it sees fit without breaking our logic.
Just as importantly, we are not over-constraining our ordering, allowing subtle ordering bugs to surface quickly and then be stamped out with a well-considered rule.

### Introduction to system configuration

When a function is added to a schedule, it is turned into a `RawSystem` and combined with a `SystemConfig` struct to create a `ConfiguredSystem`.

A system may be configured in the following ways:

- it may have any number of **labels** (which implement the `SystemLabel` trait)
  - labels are used by other systems to determine ordering constraints
  - e.g. `.add_system(gravity.label(GameLogic::Physics))`
  - labels themselves may be configured, allowing you to define high-level structure in your `App` in a single place
    - e.g `app.configure_label(GameLogic::Physics, SystemConfig::new().label(GameLogic::Physics).after(GameLogic::Input))`
- it may have ordering constraints, causing it to run before or after other systems
  - there are several variations on this, see the next section for details
  - e.g. `.add_system(player_controls.before(GameLogic::Physics))`
- it may have one or more **run criteria** attached
  - a system is only executed if all of its run criteria returns `true`
  - **states** are a special, more complex pattern that use run criteria to determine whether or not a system should run in a current state
  - e.g. `.add_system(physics.run_if(GameLogic::Physics))`

Defining system configuration in a `SystemConfig` struct, can be useful to reuse, compose and quickly apply complex configuration strategies, before applying these strategies to labels or individual systems:

```rust
#[derive(SystemLabel)]
enum PhysicsLabel {
    Forces,
    CollisionDetection,
    CollisionHandling,
}

impl Plugin for PhysicsPlugin{

    fn build(app: &mut App){
        // Import all of the `PhysicsLabel` variants for brevity
        use PhysicsLabel::*;

        // Within each frame, physics logic needs to occur after input handling, but before rendering
        let mut common_physics_config = SystemConfig::new().after(CoreLabel::InputHandling).before(CoreLabel::Rendering);

        app
        // We can reuse this shared configuration on each of our labels
        .add_label(Forces.configure(common_physics_config).before(CollisionDetection))
        .add_label(CollisionDetection.configure(common_physics_config).before(CollisionHandling))
        .add_label(CollisionHandling.configure(common_physics_config))
        // And then apply that config to each of the systems that have this label
        .add_system(gravity.label(Forces))
        // These systems have a linear chain of ordering dependencies between them
        // Systems earlier in the chain must run before those later in the chain
        .add_system_chain([broad_pass, narrow_pass].label(CollisionDetection))
        .add_system(compute_forces.label(CollisionHandling))
    }
}
```

### System ordering

The most important way we can configure systems is by telling the scheduler *when* they should be run.
The ordering of systems is always defined relative to other systems: either directly or by checking the systems that belong to a label.

There are two basic forms of system ordering constraints:

1. **Strict ordering constraints:** `strictly_before` and `strictly_after`
   1. A system cannot be scheduled until "strictly before" systems must have been completed this frame.
   2. Simple and explicit.
   3. Can cause unneccessary blocking, particularly when systems are configured at a high-level.
2. **If-needed ordering constraints:** `.before` and `.after`
   1. A system cannot be scheduled until any "before" systems that it is incompatible with have completed this frame.
   2. In 99% of cases, this is the desired behavior. Unless you are using interior mutability, systems that are compatible will always be **commutative**: their ordering doesn't matter.

Applying an ordering constraint to or from a label causes a ordering constraint to be created between all individual members of that label.
If an ordering is defined relative to a non-existent system or label, it will have no effect, emitting a warning.
This relatively gentle failure mode is important to ensure that plugins can order their systems with relatively strong assumptions that the default system labels exist, but continue to (mostly) work if those systems or labels are not present.

In addition to the `.before` and `.after` methods, you can use **system chains** to create very simple linear dependencies between the succesive members of an array of systems.

When discussing system ordering, it is particularly important to call out the `flush_commands` system.
This **exclusive system** (meaning, it can modify the entire `World` in arbitrary ways and cannot be run in parallel with other systems) collects all created commands and applies them to the `World`.

Commands (commonly used to spawn and despawn entities or add and remove components) will not take effect until this system is run, so be sure that you run a copy of the `flush_commands` system before relying on the result of a command!

This pattern is so common that a special form of ordering constraint exists for it: **command-flushed ordering constraints**.
If system `A` is `before_and_flush` system `B`, the schedule will be unsatisfiable unless there is an intervening `flush_commands` system.
Note that **this does not insert new copies of a `flush_commands` system**: instead, it is merely used to verify that your schedule has been set up correctly according the specified constraint.

### Run criteria

While ordering constraints determine *when* a system will run, **run criteria** will determine *if* it will run at all.
A run criteria is a special kind of system, which can read data from the `World` and returns a boolean value.
If its output is `true`, the system it is attached to will run during this pass of the `Schedule`;
if it is `false`, the system will be skipped.
Systems that are skipped are considered completed for the purposes of ordering constraints.
If for some reason run criteria are themselves skipped, they are considered to have returned `true`.

Let's examine a few ways we can specify run criteria:

```rust
// This function can be used as a run criteria system,
// because it only reads from the `World` and returns `bool`
fn construction_timer_finished(timer: Res<ConstructionTimer>) -> bool {
    timer.finished()
}

fn main(){
    App::new()
    .add_plugins(DefaultPlugins)
    // We can add functions as run criteria
    .add_system(update_construction_progress.run_if(construction_timer_finished))
    // We can use closures for simple one-off run criteria, 
    // which automatically fetch the appropriate data from the `World`
    .add_system(spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000))
    // The `run_if_resource_is` method is convenient syntactic sugar that generates a run criteria
    // for when you want to check the value of a resource (commonly an enum)
    .add_system(gravity.run_if_resource_equals(Gravity::Enabled))
    // Run criteria can be attached to labels: a copy of the run criteria will be applied to each system with that label
    .configure_label(GameLabel::Physics, |label: impl SystemLabel| {label.run_if_resource_equals(Paused(false))} )
    .run()
}
```

**When multiple run criteria are attached to the same system, the system will run if and only if all of those run criteria return true.**

Run criteria are evaluated "just before" the system that is attached to is run.
The state that they read from will always be "valid" when the system that they control is being run: this is important to avoid strange bugs caused by race conditions and non-deterministic system ordering.
It is impossible, for example, for a game to be paused *after* the run criteria checked if the game was paused but before the system that relied on the game not being paused completes.
In order to understand exactly what this means, we need to understand atomic ordering constraints.

### Atomic ordering constraints

**Atomic ordering constraints** are a form of ordering constraints that ensure that two systems are run directly after each other.
At least, as far as "directly after" is a meaningful concept in a parallel, nonlinear ordering.
Like the atom, they are indivisible: nothing can get between them!

To be fully precise, if we have system `A` that runs "atomically before" system `B` (`.add_system(a.atomically_before(b))`):

- `A` is strictly before `B`
- the data locks (in terms of read and write accesses to the `World`) of system `A` will not be released until system `B` has completed
- the data locks from system `A` are ignored for the purposes of checking if system `B` can run
  - after all, they're not actualy locked, merely reserved
  - competing read-locks caused by other systems will still need to complete before a data store can be mutated; the effect is to "ignore the locks of `A`", not "skip the check completely"

In addition to their use in run criteria, atomic ordering constraints are extremely useful for forcing tight interlocking behavior between systems.
They allow us to ensure that the state read from and written to the earlier system cannot be invalidated.
This functionality is extremely useful when updating indexes (used to look up components by value):
allowing you to ensure that the index is still valid when the system using the index uses it.

Of course, atomic ordering constraints should be used thoughtfully.
As the strictest of the three types of system ordering dependency, they can easily result in unsatisfiable schedules if applied to large groups of systems at once.

### States

**States** are one particularly common and powerful form of run criteria, allowing you to toggle systems on-and-off based on the value of a given resource and smoothly handle transitions between states by running cleanup and initialization systems.
Typically, these are used for relatively complex, far-reaching facets of the game like pausing, loading assets or entering into menus.

The current value of each state is stored in a resource, which must derive the `State` trait.
These are typically (but not necessarily) enums, where each distinct state is represented as an enum variant.

Each state is associated with three sets of systems:

1. **Update systems:** these systems run each frame if and only if the value of the state resource matches the provided value.
   1. `app.add_system(my_system.in_state(GameState::Playing))`
2. **On-enter systems:** these systems run once when the specified state is entered.
   1. `app.add_system(my_system.on_enter(GameState::Playing))`
3. **On-exit systems:** these systems run once when the specified stated is exited.
   1. `app.add_system(my_system.on_enter(GameState::Playing))`

Update systems are by far the simplest: they're simply powered by run criteria.
`.in_state` is precisely equivalent to `run_if_resource_is`, except with an additional trait bound that the resource must implement `State`.

On-enter and on-exit systems are stored in dedicated schedules, two per state, within the `App`.
These schedules can be configured in all of the ordinary ways, but, as they live in different schedules, ordering cannot be defined relative to systems in the main schedule.

Due to their disruptive and far-reaching effects, state transitions do not occur immediately.
Instead, they are deferred (like commands), until the next `flush_state<S: State>` exclusive system runs.
This system first runs the `on_exit` schedule of the previous state on the world, then runs the `on_enter` schedule of the new state on the world.
Once that is complete, the exclusive system ends and control flow resumes as normal.

When states are added using `App::add_state::<S: State>(initial_state)`, one `flush_state<S>` system is added to the app, with the `GeneratedLabel::StateTransition<S>` label.
You can configure when and if this system is scheduled by configuring this label, and you can add additional copies of this system to your schedule where you see fit.
Just like with commands, **state-flushed ordering constraints** can be used to verify that state transitions have run at the appropriate time.
If system `A` is `before_and_flush_state::<S>` system `B`, the schedule will be unsatisfiable unless there is an intervening `flush_state<S>` system.

Apps can have multiple orthogonal states representing independent facets of your game: these operate fully independently.
States can also be defined as a nested enum: these work as you may expect, with each leaf node representing a distinct group of systems.
If you wish to share behavior among siblings, add the systems repeatedly to each sibling, typically by saving a schedule and then using `Schedule::merge` to combine that into the specialized schedule of choice.

### Complex control flow

#### Fixed time steps

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

## Drawbacks

1. This will be a massively breaking change to every single Bevy user.
2. It will be harder to immediately understand the global structure of Bevy apps.

## Rationale and alternatives

### Doesn't label-configuration allow you to configure systems added by third-party plugins?

Yes. This is, in fact, a large part of the point.

The complete inability to configure plugins is the source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
Plugin authors cannot know all of the needs of the end user's app, and so some degree of control must be handed back to the end users.

This control is carefully limited though: systems can only be configured if they are given a public label, and all of the systems under a common label are configured as a group.
In addition, configuration defined by a private label (or directly on systems) cannot be altered, extended or removed by the app: it cannot be named, and so cannot be altered.

These limitation allows plugin authors to carefully design a public API, ensure critical invariants hold, and make serious internal changes without breaking their dependencies (as long as the public labels and their meanings are relatively stable).

### Why can't you label labels?

If each label is associated with a `SystemConfig`, and part of this struct is which labels are applied, why can't we label labels themselves?

In short: this way lies madness.
While at first it may seem appealing to be able create nested hierarchies of labels, this ends up recreating many of the problems of a stage-based design's global perspective.
If labels can themselves cause other labels to be applied, it becomes dramatically harder to track how systems are configured, and why they are configured in that way.

This is a problem for users, as it makes code much harder to read, reason about and maintain.
It is also a problem for our tools, forcing a recursive architecture when generating schedules and limiting the ability for developer tools to communicate both the how and why of a system's configuration.

### Why is if-needed the correct default ordering strategy?

Strict ordering is simple and explicit, and will never result in strange logic errors.
On the other hand, it *will* result in pointless and surprising blocking behavior, possibly leading to unsatisfiable schedules.

If-needed ordering is the corret stategy in virtually cases: in Bevy, interior mutability at the component or resource level is rare, almost never needed and results in other serious and subtle bugs.
As we move towards specifying system ordering dependencies at scale, it is critical to avoid spuriously breaking users schedules, and silent, pointless performance hits are never good.

### Why can't we just use system chaining for run criteria?

There are two reasons why this doesn't work:

1. The system we're applying a run criteria does not have an input type.
2. System chaining does not work if the chained systems are incompatible. This is far too limiting.

### Why aren't run criteria cached?

In practice, almost all run criteria are extremely simple, and fast to check.
Verifying that the cache is still valid will require access to the data anyways, and involve more overhead than simple computations on one or two resources.

### Why do we want to store multiple schedules in the `App`?

We could store these in a resource in the `World`.
Unfortunately, this seriously impacts the ergonomics of running schedules in exclusive systems due to borrow checker woes.

Storing the schedules in the `App` alleviates this, as exclusive systems are now just ordinary systems: `&mut World` is compatible with `&mut Schedules`!

## Unresolved questions

- Should we allow users to compose run criteria in more complex ways?
  - How does this work for run criteria that are not locally defined?
- If we want to support a simple `add_system_chain` API as a precursor to a system graph API, what do we rename "system chaining" to?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.

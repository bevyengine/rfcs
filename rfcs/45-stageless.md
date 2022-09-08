# Feature Name: `stageless`

## Summary

Users often have a hard time working with Bevy's system scheduling API.
Core building blocks — stages, run criteria, and states — are presented as independent but actually have tons of hidden internal coupling.

This coupling frequently comes to bite users in the form of surprising limitations, unexpected side effects, and indecipherable errors.

This RFC proposes a holistic redesign that neatly fixes these problems, with clear boundaries between system configuration, storage, execution, and flow control.

In summary:

- Store systems in a central resource.
- Make system sets (sub)graph nodes instead of containers and include them in the descriptor API.
- Make exclusive systems "normal" and use them for high-level flow control. (command application, state transitions, fixed timestep, turn queues, etc.)
- Replace run criteria with immutable, `bool`-returning conditions.
- Remove stages.

## Motivation

There are [many standing issues](https://github.com/bevyengine/bevy/discussions/2801#discussioncomment-1304027) with the current stage-centered scheduling model.
Some highlights are, in no particular order:

- Users often have a hard time just deciding what stages to put systems in.
  - Can't just think about when commands should be applied, you also have to consider run criteria and if the stage loops.
- Plugins often export systems wrapped in stages, but users can't control where those imported stages go.
- Run criteria can't be composed in any appreciable way except "piping".
  - A system can't have multiple run criteria.
  - A system can't have a single run criteria if it belongs to a `SystemSet` that has one.
  - Users can't add a state-related `SystemSet` to multiple stages because its run criteria driver will think it's finished after the first one.
  - Users can't (really) mix a fixed timestep and state transitions (unless they involve a nested schedule, but more on that later).
- Users can't identify clear patterns for implementing "turn-based" game logic.
- Users don't (or can't) really take advantage of the capabilities that the extra complexity of our stack-based state model enables.
- There's just too much API. (e.g. "`.add_system_set_to_startup_stage`")

Unfortunately, these issues remain deeply ingrained and intertwined, despite our best efforts to surface and untangle them.

To give you an idea of the challenge.
If we removed stages, all places to evaluate run criteria and apply commands would be lost, except the beginning and end of each schedule execution.
If we required immutable access and `bool` output from run criteria to enable basic composition, states would break because their implementation relies on run criteria that mutate a resource.
Likewise, if we just took away stages and `ShouldRun::*AndCheckAgain`, there could be no inner frame loops (e.g. fixed timestep).

Addressing even one problem involves major breaking changes.
Ideally, we update everything in one go, as trying to spread the breaking changes over a longer period of time brings risk of adding even more technical debt.

## User-facing explanation

### Scheduling terminology and overview

Let's define some terms.

- **system**: stateful instance of a function that can access data stored in a world
- **system set**: logical group (subgraph) of systems (can include other system sets)
- **condition**: function that must evaluate to `true` for a system (or systems in a set) to run
- **dependency**: system (set) that must complete before another system (set) can run
- **schedule** (noun): the executable form of a system set
- **schedule** (verb): specify when and under what conditions systems run
- **executor**: runs the systems in a schedule on a world
- **"ready"**: when a system is no longer waiting for dependencies to complete

To write a Bevy app, you have to specify when your systems run.
By default, systems have neither strict execution order nor any conditions for execution.
**Scheduling** is the process of supplying those properties.
To make things more ergonomic, systems can be grouped under **system sets**, which can be ordered and conditioned in the same manner as individual systems.
This allows you to easily refer to many systems and (indirectly) give properties to many systems.
Furthermore, systems and system sets can be ordered together and even grouped together *within larger sets*, meaning you can layer those properties.

In short, for any system or system set, you can define:

- its execution order relative to other systems or sets (e.g. "this system runs before A")
- any conditions that must be true for it to run (e.g. "this system only runs if a player has full health")
- which set(s) it belongs to (which define properties that affect all systems underneath)
  - if left unspecified, the system or set will be added under a default one (this is configurable)

These properties are all additive, and properties can be added to existing sets.
Adding another does not replace an existing one, and they cannot be removed.
If incompatible properties are added, the schedule will panic at startup.

### Sample

```rust
use bevy::prelude::*;

#[derive(State)]
enum GameState {
    Running,
    Paused
}

#[derive(SystemLabel)]
enum MySystems {
    Update,
    Menu,
    SubMenu,
}

fn main() {
    App::new()
        /* ... */
        // If a set does not exist when `configure_set` is called,
        // it will be implicitly created.
        .configure_set(
            // Use the same builder API for scheduling systems and system sets.
            MySystems::Menu
                // Put sets in other sets.
                .in_set(MySystems::Update)
                // Attach conditions to system sets.
                // (If this fails, all systems in the set will be skipped.)
                .run_if(state_equals(GameState::Paused))
        )
        .add_system(
            some_system
                .in_set(MySystems::Menu)
                // Attach multiple conditions to this system, and they won't conflict with Menu's conditions.
                // (All of these must return true or this system will be skipped.)
                .run_if(some_condition_system)
                .run_if(|value: Res<Value>| value > 9000)
        )
        // Bulk-add systems and system sets with some convenience macros.
        .add_systems(
          chain![
            // Macros can also accept system sets.
            MySystems::SubMenu,
            // Choose when to process commands with instances of this dedicated system.
            apply_system_buffers,
          ]
          // Configure these together.
          .in_set(MySystems::Menu)
          .after(some_system)
        )
        /* ... */
        .run();
}
```

### Deciding when systems run with dependencies

The main way you can configure systems is to say *when* they should run, using the `.before`, `.after`, and `.in_set` methods.
These properties, called *dependencies*, determine execution order relative to other systems and system sets.
These dependencies are collected and assembled to produce dependency graphs, which, along with the signature of each system, tells the executor which systems can run in parallel.
Dependencies involving system sets are later flattened into dependencies between individual pairs of systems.

If a combination of constraints cannot be satisfied (e.g. you say A has to come both before and after B), a dependency graph will be found to be **unsolvable** and return an error. However, that error should clearly explain how to fix whatever problem was detected.

An `add_systems` method and convenience macros are provided as another means to add properties in bulk.
For example, using `systems![a, b, c, ...].chain()` (or the `chain![a, b, c, ...]` shortcut) will create dependencies between the successive elements.

*Note: This is different from the "system chaining" in previous versions of Bevy. That has been renamed to "system piping" to avoid overlap.*

Bevy's `MinimalPlugins` and `DefaultPlugins` plugin groups include several built-in system sets.

```rust
#[derive(SystemLabel)]
enum Physics {
    ComputeForces,
    DetectCollisions,
    HandleCollisions,
}

impl Plugin for PhysicsPlugin {
    fn build(app: &mut App){
        // configs are reusable
        let mut fixed_after_input = Config::new()
            // built-in fixed timestep system set
            .in_set(CoreSystems::FixedUpdate)
            .after(InputSystems::ReadInputHandling);

        app
            .configure_set(
                Physics::ComputeForces
                    .configure_with(fixed_after_input)
                    .before(Physics::DetectCollisions))
            .configure_set(
                Physics::DetectCollisions
                    .configure_with(fixed_after_input)
                    .before(Physics::HandleCollisions))
            .configure_set(Physics::HandleCollisions.configure_with(fixed_after_input))
            .add_system(gravity.in_set(Physics::ComputeForces))
            .add_systems(
                chain![broad_pass, narrow_pass, solve_constraints]
                    .in_set(Physics::DetectCollisions))
            .add_system(collision_damage.in_set(Physics::HandleCollisions));
    }
}
```

### Deciding if systems run with conditions

While dependencies determine *when* systems run, **conditions** determine *if* they run at all.

Functions with compatible signatures (immutable `World` data access and `bool` output) can be attached to systems and system sets as conditions.
A system or system set will only run if all of its conditions return `true`.
If one of its conditions returns `false`, the system (or members of the system set) will be skipped.

To be clear, systems can have multiple conditions, and those conditions are not shared with others.
Each condition instance is unique and will be evaluated *at most once* per run of a schedule.
Conditions are evaluated right before their guarded system (or the first system in their guarded system set that's able to run) would be run, so their results are guaranteed to be up-to-date.
The data read by conditions will not change before the guarded system starts.

```rust
// This is just an ordinary system: timers need to be ticked!
fn tick_construction_timer(timer: ResMut<ConstructionTimer>, time: Res<Time>){
    timer.tick(time.delta());
}

// This function can serve as a condition because it does not have mutable access and it returns `bool`.
fn construction_timer_not_finished(timer: Res<ConstructionTimer>) -> bool {
    !timer.finished()
}

// As long as they're immutable, you can use queries, event readers and resources with arbitrarily complex internal logic.
fn too_many_enemies(
    population_query: Query<(), With<Enemy>>,
    population_limit: Res<PopulationLimit>,
) -> bool {
    population_query.iter().count() > population_limit
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        // You can add functions with read-only system parameters as conditions.
        .configure_set(GameSystems::Construction.run_if(construction_timer_not_finished))
        .add_systems(
            systems![tick_construction_timer, update_construction_progress]
                .in_set(GameSystems::Construction))
        .add_system(mitigate_meltdown.run_if(too_many_enemies))
        // You can use closures for simple one-off conditions.
        .add_system(spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000))
        // `resource_exists` and `resource_equals` are helper functions that produce
        // new closures that can be used as conditions.
        .add_system(gravity.run_if(resource_exists(Gravity)))
        // The systems in this set won't run if `Paused` is `true`.
        .configure_set(GameSystems::Physics.run_if(resource_equals(Paused(false))))
        .run();
}
```

### Exclusive systems

As much as everyone loves to see systems running in parallel, sometimes a little `&mut World` is necessary.
Previous versions of Bevy treated these "exclusive" systems as a separate concept, with their own set of types and traits.
They couldn't have `Local` params and could only be inserted at specific points of a stage.

That is no longer the case.
Exclusive systems are now just regular systems.
They can have `Local` params and be scheduled wherever you want.

Command application is now signaled through an exclusive system called `apply_system_buffers`.
You can add instances of this system anywhere in your schedule.
If one system depends on the effects of commands from another, make sure an `apply_system_buffers` appears somewhere between them.

```rust
use bevy::prelude::*;

#[derive(SystemLabel)]
enum MySystems {
    /* ... */
    FlushProjectiles,
    /* ... */
}

struct ProjectilePlugin;

impl Plugin for ProjectilePlugin {
    fn build(app: &mut App) {
        app
        .add_system(spawn_projectiles.before(CoreSystems::FlushPostUpdate))
        .add_systems(
            chain![
                check_if_projectiles_hit,
                despawn_projectiles_that_hit,
                // Wherever you want commands to be applied, insert an instance of `apply_system_buffers`
                // Note: You cannot use type-derived labels for ordering if duplicates exist,
                // so give this system a name if you want to order relative to it elsewhere.
                apply_system_buffers.named(MySystems::FlushProjectiles)
                fork_projectiles,
            ]
            .after(CoreSystems::FlushPostUpdate)
            .before(CoreSystems::FlushLast)
        )
    }
}
```

### Running other systems on-demand with exclusive systems

Exclusive systems have other uses as well.

All systems and system sets added to an `App` are stored within a resource called `Systems`.
If you want to run a system or system set, you have to extract it from `Systems` first.
(This lets `Systems` remain accessible when those systems are running).

`Systems` has methods to extract a system or system set given its label.
When you extract a system set, you actually receive a **schedule**.
This is the executable version of the set, containing all its systems and conditions, along with instructions to run them in the correct order.
After running a schedule, you can return it to `Systems`.
A `World::schedule_scope` method is provided for this common "extract-run-return" case.

The default `App` runner itself actually uses these methods to run the startup sequence and main update loop, so if you retrieve and run a single system somewhere, you'll effectively have written a very basic executor!

```rust
fn fancy_exclusive_system(world: &mut World) {
    // removes schedule from Systems, executes fn with it, then returns it to Systems
    world.schedule_scope(ExampleLabel, |world, mut schedule| {
        schedule.run(world);
    });
}
```

This pattern is your go-to when your scheduling needs grow beyond "each system runs once per app update".
You might want to:

- repeatedly loop over a sequence of game logic several times in a single app update (e.g. fixed timestep, action queue)
- have complex branches in your schedule (e.g. state transitions)
- change to a different threading model for some portion of your systems
- integrate an external service with arbitrary side effects, like scripting or a web server

As long as you have `&mut World`, from an exclusive system or command, you can extract anything available in the `Systems` resource and run it.

Unlike in previous versions of Bevy, states and the fixed timestep are no longer powered by run criteria.
Instead, systems that are part of states or fixed time steps are simply added under a separate system set that will be retrieved and ran by an exclusive system.
As a result, you can transition states inside the fixed timestep without issue.

### Fixed timestep

A **fixed timestep** advances a fixed number of times each second.

It's implemented as an exclusive system that runs a schedule zero or more times in a row (depending on how long the previous app update took to complete).
When you supply a constant delta time value (the literal fixed timestep) inside the encapsulated systems, the result is consistent and repeatable behavior regardless of framerate (or even the presence of a GPU).

```rust
fn main() {
    App::new()
        .add_systems(
            // Conveneniently ordering these systems relative to each other
            chain!([
               apply_forces,
               apply_acceleration,
               apply_velocity
            ])
            // Ensuring they run on a fixed timestep
            // as part of the `CoreSystems::FixedUpdate` set
            .in_set(CoreSystems::FixedUpdate)
         )
        .run();
}
```

### States

**State machines** allow for ad-hoc execution of complex logic in response to state transitions.
They're often used for relatively high-level facets of app behavior like pausing, opening menus, and loading assets.
State machines are stored as individual resources and their state types must implement the `State` trait.
It's common to see enums used for state types.

You can have multiple independently controlled states, or defined nested states with enums.
The current state of type `S` can be read from the `CurrentState<S: State>` resource.

Bevy provides a simple but convenient abstraction to link the transitions of a state `S::Variant` with system sets, specifically an `OnEnter(S::Variant)` system set and an `OnExit(S::Variant)` system set.
An exclusive system called `apply_state_transition<S: State>` can be scheduled, which will retrieve and run these schedules as appropriate when a transition is queued in the `NextState<S: State>` resource.

```rust
fn main() {
    App::new()
        .add_state::<GameState>()
        /* ... */
        // These systems only run as part of the state transition logic
        .add_system(load_map.in_set(OnEnter(GameState::Playing)))
        .add_system(autosave.in_set(OnExit(GameState::Playing)))
        // This system will be added under the `OnUpdate(GameState::Playing)` set
        .add_system(run.in_set(OnUpdate(GameState::Playing)))
        // This system will not be part of the set above, but will otherwise follow the same rules
        .add_system(jump.run_if(state_equals(GameState::Playing)))
        /* ... */
        .run();
}
```

By default, all state transitions are evaluated just before `CoreSystems::Update`, in a `CoreSystems::ApplyStateTransitions` set.
Calling `App::add_state` adds the corresponding `CurrentState` and `NextState` resources, and registers a `apply_state_transition<S>` system.

## Implementation strategy

This design can be broken down into the following steps:

- Normalize exclusive systems.
  - Enable scheduling exclusive systems together with other systems.
    - Make `&mut World` a system param, so all systems can be handled together as `System` trait objects. ([bevyengine/bevy#4166](https://github.com/bevyengine/bevy/pull/4166))
  - Deal with the fact that archetypes can now change in between systems.
    - Defer tasks borrowing a system until just before that system runs (so we can update archetype component access last second).
      - Funnel `&Scope` into spawned tasks so a task can spawn other tasks. ([bevyengine/bevy#4466](https://github.com/bevyengine/bevy/pull/4466))
      - **Alternative**: Keep spawning tasks upfront but use channels to send systems into them and back (or cancel if skipped).
    - **Alternative**: Iterate the topsorted list of systems and use e.g. [`partition_point`](https://doc.rust-lang.org/std/primitive.slice.html#method.partition_point) to identify and execute slices of parallel systems.
- Implement conditions.
  - Define the trait (i.e. `IntoRunCondition`) and blanket implement it for compatible functions.
  - Implement `.run_if` descriptor method.
  - Include condition accesses when doing ambiguity checks.
  - Include condition accesses when executor checks if systems can run.
  - Add inlined condition evaluation step in the executor.
- Implement storing and retrieving systems (and schedules) from a resource.
  - `Schedules` is effectively `Hashmap<ScheduleLabel, Schedule>`
  - Each `Schedule` owns the graph data, and cross-schedule dependencies are impossible
  - Implement a descriptor coercion trait for `L: SystemLabel` types.
  - Implement the `Systems` type as described. (See **Appendix** or [this comment](https://github.com/bevyengine/bevy/pull/4090#issuecomment-1206585499) or [prototype PR impl](https://github.com/maniwani/bevy/blob/f5f80cd195b15d3912b4d90aade8750d8d1adc2e/crates/bevy_ecs/src/schedule_v3/mod.rs).)
- Remove internal uses of "looping run criteria".
  - Convert fixed timestep and state transitions into exclusive systems that each retrieve and run a schedule.
- **Port over the rest of the engine and examples to the new API.**
- Remove stages and run criteria.
- Misc.
  - Implement bulk scheduling capabilities (`.add_systems`, descriptor wrapper enum, and convenience macros like `chain!`)
    - Rebrand "system chaining" (e.g. `ChainSystem`) into something else, like "system piping".

There is a prototype implementation of these changes (in a separate module) here: bevyengine/bevy#4391 (The **Appendix** shows several snippets from this.)

### Migration strategy

Whatever the changes, migration will be a big challenge.
The proposed changes would affect the `App` and `Plugin` methods that all engine plugins and examples use, which will be a lot to review.
At the same time, a large number of users are eagerly awaiting changes of this nature making it into the engine.

If this RFC is merged, to expedite upstreaming while easing the burden on Bevy maintainers, we propose the following migration path with 3 phases:

1. Add all the new stuff in a new module (i.e. `bevy_ecs::stageless`) that will live alongside an unchanged `bevy_ecs::schedule`, unused (and not in the prelude).
   1. Implement a trait (i.e. `StagelessAppExt`) for `App` that can hook into the new module. Ultimately temporary.
   2. Publish a migration guide for the new API.
2. Port over our engine plugins to the new methods provided through `StagelessAppExt`.
   1. Deprecate the old module to make sure everything has been migrated.
3. Replace the old module with the new module.
   1. Remove the extension trait.
   2. Give the extension trait methods back their normal names.
   3. Verify migration guide.

Phase 1 will involve the bulk of the technical effort, and needs the closest reviews.
Phase 2 and 3 will be tedious and trivial and involve very large numbers of lines changed.
We will use a global code freeze during those steps to avoid merge conflicts, and merge Phase 2 and 3 ASAP.

## Drawbacks

1. Migration: These will be major breaking changes for every Bevy user.
    - Every app and plugin will need to migrate to the new API and it doesn't share much with the old API.
2. Letting exclusive systems out of their designated insertion points makes it easier to accidentally divide large groups of parallel systems.
3. A simple state machine is a reduction in functionality compared to the state stack.
    - The proposed `apply_state_transition::<S>` system performs one transition, not an unbounded cycle.
    - There are no equivalents to:
      - `on_inactive_update`
      - `on_in_stack_update`
      - `on_pause`
      - `on_resume`
      - etc.
4. It may become harder for users to understand their app at a glance.
    - Commands processing is more explicit but less structured.
    - The conditions that guard a system can become more complex when it belongs to multiple system sets (via nesting or intersection).
    - Clear, concise error messages and graph visualization tools will become even more important.

## Rationale and alternatives

### How are system sets any different from labels?

Formally speaking, a system set is a subgraph.
It's a node in the graph of its parent set(s) (unless it's a root).

We wanted to make it clear that labels are *names* for things and not *tags*.
You can name systems and system sets.

Today, if you add multiple labels to a system, the expectation is that each label is involved in at least one dependency, so you're actually positioning things, not categorizing them.
We have made this explicit.
You position things within sets (and because sets can be "nested", we've added depth).

Although users don't normally add labels and leave them "unconstrained", such a thing does have a direct analogue in this design.
You can simply include systems in additional sets that are decoupled from the ones you'd normally execute.
In that case, you'd be using "set" as a simple abstract group instead of a scoped dependency graph.

### If users can configure system sets, can they control where systems from third-party plugins are added?

To some extent, yes.

Not being able to schedule plugin systems is a source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
We thought plugins exporting system set labels and users deciding where to position them in their schedules makes for a better balance.
A popular physics plugin [has already implemented something like this](https://github.com/dimforge/bevy_rapier/pull/169) (partly [inspired](https://discord.com/channels/691052431525675048/742569353878437978/970075559679893594) by this effort).

There are three key rules for system configuration:

1. You can always add configuration (for any publically labeled set).
2. You can never remove configuration.
3. Contradictory configuration is invalid, even if it's locally consistent.

The first rule ensures that plugins can export flexible, reusable functionality, which can be adapted to fit into the user's app.
The second rule allows plugins to enforce critical internal invariants, even in the face of additional tweaking.
The final rule is a resolution mechanism: rather than trying to guess which rule should take priority, Bevy will fail quickly, clearly and early to allow programmers to detect and fix the underlying problem.

Plugin authors can choose whatever level of logical granularity they want to expose to you *and* still ensure that important logical invariants hold.
Users will only be able schedule systems (and sets of systems) at the level the plugin makes public.
And if their public API is relatively stable, plugins could even make large internal changes without breaking user apps.

### If exclusive systems can go anywhere, won't users have a harder time resolving ambiguity errors?

You *will* have to be more careful once exclusive systems can go anywhere.

That said, the dependency graph checker won't fail to detect pairs of conflicting + ambiguously-ordered things.
The checker can point out what's ambiguous before *and* after flattening dependency graphs.
(i.e. the checker can report that system A conflicts with system set X before reporting the specific pairs of conflicting systems).

We hope system sets will help reduce the cognitive burden here and naturally lead users to patterns with fewer errors.
Likewise, if users also become responsible for scheduling sets exported by plugins, it will be within their power to resolve any errors.

Still, we have identified follow-up work to improve this.

If you have two ambiguously-ordered system sets that conflict, you probably won't get the behavior you expect if all their systems just get mixed together.
[RFC #46](https://github.com/bevyengine/rfcs/pull/46) (which spun out of this one) aims to give system sets configurable atomicity to prevent this mixing from happening, although the ambiguity would still exist.

### Why is command application scheduled with an exclusive system?

Several earlier RFCs talked about expressing more exact dependencies, e.g. `B.after_buffers_of(A)`, where such graph edges could hypothetically be used to automatically determine when to apply commands, at a minimal number of points.
We don't know how to achieve that hypothetical (graph problems are difficult).
However, if contributors find a solution later, they will be able to build on top of this design.

Therefore, although `apply_system_buffers` might not be the best way to handle this, it will let users decide when commands are applied and ensure other systems aren't running when that happens.
We think that's good enough for now.

(Maybe instead of `apply_system_buffers` applying the pending commands of *all* systems in the current schedule, we could construct `apply_system_buffers` from a tuple of system (set) labels, such that only those have their commands applied.
It's debatable whether that would increase or reduce cognitive burden.
That can be revisited later too.)

### Why did you change states from a stack to a basic FSM?

If this RFC is merged and this design is implemented, we expect most of the problems that encouraged it to vanish.
So rather than port pieces of the existing API, we think it's better to start fresh with a basic state machine and see what patterns emerge and what their limits are.
If, after migration, a significant number of users are still turning towards a plugin to recover the lost stack functionality, we can consider upstreaming it again.

A stack-based state machine should be trivial to implement though (i.e. with a loop inside the exclusive system).

### Why is using type-derived labels (`AsSystemLabel`) forbidden when there are duplicate instances of systems?

If you schedule multiple instances of a system, the one thing we *never* want to do is group them together without your explicit say-so.
To explain why, let's consider one common and significant case: `apply_system_buffers`.

Suppose a user has written the following:

```rust
fn main() {
    app
        .add_system(apply_system_buffers)
        .add_system(generates_commands.before(apply_system_buffers))
        .add_system(relies_on_commands.after(apply_system_buffers))
        .run()
}
```

The user has added three systems and, naively, created dependencies using the `apply_system_buffers` type label.

Since there's only one instance of `apply_system_buffers`, this app works like you'd expect.
But now suppose that the user adds more anonymous instances of `apply_system_buffers` or imports plugin logic that does.

Let's say the user imports a system set `X` from some plugin and `X` has another anonymous instance of `apply_system_buffers`.
If the user schedules `X` to run before (or after) their logic, their program will panic over an unsolvable dependency graph.

```rust
fn main() {
    app
        .configure_set(X.before(generates_commands))
        .add_system(apply_system_buffers)
        .add_system(generates_commands.before(apply_system_buffers))
        .add_system(relies_on_commands.after(apply_system_buffers))
        // This will panic.
        .run()
}
```

What happened here?

Well, the problem is a type-derived label will name *all* anonymous systems of that type.
Therefore, `before(apply_system_buffers)` means "before all anonymous instances of `apply_system_buffers`" even though the user was most likely only thinking about the one they added themselves.
Totally obvious, right? (/s)

Their program panicked because it saw a contradiction: `generate_commands` had to run before all `apply_system_buffers` instances and run after the one inside `X` at the same time.
The only (reliable) solution here is for the user to give their instance of `apply_system_buffers` a unique name and use that name for ordering.

So while it's natural to think of systems as "sets of one", treating their type-derived labels as system set names is subtly different and doing so would virtually guarantee errors when multiple copies exist, so we just won't allow it.

Now, we came up with three options to actually enforce that as an invariant:

1. Require users to name duplicate systems.
2. Leave duplicate systems completely anonymous. Only give the type-derived name to the first copy.
3. Error when multiple anonymous copies of a system exist and their type-derived label is used for ordering.

(1) ensures correctness and an easier debugging experience, but is too inconvenient.
(2) drops the naming requirement without losing correctness, but makes things too unclear.
(3) just makes doing the wrong thing an error.

We like (3) because it doesn't put undue burden on the user or introduce unclear implicit behavior.
Likewise, resolving the error is very simple: just name the systems.

## Unresolved questions

### What's a good convention for naming system label types?

What convention would we have for labeling internal systems and system sets?

- Would a plugin have one enum type for systems and another for system sets, or generally one enum for both?
  - If one enum for both, how would we typically name it? e.g. `CoreLabel`, `CoreSystems`, etc.

### What sugar should we use for adding multiple systems at once?

Convenience methods like `add_systems` are important for reducing boilerplate. For these methods, we need to be able to refer to collections of systems and system sets.

**In summary:**

- builder syntax: marginal improvement over adding individual systems
- array syntax: looks pretty but is literally impossible
- tuple syntax: looks pretty but is limited to 12 elements
- `vec!`-like macro syntax: looks OK but, eh, uses macros

**Conclusion:** macro syntax for practicality, likely powered by builder syntax under the hood

Below, we examine the the options by example.

#### Builder pattern

This was the strategy we used for `SystemSet`.

```rust
.add_systems(
    Nodes::new()
        .with(compute_attack), 
        .with(compute_defense),
        .with(check_for_crits),
        .with(compute_damage),
        .with(deal_damage),
        .with(check_for_death)
        .chain()
        .in_set(GameSet::Combat)
    )

.add_systems(Nodes::new().with(run).with(jump).after(InputSet::ReadInput))
```

- looks bad
  - not much better than adding things individually
  - confusing placement of brackets
  - no clear seperation between systems and the group config
- no limit to number of elements

#### Arrays

```rust
.add_systems([
        compute_attack, 
        compute_defense,
        check_for_crits,
        compute_damage,
        deal_damage,
        check_for_death,
    ]
    .chain()
    .in_set(GameSet::Combat))

.add_systems([run, jump].after(InputSet::ReadInput))
```

- looks pretty
- no limit to number of elements
- doesn't work (different functions are different types, arrays must be homogeneous)

#### Tuples

```rust
.add_systems((
        compute_attack, 
        compute_defense,
        check_for_crits,
        compute_damage,
        deal_damage,
        check_for_death
    )
    .chain()
    .in_set(GameSet::Combat)
)

.add_systems((run, jump).after(InputSet::ReadInput))
```

- looks pretty
- limited to 12 elements
- increases compile times
- relies on type system magic
  - requires invocation of the `all_tuples` macro to generate blanket impls
  - (kinda cursed, but already used for system params, systems, and bundles)

#### `vec!`-like macros

This strategy is used for the `vec![1,2,3]` macro in the standard library.

```rust
.add_systems(systems![
      compute_attack, 
      compute_defense,
      check_for_crits,
      compute_damage,
      deal_damage,
      check_for_death
   ]
   // can also use `chain![...]` for slightly more compact shortcut
   .chain() 
   .in_set(GameSet::Combat)
)

.add_systems(systems![run, jump].after(InputSet::ReadInput))
```

- looks OK (some boilerplate)
- no limit to number of elements
- relies on regular, declarative macro magic

## Future possibilities

There are a few proposals that were directly split from this RFC:

1. Unnecessary dependency removal ([RFC #47](https://github.com/bevyengine/rfcs/pull/47)).
2. Configurable system set atomicity ([RFC #46](https://github.com/bevyengine/rfcs/pull/46)).
3. Automatic (opt-in) insertion of command application and state transition systems (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
4. "Command-flushed" ordering constraints (dependencies that will ensure the pair is separated by command application).

There are also lots of related but non-urgent areas for follow-up research:

1. Improve tools for reporting and resolving system execution order ambiguities.
2. Revisit plugin definition and plugin configuration.
3. Cloning registered systems. (This probably warrants its own RFC, but it fits with the multiple worlds discussion.)
4. Add [manual dependency graph construction](https://github.com/bevyengine/bevy/pull/2381) methods.
5. Seeded, fully deterministic system execution orders: useful for debugging system order bugs and optimization work.
6. Move command queues into the `World` so that exclusive systems can drain them directly (e.g. in some user-defined order).
7. Compose conditions via arbitrary boolean expressions.
8. Support automatic insertion and removal of systems to reduce schedule clutter and support other forms of one-off logic.
9. Run schedules without `&mut World`, inferring access based on the systems inside.
10. [One-shot systems](https://github.com/bevyengine/bevy/pulls?q=is%3Apr+is%3Aopen+one-shot+system), where systems are evaluated one-at-a-time on an ad hoc basis.

## Appendix

These implementation ideas/details are here to avoid cluttering the RFC.

### `apply_system_buffers`

As command queues are owned by their systems, an exclusive system won't be able to drain their buffers by itself.
This one just signals the executor to do it.
The caveat to handling it this way is that the executor is only aware of the systems in the schedule it's running, so we should probably still have one "courtesy flush" after all the systems complete, so users can't forget to.

### `Systems`

`Systems` is a resource on the `World`.

To keep `Systems` accessible, you have to take systems and schedules out of `Systems` in order to run them.
Likewise, system insertion and removal will not immediately cause schedules to be rebuilt.
Those changes will only be propagated up to flag the system sets whose dependency graphs need to be updated (topologically sorted again).
A rebuild will remain pending until something tries to check out the `Schedule`.
This way users only incur costs for schedules they use.

Schedule extraction moves the required systems and conditions from their long-term storage (`Option`) into the selected `Schedule` container and returns it.

Even after systems or schedules have been extracted, all the graph data remains in `Systems`, so you can still add and remove things.
You still have to return extracted schedules to `Systems` in order to update them (since that process needs the systems), but if you had removed any of those systems, they will be safely dropped.

```rust
enum NodeId {
    System(u64),
    Set(u64),
}

#[derive(Resource)]
pub struct Systems {
    // `NodeId` is unique identifier, generated on insertion
    index: HashMap<SystemLabelId, NodeId>,
    next_id: u64,

    // long-term storage
    graphs: HashMap<NodeId, DependencyGraph>,
    systems: HashMap<NodeId, Option<BoxedSystem>>,
    conditions: HashMap<NodeId, Option<Vec<BoxedRunCondition>>>,
    schedules: HashMap<NodeId, Option<Schedule>>,

    // used for runtime modification of dependency graphs
    hierarchy: DiGraph<NodeId>,
    uninit_nodes: Vec<(NodeId, Config)>,
}

struct DependencyGraph {
    // the dependency graph
    base: DiGraph<NodeId>,
    // recursively-flattened version of `base` (has only system nodes and system-system edges)
    flat: DiGraph<NodeId>,
    // cached topological ordering of `flat`
    topsorted: Vec<NodeId>,
    // the union of the component access of every node inside this set
    combined_access: Access<ComponentId>,
    // `Graph` if the graph (and schedule) need to be rebuilt
    // `ScheduleOnly` if graph is up-to-date, but the schedule needs to be rebuilt
    // `None` otherwise
    rebuild: Rebuild,
}

pub struct Schedule {
    // All vectors are sorted in topological order.
    // NOTE: `RefCell<T>` could also be `Cell<Option<T>>`
    systems: Vec<RefCell<BoxedSystem>>,
    system_conditions: Vec<RefCell<Vec<BoxedRunCondition>>>,
    set_conditions: Vec<RefCell<Vec<BoxedRunCondition>>>,
    // for returning things back to `Systems`
    system_ids: Vec<NodeId>,
    set_ids: Vec<NodeId>,
    // for obeying dependencies
    system_deps: Vec<(usize, Vec<usize>)>,
    // for evaluating set conditions and skipping systems
    sets_of_systems: Vec<FixedBitSet>,
    sets_of_sets: Vec<FixedBitSet>,
    systems_of_sets: Vec<FixedBitSet>,
}
```

### Using descriptor API for both systems and system sets

It's fairly straightforward to register pure labels as system sets using the same API.

As a bonus, schedule construction will become completely order-independent as everything will be deferred until the end.

```rust
pub trait IntoConfiguredSystem<Params> {
    fn configure(self) -> ConfiguredSystem;
    fn configure_with(self, config: Config) -> ConfiguredSystem;
    fn before<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystem;
    fn after<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystem;
    fn in_set(self, set: impl SystemLabel) -> ConfiguredSystem;
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSystem;
}

pub trait IntoConfiguredSystemSet {
    fn configure(self) -> ConfiguredSystemSet;
    fn configure_with(self, config: Config) -> ConfiguredSystemSet;
    fn before<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystemSet;
    fn after<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystemSet;
    fn in_set(self, set: impl SystemLabel) -> ConfiguredSystemSet;
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSystemSet;
}

enum Order {
    Before,
    After
}

struct Config {
    sets: HashSet<SystemLabelId>,
    dependencies: Vec<(Order, SystemLabelId)>,
    conditions: Vec<BoxedRunCondition>,
}

struct ConfiguredSystem {
    system: BoxedSystem,
    config: Config,
    instance_name: SystemLabelId,
}

struct ConfiguredSystemSet {
    set: SystemLabelId,
    config: Config,
}
```

### Schedule building errors

When prototyping, you'll likely encounter schedule build errors.
Most of these will point out a reason why some graph is unsolvable.

There are two kinds of graphs the internal builder checks.

The first kind consists of the `.in_set` relationships.
These form a graph describing the hierachy of all your system sets.
The second kind consists of the `.before` and `.after` relationships.
Those form one or more dependency graphs (one for each set/schedule).
Dependencies involving system sets will also be flattened into dependencies on the systems in those sets.

So what are the errors?

1. A dependency graph contains a loop or cycle.
2. The hierarchical graph contains a loop or cycle.
3. The hierarchical graph has a transitive edge. Example:

    ```rust
    app
        .configure_set(A);
        .configure_set(B.in_set(A));
        // The next line causes a panic during graph solving.
        // `A` cannot be a parent and grandparent to `C` at same time. 
        .configure_set(C.in_set(B).in_set(A))
    ```

4. You called `.in_set` with a label that belongs to a system, rather than a system set.
5. (Optional) You have a dependency between two things that aren't siblings in a common set. That edge will not appear in either set's dependency graph, only in the flattened graph of an overarching set. This can lead to unwanted implicit ordering between systems in different sets.
6. (Optional) You referenced an "unknown" label. e.g. `.after(label)` references a label that doesn't belong to any known system or system set.
7. (Optional) You have at least one pair of ambiguously-ordered systems with conflicting data access.

(5), (6), and (7) don't inherently make a graph unsolvable, so they can be configured as ignore, warn, or error.
By default, they are all configured as warn.
(7) has additional configuration options.
See [bevyengine/bevy#4299](https://github.com/bevyengine/bevy/pull/4299) for more details.

### `IntoRunCondition`

```rust
pub trait IntoRunCondition<Params>: IntoSystem<(), bool, Params> {}

impl<Params, Marker, F> IntoRunCondition<(IsFunctionSystem, Params, Marker)> for F
where
    F: SystemParamFunction<(), bool, Params, Marker> + Send + Sync + 'static,
    Params: SystemParam + 'static,
    Params::Fetch: ReadOnlySystemParamFetch,
    Marker: 'static,
{
}
```

Any function that satisfies these constraints can be used as a condition:

- It must return a `bool`.
- It cannot have system params that grant mutable access to data.
  - **Reason**: If something has multiple conditions, the order they're evaluated in won't matter.

The following is also strongly-recommended:

- It shouldn't rely on local data (e.g. `Local<T>` or change detection queries).
  - **Reason**: Again, the order won't matter because they'll all be idempotent (otherwise, you can encounter subtle bugs).

### Evaluating conditions in the proper order

When the executor is presented with a ready system, it does the following:

- Check if access to all the data needed by the system, its conditions, and the conditions of the sets it's under (that haven't been evaluated yet) is available.
  - If no, leave the system in the ready queue and move onto the next one.
- If yes, evaluate the conditions of those not-yet-evaluated sets in hierarchical order (from the outermost to the innermost).
  - If any set's conditions return `false`, mark all the systems (incl. current one) under it as completed and all sets under it as evaluated.
  - If all of the set's conditions return `true`, mark the set as evaluated.
- If all those conditions returned `true`, now evaluate the system's conditions.
  - If any of these return `false`, mark the system as completed.
- If all the system's conditions returned `true`, run the system.
  - Mark access to the data required by the system as unavailable.
  - Create a task for the system.
  - Spawn and run that task.
- When the system task completes, refresh the available access.

Each condition is evaluated at most once.
Since most of these functions are probably simple tests, we don't spawn tasks for them, which avoids locking other systems out of the data they access.
Likewise, we don't spawn tasks for systems that get skipped. This lightweight approach minimizes task overhead.

### Implementing states

The implementation of a basic state machine can be very simple in this new scheme.

```rust
#[derive(Resource)]
pub struct CurrentState<S: State>(pub S);

impl<S: State> CurrentState<S> { /* ... */ }

#[derive(Resource)]
pub struct NextState<S: State>(pub Option<S>);

impl<S: State> NextState<S> { /* ... */ }

#[derive(SystemLabel)]
pub struct OnEnter<S: State>(S);

#[derive(SystemLabel)]
pub struct OnExit<S: State>(S);

// add (multiple instances of) this system to your schedule
pub fn apply_state_transition<S: State>(world: &mut World) {
    world.resource_scope(|world, mut current_state: Mut<CurrentState<S>>| {
        // (just assume these resources have Deref/DerefMut impls)
        if world.resource::<NextState<S>>().is_some() {
            // update the state machine
            let new = world.resource_mut::<NextState<S>>().take().unwrap();
            let old = current_state.replace(new);
            // and run the transitions
            world.schedule_scope(OnExit(old), |world, mut schedule| {
                schedule.run(world);
            });
            world.schedule_scope(OnEnter(new), |world, mut schedule| {
                schedule.run(world);
            });
        }
    });
}
```

### Change detection

Change detection requires a periodic (but very, *very* infrequent) clamping process to stay reliable.
Prior versions of Bevy check if it's time to clamp at the end of each stage.
With stages gone, we'll have to check somewhere else, most likely in the executor, at the beginning of each schedule.

### Convenience functions

We can export some helper functions to create really common conditions.

```rust
pub fn resource_exists<T>() -> impl FnMut(Option<Res<T>>) -> bool
where
    T: Resource,
{
    move |res: Option<Res<T>>| res.is_some()
}

pub fn resource_equals<T>(value: T) -> impl FnMut(Res<T>) -> bool
where
    T: Resource + PartialEq,
{
    move |res: Res<T>| *res == value
}

pub fn resource_exists_and_equals<T>(value: T) -> impl FnMut(Option<Res<T>>) -> bool
where
    T: Resource + PartialEq,
{
    move |res: Option<Res<T>>| match res {
        Some(res) => *res == value,
        None => false,
    }
}

pub fn non_send_resource_exists<T>() -> impl FnMut(Option<NonSend<T>>) -> bool
where
    T: Resource,
{
    move |res: Option<NonSend<T>>| res.is_some()
}

pub fn non_send_resource_equals<T>(value: T) -> impl FnMut(NonSend<T>) -> bool
where
    T: Resource + PartialEq,
{
    move |res: NonSend<T>| *res == value
}

pub fn non_send_resource_exists_and_equals<T>(value: T) -> impl FnMut(Option<NonSend<T>>) -> bool
where
    T: Resource + PartialEq,
{
    move |res: Option<NonSend<T>>| match res {
        Some(res) => *res == value,
        None => false,
    }
}

pub fn state_exists<S: State>() -> impl FnMut(Option<Res<CurrentState<S>>>) -> bool {
    move |current_state: Option<Res<CurrentState<S>>>| current_state.is_some()
}

pub fn state_equals<S: State>(state: S) -> impl FnMut(Res<CurrentState<S>>) -> bool {
    move |current_state: Res<CurrentState<S>>| *current_state == state
}

pub fn state_exists_and_equals<S: State>(
    state: S,
) -> impl FnMut(Option<Res<CurrentState<S>>>) -> bool {
    move |current_state: Option<Res<CurrentState<S>>>| match current_state {
        Some(current_state) => *current_state == state,
        None => false,
    }
}
```

# Feature Name: `stageless`

## Summary

Users often have a hard time working with Bevy's system scheduling API.
Core building blocks — stages, run criteria, and states — are presented as independent but actually have tons of hidden internal coupling.

This coupling frequently comes to bite users in the form of surprising limitations, unexpected side effects, and indecipherable errors.

This RFC proposes a holistic redesign that neatly fixes these problems, with clear boundaries between system configuration, storage, execution, and flow control.

In summary:

- Store schedules in a resource.
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
- **schedule** (noun): an executable system graph
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

#[derive(SystemSet)]
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
        .add_systems(
            some_system
                .in_set(MySystems::Menu)
                // Attach multiple conditions to this system, and they won't conflict with Menu's conditions.
                // (All of these must return true or this system will be skipped.)
                .run_if(some_condition_system)
                .run_if(|value: Res<Value>| value > 9000)
        )
        // Bulk-add systems
        .add_systems(
          chain![
            some_system,
            // Choose when to process commands with instances of this dedicated system.
            apply_system_buffers,
          )
          .chain()
          // Configure these together.
          .in_set(MySystems::Menu)
          .after(some_other_system)
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

An `add_systems` method is provided as another means to add properties in bulk, which accepts a collection of configured systems.
For example, tuples of systems like `(a, b, c)` can be added, and using `(a, b, c, ...).chain()` will create dependencies between the successive elements.

*Note: This is different from the "system chaining" in previous versions of Bevy. That has been renamed to "system piping" to avoid overlap.*

Bevy's `MinimalPlugins` and `DefaultPlugins` plugin groups include several built-in system sets.

```rust
#[derive(SystemSet)]
enum Physics {
    ComputeForces,
    DetectCollisions,
    HandleCollisions,
}

/// "Logical" system set for sharing fixed-after-input config
#[derive(SystemSet)]
FixedAfterInput;

impl Plugin for PhysicsPlugin {
    fn build(app: &mut App){
        app
            .configure_set(
                FixedAfterInput
                    .in_set(CoreSystems::FixedUpdate)
                    .after(InputSystems::ReadInputHandling))
            .configure_set(
                Physics::ComputeForces
                    .in_set(FixedAfterInput)
                    .before(Physics::DetectCollisions))
            .configure_set(
                Physics::DetectCollisions
                    .in_set(FixedAfterInput)
                    .before(Physics::HandleCollisions))
            .configure_set(Physics::HandleCollisions.in_set(FixedAfterInput))
            .add_system(gravity.in_set(Physics::ComputeForces))
            .add_systems(
                (broad_pass, narrow_pass, solve_constraints)
                    .chain()
                    .in_set(Physics::DetectCollisions))
            .add_systems(collision_damage.in_set(Physics::HandleCollisions));
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
            (tick_construction_timer, update_construction_progress)
                .in_set(GameSystems::Construction))
        .add_systems((
            mitigate_meltdown.run_if(too_many_enemies),
            // You can use closures for simple one-off conditions.
            spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000),
            // `resource_exists` and `resource_equals` are helper functions that produce
            // new closures that can be used as conditions.
            gravity.run_if(resource_exists(Gravity))
        ))

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

#[derive(SystemSet)]
enum MySystems {
    /* ... */
    FlushProjectiles,
    /* ... */
}

struct ProjectilePlugin;

impl Plugin for ProjectilePlugin {
    fn build(app: &mut App) {
        app
        .add_systems(spawn_projectiles.before(CoreSystems::FlushPostUpdate))
        .add_systems(
            (
                check_if_projectiles_hit,
                despawn_projectiles_that_hit,
                // Be mindful when adding exclusive systems like apply_system_buffers to your schedule.
                // These will create a hard sync point, blocking other systems from running in parallel.
                apply_system_buffers,
                fork_projectiles,
            )
            .chain()
            .after(CoreSystems::FlushPostUpdate)
            .before(CoreSystems::FlushLast)
        )
    }
}
```

### Running other schedules on-demand with exclusive systems

Exclusive systems can be used to run other schedules in an on-demand fashion.
Each system is stored within a `Schedule`, which themselves live within a public `Schedules` resource.

```rust
fn fancy_exclusive_system(world: &mut World) {
    // removes the selected schedule from Schedules, executes a fn with it, then returns it to the world
    while fancy_logic() {
        // This runs the entire schedule on the world
        world.run_schedule(MySchedule);
    });
}
```

This pattern is your go-to when your scheduling needs grow beyond "each system runs once per app update".
You might want to:

- repeatedly loop over a sequence of game logic several times in a single app update (e.g. fixed timestep, action queue)
- have complex branches in your schedule (e.g. state transitions)
- change to a different threading model for some portion of your systems
- integrate an external service with arbitrary side effects, like scripting or a web server

As long as you have `&mut World`, from an exclusive system or command, you can retrieve and run an available schedule.

Unlike in previous versions of Bevy, states and the fixed timestep are no longer powered by run criteria.
Instead, systems that are part of states or fixed time steps are simply added under a separate schedule that will be retrieved and ran by an exclusive system.
As a result, you can transition states inside the fixed timestep without issue.

### Fixed timestep

One built-in example of this pattern is a **fixed timestep**, which advances a fixed number of times each second.

This is implemented as an exclusive system that runs a schedule zero or more times in a row (depending on how long the previous app update took to complete).
When you supply a constant delta time value (the literal fixed timestep) inside the encapsulated systems, the result is consistent and repeatable behavior regardless of framerate (or even the presence of a GPU).

```rust
fn main() {
    App::new()
        // Ensuring they run on a fixed timestep
        // as part of the `CoreSchedule::FixedUpdate` schedule
        .add_systems_to(
            CoreSchedule::FixedUpdate,
            (
               apply_forces,
               apply_acceleration,
               apply_velocity
            )
            // Conveneniently ordering these systems relative to each other
            .chain()
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
The current state of type `S` can be read from the `CurrentState<S: State>` resource,
while the upcoming state can be read from `NextState<S: State>`.

Bevy provides a simple but convenient abstraction to link the transitions of a state `S::Variant` with run-on-demand schedules, specifically an `OnEnter(S::Variant)` and `OnExit(S::Variant)` schedules.
An exclusive system called `apply_state_transition<S: State>` can be scheduled, which will retrieve and run these schedules as appropriate when a transition is queued in the `NextState<S: State>` resource.

```rust
fn main() {
    App::new()
        .add_state::<GameState>()
        /* ... */
        // These systems only run as part of the state transition logic
        .add_system_to(OnEnter(GameState::Playing), load_map)
        .add_systems((
            // The .on_enter and .on_exit methods are equivalent to the pattern above
            autosave.on_exit(GameState::Playing),
            // This system will be added under the `OnUpdate(GameState::Playing)` set
            // Note that this is not a schedule:
            // it is run as part of the main game schedule,
            // but only runs if the current state is GameState::Playing
            run.in_set(OnUpdate(GameState::Playing)),
            // This system will not be part of the set above, but will otherwise follow the same rules
            jump.run_if(state_equals(GameState::Playing)),
        ))
        /* ... */
        .run();
}
```

Calling `App::add_state<S>`:

1. Adds the corresponding `CurrentState<S>` and `NextState<S>` resources
2. Adds a `OnUpdate(S::Variant)` set for each valid state. These sets are nested inside of `CoreSystems::Update` by default.
3. Registers a `apply_state_transition<S>` system, which is evaluated just before `CoreSystems::Update`, in a `CoreSystems::ApplyStateTransitions` set..

## Implementation strategy

This design can be broken down into the following steps:

- Normalize exclusive systems.
  - Enable scheduling exclusive systems together with other systems.
    - Make `&mut World` a system param, so all systems can be handled together as `System` trait objects. ([bevyengine/bevy#4166](https://github.com/bevyengine/bevy/pull/4166))
  - Deal with the fact that archetypes can now change in between systems (as there's no longer a clear line between exclusive and parallel systems).
    - Defer tasks borrowing a system until just before that system runs (so we can update archetype component access last second).
      - Funnel `&Scope` into spawned tasks so a task can spawn other tasks. ([bevyengine/bevy#4466](https://github.com/bevyengine/bevy/pull/4466))
      - **Alternative**: Keep spawning tasks upfront but use channels to send systems into them and back (or cancel if skipped).
    - **Alternative**: Iterate the topsorted list of systems, finding and execute slices of parallel systems.
- Implement conditions.
  - Define the trait (i.e. `IntoRunCondition`) and blanket implement it for compatible functions.
  - Implement `.run_if` descriptor method.
  - Include condition accesses when doing ambiguity checks.
  - Include condition accesses when executor checks if systems can run.
  - Add inlined condition evaluation step in the executor.
- Implement storing and retrieving schedules from a resource.
  - `Schedules` is effectively `Hashmap<ScheduleLabel, Schedule>`
  - Each `Schedule` owns the graph data, and cross-schedule dependencies are impossible.
  - Implement a descriptor coercion trait for `S: SystemSet` types.
- Remove internal uses of "looping run criteria".
  - Convert fixed timestep and state transitions into exclusive systems that each retrieve and run a schedule.
- **Port over the rest of the engine and examples to the new API.**
- Remove stages and run criteria.
- Misc.
  - Implement bulk scheduling capabilities (`.add_systems`, descriptor wrapper enum, system collections / tuples, and consider convenience macros like `chain!`)
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

### If users can configure system sets, can they control where systems from third-party plugins are added?

To some extent, yes.

Not being able to schedule plugin systems is a source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
We thought plugins exporting system sets and users deciding where to position them in their schedules makes for a better balance.
A popular physics plugin [has already implemented something like this](https://github.com/dimforge/bevy_rapier/pull/169) (partly [inspired](https://discord.com/channels/691052431525675048/742569353878437978/970075559679893594) by this effort).

There are three key rules for system configuration:

1. You can always add configuration (for any public set).
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

(Maybe instead of `apply_system_buffers` applying the pending commands of *all* systems in the current schedule, we could construct `apply_system_buffers` from a tuple of system sets, such that only those have their commands applied.
It's debatable whether that would increase or reduce cognitive burden.
That can be revisited later too.)

### Why did you change states from a stack to a basic FSM?

If this RFC is merged and this design is implemented, we expect most of the problems that encouraged it to vanish.
So rather than port pieces of the existing API, we think it's better to start fresh with a basic state machine and see what patterns emerge and what their limits are.
If, after migration, a significant number of users are still turning towards a plugin to recover the lost stack functionality, we can consider upstreaming it again.

A stack-based state machine should be trivial to implement though (i.e. with a loop inside the exclusive system).

### System Sets: the one way to name and group systems

System Sets are now the one way to name and refer to systems (and groups of systems). `SystemLabel` and `some_system.label(X)` have been removed in favor of sets. Even the `SystemTypeIdLabel` (such as `apply_system_buffers`) has been replaced by an equivalent `SystemTypeIdSet`.

The old `SystemLabel` approach, combined with ordering apis like `before` and `after` already resulted in set-like behavior. Consider the following:

```rust
app
    .add_systems((
        a.label(X),
        b.label(X),
        c.after(X),
    ))
```

In this app, `a` and `b` share the `X` label. When `c` adds the `after(X)` constraint, it is referring to the "unordered group" (aka a "set") of systems with the label `X`.

System Sets (in this RFC) behave in exactly the same way:

```rust
app
    .add_systems((a, b).in_set(X))
    .add_systems(c.after(X))
```

The biggest difference is that System Sets can also be ordered relative to other System Sets:

```rust
// direct ordering
app.configure_set(X.after(Y))
// indirect ordering via inheritance
app.configure_set(Z.in_set(X))
```

This is a natural extension of the ordering API, as sets were already addressable for ordering and referred to specific slices of the Schedule graph. If you can order a system relative to a set (previously known as a label), why not allow ordering the set relative to other sets?

There is one corner case worth discussing: "function system name sets", such as `apply_system_buffers`. Function names *are still conceptually and functionally set like*, especially if we allow them to be used in ordering apis.

First consider this common ordering scenario:

```rust
app
    .add_systems((
        foo,
        bar.after(foo),
    ))
```

There is one instance of `foo` and one instance of `bar` in the schedule. And we've configured `bar` to run after `foo`. No need to think about sets. The intent of the program is clear.

Now consider this ordering scenario:

```rust
app
    .add_systems((
        foo,
        bar.after(foo),
    ));

// later, maybe in a different Plugin
app.add_systems(foo.after(baz))
```

Now we can see why this api is still set-like! There are now two `foo` systems in the schedule, each identified by the `foo` name. When `bar` adds the `after(foo)` constraint, it is ambiguously referring to both systems (and implicitly needs to wait for `baz` to finish, despite having never intended to depend on that instance of foo).

This *is* a problem, but it is one inherent to the apis. There can be multiple `foo` systems and they can be ambiguously referred to as a group. As the example above illustrates, this can result in unexpected, potentially game breaking behaviors.

Therefore in the context of "function system name sets" (`SystemTypeIdSet`), ordering constraints like `before` and `after` are explicitly disallowed if there is more than one instance of the system in the schedule (via runtime checks during schedule initialization). See the next section for details.

### Why is ordering relative to type-derived sets (`SystemTypeIdSet`) forbidden when there are duplicate instances of systems?

If you schedule multiple instances of a system, the one thing we *never* want to do is group them together without your explicit say-so.
To explain why, let's consider one common and significant case: `apply_system_buffers`.

Suppose a user has written the following:

```rust
fn main() {
    app
        .add_systems((
            apply_system_buffers,
            generates_commands.before(apply_system_buffers),
            relies_on_commands.after(apply_system_buffers),
        ))
        .run()
}
```

The user has added three systems and, naively, created dependencies using the `apply_system_buffers` set (containing all instances of the apply_system_buffers function system).

Since there's only one instance of `apply_system_buffers`, this app works like you'd expect.
But now suppose that the user adds more anonymous instances of `apply_system_buffers` or imports plugin logic that does.

Let's say the user imports a system set `X` from some plugin and `X` has another anonymous instance of `apply_system_buffers`.
If the user schedules `X` to run before (or after) their logic, their program will panic over an unsolvable dependency graph.

```rust
fn main() {
    app
        .configure_set(X.before(generates_commands))
        .add_systems((
            apply_system_buffers,
            generates_commands.before(apply_system_buffers),
            relies_on_commands.after(apply_system_buffers),
        ))
        // This will panic.
        .run()
}
```

What happened here?

Well, the problem is a type-derived set will name *all* anonymous systems of that type.
Therefore, `before(apply_system_buffers)` means "before all instances of `apply_system_buffers`" even though the user was most likely only thinking about the one they added themselves.
Totally obvious, right? (/s)

Their program panicked because it saw a contradiction: `generate_commands` had to run before all `apply_system_buffers` instances and run after the one inside `X` at the same time.

One (reliable) solution here is for the user to give their instance of `apply_system_buffers` a unique name and use that name for ordering.

We came up with the following options to protect against this case:

1. Require users to name all duplicate systems.
2. Leave duplicate systems completely anonymous. Only give the type-derived name to the first copy.
3. Error when multiple anonymous copies of a system exist and their type-derived set is used for ordering.
4. Disallow before/after ordering constraints between systems using their names by not implementing SystemSet for function names.

(1) ensures correctness and an easier debugging experience, but is too inconvenient.
(2) drops the naming requirement without losing correctness, but makes things too unclear.
(3) just makes doing the (generally) wrong thing an error.
(4) prevents a common useful (and ergonomic) pattern to protect against a corner case

We like (3) because it doesn't put undue burden on the user or introduce unclear implicit behavior.
Likewise, resolving the error is very simple: just name the systems.

Internally, systems and sets are given unique identifiers and those are used for graph construction.
This means you can do `add_systems(apply_system_buffers.after(X))` multiple times or use the `(a, b, c).chain()` operation with multiple unnamed instances of the same function without having to name any of them using their types. This protects against the error in (3) for many common cases.

A system only needs to reference the potentially ambiguous `SystemTypeIdSet` when you want to do `before(name)` or `after(name)` somewhere else. And in these cases, the validation from (3) will protect users from accidentally doing something wrong.

In short, when the error in (3) is encountered:

```rust
app.add_systems((
    foo,
    foo,
    bar.after(foo),
))
```

Users should do one of the following:

1. Create a new set, add it to the duplicate system, and use that to define orders unambiguously;

    ```rust
    app.add_systems((
        foo,
        foo.in_set(X),
        bar.after(X),
    ))
    ```

2. Refactor their system registration to use APIs that order unambiguously using specific system instance ids:

    ```rust
    // chaining
    app
        .add_systems(foo)
        .add_systems((foo, bar).chain())
    // rotate bar.after(foo) to foo.before(bar)
    app.add_systems((
        foo,
        bar,
        foo.before(bar),
    ))
    ```

*Note: In case it wasn't already clear, system chaining introduces new instances of systems. It does not reuse existing ones.*

See the discussion on command-flushed ordering constraints and automatic inference of sync points in **Future Work** for ideas on how we can improve this.

### If "system types" are System Sets, won't that allow users to configure them in confusing ways?

Unless we do something to prevent this, then yes:

```rust
fn some_system(time: Res<Time>) {}

app
    .configure_set(some_system.after(X))
    .add_systems(foo.in_set(some_system))
```

To prevent this, user-facing set configuration and set-membership APIs will fail with an error message if SystemTypeIdSets (such as `some_system` in the example above) are passed in.

In the future, we might generalize the "set locking" feature to user-defined sets, which would enable Bevy plugin authors to make their sets "private" (much like "visibility" keywords in programming languages).

## Unresolved questions

### What's a good convention for naming System Sets?

What convention would we have for naming internal system sets?

For example, Bevy will have a number of "core" sets. These could use any of the following conventions:

- `enum CoreSet { Update, PostUpdate }`
- `enum CoreSets { Update, PostUpdate }`
- `enum CoreSystems { Update, PostUpdate }`
- `mod core_set { struct Update; struct PostUpdate; }`

### What sugar should we use for adding multiple systems at once?

Convenience methods like `add_systems` are important for reducing boilerplate. For these methods, we need to be able to refer to collections of systems.

**In summary:**

- builder syntax: marginal improvement over adding individual systems
- array syntax: looks pretty but is literally impossible
- tuple syntax: looks pretty but is limited to a fixed number of elements, unless tuple nesting is used (or variadic tuples are added to rust)
- `vec!`-like macro syntax: looks OK but, eh, uses macros

**Conclusion:** tuple syntax for terseness and avoiding macro-magic. `add_systems` should accept some `Into<SystemCollection>` impl, which means we can optionally support macros and/or builder syntax for those that prefer them, either officially or via 3rd party crates.

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
- limited to a fixed number of elements (the more we support, the bigger the compile time cost)
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

After this RFC is implemented, there's valuable work to be done to improve ergonomics:

1. Automatic (opt-in) insertion of command application and state transition systems (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
2. Add [manual dependency graph construction](https://github.com/bevyengine/bevy/pull/2381) methods.

Clean up related code:

1. Revisit [plugin definition and plugin configuration](https://github.com/bevyengine/bevy/issues/2160).
2. Move command queues into the `World` so that exclusive systems can drain them directly (e.g. in some user-defined order).
3. Cloning registered systems. (This probably warrants its own RFC, but it fits with the multiple worlds discussion.)

Build out solid tooling:

1. Improve tools for reporting and resolving system execution order ambiguities.
2. System graph visualization.
3. Seeded, fully deterministic system execution orders: useful for debugging system order bugs and optimization work.

Improve performance:

1. Run schedules without `&mut World`, inferring access based on the systems inside.
2. Assorted benchmarking and optimization of various scheduling strategies.

Or increase scheduling expressivity:

1. Compose conditions via arbitrary boolean expressions.
2. Unnecessary dependency removal ([RFC #47](https://github.com/bevyengine/rfcs/pull/47)).
3. "Command-flushed" ordering constraints (dependencies that will ensure the pair is separated by command application).
4. Configurable system set atomicity ([RFC #46](https://github.com/bevyengine/rfcs/pull/46)), where the data that a group of systems accesses is locked until all are completed.
5. Add [runtime insertion and removal of systems](https://github.com/bevyengine/bevy/issues/279).
6. Support automatic insertion and removal of systems to reduce schedule clutter and support other [forms of one-off logic](https://github.com/bevyengine/bevy/pull/4090).

## Appendix

These implementation ideas/details are here to avoid cluttering the RFC.

### `apply_system_buffers`

As command queues are owned by their systems, an exclusive system won't be able to drain their buffers by itself.
This one just signals the executor to do it.
The caveat to handling it this way is that the executor is only aware of the systems in the schedule it's running, so we should probably still have one "courtesy flush" after all the systems complete, so users can't forget to.

### Using descriptor API for both systems and system sets

It's fairly straightforward to register system sets using the same API.

As a bonus, schedule construction will become completely order-independent as everything will be deferred until the end.

```rust
pub trait IntoConfiguredSystem<Params> {
    fn configure(self) -> ConfiguredSystem;
    fn before<M>(self, set: impl AsSystemSet<M>) -> ConfiguredSystem;
    fn after<M>(self, set: impl AsSystemSet<M>) -> ConfiguredSystem;
    fn in_set(self, set: impl SystemSet) -> ConfiguredSystem;
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSystem;
}

pub trait IntoConfiguredSystemSet {
    fn configure(self) -> ConfiguredSystemSet;
    fn before<M>(self, set: impl AsSystemSet<M>) -> ConfiguredSystemSet;
    fn after<M>(self, set: impl AsSystemSet<M>) -> ConfiguredSystemSet;
    fn in_set(self, set: impl SystemSet) -> ConfiguredSystemSet;
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSystemSet;
}

enum Order {
    Before,
    After
}

struct Config {
    sets: HashSet<SystemSetId>,
    dependencies: Vec<(Order, SystemSetId)>,
    conditions: Vec<BoxedRunCondition>,
}

struct ConfiguredSystem {
    system: BoxedSystem,
    config: Config,
    instance_name: SystemSetId,
}

struct ConfiguredSystemSet {
    set: SystemSetId,
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

4. (Optional) You have a dependency between two things that aren't siblings in a common set. That edge will not appear in either set's dependency graph, only in the flattened graph of an overarching set. This can lead to unwanted implicit ordering between systems in different sets.
5. (Optional) You referenced an "unknown" set. e.g. `.after(set)` references a set that doesn't belong to any known system set.
6. (Optional) You have at least one pair of ambiguously-ordered systems with conflicting data access.

(4), (5), and (6) don't inherently make a graph unsolvable, so they can be configured as ignore, warn, or error.
By default, they are all configured as warn.
(6) has additional configuration options.
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

#[derive(ScheduleLabel)]
pub struct OnEnter<S: State>(S);

#[derive(ScheduleLabel)]
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
            world.run_schedule(OnExit(old));
            world.run_schedule(OnEnter(new));
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

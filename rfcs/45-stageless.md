# Feature Name: `stageless`

## Summary

The existing stage boundaries result in a large number of complications as they prevent us from operating across stages.
Moreover, several related features (run criteria, states and fixed timesteps) are rather fragile and overly complex.
This tangled mess needs to be holistically refactored: simplifying the scheduling model down into a single `Schedule`, moving towards an intent-based configuration style, taking full advantage of system labels as sets and leveraging exclusive systems to handle complex logic.

## Motivation

The fundamental challenges with the current stage-driven scheduling abstraction are [numerous](https://github.com/bevyengine/bevy/discussions/2801).
Of particular note:

- Dependencies do not work across stages.
- States do not work across stages.
- Plugins can add new, standalone stages.
- Run criteria (including states and fixed timestep run criteria!) cannot be composed.
- Run criteria cannot be reused across states or stages.
- The stack-driven state model is overly elaborate and does not enable enough use cases to warrant its complexity.
- The architecture for turn-based games and other even slightly unusual patterns is not obvious.
- We cannot easily implement a builder-style strategy for configuring systems.

Unfortunately, all of these problems are deeply interwoven. Despite our best efforts, fixing them incrementally at either the design or implementation stage is impossible, as it results in myopic architecture choices and terrible technical debt.

## User-facing explanation

This explanation is, in effect, user-facing documentation for the new design.
In addition to a few new concepts, it throws out much of the current system scheduling design that you may be familiar with.
The following elements are substantially reworked:

- schedules (simplified, each `World` can store multiple schedules in a `SystemRegistry` resource)
- run criteria (can no longer loop, are now systems)
- states (simplified, no longer purely run-criteria powered)
- fixed time steps (no longer a run criterion)
- exclusive systems (no longer special-cased)
- command processing (now performed in a `flush_commands` exclusive system)
- labels (merged with the concept of system sets, can now be directly configured)
- stages (yoten)

### Scheduling overview

Systems in Bevy are stored in a `Schedule`: a collection of **configured systems**, each of which may belong to any number of **system sets**.
In each iteration of the schedule (typically corresponding to a single frame), the `App`'s `runner` function will run the schedule,
causing the **scheduler** to run the systems in parallel using a strategy that respects all of the configured constraints.
If these constraints cannot be met (for example, a system may want to run both before and after another system), the schedule is said to be **unsatisfiable**, and the scheduler will panic.
Schedules can be nested and branched from within **exclusive systems** (which have mutable access to the entire `World`), allowing you to encode arbitrarily complex control flow using ordinary Rust constructs.

By default, each new `Schedule` is entirely unordered: systems will be selected in an arbitrary order (which is not guaranteed to be stable across passes of the schedule) and run if and only  if all of the data that they must access are free.
Just like with standard borrow checking, multiple systems can read from the same data at once, but writing to the data requires an exclusive lock.
Systems which cannot be run in parallel are said to be **incompatible**.

### The `SystemRegistry` resource stores multiple schedules

Each `World` can store multiple `Schedules` with a `SystemRegistry` resource: each with their own `impl ScheduleLabel` type, allowing you to cleanly execute coherent blocks of logic when the need arises.
For example, by default, each app stores both a startup and main schedule: the former runs only once on app creation, while the latter loops endlessly.

The main and startup schedules can be accessed using the `CoreSchedule::Main` and `CoreSchedule::Startup` labels respectively.
However, schedules are a surprisingly powerful and flexible tool: they can be used to handle initialization and cleanup logic when working with states or used ad-hoc to run game logic in complex patterns.
By default, systems are added to the main schedule.
You can control this by adding the `.in_set_schedule(ScheduleLabel::Variant)` system descriptor to your system or system set.

#### Startup systems

Startup systems are stored in their own schedule, with the `CoreSchedule::Startup` label.
When using the runner added by `MinimalPlugins` and `DefaultPlugins`, this schedule will run exactly once on app startup.

You can add startup systems with the `.add_startup_system(on_startup)` method on `App`, which is simply sugar for `.add_system(on_startup.in_set_schedule(CoreSchedule::Startup))`.

### System configuration

Each system has a few configurable parameters:

- it may have **ordering constraints**, causing it to run before or after other systems
  - there are several subtly different types of ordering constraints, see the next section for details
  - e.g. `.add_system(player_controls.before(GameSets::Physics).after(CoreSets::Input))`
- it may have one or more **run criteria** attached
  - a system is only executed if all of its run criteria return `true`
  - **states** are a special, more complex pattern that use run criteria to determine whether or not a system should run in a current state
  - e.g. `.add_system(physics.run_if_resource_equals(EnablePhysics(true)))`
- it may have one or more **labels**, allowing other systems to refer to it and enabling mass-configuration
  - e.g. `.add_system(physics.in_set(GameSets::Physics)`

This configuration is additive: adding another ordering constraint, run criteria or set does not replace the existing configuration.

System configuration can be stored in a `SystemConfig` struct.
This can be useful to reuse, compose and quickly apply complex configuration strategies, before applying these strategies to sets or individual systems.

### System set configuration

Each system set has its own associated `SystemConfig`, stored in the corresponding `Schedule`. When a system set is configured, its configuration is effectively passed down to every system under it. For example, ordering the set `A` to run before the set `B` specifies that all systems in `A` will run before any system in `B` does.

You can add many systems to the same set at once using the `App::add_systems(systems: impl SystemIterator)` method.

```rust
#[derive(SystemLabel)]
enum Physics {
    Forces,
    CollisionDetection,
    CollisionHandling,
}

impl Plugin for PhysicsPlugin{
    fn build(app: &mut App){
        // Within each frame, physics logic needs to occur after input handling, but before rendering
        let mut common_physics_config = SystemConfig::new().after(InputSet::ReadInputHandling).before(CoreSet::Rendering);

        app
        // We can reuse this shared configuration on each of our sets
        .add_set(Physics::Forces.configure(common_physics_config).before(Physics::CollisionDetection))
        .add_set(Physics::CollisionDetection.configure(common_physics_config).before(Physics::CollisionHandling))
        .add_set(Physics::CollisionHandling.configure(common_physics_config))
        // And then apply that config to each of the systems that are part of this set
        .add_system(gravity.in_set(Physics::Forces))
        // These systems have a linear chain of ordering dependencies between them
        // Systems earlier in the chain must run before those later in the chain
        // Other systems can run in between these systems
        .add_systems((broad_pass, narrow_pass).chain().in_set(Physics::CollisionDetection))
        // Add multiple systems at once to reduce boilerplate!
        .add_systems((compute_forces, collision_damage).in_set(Physics::CollisionHandling));
    }
}
```

Just like systems, system sets can be given run criteria and be part of other sets themselves, allowing users to describe properties hierarchically.

#### System ordering

The most important way we can configure systems is by telling the scheduler *when* they should be run.
The ordering of systems is always defined relative to other systems: either directly or by checking the systems that belong to a set.

Applying an ordering constraint between sets causes ordering constraints to be created between all individual members of those sets.
If an ordering is defined relative to an empty set, it will have no effect, and a warning will be emitted.
This relatively gentle failure mode is important to ensure that plugins can order their systems with relatively strong assumptions that the default system sets exist, but continue to (mostly) work if those systems or sets are not present.

In addition to the `.before` and `.after` methods, you can use **system chains** to create very simple linear dependencies between the successive members of an array of systems.
(Note to readers: this is not the same as "system chaining" in Bevy 0.7 and earlier: that concept has been renamed to "system piping".)

```rust
fn main(){
   App::new()
   // These systems are connected using a string of ordering constraints

   .add_systems((compute_attack, 
                 compute_defense,
                 check_for_crits,
                 compute_damage,
                 deal_damage,
                 check_for_death)
                 .chain()
                 // We can configure all systems in the chain at once
                 .in_set(GameSet::Combat)
   )
   .run()
}
```

#### Run criteria

While ordering constraints determine *when* a system will run, **run criteria** will determine *if* it will run at all.
Each system can have any number of run criteria, which read data from the world to control whether or not that system runs.
If (and only if) all of its run criteria return `true`, the system will run.
If any of the run criteria are `false`, the system will be skipped.
Systems that are skipped are considered completed for the purposes of ordering constraints.

Run criteria are not systems, but can be created from systems which can read (but not write) data from the `World` and returns a boolean value.

You can specify run criteria in several different ways:

```rust
// This is just an ordinary system: timers need to be ticked!
fn tick_construction_timer(timer: ResMut<ConstructionTimer>, time: Res<Time>){
    timer.tick(time.delta());
}

// This function can be used as a run criterion system,
// because it does not mutate data and returns `bool`
fn construction_timer_finished(timer: Res<ConstructionTimer>) -> bool {
    timer.finished()
}

// You can use queries, event readers and resources with arbitrarily complex internal logic
fn too_many_enemies(population_query: Query<(), With<Enemy>>, population_limit: Res<PopulationLimit>) -> bool {
   population_query.iter().count() > population_limit
}

fn main(){
    App::new()
    .add_plugins(DefaultPlugins)
    // We can add functions with read-only system parameters as run criteria
    .add_system((
       tick_construction_timer, 
       update_construction_progress)
       .chain()
       .run_if(construction_timer_finished)
    )
   .add_system(system_meltdown_mitigation.run_if(too_many_enemies))
    // We can use closures for simple one-off run criteria, 
    // which automatically fetch the appropriate data from the `World`
    .add_system(spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000))
    // The `run_if_resource_equals` method is fast, special-cased syntax that generates a run criterion
    // for when you want to check the value of a resource (commonly an enum)
    .add_system(gravity.run_if_resource_equals(Gravity::Enabled))
    // Run criteria can be attached to system sets
    .add_set(GameSet::Physics.run_if_resource_equals(Paused(false)))
    .run();
}
```

There are a few important subtleties to bear in mind when working with run criteria:

- when multiple run criteria are attached to the same system, the system will run if and only if all of those run criteria return true
- run criteria are evaluated "just before" the system that it is attached to is run
- if a run criterion is attached to a system set, a run-criterion-system will be generated for each contained system
  - this is essential to ensure that run criteria are checking fresh state without creating very difficult-to-satisfy ordering constraints
  - if you need to ensure that all systems behave the same way during a single pass of the schedule or avoid expensive recomputation, precompute the value and store the result in a resource, then read from that in your run criteria instead

Run criteria are evaluated by the executor just before the system that they are controlling is run: it is impossible to modify the data that they rely on in an observable way before the system completes.
This is important to ensure that the state of the world always matches the state expected by the system at the time it is executed.

#### States

**States** allow you to toggle systems on-and-off based on the value of a given resource and smoothly handle transitions between states by running cleanup and initialization systems.
Typically, these are used for relatively complex, far-reaching facets of the game like pausing, loading assets or entering into menus.

The current value of each state is stored in a resource, which must derive the `State` trait.
These are typically (but not necessarily) enums, where each distinct state is represented as an enum variant.

Each state is associated with three groups of systems:

1. **While-active:** these systems run each schedule iteration if and only if the value of the state resource matches the provided value.
   1. `app.add_system(apply_damage.run_in_state(GameState::Playing))`
2. **On-enter systems:** these systems run once when the specified state is entered.
   1. `app.add_system(generate_map.run_on_enter(GameState::Playing))`
3. **On-exit systems:** these systems run once when the specified state is exited.
   1. `app.add_system(autosave.run_on_exit(GameState::Playing))`

While-active systems are by far the simplest: they're simply powered by run criteria.
`.run_in_state` is precisely equivalent to `run_if_resource_equals`, except with an additional trait bound that the resource must implement `State`.

On-enter and on-exit systems are stored in dedicated schedules, two per state, within the `App's` `Schedules`.
These schedules can be configured in all of the ordinary ways, but, as they live in different schedules, ordering cannot be defined relative to systems in the main schedule.

You can change which variant of a state you are in by requesting the state as a mutable resource, then using the `State::transition_to` method within your system.
Due to their disruptive and far-reaching effects, state transitions do not occur immediately.
Instead, they are deferred (like commands), until the next `flush_state<S: State>` exclusive system runs.
This system first runs the `on_exit` schedule of the previous state on the world, then runs the `on_enter` schedule of the new state on the world.
Once that is complete, the exclusive system ends and control flow resumes as normal.
Note that commands are not automatically flushed between state transitions. If this is required, add the `flush_commands` system to your schedule.

When states are added using `App::add_state::<S: State>(initial_state)`, one `flush_state<S>` system is added to the app, as part of the `GeneratedSet::StateTransition<S>` set.
You can configure when and if this system is scheduled by configuring this set, and you can add additional copies of this system to your schedule where you see fit.

Apps can have multiple orthogonal states representing independent facets of your game: these operate fully independently.
States can also be defined as a nested enum: these work as you may expect, with each leaf node representing a distinct group of systems.

### Flushing `Commands`

Commands (commonly used to spawn and despawn entities or add and remove components) do not take effect immediately.
Instead, they are applied whenever the `flush_commands` system runs.
This exclusive system collects all created commands and applies them to the `World`, mutating it directly.

```rust
use bevy::prelude::*;

#[derive(SystemLabel)]
enum StartupSet{
   CommandFlush,
   UiSpawn,
}

/// This example has no `DefaultPlugins`,
/// so all command flushing is done manually
fn main(){
   App::new()
   // All systems that are not part of this set are implicitly before this set
   .add_system(flush_commands.in_set(CoreSet::Last))
   // Recall that this adds systems to the startup schedule, not the main one
   .add_startup_system(flush_commands.in_set(CoreSet::Last))
   // Commands will be processed in the basic flush_commands system that occurs at the end of the schedule
   .add_startup_system(spawn_player)
   // We need to customize this after it's spawned
   .add_startup_system(spawn_ui.before(StartupSet::CommandFlush).in_set(StartupSet::UiSpawn))
   .add_startup_system(customize_ui.after(StartupSet::CommandFlush))
   .add_startup_system(flush_commands.in_set(StartupSet::CommandFlush);
   // Less verbosely, we can use the `.chain()` helper method
   .add_systems((spawn_ui, flush_commands, customize_ui).chain().in_set_schedule(CoreSchedule::Startup))
   .run();
}
```

With `DefaultPlugins`, managing command flushing is likely to look more like this:

```rust
use bevy::prelude::*;

struct ProjectilePlugin;

impl Plugin for ProjectilePlugin {
  fn build(app: &mut App){
    app
    // DefaultPlugins adds three command flushes:
    // 1. CommandFlush::PreUpdate (runs after input is processed)
    // 2. CommandFlush::PostUpdate (runs after most game logic)
    // 3. CommandFlush::EndOfFrame (runs after data is extracted for rendering, at the end of the schedule)
    .add_system(spawn_projectiles.before(CommandFlush::PostUpdate))
    .add_systems(
      (check_if_projectiles_hit, despawn_projectiles_that_hit)
      .chain()
      .after(CommandFlush::PostUpdate)
    )
    // If we need to add more command flushes to make our logic work, we can manually insert them
    .add_system(flush_commands.in_set("DespawningFlush").after(CommandFlush::PostUpdate).before(CommandFlush::EndOfFrame))
    .add_system(fork_projectiles.after("DespawningFlush"));
  }
}
```

### Complex control flow

Occasionally, you may find yourself yearning for more complex system control flow than "every system runs once in a loop".
When that happens: **create an exclusive system and run a schedule in it.**

Within an exclusive system, you can freely fetch the desired schedule from the `App` with the `&Schedules` system parameter and use `Schedule::run(&mut world)`, applying each of the systems in that schedule a single time to the world of the exclusive system.
However, because you're in an ordinary Rust function you're free to use whatever logic and control flow you desire: branch and loop in whatever convoluted fashion you need!

This can be helpful when:

- you want to run multiple steps of a set of game logic during a single frame (as you might see in complex turn-based games)
- you need to branch the entire schedule to handle a particularly complex bit of game logic
- you need to integrate a complex external service, like scripting or a web server
- you want to switch the executor used for some portion of your systems

If you need absolute control over the entire schedule, consider having a single root-level exclusive system.

#### Fixed time steps

Running a set of systems (typically physics) according to a fixed time step is a particularly critical case of this complex control flow.
These systems should run a fixed number of times for each second elapsed.
The number of times these systems should be run varies (it may be 0, 1 or more each frame, depending on the time that the frame took to complete).

Simply adding run criteria is inadequate: run criteria can only cause our systems to run a single time, or not run at all.
By moving this logic into its own schedule within an exclusive system, we can loop until the accumulated time has been spent, running the schedule repeatedly and ensuring that our physics always uses the same delta-time and stays synced with the clock, even in the face of serious stutters.

## Implementation strategy

Let's take a look at what implementing this would take:

1. Completely rip out:
   1. `Stage`
   2. `Schedule`
   3. `SystemDescriptor`
   4. `IntoExclusiveSystem`
   5. `SystemContainer` and friends
2. Build out a basic schedule abstraction:
   1. Create a `ScheduleLabel` trait
   2. Store multiple schedules in a keyed `SystemRegistry` resource
   3. Support adding systems to the schedule
      1. These should be initialized by definition: see [PR #2777](https://github.com/bevyengine/bevy/pull/2777)
   4. Support adding systems to specific schedules and create startup system API
3. Rebuild core command processing machinery
   1. Begin with trivial event-style commands
   2. Explore how to regain parallelism
4. Add system configuration tools
   1. Create `SystemConfig` type
   2. System sets store a `SystemConfig`
   3. Allow systems to belong to sets
   4. Store `ConfiguredSystems` in the schedule
   5. Add `.add_systems` method, the `SystemGroup` type and the convenience methods for system groups
   6. Special-case `CoreSet::First` and `CoreSet::Last` for convenience
   7. Use [system builder syntax](https://github.com/bevyengine/rfcs/pull/31), rather than adding more complex methods to `App`
5. Re-add strict ordering constraints
6. Add run criteria
   1. Create the machinery to generate run criteria from functions
   2. Cache the accesses of run criteria on `ConfiguredSystem` in the schedule
   3. Check the values of the run-criteria of systems before deciding whether it can be run
7. Add states
   1. Create `State` trait
   2. Implement while-active states as sugar for simple run criteria
   3. Create on-enter and on-exit schedules
   4. Create sugar for adding systems to these schedules
8. Rename "system chaining" to "system piping"
   1. Usage was very confusing for new users
   2. New concept of "systems with a linear graph of ordering constraints between them" is naturally described as a chain
9. Add new examples
    1. Complex control flow with supplementary schedules
    2. Fixed time-step pattern
10. Port the entire engine to the new paradigm
    1. We almost certainly need to port the improved ambiguity checker over to make this new paradigm usable

Given the massive scope, that sounds relatively straightforward!
However, doing so will break approximately the entire engine, and tests will not pass again until step 10.

Lets explore some of the more critical and challenging details.

### Storing and configuring systems

Schedules store systems in a configured, initialized form.

```rust
struct Schedule{
   // Each schedule is associated with exactly one `World`, as systems must be initiliazed on it
   world_id: WorldId
   // We need to be able to quickly look up specific systems and iterate over them
   // `SystemId` should be a unique opaque identifier, generated on system insertion
   // We cannot use a Vec + a `usize` index as the identifier,
   // as the identifier would not be stable when systems were removed, risking serious bugs
   systems: HashMap<SystemId, ConfiguredSystem>,
   // Stores the fully initialized, efficient graph of ordering constraints
   // for all systems in the schedule.
   // The properties of each set are passed down to specific systems to cache computation results.
   // `Graph` is a bikeshed-avoidance graph storage structure,
   // the exact strategy should be benchmarked
   ordering_constraints: Graph<SystemId>,
   // Set configuration must be stored at the schedule level,
   // rather than attached to specific set instances
   sets: HashMap<Box<dyn SystemLabel>, SystemSetConfig>,
   // Used to check if mutation is allowed
   currently_running: bool,
}
```

Configured systems are composed of a raw system and its configuration.

```rust
struct ConfiguredSystem{
   raw_system: System,
   config: SystemConfig,
}
```

Raw systems store metadata, cached state, `Local` system parameters and the function pointer to the actual function to be executed:

```rust
struct System {
   function: F,
   // Contains all metadata and state
   meta: SystemMeta
   last_change_tick: u32,
}
```

System configuration stores ordering dependencies, run criteria and system sets.
It can be created raw, or stored in association with either system sets or systems.

```rust
enum OrderingTarget {
   Label(Box<dyn SystemLabel>),
   System(SystemId),
}

struct SystemConfig {
   strict_ordering_before: Vec<OrderingTarget>,
   strict_ordering_after: Vec<OrderingTarget>,
   // The `TypeId` here is the type id of the flushing system
   flushed_ordering_before: Vec<(TypeId, OrderingTarget)>,
   flushed_ordering_after: Vec<(TypeId, OrderingTarget)>,
   run_criteria: Vec<SystemId>,
   // The combined access requirements of the system and its run criteria
   joint_access: FilteredAccessSet<ComponentId>,
   sets: Vec<Box dyn SystemLabel>,
}

struct SystemSetConfig {
   system_config: SystemConfig,
}

```

### System set configuration

When building the dependency graph, all edges (constraints) between labels are expanded into edges between the systems nested within them.

### Run criteria

Run criteria are systems with several key constraints:

- They cannot have system params that give mutable access to data.
  - Required so we don't have order to them.
  - Read-only access avoids very surprising side effects.
- They cannot capture or store local data.
  - Required to avoid user footguns: local data would be updated even if the guarded system ultimately doesn't run.
- They must return a `bool`.
  - Required so we know whether the system should run!

It's worth calling out that run criteria cannot be shared. Attaching the same run criteria to multiple systems will create a separate instance for each system. This is to ensure that systems and their run criteria are always evaluated as an atomic unit.

When the executor is presented with a system, it does the following:

- Check if the joint data access of the system, its run criteria, and the run criteria of its not-yet-seen sets is available.
  - If not, the system is left in the queue and the next system is checked.
- If the joint access is available, then in hierarchical order, evaluate the run criteria of each not-yet-seen set.
  - Mark the set as seen.
  - If any run criteria return `false`, skip *all* systems nested under the set. Also mark all sets nested under it as seen.
- If all set run criteria passed, evaluate the system's run criteria.
  - Skip the system if any run criteria return `false`.
- If all run criteria have returned `true`:
  - "Lock" the data access required by the system.
  - Spawn a task to run the system.
  - Release its "locks" when the task completes.

This lightweight approach minimizes task overhead. Since we don't spawn tasks for run criteria (which are extremely likely to be simple tests), we don't have to lock other systems out of the data they access. Likewise, we don't spawn tasks for systems that were skipped.

### Flushed ordering constraints

Being able to assert that commands are processed (or state is flushed) between two systems is a critical requirement for being able to specify logically-meaningful ordering of systems without a global view of the schedule.

However, the exact logic involved in doing this is quite complex.
We need to:

1. **When validating the schedule:** ensure that a flushing system of the appropriate type *could* occur between the pair of systems.
2. **When executing the schedule:** ensure that a flushing system of the appropriate type *does* occur between the pair of systems.

For this RFC, let's begin with the very simplest strategy: for each pair of systems with a flushed ordering constraint, ensure that a flushing system of the appropriate type is strictly after the first system, and strictly before the second.

We can check if the flushing system is of the appropriate type by checking the type id of the function pointer stored in the `System`.
This is cached in the `system_function_type_map` field of the `Schedule` for efficient lookup, yielding the relevant `SystemIds`.

With the ids of the relevant systems in hand, we can check each constraint: for each pair of systems, we must check that at least one of the relevant flushing systems is between them.
To do so, we need to check both direct *and* transitive strict ordering dependencies, using a depth-first search to walk backwards from the later system using the `CachedOrdering` until a flushing system is reached, and then walking back further from that flushing system until the originating system is reached.

### Schedule unsatisfiability

Currently there is only one way in which a schedule can be unsatisfiable:

1. The graph of ordering constraints contains a cycle.

However, this list is likely to expand over time.

### Change detection

The reliable change detection users have come to know and love stays the same. The only difference with this architecture is when/where we check if a `check_tick` scan is needeed. Each world still has an atomic counter that increments each time a system runs (skipped systems do not count) and tracks change ticks for its components and systems.

For a system to detect changes (assuming one of its queries has a `Changed<T>` filter), its tick and the change ticks of any matched components are compared against the current world tick. If the system's last run is older than a change, the query will yield that component value.

```rust
fn is_changed(world_tick: u32, system_tick: u32, change_tick: u32) -> bool {
   let ticks_since_change = world_tick.wrapping_sub(change_tick);
   let ticks_since_system = world_tick.wrapping_sub(system_tick);
   ticks_since_system > ticks_since_change
}
```

To ensure graceful operation, change ticks are periodically scanned, and any older than a certain threshold are clamped to prevent their age from overflowing and triggering false positives.

We perform a scan once at least `N` (currently ~530 million) ticks (and no more than `2N - 1`) have elapsed since the previous scan. The latter can be circumvented if exclusive and chained systems are abused in strange ways, but since `N` is so large, users are extremely unlikely to encounter that in practice.

```rust
pub const MAX_CHANGE_AGE: u32 = u32::MAX - (2 * N - 1);

fn check_tick(world_tick: u32, saved_tick: &mut u32) {
   let age = world_tick.wrapping_sub(*saved_tick);
   if age > MAX_CHANGE_AGE {
      *saved_tick = world_tick.wrapping_sub(MAX_CHANGE_AGE);
   }
}
```

## Drawbacks

1. This will be a massively breaking change to every single Bevy user.
   1. Any non-trivial control over system ordering will need to be completely reworked.
   2. Users will typically need to think a bit harder about exactly when they want their gameplay systems to run. In most cases, they should just add it to the `CoreSet::Update` set, which will place them after input and before rendering.
2. It may be harder to immediately understand the global structure of Bevy apps.
   1. Powerful system debugging and visualization tools become even more important (but easier to write).
   2. It may become harder to reason about exactly when command flushing occurs.
   3. If a single system belongs to multiple sets it can suddenly create wide-spread consequences.
3. State transitions are no longer queued up in a stack.
   1. This also removes "in-stack" and related system groups / logic.
   2. This can be easily added later, or third-party plugins can create their own abstraction.
4. When a state transition occurs, the systems in the while-active set are no longer applied immediately.
   1. Instead, the normal flow of logic continues.
   2. Users can duplicate critical logic in the while-active and on-enter collections.
   3. Similarly, arbitrarily long chains of state transitions are no longer be processed in the same schedule pass.
   4. However, users can add as many copies of the `flush_state` system as they would like, and loop it within an exclusive system.

## Rationale and alternatives

### Doesn't set-configuration allow you to configure systems added by third-party plugins?

Yes. This is a large part of the point.

The difficulties configuring plugins is a source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
Plugin authors cannot know all of the needs of the end user's app, and so some degree of control must be handed back to the end users.

This control is carefully limited though: systems can only be configured if they are part of a public set, and all of the systems under a common set are configured as a group.
In addition, configuration defined by a private set (or directly on systems) cannot be altered, extended or removed by the app: it cannot be named, and so cannot be altered.

This limitation allows plugin authors to carefully design a public API, ensure critical invariants hold, and make serious internal changes without breaking their dependencies (as long as the public sets and their meanings are relatively stable).

### Why aren't run criteria cached?

In practice, almost all run criteria are extremely simple, and fast to check.
Verifying that the cache is still valid will require access to the data anyways, and involve more overhead than simple computations on one or two resources.

In addition, this is required to guarantee atomicity.
We could add atomic groups to this proposal to get around this, but that is another large research project.

## What sugar should we use for adding multiple systems at once?

When using convenience methods like `add_systems`, we need to be able to refer to collections of systems.
This will become increasingly important as more complex graph-builder APIs are added.

**In summary:**

- array syntax: optimal but literally impossible
- tuple syntax: magic but pretty
- builder syntax: way too verbose for a pattern that's supposed to increase convenience
- `vec!`-style macro: not any less magic than tuple syntax, not as pretty

**Conclusion:** tuple syntax, powered by builder syntax under the hood that users can use instead when it's more convenient.

Below, we examine the the options by example.

### Arrays

```rust
.add_systems([compute_attack, 
                   compute_defense,
                   check_for_crits,
                   compute_damage,
                   deal_damage,
                   check_for_death]
                   .chain()
                   .in_set(GameSet::Combat))

.add_systems([run, jump].after(InputSet::ReadInput))
```

Very pretty. Doesn't work because types are heterogenous though.

### Tuples

```rust
.add_systems((compute_attack, 
              compute_defense,
              check_for_crits,
              compute_damage,
              deal_damage,
              check_for_death)
              .chain()
              .in_set(GameSet::Combat)
            )

.add_systems((run, jump).after(InputSet::ReadInput))
```

Also pretty! Mildly cursed and definitely unidiomatic, but we use this flavor of cursed type system magic already for bundles.

Requires another invocation of the `all_tuples` macro to generate impls.

### Builder pattern

This was the strategy we used for `SystemSet`.

```rust
.add_systems(SystemGroup:::new()
               .with(compute_attack), 
               .with(compute_defense),
               .with(check_for_crits),
               .with(compute_damage),
               .with(deal_damage),
               .with(check_for_death)
               .chain()
               .in_set(GameSet::Combat)
            )

.add_systems(SystemGroup::new().with(run).with(jump).after(InputSet::ReadInput))
```

Oof. Very verbose and cluttered, even with a change from `with_system` to `with`.
Placement of brackets can be confusing, and there's no clear seperation between systems and their shared config.

This is particularly painful for small groups, and is not significantly more ergonomic than just adding the systems individually.

### Vec-style macro

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
   .chain()
   .in_set(GameSet::Combat)
)

.add_systems(systems![run, jump].after(InputSet::ReadInput))
```

Reasonably pretty, but still more verbose than the tuple option.
Requires boilerplate invocation of a macro every single time these methods are used.
Not any less magic than the tuples.

## Unresolved questions

- What syntactic strategy should we use for `app.add_systems`?

## Future possibilities

There are a few proposals that should be considered immediately, hand-in-hand with this RFC:

1. If-needed ordering constraints ([RFC #47](https://github.com/bevyengine/rfcs/pull/47)).
2. Opt-in automatic insertion of flushing systems for command and state (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
3. Schedule commands and schedule merging.
   1. This is complex enough to warrant its own RFC, and should probably be considered in concert with multiple worlds.

In addition, there is quite a bit of interesting but less urgent follow-up work:

1. Atomic groups ([RFC #46](https://github.com/bevyengine/rfcs/pull/46)), to ensure that systems are executed in a single block.
2. Opt-in stack-based states (possibly in an external crate).
3. More complex strategies for run criteria composition.
   1. This could be useful, but is a large design that can largely be considered independently of this work.
   2. How does this work for run criteria that are not locally defined?
4. A more cohesive look at plugin definition and configuration strategies.
5. A [graph-based system ordering API](https://github.com/bevyengine/bevy/pull/2381) for dense, complex dependencies.
6. Warn if systems that emit commands do not have an appropriate command-flushing ordering constraint.
7. Run schedules without exclusive world access, inferring access based on the contents of the `Schedule`.
8. Automatically add and remove systems based on `World` state to reduce schedule clutter and better support one-off logic.
9. Tooling to force a specific schedule execution order: useful for debugging system order bugs and precomputing strategies.
10. Better tools to tackle system execution order ambiguities.
11. Command-flushed ordering constraints.

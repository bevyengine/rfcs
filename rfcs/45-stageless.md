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

- schedules (simplified, each `App` can store multiple dynamically modifiable schedules)
- run criteria (can no longer loop, are now evaluated immediately before the systems they control)
- states (simplified, no longer purely run-criteria powered)
- fixed time steps (no longer a run criterion)
- exclusive systems (no longer special-cased)
- command processing (now performed in a `flush_commands` exclusive system)
- labels (can now be directly configured)
- system sets (use `.add_systems` instead)
- stages (yoten)

### Scheduling overview and intent-based configuration

Systems in Bevy are stored in a `Schedule`: a collection of **configured systems**.
In each iteration of the schedule (typically corresponding to a single frame), the `App`'s `runner` function will run the schedule,
causing the **scheduler** to run the systems in parallel using a strategy that respects all of the configured constraints.
If these constraints cannot be met (for example, a system may want to run both before and after another system), the schedule is said to be **unsatisfiable**, and the scheduler will panic.
Schedules can be nested and branched from within **exclusive systems** (which have mutable access to the entire `World`), allowing you to encode arbitrarily complex control flow using ordinary Rust constructs.

By default, each new `Schedule` is entirely unordered: systems will be selected in an arbitrary order (which is not guaranteed to be stable across passes of the schedule) and run if and only  if all of the data that they must access are free.
Just like with standard borrow checking, multiple systems can read from the same data at once, but writing to the data requires an exclusive lock.
Systems which cannot be run in parallel are said to be **incompatible**.

In Bevy, schedules are ordered via **intent-based configuration**: record the exact, minimal set of constraints needed for a scheduling strategy to be valid, and then let the scheduler figure it out!
As the number of systems in your app grows, the number of possible scheduling strategies grows exponentially, and the set of *valid* scheduling strategies grows almost as quickly.
However, each system will continue to work hand-in-hand with a small number of closely-related systems, interlocking in a straightforward and comprehensible way.

By configuring our systems with straightforward, local rules, we can allow the scheduler to pick one of the possible valid paths, optimizing as it sees fit without breaking our logic.
Just as importantly, we are not over-constraining our ordering. Subtle ordering bugs will surface quickly and then be stamped out with a well-considered rule.

### The `App` stores multiple schedules

Each `App` can store multiple `Schedules` in a `HashMap<Box<dyn ScheduleLabel>, Schedule>`.
Schedules allow you to cleanly execute coherent blocks of logic without sacrificing parallelism.

By default, each app stores both a startup (`DefaultSchedule::Startup`) and main (`DefaultSchedule::Main`) schedule: the former runs only once on app creation, while the latter loops endlessly.
However, schedules are a surprisingly powerful and flexible tool: they can be used to handle initialization and cleanup logic when working with states or used ad-hoc to run game logic in complex patterns.

By default, systems are added to the main schedule.
You can control this by adding the `.to_schedule(ScheduleLabel::Variant)` system descriptor to your system or label.
To insert a new scedule into the `Schedules` collection, use `app.insert_schedule(label, schedule)`.

To read (but not write) the schedules stored in your app, request `&Schedules` as a system parameter.
Unsurprisingly, this never conflicts with entities or resources in the `World`, as they are stored one level higher.

#### Startup systems

Startup systems are stored in their own schedule, with the `DefaultSchedule::Startup` label.
When using the runner added by `MinimalPlugins` or `DefaultPlugins`, this schedule will run exactly once on app startup.

You can add startup systems with the `.add_startup_system(on_startup)` method on `App`, which is simply sugar for `.add_system(on_startup.to_schedule(DefaultSchedule::Startup))`.

### Introduction to system configuration

Each system has a few configurable parameters:

- it may have **ordering constraints**, causing it to run before or after other systems
  - there are several subtly different types of ordering constraints, see the next section for details
  - e.g. `.add_system(player_controls.before(GameLabels::Physics).after(CoreLabels::Input))`
- it may have one or more **run criteria** attached
  - a system is only executed if all of its run criteria return `true`
  - **states** are a special, more complex pattern that use run criteria to determine whether or not a system should run in a current state
  - e.g. `.add_system(physics.run_if_resource_equals(EnablePhysics(true)))`
- it may have one or more **labels**, allowing other systems to refer to it and enabling mass-configuration
  - e.g. `.add_system(physics.label(GameLabels::Physics)`

This configuration is additive: adding another ordering constraint, run criteria or label does not replace the existing configuration.

System configuration can be stored in a `SystemConfig` struct. This can be useful to reuse, compose and quickly apply complex configuration strategies, before applying these strategies to labels or individual systems.

### Label configuration

Each system label has its own associated `SystemConfig`, stored in the corresponding `Schedule`.
When a label is configured, that configuration is passed down in an additive fashion to each system with that label.

You can apply the same label(s) to many systems at once using the `App::add_systems(systems: impl SystemIterator)` method.

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
        let mut common_physics_config = SystemConfig::new().after(CoreLabel::InputHandling).before(CoreLabel::Rendering);

        app
        // We can reuse this shared configuration on each of our labels
        .configure_label(Physics::Forces.configure(common_physics_config).before(Physics::CollisionDetection))
        .configure_label(Physics::CollisionDetection.configure(common_physics_config).before(Physics::CollisionHandling))
        .configure_label(Physics::CollisionHandling.configure(common_physics_config))
        // And then apply that config to each of the systems that have this label
        .add_system(gravity.label(Physics::Forces))
        // These systems have a linear chain of ordering dependencies between them
        // Systems earlier in the chain must run before those later in the chain
        // Other systems can run in between these systems
        .add_systems((broad_pass, narrow_pass).chain().label(Physics::CollisionDetection))
        // Add multiple systems as once to reduce boilerplate!
        .add_systems((compute_forces, collision_damage).label(Physics::CollisionHandling));
    }
}
```

Labels cannot themselves be labelled: composition is generally clearer than ad-hoc inheritance, and this allows us to compute the configuration for each system more quickly.
If you need to copy the behavior of a label, use `.configure_label(TargetLabel.configure(SourceLabel.config()))`.
If you need to ensure that your systems are treated as if they were nested within another label to ensure that other systems have the correct relative ordering, also add that label to your systems.

### System ordering

The most important way we can configure systems is by telling the scheduler *when* they should be run.
The ordering of systems is always defined relative to other systems: either directly or by checking the systems that belong to a label.

Applying an ordering constraint to or from a label causes a ordering constraint to be created between all individual members of that label.
If an ordering is defined relative to a non-existent system or an unused label, it will have no effect, emitting a warning.
This relatively gentle failure mode is important to ensure that plugins can order their systems with relatively strong assumptions that the default system labels exist, but continue to (mostly) work if those systems or labels are not present.

In addition to the `.before` and `.after` methods, you can use **system chains** to create very simple linear dependencies between the successive members of an array of systems.
(Note to readers: this is not the same as "system chaining" in Bevy 0.6 and earlier: that concept has been renamed to "system piping".)

```rust
fn main(){
   App::new()
   // This systems are connected using a string of ordering constraints
   .add_systems((compute_attack, 
                 compute_defense,
                 check_for_crits,
                 compute_damage,
                 deal_damage,
                 check_for_death)
                 .chain()
                 // We can configure and label all systems in the chain at once
                 .label(GameLabel::Combat)
   )
   .run()
}
```

#### Ordering with `Commands`

Commands (commonly used to spawn and despawn entities or add and remove components) do not take effect immediately.
Instead, they are applied whenever the `flush_commands` system runs.
This exclusive system collects all created commands and applies them to the `World`, mutating it directly.

This pattern is so common that a special form of ordering constraint exists for it: **command-flushed ordering constraints**.
If system `A` is `before_and_flush` system `B`, the schedule will be unsatisfiable unless there is an intervening `flush_commands` system.
Note that **this does not insert new copies of a `flush_commands` system** (yet).
Instead, it has two purposes:

- acting like an assert statement, ensuring that the schedule is set up the way you expect it to be (even after refactors)
- allows plugin authors to safely export configurable groups of systems that can be inserted into the schedule where the user wishes
  - without this constraint, requirements that the user place one group before a command sync and another after would silently break

```rust
use bevy::prelude::*;

#[derive(SystemLabel)]
enum StartupLabel{
   CommandFlush,
   UiSpawn,
}

/// This example has no `DefaultPlugins`,
/// so all command flushing is done manually
fn main(){
   App::new()
   // All systems that do not have this label are implicitly before this label
   .add_system(flush_commands.label(CoreLabel::Last))
   // Recall that this adds systems to the startup schedule, not the main one
   .add_startup_system(flush_commands.label(CoreLabel::Last))
   // Commands will be processed in the basic flush_commands system that occurs at the end of the schedule
   .add_startup_system(spawn_player)
   // We need to customize this after it's spawned
   .add_startup_system(spawn_ui.before(StartupLabel::CommandFlush).label(StartupLabel::UiSpawn))
   .add_startup_system(customize_ui.after(StartupLabel::CommandFlush).after_and_flush(StartupLabel::UiSpawn))
   .add_startup_system(flush_commands.label(StartupLabel::CommandFlush);
   // Less verbosely, we can use the `.chain()` helper method
   .add_systems((spawn_ui, flush_commands, customize_ui).chain().to_schedule(CoreSchedule::Startup))
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
      .after_and_flush(spawn_projectiles).after(CommandFlush::PostUpdate)
    )
    // If we need to add more command flushes to make our logic work, we can manually insert them
    .add_system(flush_commands.label("DespawningFlush").after(CommandFlush::PostUpdate).before(CommandFlush::EndOfFrame))
    .add_system(fork_projectiles.after("DespawningFlush"));
  }
}
```

### Run criteria

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
    // Run criteria can be attached to labels: a copy of the run criteria will be applied to each system with that label
    .configure_label(GameLabel::Physics.run_if_resource_equals(Paused(false)))
    .run();
}
```

There are a few important subtleties to bear in mind when working with run criteria:

- when multiple run criteria are attached to the same system, the system will run if and only if all of those run criteria return true
- run criteria are evaluated "just before" the system that it is attached to is run

- if a run criterion is attached to a label, a run-criterion-system will be generated for each system that has that label
  - this is essential to ensure that run criteria are checking fresh state without creating very difficult-to-satisfy ordering constraints
  - if you need to ensure that all systems behave the same way during a single pass of the schedule or avoid expensive recomputation, precompute the value and store the result in a resource, then read from that in your run criteria instead

Run criteria are evaluated by the executor just before the system that they are controlling is run: it is impossible to modify the data that they rely on in an observable way before the system completes.
This is important to ensure that the state of the world always matches the state expected by the system at the time it is executed.

### States

**States** are one particularly common and powerful form of run criteria, allowing you to toggle systems on-and-off based on the value of a given resource and smoothly handle transitions between states by running cleanup and initialization systems.
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
Note that commands are not automatically flushed between state transitions: if this is required, add a copy of `flush_commands` to your schedule.

When states are added using `App::add_state::<S: State>(initial_state)`, one `flush_state<S>` system is added to the app, with the `GeneratedLabel::StateTransition<S>` label.
You can configure when and if this system is scheduled by configuring this label, and you can add additional copies of this system to your schedule where you see fit.
Just like with commands, **state-flushed ordering constraints** can be used to verify that state transitions have run at the appropriate time.
If system `A` is `before_and_flush_state::<S>` system `B`, the schedule will be unsatisfiable unless there is an intervening `flush_state<S>` system.

Apps can have multiple orthogonal states representing independent facets of your game: these operate fully independently.
States can also be defined as a nested enum: these work as you may expect, with each leaf node representing a distinct group of systems.
If you wish to share behavior among siblings, add the systems repeatedly to each sibling, typically by saving a schedule and then using `Schedule::merge_schedule` to combine that into the specialized schedule of choice.

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

### Tools to help you work with schedules

Bevy provides some tools to help you keep your schedule properly woven as your game grows in complexity and is refactored.

#### Ambiguous execution order

The first of these is a tool that reports pairs of systems with an **ambiguous execution order**.
A pair of systems has an **ambiguous execution order** if they are incompatible and do not have any ordering constraints between them, either directly or indirectly.
This can only happen if one system could read a piece of data that the other could write.
In some passes of the schedule, you will read fresh, updated state, and in other passes you will read state from the pass before.

Most of the time, this is a logic bug (although it may just result in a one-frame delay).
By default, whenever schedules are constructed, Bevy will report how many execution order ambiguities it found, allowing you to watch for sudden, unexpected jumps in the number of ambiguities due to a refactoring of your schedule structure.
You can change this behavior by configuring the `ExecutionOrderAmbiguities` resource.

```rust
enum ExecutionOrderAmbiguities{
   /// The schedule will not be checked for ambiguities
   ///
   /// This behavior will very slightly reduce startup time and eliminate annoying warnings during prototyping.
   Allow,
   /// The number of ambiguities is reported to the console
   ///
   /// If this number suddenly jumps, you have likely just removed a foundational part of your schedule structure.
   WarnCount,
   /// The details of each ambiguity is reported to the console
   ///
   /// This can be very useful for debugging. All stricter levels also produce the same verbose output
   WarnVerbose,
   /// The schedule will panic unless amibguities are explicitly allow by a shared label
   ///
   /// Some ambiguities are inconsequential, and can be ignored.
   Deny, 
   /// The schedule will panic if an ambiguity is found
   ///
   /// This level is likely too strict unless perfect determinism is required
   Forbid,
}
```

This resource effectively acts as a lint for your schedule, allowing you to control the level of strictness you desire.
Execution order ambiguities can be explicitly allowed by giving the conflicting systems a shared label, and then configuring that label to `.allow_ambiguities`.
This can be helpful to clean up execution order ambiguities that you genuinely do not care about, or when working with operations (such as addition of integers) whose operations are genuinely commutative.
Note that execution order ambiguities are based on **hypothetical incompatibility**: the scheduler cannot know that entities with both a `Player` and `Tile` component will never exist.
You can fix these slightly silly execution order ambiguities by adding `Without` filters to your queries.

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
   2. Store multiple schedules in an `App` using a dictionary approach
   3. Support adding systems to the schedule
      1. These should be initialized by definition: see [PR #2777](https://github.com/bevyengine/bevy/pull/2777)
   4. Support adding systems to specific schedules and create startup system API
   5. Special-cased data access control for "the entire world" and `Schedules` to support exclusive systems
3. Rebuild core command processing machinery
   1. Begin with trivial event-style commands
   2. Explore how to regain parallelism
4. Add system configuration
   1. Create `SystemConfig` type
   2. Systems labels store a `SystemConfig`
   3. Allow systems to be labelled
   4. Store `ConfiguredSystems` in the schedule
   5. Add `.add_systems` method, the `SystemGroup` type and the convenience methods for system groups
   6. Special-case `CoreLabel::First` and `CoreLabel::Last` for convenience
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
   // All labels are converted to specific systems.
   // `Graph` is a bikeshed-avoidance graph storage structure,
   // the exact strategy should be benchmarked
   ordering_constraints: Graph<SystemId>,
   // Label configuration must be stored at the schedule level,
   // rather than attached to specific label instances
   labels: HashMap<Box<dyn SystemLabel>, LabelConfig>,
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

System configuration stores ordering dependencies, run criteria and labels.
It can be created raw, or stored in association with either system labels or systems.

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
   // This vector is always empty when this struct is used for labels
   labels: Vec<Box dyn SystemLabel>,
}

struct LabelConfig {
   system_config: SystemConfig,
   allow_ambiguities: bool,
}

```

### Label configuration

When a property is applied to a label, it is ultimately applied to each system in the label, as if it had been labelled individually.
Of course, properties that only make sense at a group level (such as whether ambiguities within that label are allowed) are not propagated down, as they are not defined for inidividual systems.

When building the constraint graph, all labels are expanded into the set of systems that they contain.
Thus, adding an ordering constraint between label N and M is the same as adding a constraint between each `n * m` pair of systems.

### Run criteria

Run criteria can access arbitrary data from the world, have their own fetches and are run on the world.
While they reuse much of the system machinery, they have several important constraints:

- they cannot mutate data
  - required to avoid having to order run criteria relative to each other
- they cannot capture or store local data
  - required to avoid user footguns
  - required to efficiently special-case their execution
- they cannot be configured
  - required to avoid user footguns
  - required to efficiently special-case their execution
- they must return a `bool`, so we know whether the system they're controlling should run!

It's worth calling out that run criteria will always be evaluated seperately, even if applied to an entire label.
This is required to ensure that run criteria are evaluated atomically with the systems they control.

When the executor is presented with a system, it must choose whether or not the system can be run:

- check that the joint data access of the system and its run criteria are available
  - if not, the system is put back into the queue and the next system is checked
- evaluate each run criteria on the `World`, in an arbitrary order
  - return early and skip the system if any run criteria return `false`
- if and only if all of the run criteria have returned `true`:
  - spawn a task
  - lock the data access required by the system
  - run the system
  - release the locks when the system completes

This approach is very light-weight (as the overwhelming majority of run criteria are extremely simple).
It allows us to avoid ever locking data for run criteria, and skip spawning a new task for systems that are skipped.

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

Schedules can be unsatisfiable for several reasons:

1. The graph of label ordering constraints contains a cycle.
   1. Each label is treated as a node, and each type of ordering constraint is treated as the same type of edge.
2. The cached schedule graph contains a cycle.

### Change detection

The reliable change detection users have come to know and love is completely unaffected by these architectural changes. Each world still has an atomic counter that increments each time a system runs (skipped systems do not count) and tracks change ticks for its components and systems.

For a system to detect changes (assuming one of its queries has a `Changed<T>` filter), its tick and the change ticks of any matched components are compared against the current world tick. If the system's last run is older than a change, the query will yield that component value.

```rust
fn is_changed(world_tick: u32, system_tick: u32, change_tick: u32) -> bool {
   let ticks_since_change = world_tick.wrapping_sub(change_tick);
   let ticks_since_system = world_tick.wrapping_sub(system_tick);
   ticks_since_system > ticks_since_change
}
```

To ensure graceful operation, change ticks are periodically scanned, and those older than a certain threshold are clamped to prevent their age from overflowing. Because of this, a system will never falsely report or miss a change provided its age and the age of the changed component have not *both* saturated.

Every schedule will perform a scan once at least `N` (currently several million) ticks have elapsed since its previous scan, and no more than `2N - 1` ticks should occur between scans. The latter can be circumvented if nested schedules are used in strange ways, but most users aren't likely to encounter these edge cases in practice.

```rust
pub const CHANGE_DETECTION_MAX_DELTA: u32 = u32::MAX - (2 * N);

fn check_tick(world_tick: u32, saved_tick: &mut u32) {
   let delta = world_tick.wrapping_sub(*saved_tick);
   if delta > CHANGE_DETECTION_MAX_DELTA {
      *saved_tick = world_tick.wrapping_sub(CHANGE_DETECTION_MAX_DELTA);
   }
}
```

## Drawbacks

1. This will be a massively breaking change to every single Bevy user.
   1. Any non-trivial control over system ordering will need to be completely reworked.
   2. Users will typically need to think a bit harder about exactly when they want their gameplay systems to run. In most cases, they should just add the `CoreLabel::AppLogic` label to them, which will place them after input and before rendering.
2. It will be harder to immediately understand the global structure of Bevy apps.
   1. Powerful system debugging and visualization tools become even more important.
   2. Labels do not have a hierarchy and can be entangled.
   3. If multiple labels are added to a single system it can suddenly create wide-spread consequences (particularly with strict ordering constraints).
3. State transitions are no longer queued up in a stack.
   1. This also removes "in-stack" and related system groups / logic.
   2. This can be easily added later, or third-party plugins can create their own abstraction.
4. When a state transition occurs, the systems in the while-active set are no longer applied immediately.
   1. Instead, the normal flow of logic continues.
   2. Users can duplicate critical logic in the while-active and on-enter collections.
   3. Similarly, arbitrarily long chains of state transitions are no longer be processed in the same schedule pass.
   4. However, users can add as many copies of the `flush_state` system as they would like, and loop it within an exclusive system.
5. It will become harder to reason about exactly when command flushing occurs.
6. Ambiguity sets are not directly accounted for in the new design.
   1. They need more thought and are rarely used; we can toss them back on later.

## Rationale and alternatives

### Doesn't label-configuration allow you to configure systems added by third-party plugins?

Yes. This is, in fact, a large part of the point.

The complete inability to configure plugins is the source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
Plugin authors cannot know all of the needs of the end user's app, and so some degree of control must be handed back to the end users.

This control is carefully limited though: systems can only be configured if they are given a public label, and all of the systems under a common label are configured as a group.
In addition, configuration defined by a private label (or directly on systems) cannot be altered, extended or removed by the app: it cannot be named, and so cannot be altered.

These limitation allows plugin authors to carefully design a public API, ensure critical invariants hold, and make serious internal changes without breaking their dependencies (as long as the public labels and their meanings are relatively stable).

### Why can't we just use system piping (previously system chaining) for run criteria?

There are two reasons why this doesn't work:

1. The system we're applying a run criteria to does not have an input type.
2. System piping does not work if the `SystemParam` of the input and output systems are incompatible.

### Why aren't run criteria cached?

In practice, almost all run criteria are extremely simple, and fast to check.
Verifying that the cache is still valid will require access to the data anyways, and involve more overhead than simple computations on one or two resources.

In addition, this is required to guarantee atomicity.
We could add atomic groups to this proposal to get around this, but adding multiple atomic ordering constraints originating from a single system is extremely prone to dead-locks.
We should not hand users this foot-gun.

### Why do we want to store multiple schedules in the `App`?

We could store these in a resource in the `World`.
Unfortunately, this seriously impacts the ergonomics of running schedules in exclusive systems due to borrow checker woes.

Storing the schedules in the `App` alleviates this, as exclusive systems are now just ordinary systems: `&mut World` is compatible with `&Schedules`!

### What syntax should we use for collections of systems?

When using convenience methods like `add_systems`, we need to be able to refer to conveniently refer to collections of systems.
This will become increasingly important as more complex graph-builder APIs are added.

**In summary:**

- array syntax: optimal but literally impossible
- tuple syntax: magic but pretty
- builder syntax: way too verbose for a pattern that's supposed to increase convenience
- `vec!`-style macro: not any less magic than tuple syntax, not as pretty

**Conclusion:** tuple syntax, powered by builder syntax under the hood that users can use instead when it's more convenient.

Below, we examine the the options by example.

#### Arrays

```rust
.add_systems([compute_attack, 
                   compute_defense,
                   check_for_crits,
                   compute_damage,
                   deal_damage,
                   check_for_death]
                   .chain()
                   .label(GameLabel::Combat))

.add_systems([run, jump].after(CoreLabel::Input))
```

Very pretty. Doesn't work because types are heterogenous though.

#### Tuples

```rust
.add_systems((compute_attack, 
              compute_defense,
              check_for_crits,
              compute_damage,
              deal_damage,
              check_for_death)
              .chain()
              .label(GameLabel::Combat)
            )

.add_systems((run, jump).after(CoreLabel::Input))
```

Also pretty! Mildly cursed and definitely unidiomatic, but we use this flavor of cursed type system magic already for bundles.

Requires another invocation of the `all_tuples` macro to generate impls.

#### Builder pattern

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
               .label(GameLabel::Combat)
            )

.add_systems(SystemGroup::new().with(run).with(jump).after(CoreLabel::Input))
```

Oof. Very verbose and cluttered, even with a change from `with_system` to `with`.
Placement of brackets can be confusing, and there's no clear seperation between systems and their shared config.

This is particularly painful for small groups, and is not significantly more ergonomic than just adding the systems individually.

#### Vec-style macro

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
   .label(GameLabel::Combat)
)

.add_systems(systems![run, jump].after(CoreLabel::Input))
```

Reasonably pretty, but still more verbose than the tuple option.
Requires boilerplate invocation of a macro every single time these methods are used.
Not any less magic than the tuples.

## Unresolved questions

- Should while-active systems of states be handled using run criteria?
  - Currently, all of the while-active systems of the next state will be handled on the same frame
- Do we care about reimplementing the state stack as a first-party tool?
- How does this fit into the planned pipelining work?
  - Should be answered in the Multiple Worlds RFC.
- Is automatic inference of sync points required in order to make this design sufficiently ergonomic?

## Future possibilities

There are a few proposals that should be considered immediately, hand-in-hand with this RFC:

1. If-needed ordering constraints ([RFC #47](https://github.com/bevyengine/rfcs/pull/47)).
2. Opt-in automatic insertion of flushing systems for command and state (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
3. Multiple worlds (see [RFC #16](https://github.com/bevyengine/rfcs/pull/16), [RFC #43](https://github.com/bevyengine/rfcs/pull/43)), as a natural extension of the way that apps can store multiple schedules.
   1. This is needed to ensure the `SubApps` introduced for rendering pipelining work.
4. Schedule commands and schedule merging.
   1. This is complex enough to warrant its own RFC, and should probably be considered in concert with multiple worlds.

In addition, there is quite a bit of interesting but less urgent follow-up work:

1. Atomic groups ([RFC #46](https://github.com/bevyengine/rfcs/pull/46)), to ensure that systems are executed in a single block.
2. Opt-in stack-based states (possibly in an external crate).
3. More complex strategies for run criteria composition.
   1. This could be useful, but is a large design that can largely be considered independently of this work.
   2. How does this work for run criteria that are not locally defined?
4. A more cohesive look at plugin definition and configuration strategies.
5. A graph-based system ordering API for dense, complex dependencies.
6. Warn if systems that emit commands do not have an appropriate command-flushing ordering constraint.
7. Run schedules without exclusive world access, inferring access based on the contents of the `Schedule`.
8. Automatically add and remove systems based on `World` state to reduce schedule clutter and better support one-off logic.
9. Tooling to force specific schedule execution orders: useful for debugging system order bugs and precomputing strategies.

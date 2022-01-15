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
- Fixed timestep run criteria do not correctly handle the logic needed for robust game physics.
- The architecture for turn-based games and other even slightly unusual patterns is not obvious.
- We cannot easily implement a builder-style strategy for configuring systems.

Unfortunately, all of these problems are deeply interwoven. Despite our best efforts, fixing them incrementally at either the design or implementation stage is impossible, as it results in myopic architecture choices and terrible technical debt.

## User-facing explanation

This explanation is, in effect, user-facing documentation for the new design.
In addition to a few new concepts, it throws out much of the current system scheduling design that you may be familiar with.
The following elements are substantially reworked:

- schedules (flattened)
- run criteria (can no longer loop, are now systems)
- states (simplified, no longer purely run-criteria powered)
- fixed time steps (no longer a run criteria)
- exclusive systems (no longer special-cased)
- command processing (now performed in a `flush_commands` exclusive system)
- labels (can now be directly configured)
- system sets (now just used to mass-apply a label to a group of systems)
- stages (yoten)

### Scheduling overview and intent-based configuration

Systems in Bevy are stored in a `Schedule`: a collection of **configured systems**.
In each iteration of the schedule (typically corresponding to a single frame), the `App`'s `runner` function will run the schedule,
causing the **scheduler** to run the systems in parallel using a strategy that respects all of the configured constraints.
If these constraints cannot be met (for example, a system may want to run both before and after another system), the schedule is said to be **unsatisfiable**, and the scheduler will panic.
Schedules can be nested and branched from within **exclusive systems** (which have mutable access to the entire `World`), allowing you to encode arbitrarily complex control flow using ordinary Rust constructs.

In the beginning, each `Schedule` is entirely unordered: systems will be selected in an arbitrary order and run if and only if all of the data that it must access is free.
Just like with standard borrow checking, multiple systems can read from the same data at once, but writing to the data requires an exclusive lock.
Systems which cannot be run in parallel are said to be **incompatible**.

In Bevy, schedules are ordered via **intent-based configuration**: record the exact, minimal set of constraints needed for a scheduling strategy to be valid, and then let the scheduler figure it out!
As the number of systems in your app grows, the number of possible scheduling strategies grows exponentially, and the set of *valid* scheduling strategies grows almost as quickly.
However, each system will continue to work hand-in-hand with a small number of closely-related systems, interlocking in a straightforward and comprehensible way.

By configuring our systems with straightforward, local rules, we can allow the scheduler to pick one of the possible valid paths, optimizing as it sees fit without breaking our logic.
Just as importantly, we are not over-constraining our ordering, allowing subtle ordering bugs to surface quickly and then be stamped out with a well-considered rule.

### The `App` stores multiple schedules

The `App` can store multiple `Schedules` in a `HashMap<Box<dyn ScheduleLabel>, Schedule>` storage.
This is used for the enter and exit schedules of states, but can also be used to store, mutate and access additional schedules.
For safety reasons, you cannot mutate schedules that are currently being run: instead, you can defer their modification until the end of the main loop using `ScheduleCommands`.

The main and startup schedules can be accessed using the `DefaultSchedule::Main` and `DefaultSchedule::Startup` labels respectively.
By default systems are added to the main schedule.
You can control this by adding the `.to_schedule(MySchedule::Variant)` system descriptor to your system.

You can access the schedules stored in the app using the `&Schedules` or `&mut Schedules` system parameters.
Unsurprisingly, these never conflict with entities or resources in the `World`, as they are stored one level higher.

#### Startup systems

Startup systems are stored in their own schedule, with the `DefaultSchedule::Startup` label.
When using the runner added by `MinimalPlugins` and `DefaultPlugins`, this schedule will run exactly once on app startup.

You can add startup systems with the `.add_startup_system(on_startup)` method on `App`, which is simply sugar for `.add_system(on_startup.to_schedule(DefaultSchedule::Startup))`.

### Introduction to system configuration

A system may be configured in the following ways:

- it may have any number of **labels** (which implement the `SystemLabel` trait)
  - labels are used by other systems to determine ordering constraints
  - e.g. `.add_system(gravity.label(GameLogic::Physics))`
  - labels themselves may be configured, allowing you to define high-level structure in your `App` in a single place
    - e.g `app.configure_label(GameLogic::Physics.after(GameLogic::Input))`
- it may have ordering constraints, causing it to run before or after other systems
  - there are several subtly different types of ordering constraints, see the next section for details
  - e.g. `.add_system(player_controls.before(GameLabels::Physics).after(CoreLabels::Input))`
- it may have one or more **run criteria** attached
  - a system is only executed if all of its run criteria return `true`
  - **states** are a special, more complex pattern that use run criteria to determine whether or not a system should run in a current state
  - e.g. `.add_system(physics.run_if(GameLogic::Physics))`

System configuration can be stored in a `SystemConfig` struct. This can be useful to reuse, compose and quickly apply complex configuration strategies, before applying these strategies to labels or individual systems.

If you just want to run your game logic systems in the middle of your schedule, after input is processed but before rendering occurs, add the premade `CoreLabel::AppLogic` to them.

### Label configuration

Each system label has its own associated `SystemConfig`, stored in the corresponding `Schedule`.
This configuration is applied in an additive fashion to each system with that label.

You can define labels in a standalone fashion, configuring them at the time of creation:

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
		// Other systems can run in between these systems;
		// use `add_atomic_system_chain` if this is not desired
        .add_system_chain([broad_pass, narrow_pass].label(Physics::CollisionDetection))
        // System sets apply a set of labels to a collection of systems
        // and are helpful for reducing boilerplate
        .add_system_set([compute_forces, collision_damage], Physics::CollisionHandling);
    }
}
```

You can apply the same label(s) to many systems at once using **system sets** with the `App::add_system_set(systems: impl SystemIterator, labels: impl SystemLabelIterator)` method.

### System ordering

The most important way we can configure systems is by telling the scheduler *when* they should be run.
The ordering of systems is always defined relative to other systems: either directly or by checking the systems that belong to a label.

There are two basic forms of system ordering constraints:

1. **Strict ordering constraints:** `strictly_before` and `strictly_after`
   1. A system cannot be scheduled until "strictly before" systems must have been completed during this iteration of the schedule.
   2. Simple and explicit.
   3. Can cause unneccessary blocking, particularly when systems are configured at a high-level.
2. **If-needed ordering constraints:** `.before` and `.after`
   1. A system cannot be scheduled until any "before" systems that it is incompatible with have completed during this iteration of the schedule.
   2. In the vast majority of cases, this is the desired behavior. Unless you are using interior mutability, systems that are compatible will always be **commutative**: their ordering doesn't matter.

Applying an ordering constraint to or from a label causes a ordering constraint to be created between all individual members of that label.
If an ordering is defined relative to a non-existent system or label, it will have no effect, emitting a warning.
This relatively gentle failure mode is important to ensure that plugins can order their systems with relatively strong assumptions that the default system labels exist, but continue to (mostly) work if those systems or labels are not present.

In addition to the `.before` and `.after` methods, you can use **system chains** to create very simple linear dependencies between the succesive members of an array of systems.
(Note to readers: this is not the same as "system chaining" in Bevy 0.6 and earlier: that concept has been renamed to "system handling".)

When discussing system ordering, it is particularly important to call out the `flush_commands` system.
This **exclusive system** (meaning, it can modify the entire `World` in arbitrary ways and cannot be run in parallel with other systems) collects all created commands and applies them to the `World`.

Commands (commonly used to spawn and despawn entities or add and remove components) will not take effect until this system is run, so be sure that you run a copy of the `flush_commands` system before relying on the result of a command!

This pattern is so common that a special form of ordering constraint exists for it: **command-flushed ordering constraints**.
If system `A` is `before_and_flush` system `B`, the schedule will be unsatisfiable unless there is an intervening `flush_commands` system.
Note that **this does not insert new copies of a `flush_commands` system**: instead, it is merely used to verify that your schedule has been set up correctly according the specified constraint.

### Run criteria

While ordering constraints determine *when* a system will run, **run criteria** will determine *if* it will run at all.
A run criteria is a special kind of system, which can read (but not write) data from the `World` and returns a boolean value.
If its output is `true`, the system it is attached to will run during this pass of the `Schedule`;
if it is `false`, the system will be skipped.
Systems that are skipped are considered completed for the purposes of ordering constraints.
Run criteria cannot be labelled, otherwise ordered or themselves have run criteria: order other systems relative to the system they control instead.

Let's examine a few ways we can specify run criteria:

```rust
// This function can be used as a run criteria system,
// because it only reads from the `World` and returns `bool`
fn construction_timer_finished(timer: Res<ConstructionTimer>) -> bool {
    timer.finished()
}

// Timers need to be ticked!
fn tick_construction_timer(timer: ResMut<ConstructionTimer>, time: Res<Time>){
	timer.tick(time.delta());
}

fn main(){
    App::new()
    .add_plugins(DefaultPlugins)
	// We can add functions with read-only system parameters as run criteria
	.add_system_chain([tick_construction_timer, 
					   update_construction_progress.run_if(construction_timer_finished)])
    // We can use closures for simple one-off run criteria, 
    // which automatically fetch the appropriate data from the `World`
    .add_system(spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000))
    // The `run_if_resource_equals` method is convenient syntactic sugar that generates a run criteria
    // for when you want to check the value of a resource (commonly an enum)
    .add_system(gravity.run_if_resource_equals(Gravity::Enabled))
    // Run criteria can be attached to labels: a copy of the run criteria will be applied to each system with that label
    .configure_label(GameLabel::Physics.run_if_resource_equals(Paused(false)))
    .run();
}
```

There are a few important subtleties to bear in mind when working with run criteria:

- when multiple run criteria are attached to the same system, the system will run if and only if all of those run criteria return true
- run criteria are evaluated "just before" the system that is attached to is run
- if a run criteria is attached to a label, a run criteria system will be generated for each system that has that label
  - this is essential to ensure that run criteria are checking fresh state without creating very difficult to satify ordering constraints
  - if you need to ensure that all systems behave the same way during a single pass of the schedule or avoid expensive recomputation, precompute the value and store the result in a resource, then read from that in your run criteria instead

The state that run criteria read from will always be "valid" when the system that they control is being run: this is important to avoid strange bugs caused by race conditions and non-deterministic system ordering.
It is impossible, for example, for a game to be paused *after* the run criteria checked if the game was paused but before the system that relied on the game not being paused completes.
In order to understand exactly what this means, we need to understand atomic ordering constraints.

### Atomic ordering constraints

**Atomic ordering constraints** are a form of ordering constraints that ensure that two systems are run directly after each other.
At least, as far as "directly after" is a meaningful concept in a parallel, nonlinear ordering.
Like the atom, they are indivisible: nothing can get between them!

To be more precise, if a system `A` is `atomically_before` system `B`, nothing (other than system `B`) can mutate the data that system `A` could access.

In addition to their use in run criteria, atomic ordering constraints are extremely useful for forcing tight interlocking behavior between systems.
They allow us to ensure that the state read from and written to the earlier system cannot be invalidated.
This functionality is extremely useful when updating indexes (used to look up components by value):
allowing you to ensure that the index is still valid when the system using the index uses it.

Of course, atomic ordering constraints should be used thoughtfully.
As the strictest of the three types of system ordering dependency, they can easily result in unsatisfiable schedules if applied to large groups of systems at once.
For example, consider the simple case, where we have an atomic ordering constraint from system A to system B.
If for any reason commands must be flushed between systems A and system B, we cannot satisfy the schedule.
The reason for this is quite simple: the lock on data from system A will not be released when it completes, but we cannot run exclusive systems unless all locks on the `World's` data are clear.
As we add more of these atomic constraints (particularly if they share a single ancestor), the risk of these problems grows.
In effect, atomically-connected systems look "almost like" a single system from the scheduler's perspective.

On the other hand, atomic ordering constraints can be helpful when attempting to split large complex systems into multiple parts.
By guaranteeing that the initial state of your complex system cannot be altered before the later parts of the system are complete,
you can safely parallelize the rest of the work into seperate systems, improving both performance and maintainability.

### States

**States** are one particularly common and powerful form of run criteria, allowing you to toggle systems on-and-off based on the value of a given resource and smoothly handle transitions between states by running cleanup and initialization systems.
Typically, these are used for relatively complex, far-reaching facets of the game like pausing, loading assets or entering into menus.

The current value of each state is stored in a resource, which must derive the `State` trait.
These are typically (but not necessarily) enums, where each distinct state is represented as an enum variant.

Each state is associated with three sets of systems:

1. **Update systems:** these systems run each schedule iteration if and only if the value of the state resource matches the provided value.
   1. `app.add_system(apply_damage.in_state(GameState::Playing))`
2. **On-enter systems:** these systems run once when the specified state is entered.
   1. `app.add_system(generate_map.on_enter(GameState::Playing))`
3. **On-exit systems:** these systems run once when the specified state is exited.
   1. `app.add_system(autosave.on_exit(GameState::Playing))`

Update systems are by far the simplest: they're simply powered by run criteria.
`.in_state` is precisely equivalent to `run_if_resource_equals`, except with an additional trait bound that the resource must implement `State`.

On-enter and on-exit systems are stored in dedicated schedules, two per state, within the `App's` `Schedules`.
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

Occasionally, you may find yourself yearning for more complex system control flow than "every system runs once in a loop".
When that happens: **reach for an exclusive and run a schedule in it.**

Within an exclusive system, you can freely fetch the desired schedule from the `App` with the `&Schedules` (or `&mut Schedules`) system parameter and use `Schedule::run(&mut world)`, applying each of the systems in that schedule a single time to the world of the exclusive system.
However, because you're in an ordinary Rust function you're free to use whatever logic and control flow you desire: branch and loop in whatever convoluted fashion you need!

This can be helpful when:

- you want to run multiple steps of a set of game logic during a single frame (as you might see in complex turn-based games)
- you need to branch the entire schedule to handle a particularly complex bit of game logic
- you need to integrate a complex external service, like scripting or a web server
- you want to switch the executor used for some portion of your systems

If you need absolute control over the entire schedule, consider having a single root-level exclusive system.

#### Fixed time steps

Running a set of systems (typically physics) according to a fixed time step is a particularly critical case of this complex control flow.
These systems should run a fixed number of times for each wall-clock second elapsed.
The number of times these systems should be run varies (it may be 0, 1 or more each frame, depending on the time that the frame took to complete).

Simply adding run criteria is inadequate: run criteria can only cause our systems to run a single time, or not run at all.
By moving this logic into its own schedule within an exclusive system, we can loop until the accumulated time has been spent, running the schedule repeatedly and ensuring that our physics always uses the same elapsed delta-time and stays synced with the wall-clock, even in the face of serious stutters.

## Implementation strategy

Let's take a look at what implementing this would take:

1. Completely rip out:
   1. `Stage`
   2. `Schedule`
   3. `SystemDescriptor`
   4. `IntoExclusiveSystem`
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
   5. Add `.add_system_set` method
   6. Use [system builder syntax](https://github.com/bevyengine/rfcs/pull/31), rather than adding more complex methods to `App`
5. Add basic ordering constraints: basic data structures and configuration methods
   1. Begin with strict ordering constraints: simplest and most fundamental
   2. Add if-needed ordering constraints
6. Add atomic ordering constraints
7. Add run criteria
   1. Create `IntoRunCriteriaSystem` machinery
   2. Store the run criteria of each `ConfiguredSystem` in the schedule
   3. Add atomic ordering constraints
   4. Check the values of the run-criteria of systems before deciding whether it can be run
   5. Add methods to configure run criteria
8. Add states
   1. Create `State` trait
   2. Implement on-update states as sugar for simple run criteria
   3. Create on-enter and on-exit schedules
   4. Create sugar for adding systems to these schedules
9. Rename "system chaining" to "system handling"
   1. Usage was very confusing for new users
   2. Almost exclusively used for error handling
   3. New concept of "systems with a linear graph of ordering constraints between them" is naturally described as a chain
10. Add new examples
    1. Complex control flow with supplementary schedules
    2. Fixed time-step pattern
11. Port the entire engine to the new paradigm
    1. We almost certainly need to port the improved ambiguity checker over to make this reliable

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
   // Label configuration must be stored at the schedule level,
   // rather than attached to specific label instances
   labels: HashMap<Box<dyn SystemLabel>, SystemConfig>,
   // Used to quickly look up system ids by the type ids of their underlying function
   // when checking atomic ordering constraints
   system_function_type_map: HashMap<TypeId, Vec<SystemId>>,
   // Used to check if mutation is allowed
   currently_running: bool,
}
```

Configured systems are composed of a raw system and its configuration.

```rust
struct ConfiguredSystem{
   raw_system: RawSystem,
   config: SystemConfig,
   // All of the ordering constraints from `SystemConfig` 
   // are converted into individual systems.
   // Only the systems that this system are after are stored here.
   // This is what is actually checked by the scheduler to see if this system can be run.
   // All if-needed orderings are either converted to strict orderings at this step or removed.
   cached_ordering: CachedOrdering, 
}
```

Raw systems store metadata, cached state, `Local` system parameters and the function pointer to the actual function to be executed:

```rust
struct RawSystem {
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
   if_needed_ordering_before: Vec<OrderingTarget>,
   atomic_ordering_before: Vec<OrderingTarget>,
   strict_ordering_after: Vec<OrderingTarget>,
   if_needed_ordering_after: Vec<OrderingTarget>,
   atomic_ordering_after: Vec<OrderingTarget>,
   // The `TypeId` here is the type id of the flushing system
   flushed_ordering_before: Vec<(TypeId, OrderingTarget)>,
   flushed_ordering_after: Vec<(TypeId, OrderingTarget)>,
   run_criteria: Vec<SystemId>,
   // This vector is always empty when this struct is used for labels
   labels: Vec<Box dyn SystemLabel>,
}
```

### Atomic ordering constraints

To be fully precise, if we have system `A` that runs "atomically before" system `B` (`.add_system(a.atomically_before(b))`):

- `A` is strictly before `B`
- the "write" data locks of system `A` are relaxed to "read" data locks upon system `A`'s completion
- "read" data locks of system `A` will not be released until system `B` (and all other atomic dependents) have completed
- the data locks from system `A` are ignored for the purposes of checking if system `B` can run
  - after all, they're not actualy locked, merely reserved
  - competing read-locks caused by other systems will still need to complete before a data store can be mutated; the effect is to "ignore the locks of `A`", not "skip the check completely"
  - if and only if `A` has multiple atomic dependents, they cannot mutate the data read by `A`, as that would invalidate the state created by `A` that other dependents may be relying on

In addition, atomic ordering constraints introduce a new form of schedule unsatisfiability.
If a system must run between `A` and `B`, but can mutate data that `A` has access to, the schedule is unsatisfiable.

### Flushed ordering constraints

Being able to assert that commands are processed (or state is flushed) between two systems is a critical requirement for being able to specify logically-meaningful ordering of systems without a global view of the schedule.

However, the exact logic involved in doing this is quite complex.
We need to:

1. **When validating the schedule:** ensure that a flushing system of the appropriate type *could* occur between the pair of systems.
2. **When executing the schedule:** ensure that a flushing system of the appropriate type *does* occur between the pair of systems.

For this RFC, let's begin with the very simplest strategy: for each pair of systems with a flushed ordering constraint, ensure that a flushing system of the appropriate type is strictly after the first system, and strictly before the second.

We can check if the flushing system is of the appropriate type by checking the type id of the function pointer stored in the `RawSystem`.
This is cached in the `system_function_type_map` field of the `Schedule` for efficient lookup, yielding the relevant `SystemIds`.

With the ids of the relevant systems in hand, we can check each constraint: for each pair of systems, we must check that at least one of the relevant flushing systems is between them.
To do so, we need to check both direct *and* transitive strict ordering dependencies, using a depth-first search to walk backwards from the later system using the `CachedOrdering` until a flushing system is reached, and then walking back further from that flushing system until the originating system is reached.

## Drawbacks

1. This will be a massively breaking change to every single Bevy user.
   1. Any non-trivial control over system ordering will need to be completely reworked.
   2. Users will typically need to think a bit harder about exactly when they want their gameplay systems to run. In most cases, they should just add the `CoreLabel::AppLogic` label to them.
2. It will be harder to immediately understand the global structure of Bevy apps.
   1. Powerful system debugging and visualization tools become even more important.
3. State transitions are no longer queued up in a stack.
   1. By default, arbitrarily long chains of state transitions are no longer be processed in the same schedule pass.
   2. However, users can add as many copies of the `flush_state` system as they would like, and loop it within an exclusive system.
   3. This also removes "in-stack" and related system groups / logic.
   4. This can be easily added later, or third-party plugins can create their own abstraction.
4. It will become harder to reason about exactly when command flushing occurs.
5. Ambiguity sets are not directly accounted for in the new design.
   1. They need more thought and are rarely used; we can toss them back on later.

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

### Why can't we just use system handling (previously system chaining) for run criteria?

There are two reasons why this doesn't work:

1. The system we're applying a run criteria does not have an input type.
2. System handling does not work if the `SystemParam` of the original and handling systems are incompatible. This is far too limiting.

### Why aren't run criteria cached?

In practice, almost all run criteria are extremely simple, and fast to check.
Verifying that the cache is still valid will require access to the data anyways, and involve more overhead than simple computations on one or two resources.

In addition, adding multiple atomic ordering constraints originating from a single system is extremely prone to dead-locks; we should not hand users this foot-gun.

### Why do we want to store multiple schedules in the `App`?

We could store these in a resource in the `World`.
Unfortunately, this seriously impacts the ergonomics of running schedules in exclusive systems due to borrow checker woes.

Storing the schedules in the `App` alleviates this, as exclusive systems are now just ordinary systems: `&mut World` is compatible with `&mut Schedules`!

## Unresolved questions

- Are the access restrictions imposed by atomic ordering constraints correct and minimal?
- Should on-update systems of stages be handled using run criteria?
  - Currently, all of the update systems of the next state will be handled on the same frame
- Is automatic inference of sync points required in order to make this design sufficiently ergonomic?
- What is the best way to handle the migration process from an organizational perspective?
  - Get an RFC approved, and then merge a massive PR developed on a off-org branch?
    - Straightforward, but very hard to review
    - Risks divergence and merge conflicts
  - Use an official branch with delegated authority?
    - Easier to review, requires delegation, ultimately creates a large PR that needs to be reviewed
    - Risks divergence and merge conflicts
  - Develop `bevy_schedule3` on the `main` branch as a partial fork of `bevy_ecs`?
    - Some annoying machinery to set up
    - Requires more delegation and trust to ECS team
    - Avoids divergence and merge conflicts
    - Clutters the main branch

## Future possibilities

Despite the large scope of this RFC, it leaves quite a bit of interesting follow-up work to be done:

1. Opt-in automatic insertion of flushing systems for command and state (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
2. First-class indexes, built using atomic ordering constraints (and likely automatic inference).
3. Multiple worlds (see [RFC #16](https://github.com/bevyengine/rfcs/pull/43), [RFC #43](https://github.com/bevyengine/rfcs/pull/43)), as a natural extension of the way that apps can store multiple schedules.
4. Opt-in stack-based states (likely in an external crate).
5. More complex strategies for run criteria composition.
   1. This would be very useful, but is a large design that can largely be considered independently of this work.
   2. How does this work for run criteria that are not locally defined?
6. A more cohesive look at plugin definition and configuration strategies.
7. A graph-based system ordering API for dense, complex dependencies.
8. Store systems in the `World` as entities?
9. Warn if systems that emit commands do not have an appropriate command-flushing ordering constraint.

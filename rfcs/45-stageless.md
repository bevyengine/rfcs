# Feature Name: `stageless`

## Summary

Bevy's current scheduling architecture (as of 0.7) has a large number of issues due to internal complexity.
Certain concepts and behaviors are unnecessarily fused together and certain building blocks—stages, run criteria, states—are presented as independent but actually have significant internal coupling.
These properties frequently lead users into encounters with unexpected side effects and indecipherable errors.

This is a holistic redesign that decouples system storage from system scheduling, makes system labels more useful, slims down the app builder API, leverages exclusive systems to handle complex control flow (gives a clear purpose to everything in general), and more.

## Motivation

The fundamental challenges with the current stage-centered scheduling model are [numerous](https://github.com/bevyengine/bevy/discussions/2801).
Of particular note:

- Stages entangle ordering and command processing.
- Plugin authors have no tools to export their systems other than wrapping them in stages, but the scheduling of imported stages can't be controlled by the user.
- Systems and system sets cannot have multiple run criteria.
- There's no clear pattern for users to architect turn-based games.
- There's no clear pattern for users to make a state "span across" multiple stages with our state machine model.
- Our state machine model (a stack) doesn't enable enough use cases to warrant its complexity.
- There are just too many types and methods. (looking at you, "`.add_system_set_to_startup_stage`")

Unfortunately, all of these problems are deeply interwoven. Despite our best efforts, fixing them incrementally at either the design or implementation stage is impossible, as it results in myopic architecture choices and terrible technical debt.

## User-facing explanation

### Scheduling terminology and overview

Let's define some terms so that hopefully we're all on the same page.

- **system**: a stateful instance of a function that accesses data stored in a world
- **system set**: a system label that represents a group of systems and other sub-sets
- **condition**: a special kind of system that guards the execution of a system (set)
- **dependency**: a system (set) that must complete before the system (set) of interest can run
- **scheduler**: the programmer, who is attempting to control when and under what conditions systems run
- **schedule** (verb): specify when and under what conditions systems run
- **schedule** (noun): the executable form of a system set
- **executor**: executes a schedule on a world
- **"ready"**: when a system is no longer waiting for dependencies to complete

To build a Bevy app, users have to specify when their systems should run. By default, systems have no strict order nor conditions for execution. The process of specifying those is called **scheduling**. To make things more ergonomic, systems can be grouped into **system sets**, which can be ordered and conditioned in the same manner as systems. Furthermore, systems and system sets can be ordered together and grouped together *within larger sets*, which allows users to describe logical hierarchies.

In short, for any system or system set, users can:

- add execution order constraints relative to other systems or sets (e.g. "this system runs before A")
- add conditions that must be true for it to run (e.g. "this system only runs if a player has full health")
- add it to sets (which define common behavior for all systems within that set)

These properties are all additive.
Adding another does not replace an existing one, and they cannot be removed.

Example:

```rust
use bevy::prelude::*;

#[derive(State)]
enum GameState {
    Playing,
    Paused
}

#[derive(SystemLabel)]
enum GameSet {
    Menu,
    SubMenu,
}

fn main() {
    App::new()
        /* ... */
        // helper function that registers state machine resource
        // and adds OnEnter and OnExit sets for its states
        .add_states([GameState::Playing, GameState::Paused])
        .add_set(
            // systems and sets have the same API for scheduling
            GameSet::Menu
                // sets can go inside other sets
                .in_set(CoreSet::Update)
                // sets can have conditions, these must be true for the systems inside to run
                .run_if(state_equals(GameState::Paused))
        )
        .add_system(
            some_system
                .in_set(GameSet::Menu)
                // systems and sets can have multiple conditions (they must all be true)
                .run_if(some_condition_system)
                .run_if(|value: Res<Value>| value > 9000)
        )
        // multiple systems and sets can be added at once
        .add_many(
          // there are macros available for describing a linear sequence of systems and sets
          chain![
            GameSet::SubMenu,
            // process commands using a dedicated system
            apply_buffers.named("FlushMenu"),
          ]
          // configure all nodes in the chain at once
          .in_set(GameSet::Menu)
          .after(some_system)
        )
        /* ... */
        .run();
}
```

### Configuring run order

The simplest way users can configure systems is by saying *when* they should run using the `.before`, `.after`, and `.in_set` methods. These constraints are relative to another system or set. Run order constraints involving sets are eventually flattened into constraints between the individual systems. Any constraints the user specifies are used to derive dependency graphs, which (along with the data access) tells the engine where systems can run in parallel. The only limit is that every graph must be **solvable**. If any relationship constraint cannot be met (e.g. A was configured to run both before and after system B), it will result in an error.

Bevy provides the `App::add_many` method for configuring multiple systems and sets at once. Using `systems![a, b, c, ...].chain()` (or the `chain![a, b, c, ...]` shortcut) will create dependencies between the successive elements.
(Note: This is different from "system chaining" in Bevy 0.7 and earlier. That concept has been renamed to "system piping" to avoid overlap.)

```rust
fn main() {
    App::new()
        .add_many(chain![compute_damage, deal_damage, check_for_death].in_set(GameSet::Combat))
        .run()
}
```

A standard Bevy app will have built-in system sets, provided by either `MinimalPlugins` or `DefaultPlugins`.
In those apps, if no set is specified, then by default systems and sets are added to `CoreSet::Update`.

```rust
#[derive(SystemLabel)]
enum Physics {
    ComputeForces,
    DetectCollisions,
    HandleCollisions,
}

impl Plugin for PhysicsPlugin {
    fn build(app: &mut App){
        // Scheduling configs are reusable.
        let mut shared_config = Config::new()
            // Built-in fixed update set (replaces FixedTimestep).
            .in_set(CoreSet::FixedUpdate)
            .after(InputSet::ReadInputHandling);

        app
            .add_set(Physics::ComputeForces.configure_with(shared_config).before(Physics::DetectCollisions))
            .add_set(Physics::DetectCollisions.configure_with(shared_config).before(Physics::HandleCollisions))
            .add_set(Physics::HandleCollisions.configure_with(shared_config))
            .add_system(gravity.in_set(Physics::ComputeForces))
            .add_many(chain![broad_pass, narrow_pass, solve_constraints].in_set(Physics::DetectCollisions))
            .add_system(collision_damage.in_set(Physics::HandleCollisions));
    }
}
```

### Configuring run conditions (previously run criteria)

While ordering constraints determine *when* a system runs, **run conditions** determine *if* it runs at all. Run conditions consist of systems that have read-only data access and return `bool`.
A system or set can have any number of conditions, and it will only run if all of them return `true`. 
If any condition returns `false`, the system (or systems under the set) will be skipped.

Conditions are each evaluated *at most once* during a single pass of a schedule.
The evaluation occurs just before the system (or the first system encountered that is part of the set of interest) would be run.
Because of this, results are guaranteed to be up-to-date. The state read by the conditions cannot be modified in an observable way before the guarded system starts.

```rust
// This is just an ordinary system: timers need to be ticked!
fn tick_construction_timer(timer: ResMut<ConstructionTimer>, time: Res<Time>){
    timer.tick(time.delta());
}

// This function can be used as a condition because it does not mutate data and returns `bool`.
fn construction_timer_not_finished(timer: Res<ConstructionTimer>) -> bool {
    !timer.finished()
}

// You can use queries, event readers and resources with arbitrarily complex internal logic.
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
        .add_set(GameSet::Construction.run_if(construction_timer_not_finished))
        .add_many(systems![tick_construction_timer, update_construction_progress].in_set(GameSet::Construction))
        .add_system(system_meltdown_mitigation.run_if(too_many_enemies))
        // We can use closures for simple one-off conditions, 
        // which automatically fetch the appropriate data from the `World`.
        .add_system(spawn_more_enemies.run_if(|difficulty: Res<Difficulty>| difficulty >= 9000))
        // The `resource_exists` and `resource_equals` helper functions for generating
        // simple conditions (well, closures that convert into conditions).
        .add_system(gravity.run_if(resource_exists(Gravity)))
        // Conditions can be attached to system sets, so you can skip the whole set.
        .add_set(GameSet::Physics.run_if(resource_equals(Paused(false))))
        .run();
}
```

### Running scheduled systems

The systems, sets, and scheduling data added to an `App` is stored on the `World` in the `SystemRegistry` resource. This resource is also responsible for generating and maintaining the dependency graphs for the stored system sets.

In order to run a system or system set on the `World`, the system(s) must be extracted from the `SystemRegistry`. (This lets the `SystemRegistry` remain accessible by the running systems).

To do that, the `SystemRegistry` has methods to export a system or schedule given a label.
A **schedule** is an executable version of a system set, containing all the systems and conditions, as well as some metadata for an **executor** to run those in the correct order (efficiently).
After the schedule has been run, it can be returned to the `SystemRegistry` using another method.
This is the same process the default `App` runner uses to execute the startup sequence and main update loop.

### Command processing

Commands are arbitrary world modifications that are most commonly used to spawn or despawn entities and add or remove components.
Since those types of changes have unpredictable side effects on stored component data, commands are deferred until the next scheduled `apply_buffers` exclusive system runs.
If a system depends on the effects of commands in another system, you should make sure that there is a `apply_buffers` between them.

```rust
use bevy::prelude::*;

struct ProjectilePlugin;

impl Plugin for ProjectilePlugin {
    fn build(app: &mut App) {
        app
        .add_system(spawn_projectiles.before(CoreSystem::FlushPostUpdate))
        .add_many(
            chain![
                check_if_projectiles_hit,
                despawn_projectiles_that_hit,
                // Wherever you want to sync and apply commands, insert an instance of `apply_buffers`.
                apply_buffers.named("FlushProjectiles")
                fork_projectiles,
            ]
            .after(CoreSystem::FlushPostUpdate)
            .before(CoreSystem::FlushLast)
        )
    }
}
```

### Exclusive systems and control flow

Exclusive systems come in handy when users find themselves in need of something more complex than "each system runs once per schedule pass".
In those cases, you can **create an `&mut World` context like an exclusive system or command and run a schedule inside it.**
You're free to extract any schedule available in the `SystemRegistry` resource and run it on the world.

This is useful when you want to:

- repeat a sequence of game logic multiple times in a single frame
- have a branch in your schedule to handle some complex bit of game logic
- switch the executor used for some portion of your systems
- integrate an external service with arbitrary side effects, like scripting or a web server

Fixed timestep simulations and state machines are canonical examples of the first two bullet points.

### Fixed Timestep

A **fixed timestep** simulation advances a fixed number of times for each second elapsed. This most often used for physics (core gameplay in general).

When implemented in the main thread, it takes the form of a conditional loop that runs zero or more times in a frame (depending on how long the previous frame took to complete). When combined with using a constant `dt` value (equal to the fixed timestep) inside the simulation systems, the result produces consistent and repeatable behavior regardless of framerate (or presence of a GPU), even in the face of serious frame stutters.

*Unlike* current Bevy, this process no longer involves conditions (run criteria). Instead, the simulation logic is simply added to its own set that an exclusive system extracts and repeatedly runs in a loop.

### States

**Finite-state machines** allow ad-hoc execution of logic in response to state transitions. Typically, FSMs are used for relatively complex, high-level facets of an app like pausing, opening menus, and loading assets. FSMs are stored as individual resources and their state types must implement the `State` trait. State types are most often enums (or nested enums), where each enum variant represents a different state.

Each state has two associated system sets: an **on-enter** set that runs when the state is entered, and an **on-exit** set that runs when the state is exited. Neither set is part of another set.

```rust
fn main() {
    App::new()
        /* ... */
        .add_system(load_map.in_set(OnEnter(GameState::Playing)))
        .add_system(autosave.in_set(OnExit(GameState::Playing)))
        /* ... */
        .run();
}
```

You can queue a state transition by getting a mutable reference to the state machine resource (`ResMut<Fsm<S>>`) and then using the `Fsm::queue_transition` method within your system.

Since state transitions can be as open-ended as commands, they're deferred in a similar fashion until the next execution of an `apply_state_transition<S: State>` exclusive system.
This system first extracts the FSM using `World::resource_scope`, next runs the "on-exit" schedule of the current state, then runs the "on-enter" schedule of the queued state, and finally updates the FSM's state.

Note that commands are not automatically processed in or between those steps.
Add instances of the `apply_buffers` system to one or both sets if that is required.

## Implementation strategy

Let's take a look at what implementing this would take:

- Scheduling
  - Delete `SystemSet`, `Stage`, `Schedule`, and `SystemContainer`.
  - Add `&mut World` as a `SystemParam`.
  - Delete all the special exclusive system types and traits.
  - Go full [builder syntax](https://github.com/bevyengine/rfcs/pull/31) on descriptors instead of adding more `App` methods.
    - Rename the `.label` method on `IntoSystemDescriptor` trait to `.in_set`.
    - Add method to import saved config.
    - Add equivalent `IntoSetDescriptor` trait for types that impl `SystemLabel`.
    - Add `.named` method on `IntoSystemDescriptor` so users can give systems unique instance names.
    - Add `Descriptor` enum that generalizes system and set descriptors (for bulk scheduling).
- Storage
  - Build the `SystemRegistry` type.
    - Add method to change which set is default (i.e. `CoreSet::Update`) if none are specified.
      - Need special-casing for sets which that not have a parent, like state transitions.
    - Add `.add_many` method and some macros for convenient bulk scheduling.
- Run conditions (previously run criteria)
  - Define `IntoRunCondition` trait that can only be impl by compatible functions.
  - Update graph checker to include the access of run conditions when checking for ambiguous order between systems.
  - Update executor to include the access of run conditions when checking if systems can run.
- Commands
  - Add `apply_buffers` exclusive system.
    - This system just toggles a signal resource while the executor itself still flushes the buffers.
    - **Note**: The executor only sees the systems in the schedule being executed. Other systems would not be processed.
- Executor
  - Defer task creation until we know a system can and should run.
- States
  - Add `State` trait and derive macro.
  - Add finite-state machine resource type.
  - Add `OnEnter<S: State>(S)` and `OnExit<S: State>(S)` types and impl `SystemLabel` for them.
  - Add `apply_state_transition::<S: State>` exclusive system that will run the transition sets.
  - Add sugar:
    - Add `.add_states` method (to `SystemRegistry` and `App`) that registers the "on-enter" and "on-exit" sets for multiple states.
    - Add `state_equals` function that generates an `IntoRunCondition`-compatible closure for checking "if active".
- Misc.
  - Rename "system chaining" to "system piping" so we can use "chain" to describe the macro/type that applies sequential ordering constraints.
- Port the rest of the engine and examples to the new API.

Given the massive scope, that sounds relatively straightforward!
However, doing so will break approximately the entire engine, and tests will not pass again until everything has been ported.

Let's explore some implementation details.

### Scheduling

Systems and their scheduling information are bundled and stored.
The actual dependency graph construction is deferred (which means scheduling methods are order-independent).

```rust
enum Order {
    Before,
    After
}

struct Config {
    sets: HashSet<BoxedSystemLabel>,
    edges: Vec<(Order, BoxedSystemLabel)>,
    conditions: Vec<BoxedRunCondition>,
}

struct ConfiguredSystem {
    system: BoxedSystem,
    config: Config,
    name: Option<BoxedSystemLabel>,
}

struct ConfiguredSet {
    set: BoxedSystemLabel,
    config: Config,
}

trait IntoConfiguredSystem<Params> {
    fn configure(self) -> ConfiguredSystem;
    fn configure_with(self, config: Config) -> ConfiguredSystem;
    /// Configures the system to run before the system or set given by `label`.
    fn before<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystem;
    /// Configures the system to run after the system or set given by `label`.
    fn after<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSystem;
    /// Configures the system to run as part of `set`.
    fn in_set(self, set: impl SystemLabel) -> ConfiguredSystem;
    /// Configures the system to run only if `condition` returns `true`.
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSystem;
    /// Sets `name` as the unique name for this system.
    fn named(self, name: impl SystemLabel) -> ConfiguredSystem;
}

trait IntoConfiguredSet {
    fn configure(self) -> ConfiguredSet;
    fn configure_with(self, config: Config) -> ConfiguredSet;
    /// Configures the system set to run before the system or set given by `label`.
    fn before<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSet;
    /// Configures the system set to run after the system or set given by `label`.
    fn after<M>(self, label: impl AsSystemLabel<M>) -> ConfiguredSet;
    /// Configures the system set to run as part of `set`.
    fn in_set(self, set: impl SystemLabel) -> ConfiguredSet;
    /// Configures the system set to run only if `condition` returns `true`.
    fn run_if<P>(self, condition: impl IntoRunCondition<P>) -> ConfiguredSet;
}
```

### Scheduling errors

There are several reasons an app may error due to improper scheduling. Most are related to graph solvability.

1. Your dependencies contain a loop or cycle.
2. Your hierarchy contains a loop or cycle.
3. Your hierarchy has a transitive edge (something is both in some set and not at the same time.)
4. You called `.in_set` with a system's label.
5. (User-configurable) You have a dependency between two things that aren't in the same set.
6. (User-configurable) You referenced an "unknown" label (you forgot to add the system or set somewhere).
7. (User-configurable) You have two things with conflicting data access and ambiguous order.

By default, if something is ordered relative to an "unknown" system or set, that dependency will be ignored and the user will get a warning. We warn instead of error so that users can order their systems relative to systems and sets from plugins (particularly the default plugins) whether they exist or not.

### Storage

```rust
struct SystemRegistry {
    // Systems can only be run on the `World` that initialized them.
    world_id: Option<WorldId>,
    // Maps labels to identifiers.
    // We need to be able to quickly look up specific systems and sets or iterate over them.
    // `RegId` is unique identifier, generated on insertion.
    ids: HashMap<BoxedSystemLabel, RegId>,
    // ID of next thing to be registered.
    next_id: RegId,
    // Permanent system storage.
    systems: HashMap<RegId, Option<BoxedSystem>>,
    // Permanent conditions storage.
    conditions: HashMap<RegId, Option<Vec<BoxedRunCondition>>>,
    // Temporary storage and cached metadata for executing registered sets.
    schedules: HashMap<RegId, Option<Schedule>>,
    // Graph-related metadata for sets. Used for run-time modification.
    set_metadata: HashMap<RegId, SetMetadata>,
    // The "global" hierarchy of everything in the registry. Used for run-time modification.
    // `Graph` is a bikeshed-avoidance graph storage structure.
    hierarchy: Graph<RegId>,
    // Stuff that hasn't been initialized yet.
    uninit_nodes: Vec<(RegId, Config)>,
}

enum RegId {
    System(u64),
    Set(u64),
}

struct SetMetadata {
    // The graph of the systems and sets directly under this set.
    graph: Graph<RegId>,
    // The cached topological order for `graph`.
    topsort: Vec<RegId>,
    flattened: Graph<RegId>,
    flattened_topsort: Vec<RegId>,
    hierarchy: Graph<RegId>,
    hierarchy_topsort: Vec<RegId>,
    // The combined component access of everything under this set.
    combined_access: FilteredAccessSet<ComponentId>,
    // True if the schedule for this set needs to be regenerated.
    modified: bool,
}
```

### Run conditions (Run criteria)

Conditions are systems with several key constraints:

- They must return a `bool`.
- They cannot have system params that grant mutable access to data.
  - We don't have order to them.
  - Avoids side effects.
- They cannot capture or store local data.
  - Avoid footgun: local data would be modified even if another condition stops the guarded system from running.

When the executor is presented with a ready system, we do the following:

- Check that all the data accessed by the system, its run conditions (RC), and the RC of any sets it is under (that haven't been evaluated yet) is available.
  - If not, leave the system in the ready queue and move onto the next one.
- If yes, evaluate the RC of the non-evaluated sets that the system is under in hierarchical order (from the outermost set to the innermost set).
  - If any set's RC return false, mark all the systems (incl. current one) and sets under that set as completed/evaluated.
  - If all set's RC return true, mark the set as evaluated.
- If all the sets' RC returned true, now evaluate the system's RC.
  - If any of these return false, mark the system as completed.
- If all the system's RC returned true, run the system.
  - "Lock" the world data access required by the system.
  - Spawn a task for it.
  - Run that task.
  - Release the "lock" when the task completes.

Each condition is evaluated at most once during a single schedule pass. This lightweight approach minimizes task overhead. Since we don't spawn tasks for RC (which are extremely likely to be simple tests), we avoid locking other systems out of the data they access. Likewise, we avoid spawning tasks for systems that get skipped.

### Condition helper functions

Common conditions can be easily created using some ergonomic helper functions.

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

pub fn state_machine_exists<S: State>() -> impl FnMut(Option<Res<Fsm<S>>>) -> bool {
    move |fsm: Option<Res<Fsm<S>>>| fsm.is_some()
}

pub fn state_equals<S: State>(state: S) -> impl FnMut(Res<Fsm<S>>) -> bool {
    move |fsm: Res<Fsm<S>>| *fsm.current_state() == state
}

pub fn state_machine_exists_and_equals<S: State>(
    state: S,
) -> impl FnMut(Option<Res<Fsm<S>>>) -> bool {
    move |fsm: Option<Res<Fsm<S>>>| match fsm {
        Some(fsm) => *fsm.current_state() == state,
        None => false,
    }
}
```

### Change detection

The reliable change detection users have come to know and love stays the same.

There is one change, but it's about an internal implementation detail.

Change detection requires a periodic (but very, *very* infrequent) cleanup process to stay reliable. In Bevy 0.7 and earlier, an app checks if it's time to perform this cleanup at the end of each stage. Since stages no longer exist, now the app checks in each `apply_buffers` system and at the end of each pass of a schedule. (The check itself is trivial.)

## Drawbacks

1. This will be a massively breaking change to every single Bevy user.
    - Everybody is gonna need to port their apps to the new API.
    - Since they have more control, users will need to give more thought about when their systems should run.
2. It may become harder to understand an app schedule at a glance.
    - In particular, it may become harder to understand when commands are processed.
    - Better debugging and visualization tools become even more important (but easier to write).
    - Extra caution must be taken around systems that belong to multiple sets.
3. State transitions are no longer queued up in a stack.
    - This also removes "in-stack" and related system groups / logic.
    - With the simplified run criteria, the stack is trivial to re-implement, but preferably a third-party plugin does it.
    - The `apply_state_transition` system performs a single transition, doesn't do an endless cycle in one schedule pass.
4. States no longer have an "on update".
    - If a system must run on the same frame as the transition, it has to be in "on enter" and "on exit" (or your own DIY equivalent).

## Rationale and alternatives

### Why did you call them "sets" instead of "labels" or "categories"?

The word "set" better describes what labeling actually means for scheduling.
Sets aren't just an abstract tag, they have a "physical" location in the overall graph (a set is a "supernode" that encapsulates all the nodes inside it).
Likewise, sets that intersect (share nodes) can't be made to depend on each other.

### If users can configure system sets, can they mess with systems added by third-party plugins?

Yes, somewhat, and it's intentional.

Not being able to schedule plugin systems is a source of [significant user pain](https://github.com/bevyengine/bevy/issues/2160).
Instead of the current status quo, we thought a better pattern would be a plugin exporting its top-level set label (at the very least) and the user being responsible for deciding where to put it.

As it's now possible to define set hierarchies, plugin authors can decide exactly what they want to expose to users in their "public API" *and* still ensure any key logical invariants hold.
Users will only be able schedule what the plugin author chooses to make public.
Plugin authors could even make massive internal changes without breaking user apps (as long as the "public API" is relatively stable).

### Why is there no "on-update" system set for states?

Conceptually, a set occupies some position in a schedule.
However, simply being in a state does not.
Any system anywhere can be conditioned to only run if a certain state is active.
If there was a builtin "on-update" set, it could only exist in one place.
That, along with it being trivial to create a set with the condition that a state must be active, gave little reason to do it.

```rust
fn main() {
    App::new()
        /* ... */
        .add_system(
            my_system
            // A state being active has no inherent position in the schedule.
            // Any system anywhere can be told to only run if a certain state is active.
            .in_set(CoreSet::PostUpdate)
            // This helper generates a condition that returns `true` if the FSM is currently `GameState::Playing`.
            .only_if(state_equals(GameState::Playing))
        )
        /* ... */
        .run();
    )
}
```

## Unresolved questions

### What sugar should we use for adding multiple systems at once?

Convenience methods like `add_many` are important for reducing boilerplate. For these methods, we need to be able to refer to collections of systems and system sets.

**In summary:**

- builder syntax: marginal improvement over adding individual systems

- array syntax: looks pretty but is literally impossible
- tuple syntax: looks pretty but is limited to 12 elements
- `vec!`-like macro syntax: looks OK but, uh, uses macros

**Conclusion:** macro syntax, powered by builder syntax under the hood

Below, we examine the the options by example.

#### Builder pattern

This was the strategy we used for `SystemSet`.

```rust
.add_many(
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

.add_many(Nodes::new().with(run).with(jump).after(InputSet::ReadInput))
```

- looks bad
  - not much better than adding things individually
  - confusing placement of brackets
  - no clear seperation between systems and the group config
- no limit to number of elements

#### Arrays

```rust
.add_many([
        compute_attack, 
        compute_defense,
        check_for_crits,
        compute_damage,
        deal_damage,
        check_for_death,
    ]
    .chain()
    .in_set(GameSet::Combat))

.add_many([run, jump].after(InputSet::ReadInput))
```

- looks pretty
- no limit to number of elements
- doesn't work (different functions are different types, arrays must be homogoneous)

#### Tuples

```rust
.add_many((
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

.add_many((run, jump).after(InputSet::ReadInput))
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
.add_many(systems![
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

.add_many(systems![run, jump].after(InputSet::ReadInput))
```

- looks OK (some boilerplate)
- no limit to number of elements
- relies on macro magic

## Future possibilities

There are a few proposals that should be considered immediately, hand-in-hand with this RFC:

1. If-needed ordering constraints ([RFC #47](https://github.com/bevyengine/rfcs/pull/47)).
2. Atomic groups (sets) ([RFC #46](https://github.com/bevyengine/rfcs/pull/46)), to ensure that systems are executed as a single block.
3. Opt-in automatic insertion of command and state flushing systems (see discussion in [RFC #34](https://github.com/bevyengine/rfcs/pull/34)).
4. Command-flushed ordering constraints.

In addition, there is quite a bit of interesting but less urgent follow-up work:

1. Cloning registered systems to another registry. (This probably warrants its own RFC, and should probably be considered in concert with multiple worlds.)
2. Reintroduce stack-based states (ideally in an external crate).
3. More complex boolean expressions for composing run conditions.
4. A more cohesive look at plugin definition and configuration strategies.
5. A label-free [graph-based system ordering API](https://github.com/bevyengine/bevy/pull/2381) for specifying dense, complex dependencies.
6. Run schedules without `&mut World`, inferring access based on the contents of the `Schedule`.
7. Automatic insertion and removal of systems based on `World` state to reduce schedule clutter and better support one-off logic.
8. Tools to force a specific schedule execution order: useful for debugging system order bugs and precomputing strategies.
9. Better tools to tackle system execution order ambiguities.
10. Instead of triggering command sync with a resource, use a system that can actually drain the system command queues, which are stored on the `World` (i.e. in some interior mutable resource). This would potentially allow for people to play with alternative methods for applying commands i.e. parallelization.

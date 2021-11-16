# Feature Name: `many-worlds`

## Summary

Bevy apps: now with more worlds. And more schedules to match!
Transfer entities between worlds, clone schedules and spin up and down worlds with the `AppCommands`.
New `GlobalRes` resource type, which is accessible across worlds.

## Motivation

The ability to have multiple worlds (and unique groups of systems that run on them) is a very useful, reusable building block for separating parts of the app that interact very weakly, if at all.
Some of the more immediate use cases include:

1. A more unified design for pipelined rendering.
2. Entity staging grounds, where complete or nearly-complete entities can be stored for quick later use (see [bevy #1446](https://github.com/bevyengine/bevy/issues/1446)).
3. Entity purgatories, where despawned entities can be kept around while their components are inspected (see [bevy #1655](https://github.com/bevyengine/bevy/issues/1655)).
4. A trivially correct structure for parallel world simulation for scientific simulation use cases.
5. Look-ahead simulation for AI, where an agent simulates a copy of the world using the systems without actually affecting game state.
6. Isolation of map chunks.
7. Temporarily disabling entities in a system-agnostic fashion.

Currently, this flavor of design requires abandoning `bevy_app` completely, carefully limiting *every* system used (including base / external systems) or possibly some very advanced shenanigans with exclusive systems.

A more limited "sub-world" design (see #16 for further discussion) is more feasible, by storing a `World` struct inside a resource, but this design is harder to reason about, unnecessarily hierarchical and inherently limited.

## User-facing explanation

When you need to fully isolate entities from each other (or make sure logic doesn't actually run on some chunk of them), you can create **multiple worlds** in your Bevy app.

At this point, you should be familiar with the basics of both `Worlds` and `Schedules`.
Your `App` can store any number of these, each corresponding to a particular `WorldLabel`.

During each pass of the game loop, each schedule runs in parallel, modifying the world it is assigned to.
These worlds are fully isolated: they cannot view or modify any other world, and stages progress independently across schedules.
Then, the **global world** (which stores common, read-only data) is processed, and any systems in its schedule are run.
Finally, at the end of each game loop, all of the worlds wait for a synchronization step, and apply any `AppCommands` that were sent.

`AppCommands` allow you to communicate between worlds in various fashions.
You might:

- `spawn_world` or `despawn_world`, to create and manage new worlds
- `add_schedule`, `clone_schedule`, `reassign_schedule` or `remove_schedule` to apply logic to these new worlds
- `move_resource`, `send_other_world_command` or `send_other_world_event` to communicate between worlds
- transfer entities between worlds with `move_entities` or `move_query`
- update the value of global resources that are shared between worlds using `insert_global`

### WorldLabels

Each world has its own label, which must implement the `WorldLabel` trait in the same fashion as the system labels you're already familiar with.

By default, three worlds are added to your app: `CoreWorld::Global`, `CoreWorld::Main` and `CoreWorld::Rendering`.
By convention, the main world should contain the vast majority of your logic, while the global world should only contain global resources (and other read-only data) that need to be accessed across worlds.

### Global resources

Each app has its own collection of global resources, which can be viewed (but not modified) in each of the worlds.
This allows for convenient, non-blocking sharing of things like asset collections, settings and input events.

If needed, you can get a read-only view into the entire global world using the `GlobalWorld` system parameter.
This is non-blocking, as the global world cannot be modified in any fashion while other schedules are active.

### Example: parallel simulations

Suppose you're attempting to run a complex scientific simulation in parallel, and want to slightly vary the starting conditions or rules.
Because these simulations don't need to interact in any meaningful way, this is an ideal situation in which to use multiple worlds.

```rust
use bevy::prelude::*;

#[derive(WorldLabel)]
struct SimulationWorld(usize);


// These systems will be run in each of ours imulation worlds
struct SimulationLogicPlugin;
impl Plugin for SimulationLogicPlugin{
   fn build(self, app: &mut App){
      app
         .add_system(births_system)
         .add_system(deaths_system)
         .add_system(migration_system)
         // This system will send events to CoreWorld::Global
         // so we can collect the simulation state for reporting
         .add_system(report_status_system);
   }
}

fn main(){

   const N_WORLDS = 10;
   let birth_rates: Vec<BirthRate> = BirthRate::random(10);
   let death_rates: Vec<DeathRate> = DeathRate::random(10);

   let mut app = App::new();
   // These system is added to the global world's schedule,
   // which will run after all of our simulation world
   app.add_plugins(DefaultPlugins.to_world(CoreWorld::Global))
   // This system will collect information send by events from each
   // report_status_system in our parallel worlds,
   // allowing us to watch the simulation progress over time in our window
   app.add_system(plot_simulations_system.to_world(CoreWorld::Global));

   for i in 0..N_WORLDS {
      // Create a new world
      app.spawn_world(SimulationWorld(i))
         // Sets the currently active world for the App,
         // which controls the schedule to which systems are added
         // and where resources are inserted
         .set_world(SimulationWorld(i))
         // This ensures basic functionality like `Time` functions properly
         // for each of our worlds
         .add_plugins(MinimalPlugins)
         // Add the simulation configuration as resources
         // We could have also passed this into our plugin
         // using a method which returns a SimulationLogicPlugin
         .insert_resource(birth_rates[i])
         .insert_resource(death_rates[i])
         // Contains all of our logic,
         // and will run on each simulation world in parallel
         .add_plugin(SimulationLogicPlugin);
   }

   // The Main world isn't being used, so we may as well remove it
   app.remove_world(CoreWorld::Main);
   app.remove_schedule(CoreWorld::Main);

   // Now that everything is constructed, run the app!
   app.run();
}
```

### Example: chunking game state

In this example, we're demonstrating the structure of a simple game that contains several planets, each modelled as their own `World`.
Planets cannot interact, but we can send entities between these worlds using `AppCommands`.

```rust
use bevy::prelude::*;

// This is a dummy world label
// used to store the schedule that controls our logic
// which is replicated across our planets
#[derive(WorldId)]
struct GameLogic;

// In this App, we're using the main world to run our menu UI
// and then running game logic on each of our planet worlds independently.
// The active world is set with the ActiveWorld global resource
// to correspond to 
fn main(){
   App::new()
      // Added to CoreWorld::Main by default
      .add_plugin(MenuPlugin)
      .set_world(CoreWorld::Global)
      // We want a consistent unit of time across all of our worlds, so it's stored as a Global
      .add_plugins(MinimalPlugins.to_world(CoreWorld::Global))
      // We need an initial world for the player to spawn in
      .add_startup_system(generate_initial_world_system)
      // Respond to NewWorld events
      .add_system(new_world_system.to_world(Core))
      // Storing the game logic in a consistent place
      .add_schedule(GameLogic)
      .set_world(GameLogic)
      .add_plugin(LogicPlugin)
      .add_plugin(InputPlugin)
      .add_plugin(AudioPlugin)
      .run();
}

// Events that trigger our app to generate a new world
struct NewWorld;

fn generate_initial_world_system(mut app_commands: AppCommands){
   // We need to send this event to CoreWorld::Global,
   // since that's where new_world_system runs.
   // We could use a custom AppCommands instead of this indirection with events
   // but that is a bit more involved and harder to intercept in other systems
   app_commands.send_event(NewWorld, CoreWorld::Global);
}

// More NewWorld events are generated when we explore the galaxy
fn new_world_system(events: EventReader<NewWorld>, mut app_commands: AppCommands){
   for event in events.iter(){
      app_commands
      .spawn_world(event.world_label)
      // Each world needs its own copy of the logic
      // so then it can keep running when the player isn't looking
      .clone_schedule(GameLogic, event.world_label);      
   }
}

// This event is sent whenever a system 
struct BeamMeUp{
   // The entities to move between worlds
   entities: HashSet<Entities>,
   // This type should have a blanket impl for the underlying label,
   // purely for convenience
   target_world: Box<dyn WorldLabel>,
}

/// Move entities between worlds
// This system lives in LogicPlugin, and is replicated across each planet's worlds
fn transport_between_planets_system(events: EventReader<BeamMeUp>, app_commands: AppCommands, current_world: WorldLabel){
   for event in events.iter(){
      app_commands.move_entities(event.entities, current_world, event.target_world);
   }
}

/// Change which world entities are being fetched from for pipelined rendering and audio
// This system lives in LogicPlugin, and is replicated across each planet's worlds
// and is triggered when the player changes which world they want to watch via the UI
fn swap_planet_view(events: Events<SwapToWorld>, mut app_commands: AppCommands, current_world: WorldLabel){
   if events.iter().next().is_some(){
      // Demonstrates a hypothetical integration with pipelined-rendering
      // (and eventually audio), where we can swap which world the entities
      // to visualize and "audioize" are coming from
      app_commands.set_source_world(current_world);
   }
}
```

### Advanced example: world-model AI

In this example, we're using a multiple worlds approach in order to allow complex agents to simulate the world in order to plan ahead.
By taking this strategy, we can ensure that the agents always have an accurate model of the rules of the game without code duplication.

```rust
use bevy::prelude::*;

#[derive(WorldLabel)]
struct GameRulesWorld;

#[derive(WorldLabel)]
struct SimulationWorld(Uuid);

fn main(){
   App::new()
   // By storing all of our logic in its own world, we can quickly duplicate it
   .spawn_world(GameRulesWorld)
   .set_world(GameRulesWorld)
   .add_plugin(GameLogicPlugin)
   // We don't need or want all of the player-facing systems in our simulations
   .set_world(CoreWorld::Main)
   .add_plugins(DefaultPlugins)
   .add_plugin(UiPlugin)
   .add_plugin(RenderingPlugin)
   .add_plugin(InputPlugin)
   // We're sticking to a one-deep search for sanity here
   .add_system(simulate_action_system)
   .add_system(choose_action_system)
   .run();
}

// This system is contained within `GameLogicPlugin`
fn simulate_action_system(potential_actions: Res<PotentialActions>, mut app_commands: AppCommands, sim_query: Query<((), With<Simulate>)>){
   // Create a new world for each potential action
   for proposed_action in potential_actions.iter() {
      // These identifiers are uniquely generated and ephemeral
      let sim_world = SimulationWorld::new();

      // Making a simulation world that's identical to the main world in all relevant ways
      app_commands
         .spawn_world(sim_world)
         .clone_schedule(GameRulesWorld, sim_world)
         // As above, blocked by bevy #1515
         .clone_entities_to_world(sim_query.entities(), sim_world)
         // This system will evaluate the world state, send it back with an event reporting the values
         .add_system(evaluate_action_system.to_world(sim_world))
         .send_event(proposed_action, sim_world);
   }
}

struct EvaluateAction {
   proposed_action: Action,
   score: Score,
}

// This system somehow evaluates the goodness of the world state
// and then sends that information back to the main world
fn evaluate_action_system(mut proposed_actions: EventReader<ProposedActions>,
   app_commands: AppCommands, score: Res<Score>,
   current_world_label: CurrentWorld){
   
   // Exactly one proposed action should be sent per world
   let action: Action = proposed_actions.iter().next().unwrap().into();

   let evaluation = EvaluateAction{
      action,
      score: *score,
   }

   // Report back to the main world
   app_commands.send_event(evaluation, main_world);
   // We're only looking ahead one step, so we're done with the world now
   app_commands.despawn_world(current_world_label);
}

// Run in the main world, once we've evaluated our actions
fn choose_action_system(mut eval_events: EventReader<EvaluateAction>, mut action_events: EventWriter<Action>){

   let mut best_action = Action::Pass;
   let mut best_score = Score(0);

   for evaluation in eval_events.iter(){
      if evaluation.score >= best_score {
         best_action = evaluation.action;
      }
   }

   action_events.send(best_action);
}
```

### Advanced: custom strategies for multiple worlds

You can modify the default strategy described above by setting up your own custom runner, which can be used to control the execution of schedules in arbitrary ways.

Note that schedules are initialized on particular worlds, and will not run on a world with a mismatched `WorldLabel`.

## Implementation strategy

`App`'s new struct under this proposal would be:

```rust
pub struct App {
 // WorldId is not very readable; we can steal labelling tech from systems
 pub worlds: HashMap<WorldLabel, World>,
  // Each Schedule must have a `World`, but not all worlds need schedules
 pub schedules: HashMap<WorldLabel, Schedule>,
 // Controls where app
 // This is important to preserve the current ergonomics in the simple case
 // Defaults to `CoreWorld::Main`
 pub current_world: Box<dyn WorldLabel>,
  // This is needed in order to correctly hand out read-only access
 pub global_world: Box<dyn WorldLabel>,
 // Which world the entities to be used for rendering and sound come from
 // The relevant components will be automatically copied and moved 
 // to the appropriate worlds in a pipelined fashion
 // Defaults to `CoreWorld::Main`
 pub source_world: Box<dyn WorldLabel>,
 // Exactly the same as before
 pub runner: Box<dyn Fn(App) + 'static, Global>,
}
```

The standard main loop would have the following basic structure:

1. Create a `Global<ComputeTaskPool>`, which all worlds can access when they need threads to run their systems.
2. For each `WorldLabel` in `app.world.keys()`, initialize the corresponding schedule with the correct label.
3. For each world, run the corresponding schedule if it exists.
   1. Each world is fully independent, and each scheduler can request more threads as needed in a greedy fashion.
4. Run the global schedule.
5. At the end of each loop, put the entire `App` back together and run any `AppCommands`.
   1. Be sure to apply standard `Commands` to each world.
6. Go to 2.

This should be programmed using the default (and minimal) runners.
Custom schedule and synchronization behavior can be added with a custom runner.

### Cloneable schedules

Schedules must be cloneable in order to reasonably spin up new worlds.

When schedules are cloned, they must become uninitialized again.

### AppCommands

`AppCommands` directly parallel `Commands` API and design where possible, likely abstracting out a trait.

They operate on `&mut App`.

The initial set of `AppCommands` needed for this RFC are:

- `spawn_world(label: impl WorldLabel)`
- `despawn_world(label: impl WorldLabel)`
- `set_world(label: impl WorldLabel)`
  - sets `App::current_world`
- `set_global_world`
- `set_source_world`
- `clone_world(old_label: impl WorldLabel, new_label: impl WorldLabel)`
- `new_schedule(label: impl WorldLabel, schedule: Schedule)`
  - if systems are added to a world without an existing schedule, make one
- `clone_schedule(old_label: impl WorldLabel, new_label: impl WorldLabel)`
- `reassign_schedule(old_label: impl WorldLabel, new_label: impl WorldLabel)`
- `remove_schedule(label: impl WorldLabel)`
- `move_entity(entities: Entity, origin_world: impl WorldLabel, destination_world: impl WorldLabel)`
- `move_entities(entities: HashSet<Entity>, origin_world: impl WorldLabel, destination_world: impl WorldLabel)`
- `move_query::<Q: WorldQuery, F: WorldQuery + FilterFetch>(origin_world: impl WorldLabel, destination_world: impl WorldLabel)`
  - moves every matching entity to the destination world
- `move_resource::<R: Resource>(origin_world: impl WorldLabel, destination_world: impl WorldLabel)`
- `send_events::<E>(event: E, destination_world: impl WorldLabel)`
- `send_events::<E>(events: Events<E>, destination_world: impl WorldLabel)`
- `send_commands(commands: Commands, destination_world: impl WorldLabel)`
- `insert_global(res: impl Resource)`
  - Also used for mutating global resources
- `init_global::<R: Resource + FromWorld>()`
- `remove_global::<R: Resource>()`

All of these methods should have an equivalent on `App`, which should be the canonical form called by these commands.

Schedule modifying commands are a natural extension, but are well outside of the scope of this RFC.

### Global resources

In order to avoid contention issues and scheduler entanglement, global resources are read-only to ordinary worlds.
Use `AppCommands::insert_global_resource` to modify them.
They are accessible via `GlobalRes<MyResource>` system parameters in standard systems.

Under the hood, each other world gets a `&` to these.
This simplifies system parameter dispatch, and allows them to be accessed in exclusive systems using `World::get_global_resource::<R: Resource>`.

Global resources are stored in their own world, with a corresponding schedule and the `CoreWorld::Global` label (the default world gets `CoreWorld::Main`).
The default runner special-cases the execution order of our worlds, in order to safely hand out non-blocking read-only access to this world.

### WorldLabel

Steal implementation from system and stage labels.
This should probably be abstracted out into its own macro at this point.

Remove `WorldId` in favor of an internal `WorldLabel`, which is exposed as a system parameter so systems can tell which world they're running on.

## Drawbacks

- added complexity to the core design of `App`
- this feature will be ignored by many games with less complex data flows
- makes the need for plugin configurability even more severe
- sensible API is blocked on [system builder syntax](https://github.com/bevyengine/bevy/pull/2736)
  - until then, we *could* make methods for `add_system_set_to_stage_to_world`...
- `set_world` makes the `App`'s [order-dependence](https://github.com/bevyengine/bevy/issues/1255) stronger
  - this is mitigated by the standard plugin architecture
  - the API created by this is very easy to read and reason about
  - this surface-level API should be easy to redesign if we decide to coherently tackle that problem
- heavier use of custom runners increases the need for better reusability of the `DefaultPlugins` `winit` runner; this is a nuisance to hack on for the end user and custom runners for "classical games" are currently painful

## Rationale and alternatives

### Why not sub-worlds?

Ownership semantics are substantially harder to code and reason about in a hierarchical structure than in a flat one.

They are significantly less elegant for several use cases (e.g. scientific simulation) where no clear hierarchy of worlds exists.
In cases where this hierarchy does exist, the presence of a global world and schedule should handle the use cases well.

Things will also get [very weird](https://gatherer.wizards.com/pages/card/details.aspx?name=Shahrazad) if your sub-worlds are creating sub-worlds (as is plausible in an AI design), compared to a flat structure.

Finally, the ergonomics, discoverability and potential for optimization of a custom-built API for this very expressive feature is much better than `Res<World>`-based designs.

### Why are schedules associated with worlds?

Each system must be initialized from the world it belongs to, in a reasonably expensive fashion.
We should cache this, which means persistently associating the schedules which store our systems with worlds.

### Why not store a unified `Vec<(World, Schedule)>`?

This makes our central, very visible API substantially less clear for beginners, and is a pain to work with.
By moving to a label system, we can promote explicit, extensible designs with type-safe, human readable code.

In addition, the tightly-coupled design makes the following relatively common tasks much more frustrating:

- storing schedules that are not associated with any meaningful world, to later be cloned
- storing data in worlds that have no meaningful associated schedule, to be operated on using `AppCommands`
- swapping schedules between worlds

### Why can't we have multiple schedules per world?

This makes the data access / storage model slightly more complex.
More importantly, it raises the question of "how do we run multiple schedules on the world".
We could invent an entire DSL to describe all of the different strategies one might use, or force users to write custom runners that are schedule-aware.

But, we already have that! This is the entire reasoning behind the ability to nest and compose schedules.
Those tools may be somewhat lacking (e.g. the ability to properly loop schedules or elegantly describe turn-based games),
but that problem should be addressed properly, not papered over.

## Why is this design at the `bevy_app` level, not `bevy_ecs`?

Fundamentally, we need to store schedules and worlds together, and have a way to execute logic on them in a sensible, controllable way.
`bevy_ecs` has no such abstractions for this, in favor of allowing end users to roll their own wrapping data structures as desired.

For the most part, this is trivial to replicate externally: `App` is still very simple, the hard part is in getting the design right.
`AppCommands` are the only technically-challenging bit, but those are easy to replicate in a fixed fashion using an external control flow in the event that they are needed, or they can be stolen from `bevy_app` as needed.

## Prior art

[#16](https://github.com/bevyengine/rfcs/pull/16) covers a related but mutually incompatible sub-world proposal.

[Unity](https://docs.unity3d.com/Packages/com.unity.entities@0.0/manual/ecs_in_detail.html#world) has multiple worlds:

- possible uses cases are discussed [here](https://forum.unity.com/threads/what-should-be-the-motivation-for-creating-separate-ecs-worlds.526277/)
- docs are thin
- usability seems very poor

## Unresolved questions

1. Can we get `App`-level storage of `AppCommandQueue` working properly? Very much the same problem as [bevy #3096](https://github.com/bevyengine/bevy/issues/3096) experienced with `Commands`.
2. Is the ability to clone entities between worlds critical / essential enough to solve [bevy #1515](https://github.com/bevyengine/bevy/issues/1515) as part of this feature (or before attempting it)?
3. Should `AppCommands` take a `&mut App`, or something better scoped to reduce borrow-check complexity?
4. How do we ensure that we can move entities and resources from one `World` to another outside of `bevy_app` without cloning?
   1. This is important to ensure that stand-alone `bevy_ecs` users can use this feature set smoothly.
5. How can we improve the end user experience when working with custom runners?
   1. This problem already exists: modifying the `winit` strategy is very painful.
   2. Split between "schedule logic" and "interfacing logic"?
6. When writing runners, how precisely do we specify world sync points?
7. How do we preserve the validity of cross-entity references (relations) when transferring resources and components between worlds?

## Future possibilities

1. Dedicated threads per world.
   1. Could be very useful for both audio and input.
   2. Needs profiling and design.
   3. May break on web.
   4. May behave poorly with `NonSend` resources in complex ways.
2. Built-in support for some of the more common use cases.
   1. Feature needs time to bake first, and immediate uses are appealing to end users.
   2. Entity purgatories would be interesting to have at a component level.
3. `NonSendGlobal` resources.
4. Ultra-exclusive systems could be added to allow for a system-style API to perform work across worlds or directly modify schedules.
   1. Some trickery would be required to avoid self-referential schedule modification.
5. More fair, responsive or customizable task pool strategies to better predict and balance work between worlds.
6. Use of app commands to modify schedules, as explored in [bevy #2507](https://github.com/bevyengine/bevy/pull/2507).
   1. We could *maybe* even pause the app, and then modify the runner.
7. Fallible `AppCommands` using the technology in [bevy #2241](https://github.com/bevyengine/bevy/pull/2241).
8. Worlds as a staging ground for scenes in a prefab workflow.

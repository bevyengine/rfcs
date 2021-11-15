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
7. Roll-back networking.

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

By default, two worlds are added to your app: `CoreWorld::Global` and `CoreWorld::Main`.
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

### Example: entity prefab staging

In this example, we're storing ready-made entities in our global world, and rapidly pulling them into our default world as needed.
This allows us to store complex, procedurally-generated entities in a reusable fashion,
without worrying that they will be accidentally broken by the game's logic.

```rust
fn main(){
   App::new()
   .add_plugins(DefaultPlugins)
   // Events to spawn prefabs are generated by systems in here,
   // and sent across worlds using AppCommands
   .add_plugin(GamePlugin)
   // This plugin will contain a number of startup systems
   // which generate entities in complex ways
   .add_plugin(PrefabInitializationPlugin.to_world(CoreWorld::Global))
   // This system will read prefab-spawning events sent to it from the main world
   .add_system(spawn_prefabs_system.exclusive_system().to_world(CoreWorld::Global))
   .run();
}

struct SpawnPrefabEvent {
   // You can store whatever information you'd like in these events
   // in order to make looking up the right prefab easy
   identifier: Identifier
}

// An example system which uses the prefab system
fn spawn_marine(mut app_commands: AppCommands){
   let spawn_events = Events::<SpawnPrefabEvent>::new();

   app_commands.send_events(spawn_events, CoreWorld::Global);
}

// This system allows us to read main-world SpawnPrefabEvents
// and send over the appropriate entities
fn spawn_prefabs_system(global_world: &mut World){
   // These were sent from the main world
   let spawn_events = global_world.get_resource::<Events<SpawnPrefabEvent>>();
   // This handmade resource allows users to quickly look up entities by their identifier
   let prefab_index = global_world.get_resource::<PrefabIndex>();

   // This convenience method is blocked by bevy #3096
   // but that problem is bascially orthogonal to this RFC
   let app_command_queue = world.app_commands();

   // Please pretend borrow-check is happy for the sake of discussion
   // Fighting with it without a compiler is no fun :(
   for spawn_event in spawn_events.iter(){
      let prefab_entity = prefab_index.get(event.identifier);
      // Blocked by bevy #1515 but specific to this use case
      let cloned_entity = global_world.clone_entity(prefab_entity).id();
      // Sends a copy of the prefab to the main world as requested
      app_commands.move_entity(cloned_entity, CoreWorld::Global, CoreWorld::Main);
   }
}
```

This particular set-up will result in a one frame delay for spawning prefabs.
On the first frame, the spawn event is sent from the main world to the global world.
On the second frame, the entity is cloned and sent back.

If this is a deal-breaker, you could write a custom runner that has an additional sync point each frame.

### Advanced example: world-model AI

In this example, we're using a multiple worlds approach in order to allow complex agents to simulate the world in order to plan ahead.
By taking this strategy, we can ensure that the agents always have an accurate model of the rules of the game without code duplication.

```rust
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
   .run();
}

// This system is contained within `GameLogicPlugin`
fn simulate_action(action: AiActions, app_commands: AppCommands){
   app_commands
   .spawn_world(SimulationWorld::new())

   // FIXME: figure out the crazy structure needed to make this actually work,
   // or swap to a simpler example
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
 // This is important to preserve the current ergonomics in the simple case
 // This is initialized to `CoreWorld::Main`
 pub current_world: WorldLabel,
  // This is needed in order to correctly hand out read-only access
 pub global_world: WorldLabel,
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

Remove `WorldId`.

## Drawbacks

- added complexity to the core design of `App`
- this feature will be ignored by many games with less complex data flows
- makes the need for plugin configurability even more severe
- sensible API is blocked on [system builder syntax](https://github.com/bevyengine/bevy/pull/2736)
  - until then, we *could* make methods for `add_system_set_to_stage_to_world`...
- heavier use of custom runners increases the need for better reusability of the `DefaultPlugins` `winit` runner; this is a nuisance to hack on for the end user and custom runners for "classical games" are currently painful

## Rationale and alternatives

### Why not sub-worlds?

Ownership semantics are substantially harder to code and reason about in a hierarchical structure than in a flat one.

They are significantly less elegant for several use cases (e.g. scientific simulation) where no clear hierarchy of worlds exists.

Things will also get [very weird](https://gatherer.wizards.com/pages/card/details.aspx?name=Shahrazad) if your sub-worlds are creating sub-weorlds (as is plausible in an AI design), compared to a flat structure.

### Why are schedules associated with worlds?

Each system must be initialized from the world it belongs to, in a reasonably expensive fashion.
Free-floating schedules,

### Why not store a unified `Vec<(World, Schedule)>`?

This makes our central, very visible API substantially less clear for beginners, and is a pain to work with.
By moving to a label system, we can promote explicit, extensible designs with type-safe, human readable code.

In addition, this design prevents us from storing schedules which are not associated with any current world, which is annoying for some architectures which aggressively spawn and despawn worlds.

### Why can't we have multiple schedules per world?

This makes the data access / storage model slightly more complex.
No pressing use cases have emerged for this, especially since schedules can be nested.
If one arises, swapping to a multimap should be easy.

The ability to clone schedules should give us almost everything you might want to do with this.

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

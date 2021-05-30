# Feature Name: `ui-systems-callbacks`

## Summary

App logic is dramatically clearer when it flows from input to actions to reactions.
This is handled quite simply (from an end user perspective) with a unified `Action` event type on UI elements and automatic input dispatching.
To execute logic triggered by interacting with the UI, use ordinary systems in concert with a general purpose event handling paradigm to store custom behavior on individual UI entities as components that tie into well-defined events in a principled way.

## Motivation

Bevy's ECS is a powerful and expressive tool for arbitrary computation and scheduling.
However, it tends to struggle badly when one-off behaviors are needed, resulting in both excessive boilerplate and a huge proliferation of systems that must run each and every frame.

This is particularly relevant when it comes to designing the logic behind user interfaces, which are littered with special-cased, complex functions with very low performance demands.

## User-facing explanation

When building user interfaces in Bevy, data flows through three conceptual stages:

1. **Input.** Raw keyboard, mouse, joystick events and so on are received. These are handled by various **input dispatching** systems, and converted into actions, with tangible game-specific meaning.
2. **Action.** Entities in our world (whether they're game objects or interactive **UI elements**) receive actions and change data.
3. **Reaction.** Other systems watch for changes or events produced by the UI elements that changed, and react to finish the tasks they started.

In this chapter, we're going to discuss patterns you can use in Bevy to design user interfaces that map cleanly to this data flow, allowing them to be modified, built upon and debugged without the spaghetti.

Once we've added an interactive UI element to our `World` (think a button, form or mini-map), we need to get actions to it in some form.
The simplest way to do this would be to listen to the input events yourself, and then act if an appropriate event is heard.

```rust
// This system is added to CoreStage::Ui to ensure that it runs at the appropriate time
fn my_button(mut query: Query<(&Interaction, &mut Counter), 
                     (With<MyButton>, Changed<Interaction>)>){
 // Extract the components on the button in question
 // and see if it was clicked in the last frame
 let (interaction, mut counter) = query.single_mut().unwrap();
 if *interaction == Interaction::Clicked {
  *counter += 1;
 }
}
```

However, this conflation of inputs and actions starts to get tricky if we want to add a keybinding that performs the same behavior.
Do we duplicate the logic? Mock the mouse input event to the correct button?
Instead, the better approach is to separate inputs from actions, and use the built-in **event-queue** that comes with our `ButtonBundle`.

```rust
// Each of our buttons have an `Events<Action>` component,
// which stores a data-less event recording that it has been clicked.
// Adding your own `Events<T>` components with special data to UI elements is easy;
// simply add it to your custom bundle on spawn.

// Action events are automatically added to entities 
// with an Interaction component when they are clicked on
// Here, we're adding a second route to the same end, triggering when "K" is pressed
fn my_button_hotkey(mut query: Query<&mut EventWriter<Actions>, With<MyButton>>, keyboard_input: Res<KeyboardInput>){
 if keyboard_input.just_pressed(KeyCode::K){
  let button_action_writer = query.single_mut().unwrap();
  // Sends a single, dataless event to our MyButton entity
  button_action_writer.send(Action);
 }
}

// We can use the EventReader<Action> sugar to ergonomically read the events stored on our buttons
fn my_button(mut query: Query<(&mut EventReader<Action>, &mut Counter), With<MyButton>>){
 // Extract the components on the button in question
 // and see if it was clicked in the last frame
 let (actions, mut counter) = query.single_mut().unwrap();
 for _ in actions {
  *counter += 1;
 }
}
```

As you can see, decoupling inputs and actions in this way makes our code more robust (since it can handle multiple inputs in a single frame), and dramatically more extensible, without adding any extra boilerplate.
If we wanted to, we could add another system that read the same `Events<Action>` component on our `MyButton` entity, reading these events completely independently to perform new behavior each time either the button was clicked or "K" was pressed.

Finally, we can use this decoupling to ensure that only valid inputs get turned into actions, by making sure that our systems runs after the `bevy::input::SystemLabels::InputDispatch` system label during the `CoreStage::PreUpdate` stage.
Input is converted to actions during systems with those labels, so we can intercept it before it is seen by any systems in our `Update` or `Input` stages.

```rust
use bevy::prelude::*;
use bevy::input::SystemLabels;

fn main(){
 App::build()
  .add_system_to_stage(CoreStage::PreUpdate, 
   verify_cooldowns.system().after(SystemLabels::InputDispatch))
  .run();
}

/// Ignores all inputs to Cooldown-containing entities that are not ready
fn verify_cooldowns(mut query: Query<&Cooldown, &mut Events<Action>>){
 for cooldown, mut actions in query.iter_mut(){
  if !cooldown.finished{
   actions.clear();
  }
 }
}
```

### Generalizing behavior

Of course, we don't *really* want to make a separate system and marker component for every single button that we create.
Not only does this result in heavy code duplication, it also imposes a small (but non-zero) overhead for each system in our schedule every tick.
Furthermore, if the number of buttons isn't known at compile time, we *can't* just make more systems.

Commonly though, we will have many related UI elements: unit building an RTS, ability buttons in a MOBA, options in a drop-down menu.
These will have similar, but not identical behavior, allowing us to move data from the *components* of the entity into an *event*, commonly of the exact same type.
In this way, we can use a single system to differentiate behavior.

```rust
// Operates over any ability buttons we may have in a single system
fn ability_buttons(mut query: Query<(&mut EventReader<Action>, &mut Timer<Cooldown>, &Ability)>, time: Res<Time>, mut ability_events: EventWriter<Ability>){
 for actions, cooldown, ability in query.iter_mut(){
  // Tick down our cooldown timers
  cooldown.tick(time.delta());
  
  for _ in actions {
   // You can only use abilities that are off cooldown!
   if cooldown.finished(){
    // Creates a global ability event using the data for other systems to handle
    // This should include the originating `Entity` as a field of `Ability`, 
    // along with other data about its effects
    ability_events.send(ability);
   }
  }
 }
}
```

We can extend this pattern further, with the use of **generic systems**, allowing us to quickly create systems for new types of interactable UI elements.

```rust
use bevy::prelude::*;

fn main(){
 App::build()
  .add_event::<NewPuzzle>()
  .add_system_to_stage(CoreStage::Input, puzzle_button::<NewPuzzle>.system())
  .add_event::<ResetPuzzle>()
  .add_system_to_stage(CoreStage::Input, puzzle_button::<ResetPuzzle>.system())  
  .add_event::<WriteNumber>()
  .add_system_to_stage(CoreStage::Input, puzzle_button::<WriteNumber>.system())
  .run()
}

// Dataless structs that double as components and events
struct NewPuzzle;
struct ResetPuzzle;
// Also passes the correct number stored in the button into the puzzle game's logic
struct WriteNumber(u8);

/// Sends the event type associated with the button when pressed
/// using the data stored on the component of that type
fn puzzle_button<T: Component + Clone>(
    query: Query<(&Interaction, &T)>,
    mut event_writer: EventWriter<T>,
) {
    for (interaction, marker) in query.iter() {
        if *interaction == Interaction::Clicked {
            event_writer.send(marker.clone());
        }
    }
}
```

### Specializing behavior

In certain cases though, you may need to have a very large number of UI elements, each with entirely custom behavior.
Rather than running one (or more!) system per button, you can use a more advanced **event handler** pattern;
describing *what* each entity should do each time a specific event occurs.

There are four intuitive steps to doing so:

1. Create a custom events type `T` that stores all the data you need.
2. Register this type in your app using `AppBuilder::add_event::<T>()`. This has three effects:
   1. Registers the `Events<T>` resource in the app in its default (empty) state.
   2. Adds an `cleanup_events::<T>()` system to `CoreStage::PostUpdate`, which causes the events to automatically be cleaned up after two frames using a double-buffer system.
   3. Adds an `handle_events::<T>()` system to `CoreStage::Update`, which automatically calls the function stored in `EventHandler<T>` once for each event received in the corresponding storage (be it a global resource or a component of a particular entity).
3. Create a way for your events to be generated.
   1. For most UI applications, you'll simply be using the standard `Events<Action>` component on every interactive UI element.
   2. This component is automatically filled with events when buttons are pressed, or other user interactions occur.
4. Define the behavior that should occur when the event occurs, and add it as a `EventHandler<T>` struct, added as either a resource or a component.

Steps 1 through 3 should be completely familiar to you from working with other types of events, so lets focus on the last step.
By examining the definition of `EventHandler` struct, we can see that it stores a function which consumes an event and mutates the world:

```rust
// These trait bounds are required as events could be stored as either a component or resource
struct EventHandler<T: Component + Resource> {
 // Pass in particular data that you want to use to customize the effects in the event T
 func: Fn(&T, &mut World),
}
```

When the `handle_events::<T>` system detects an appropriate event, it runs that function at the end of the current stage using the `handle_event::<T>)` command.
Let's take a look at exactly how that system works:

```rust
/// Automatically handles events of type `T` according to the logic in the corresponding `EventHandler`
fn <T: Component + Resource> handle_events::<T>(mut commands: Commands,
mut query: Query<(&mut EventReader<T>, &EventHandler<T>)>, 
  event_res: EventReader<Option<T>>, event_handler_res: EventHandler<Option<T>>){
  // Only handle resource events if both the global resource and global handler exist 
  if let mut Some(events) = event_res && let Some(handler) = event_handler_res {
    for event in events{
      commands.handle_events::<T>(event, handler.func);
    }
  }

  // Repeat the same logic with components
  for (mut events, handler) in query.iter_mut() {
    for event in events {
      commands.handle_events::<T>(event, handler.func);
    }
  }
}
```

When the `handle_event` command is applied, the function you've stored in your handler is called, using the event you passed in along with the `World` as inputs, altering the world as it pleases.

Working directly with access to the world may seem intimidating, but it's not too hard with a bit of practice and most of the API mirrors that of the familiar `Commands`.
When writing these functions (or closures!), just be mindful of the required type signature: it must take in `&T` (your underlying event type) and `&mut World` (all of the data in our app) as function arguments and have no return value.

Let's lay out what a few simple bits of functionality would look like.

Once you've defined the desired functionality, adding event handling to your UI elements is straightforward: simply add an `EventHandler<Action>` struct as a component, storing the desired behavior in a function (including a closure).

```rust
fn despawn_self(action: &Action, world: &mut World) {
  world.despawn(action.entity);
}

// The _action variable is never used, so it is prefixed with an _ by convention
fn set_difficulty_hard(_action: &Action, world: &mut World) {
  let mut difficulty = world.get_resources_mut::<Difficulty>().unwrap();
  *difficulty = Difficulty::Hard;
}

fn toggle_pvp(action: &Action, world: &mut World){
  let mut player_entity = world.get_entity_mut(actions.entity).expect("This entity should still exist!");
  if player_entity.contains::<InCombat>(){
    player_entity.remove::<InCombat>();
  } else {
    player_entity.insert(InCombat);
  }
}

fn spawn_slime(action: &Action, world: &mut World){
  // You can simply use .query instead if you have no filters
  let player_position = world.query_filtered<&Position, With<Player>>().single().unwrap();
  world.spawn().insert_bundle(BigSlimeBundle::new(player_position));
}

fn supercharge_towers(action: &Action, world: &mut World){
  let button_cooldown = world.entity_mut(action.entity).get_mut::<Cooldown>().unwrap();
  button_cooldown.reset();

  let mut tower_query = world.query_filtered<(&mut AttackSpeed, &mut Damage), With<Tower>>();
  for (mut attack_speed, mut damage) in tower_query.iter_mut(){
    *attack_speed *= 2.0;
    *damage *= 3;
  }
}
```

Let's pull this together into a single complete but minimal example to demonstrate the lifecycle.

```rust
use bevy::prelude::*;

fn main(){
  App::build()
    .add_plugins(DefaultPlugins)
    // This is autaomtically done as part of DefaultPlugins
    // .add_event::<Action>()
    .add_startup_system(create_growing_button.system())
    .run();
}

fn grow_on_action((action: &Action, world: &mut World){
  let mut transform = world.entity_mut(action.entity).get_mut::<Tranform>().unwrap();
  transform.scale *= 1.1; 
}

fn create_growing_button(mut commands: Commands){
  // Create an ordinary button
  commands.spawn_bundle(ButtonBundle::default())
  // Add our custom event handler to it as a component
  .insert(EventHandler::<Action>::new(grow_on_action));
}
```

As you can see, this is a simple but expressive way to create custom effects for your interactive UI elements without adding additional systems.
Be wary though: commands (like those used by event handlers) are generally less performant due to their inability to be parallelized,
and only take effect at the end of the current stage.

### Reacting to UI

Once your UI has responded to actions, you may want to respond to its consequences in a downstream fashion.
There are three good tools to do so:

1. **Change detection:** Using `Changed<T>` query filters (or `.is_changed()` for resources), respond to changes in the data.
2. **Global events:** Emit and then listen for an event as a resource using `EventWriter` and `EventReader` as system parameters.
3. **Entity-specific events:** Emit and then listen for an event as a component using `EventWriter` and `EventReader` as query parameters.

Change detection should be your default tool for simple cases: it is fast, perfectly reliable and does not involve the creation of any new types.
Simply read the new value of the data and respond accordingly.

Events are useful when you need to store more than one possible event, or want to encode additional data.
Use entity-specific events when the effects of your action are well-localized to a single entity, and global events otherwise.

## Implementation strategy

This proposal's functionality depends on:

1. Per-entity events: [PR](https://github.com/bevyengine/bevy/pull/2116), [perf improvements](https://github.com/bevyengine/bevy/pull/2073). This is essential to achieving nice ergonomics around input mapping and action responses.
2. A standardized label for input dispatch that's used by core and community plugins, and a new `CoreStage::Input`. These are essential to help reduce system ordering headaches.
3. A simple `handle_events` command, that works as outlined above.
4. \[Optional\] Automatic, procedural labels of generic systems for `cleanup_events` and `handle_events`, allowing you to specify which order user-defined systems run in relative to these important triggers.
5. \[Optional\] The ability to configure before / after system ordering via their labels, allowing you to specify the relative order of event-handling systems.

5 and 6 are optional because they can be bypassed by manually adding the appropriate systems.
That's not a *great* solution though, as it's highly non-obvious, fragile, and heavy on boilerplate.

## Drawbacks

1. EventHandlers have arbitrary power. If not carefully managed, this could create terrible spaghetti.
2. Handling events in this way, like other commands, operate sequentially. This is problematic for high performance applications.
3. Commands do not take effect until the stage boundary.
4. Serialization of components that store functions is likely to be challenging.
5. The ability to further constrain ordering of systems by their labels increases user control over the internal details of plugins, although it should be impossible to create internal breaks in this way due to the additive nature.

## Rationale and alternatives

### Why is it helpful to use the ECS for our UI logic?

1. Familiar to Bevy programmers.
2. Trivially integrated with game logic.
3. Incredibly expressive.
4. Benefits from other engine improvements and reduces maintenance burden.

### Why do we need to add this as an engine feature?

While very little code will need to be added *specifically* for this pattern,
it is vitally important to demonstrate complex patterns in an opinionated way to users.

This allows for the creation of a standardized, interoperable ecosystem,
and guides users towards a sensible, performant and maintainable set of patterns when building their own user interfaces.

### Why do we want event handling?

UI, much like scripting, tends to involve a large number of special-cased behaviors, in direct opposition to the natural patterns promoted by the ECS architecture.
The event handling pattern allows us to express this logic in a sane and maintainable fashion.

Theoretically, everything that users could do with this pattern could be done with custom-built systems.
However, this clutters our scheduler (possibly hurting performance), reduces clarity and hurts compile times due to a huge number of one-off types.
Furthermore, by storing specialized logic as data directly on UI entities it becomes dramatically easier to debug and reason about customized behavior as there is one obvious place to look, rather than checking each component's corresponding systems.

### Why don't we need a more complex reactivity model?

Most reaction chains in UI are shockingly short.
The most complex patterns are:

1. Changing UI appearance en-masse: styles and themes should be used for this.
2. Changing which UI elements are displayed (e.g. swapping tabs, pulling up a menu): this is precisely what `on_enter` and `on_exit` system sets in `States` are intended to solve.
3. Handling layout changes: this should be done automatically in a single system (or group of systems) that runs in `CoreStage::PostUpdate`, rather than being scatter across our logic.

For everything else, change detection and events should be more than adequate.

By sticking to a straightforward imperative model of UI behavior combined with decoupled reactivity,
we can make complex user interfaces dramatically easier to reason about, refactor and debug.

## Unresolved questions

- How do we serialize and deserialize component that store a function, like `EventHandler<T>`?
- How can we ergonomically control the order in which event handling commands are executed for entities triggering on the same type of event?

## Future work

1. We may want to loop over UI in some way to ensure that everything is resolved properly.
2. The ergonomics and performance of the event handling pattern will be improved with other possible improvements to commands, namely more immediate processing, parallel execution and better control over execution order.
3. Accessing and modifying the behavior of other entities within a UI hierarchy *can* be done as is, but will be much more ergonomic with advanced relations features like the ability to query for data on the target entity.
4. The ergonomics and configurability of event registration should be improved, especially if we're tying an additional system to it and making more aggressive use of events.
As a start, events could be automatically registered (see [discussion on making Component opt-in](https://github.com/bevyengine/bevy/issues/1843#issuecomment-850767526)) and the strategy for clearing events could be varied in transparent ways.

### General application of event handling

The event handler pattern, once introduced to UI, immediately lends itself to other highly customized, scripting-like applications within the game's logic.

One might create events for `Hit`, `Death` and `Poisoned` to create extremely customizable behavior in an RPG (without relying on a new system for every single enemy type and mechanic).
Or you could use it for controlling puzzle logic in an action game, or tracking achievements in a shooter.

Cases like this last one make it particularly relevant

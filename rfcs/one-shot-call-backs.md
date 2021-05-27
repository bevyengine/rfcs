# Feature Name: `one-shot-callbacks`

## Summary

When combined with ordinary systems, one-shot systems stored as components on UI entities offer a powerful and expressive callback-like paradigm for one-off UI behavior.
In order to reduce the importance of system ordering and avoid data integrity bugs, these systems run within a dedicated looping substage.

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
fn ability_buttons(mut query: Query<(&mut EventReader<Action>, &mut Timer<Cooldown>, &Ability), time: Res<Time>, mut ability_events: EventWriter<Ability>){
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
Rather than running one (or more!) system per button, you can use a more advanced **callback pattern** using `Callback` components which store a command.
This command is applied once for each time an `Action` event is received by your entity, taking effect at the end of the current stage.

Under the hood, these are processed by the `callback_system` function found in `CoreStage::Ui`. The `Callback` type and this built-in system are *remarkably* simple:

```rust
/// `Callback` components are automatically run as commands when 
enum Callback {
	/// Commands that affecting the global state broadly
	Command(Commands),
	/// Commands that affect a single entity, stored in this enum
	EntityCommand(Entity, EntityCommands),
	/// Commands that affect only the entity that has this component
	SelfCommand(EntityCommands),
}

/// Applies the `Callback` component of entities once for each `Action` event that they have
fn callback_system(mut commands: Commands, mut query: Query<(Entity, &mut EventReader<Action>, &Callback)>){
	for (self_e, mut actions, callback) in query.iter_mut(){
		// For each Action (triggered by inputs) that our entity receives
		for _ in actions {
			// Run the command referenced in the `Callback` component of our entity
			// at the end of the current stage
			match Callback {
				Callback::Command(command) => commands.apply(command),
				EntityCommand::EntityCommand(e, e_command) => commands.entity(e).apply(e_command),
				Callback::SelfCommand(e_command) => commands.entity(self_e).apply(e_command),
			}
		}
	}
}
```

Adding callbacks to your UI elements is straightforward: simply add the appropriate `Callback` struct as a component.

```rust
// This button will spawn a new unit when pressed
commands.spawn_bundle(ButtonBundle::default())
	.with(Callback::Command(Commands::new().spawn_bundle(UnitBundle::default())));

// This button will overwrite the value of the `GameDifficulty` resource when pressed
commands.spawn_bundle(ButtonBundle::default())
	.with(Callback::Command(Commands::new().insert_resource(GameDifficulty::Hard));

// This button will add the `InCombat` marker component to the `player_entity` Entity when pressed
commands.spawn_bundle(ButtonBundle::default())
	.with(Callback::EntityCommand(player_entity, EntityCommands::new().insert(InCombat)));

// This button will despawn itself when pressed
commands.spawn_bundle(ButtonBundle::d, efault())
	.with(Callback::SelfCommand(EntityCommands::new().despawn()));
```

Remember that you can create your own **custom commands**, giving you the ability to express arbitrary logic using callbacks.
For moderately complex, one-off cases though, you may prefer to combine `Callback` with the `run_system` command to quickly execute systems of your own design in a one-shot fashion when UI elements are activated.

```rust
/// Powers up all of our towers when this system runs
fn supercharge_towers(mut query: Query<(&mut Damage, &mut AttackSpeed), With<Tower>>){
	for mut damage, mut attack_speed in query.iter_mut(){
		*damage *= 2.0;
		*attack_speed *= 2.0; 
	}
}

/// Runs the supercharge_towers system once when this button is activated
commands.spawn_bundle(ButtonBundle::default())
	.with(Callback::Command(Commands::new().run_system(supercharge_towers.system())));
```

### Reacting to UI



TODO: discuss change detection, global events, and entity-specific events

## Implementation strategy

This proposal's functionality depends on:

1. Per-entity events: [PR](https://github.com/bevyengine/bevy/pull/2116), [perf improvements](https://github.com/bevyengine/bevy/pull/2073)
2. Implementing a command chaining API.
3. A standardized label for input dispatch that's used by core and community plugins.
4. \[Optional\] The ability to run one-shot systems with commands: [issue](https://github.com/bevyengine/bevy/issues/2192), [PR](https://github.com/bevyengine/bevy/pull/2234).

In addition, we should aim to solve the **UI loop problem** as part of this proposal.

### Command chaining

TODO: explain exactly how this API should work.

### UI loop

TODO: discuss UI loop problem, and propose substage solution.

## Drawbacks

1. Callbacks have arbitrary power. If not carefully managed, this could create terrible spaghetti.
2. Callbacks, like other commands, operate sequentially. This is problematic for high performance applications.
3. There are several equivalent ways to achieve the same outcome. This choice is mostly dictated by ergonomics and subtle (but typically irrelevant) perf considerations.
4. Serialization of callback components is likely to be challenging.
5. A looping UI-specific substage will reduce parallelism, and force opinions on end-user code.
6. Looping stages expose weaknesses in the `State` dataflow model: a state's run criteria might have returned `No` by the time the UI is ready to switch.

## Rationale and alternatives

### Dataflow desiderata

1. No special-casing.
2. Flexible.
3. Opinionated.
4. Efficient.

## Unresolved questions

- How do we serialize and deserialize callback components?
- How can we ergonomically control the order in which callback commands are executed?

## Future work

1. The UI loop problem may be better addressed by multiple worlds and schedules, see #16.
2. The `EntityCommand` variant of `Callback` may be better handled using `Relations` in some form in the future.
3. The ergonomics and performance of the callback pattern will be improved with other possible improvements to commands, namely more immediate processing, parallel execution and better control over execution order.

### Scripting-like gameplay code using callbacks

TODO: discuss `CallbackWrapper` trait and its use to make scripting-like hooks with `callback_hook::<C>` system

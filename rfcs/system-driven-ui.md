# Feature Name: `system-driven-ui`

## Summary

UI widgets' behavior is defined entirely by the components they have, and the data within those components.
The logic to execute the UI is handled by Bevy systems, operating in parallel over the relevant UI widgets.

## Motivation

In larger teams, UI is commonly built with visual tools, rather than from code, which enables non-programming team members like artists and UI designers to participate directly.
As a result, an intermediate asset-like representation is needed, allowing large teams to organize assets cleanly rather than scattering them across the code base.

Doing so pushes us towards a data-driven workflow that fits naturally with our ECS, and in particular, scenes.
By controlling the behavior of our widgets with components and systems, we can support this use case while allowing for seamless integration of code-driven and asset-driven UI workflows while allowing for performant and easily extensible widget behavior.

## Guide-level explanation

The **UI data flow** in Bevy has four conceptual stages:

1. **Dispatching** raw input events to specific widgets.
2. **Handling** raw input events by converting into widget-specific events.
3. **Responding** to widget-specific events with widget logic.
4. **Reading** UI state.

Widgets, as previously discussed in [UI Building Blocks and Styles](https://github.com/bevyengine/rfcs/pull/1) are simply entities with various components on them.
As discussed there, Bevy does not draw a hard-line between "UI" and "not UI"; these concepts and patterns are also useful for handling input to game entities.

### Dispatching input to widgets

Pretty as they may be, user interfaces are ultimately designed to, well, interface with users.
In Bevy, that means they must listen for **input events**, created by your mouse, keyboard, joystick, touch or hand-rolled hum-to-move input pipeline.
But once we receive an input event, what are we to make of it?

For simple prototypes, it makes sense to take a simple approach: just listen to the event stream you care about directly.

```rust
fn jump(query: Query<&mut Velocity, With<Player>>, keyboard_input: Res<Input<KeyCode>>) {
    if keyboard_input.just_pressed(KeyCode::Space) {
        let mut vel = query.single_mut().unwrap();
        vel += Velocity::new(0.0, 1.0);
    }
}
```

This feels natural when building out simple gameplay systems, but if you were so inclined, you could extend this out to classical "widget UI", read the mouse's position and then fire off a button event if the mouse was over a button at the time it was clicked.

While refreshingly simple and perfectly modular, this approach runs into some challenges as you attempt to scale it out to more complex games:

1. Your input logic is scattered across your code-base, making writing a rebinding interface very hard.
2. Supporting multiple input methods (e.g. mouse and hotkeys or gamepad and keyboard) becomes very complex. Do you duplicate these systems? Do you read in two input streams?
3. Handling input events that rely on information from other entities and systems is very frustrating.
   1. Consider overlapping clickable UI widgets; you can't rely on a button's position alone to tell you whether it's correct to execute as that section may be covered.
   2. In game play systems, actors can typically only do one thing at once, or combinations result in different non-additive behavior.
4. Each system listens to the whole input stream of events. When you have 100 different systems, and they each discard 99% of all candidate input events, this creates unnecessary performance drag.

This is where Bevy's **central input dispatch** comes in.

TODO: explain.

### Handling inputs

TODO: write.



### Widget logic

TODO: complete.

Just like when we're using the ECS to control our game's logic, these components both store data and control behavior, by determining which systems apply to each particular `Widget` entity.
Suppose we want to create a radio-button widget.
Rather than trying to create an object with a `RadioButtonWidget` type, like we might in an object-oriented UI framework,
we create an entity with all of the components required to accomplish the desired behavior.
Think of components as behaving in an analogous fashion to traits, with the radio button behavior having trait bounds for each of the required components.

Breaking it down into the constituents, we want our radio button to:

1. Store internal state in an accessible way.
2. Respond to input.
3. Be rendered into several radio buttons.

Turning this into code, we get a `RadioButtons` component and a couple of supporting systems.

```rust
struct RadioButtons {
    // These can't be pub fields, and instead must rely on accessor methods
    // to ensure internal invariants (like only state is one of options) hold
    options: Vec<Text>,
    state: Text,
}

// TODO: complete me
fn handle_radio_button_input(radio_queries: ) {}


// TODO: complete me
fn render_radio_buttons(query: Query<&RadioButton, &mut Sprite, >) {

}

```

### Reading UI state

TODO: refresh.

Widgets can be wired to each other and the game state in two common ways:

1. Directly reading the state of our widget's components.
2. Using events emitted by interacting with the widgets.

Read the component's state when you care about its current state, use event channels when you want to detect events being fired off.

As you move beyond trivial UIs, you may need to disambiguate between multiple widgets of the same type.
To do so, isolate the correct widget's state using marker components.

As your widgets grow further, you may want to scale this pattern dynamically, rather than relying on specific marker components.
To do so, your consuming systems should point to other widgets directly, by storing references to their `Entity` identifier.

Let's take a look at each of these patterns with some examples:

```rust
// In this example, we're wiring up our logic directly to the state of our singleton widget
fn set_background_color(color_selector_query: Query<&ColorSelector>, bg_color: ResMut<ClearColor>) {
    let color = color_selector_query.single().unwrap();
    *bg_color = color;
}

// This is a trivial example of how you might listen to an event emitted by a widget's interactions
fn hello_button(button_events: EventReader<ButtonEvent>) {
    for _ in button_events.iter() {
        info!("Hello!");
    }
}

// Sometimes, the correct behavior may be to simply mirror UI state into a resource
// Then access that for downstream systems
fn dark_mode_toggle(
    query: Query<&Toggle, With<LighDarkWidget>>,
    mut state: ResMut<State<LightDarkMode>>,
) {
    // This example extends the light-dark mode example in PR #1: UI styling
    let toggle = query.single().unwrap();

    state.active() = match toggle {
        true => LightDarkMode::Dark,
        false => LightDarkMode::Light,
    }
}

// Here, we're disambiguating between multiple widgets with a Selector by using a marker component
fn build_unit(
    mut commands: Commands,
    build_events: EventRader<BuildEvent>,
    selector_query: Query<&Selector, With<BuildSelector>>,
) {
    let unit_type = selector_query.single().unwrap().unit_type;
    for event in build_events.iter() {
        commands
            .spawn()
            .insert_bundle(UnitBundle::new(unit_type))
            .insert(event.transform.clone());
    }
}

// In this example, we have many similar UI elements
// and we're looking to control the state of the corresponding UI entity
// Note that we're using a game data -> UI data flow here
fn update_healthbars(
    unit_query: Query<(&CurrentHealth, &MaxHealth, &Transform, &HealthBarEntity), Without<Widget>>,
    mut healthbar_query: Query<(&mut FillingBar, &mut Transform), With<Widget>>,
) {
    for (current, max, transform, hb_entity) in unit_query.iter() {
        let (mut hb_bar, mut hb_transform) = healthbar_query.get(hb_entity).unwrap();
        *hb_bar = health_to_bar(current, max);
        *hb_transform = offset_health_bar(transform);
    }
}
```

When building UI logic that requires chains of events triggering one after another, you should be careful to ensure that upstream systems are processed earlier in the game-loop than their downstream systems.
Failing to do so properly may result in invalid state, input lag or visual flickering.
Standard system ordering tools (e.g. `.before()` and `.after()`) work well for this.

## Reference-level explanation

### Dispatching input to widgets

TODO: complete and clean up. Needs architecture help still :/

In order to unify various input methods, handle overlapping widgets, and avoid duplicating work, we need a central approach to UI dispatching.
UI events have two core fields:

1. What is being interacted with?
   1. Screen-space coordinates.
    Used for mouse, touch and joystick selectors.
    Note that world-space UI elements may be dispatched to in this way as well, requiring recomputation whenever they move or the camera changes.
   2. Selected UI elements.
    Commonly used for keyboard control.
2. What signal is being sent?

During dispatching, we must unify these signals across the various input streams, and convert them into logic that our widgets can understand,
stored as events on a component of the widget itself.
Finally, our widgets can take their new unified event stream and process it independently in their own systems.

```rust
pub struct InteractableMap {
    // Contains some data structure that allows us to efficiently map screen space coordinates to a UI widget
    // This is probably done with a pixel-resolution array
}

impl InteractableMap(){
    pub fn get_from_coordinates(&self, position: &GlobalTransform) -> Entity {
        // Uses this data structure to map between screen space to a UI widget
    }

    // Analagous function for keyboard / gamepad buttons

    // 
}

// The Bounds component doesn't exist yet; it should be a tight polygon around the object in a 2D plane, using screen-space coordinates
// Screen-space bounds would be constant; world-space bounds need to be updated when the UI changes
fn build_interactable_map(
    // PERF: this should be done using change detection and iterative rebuilding from Bounds instead
    world_space_query: Query<(&Bounds), (Without<ScreenSpace>, With<Interactable>)>,
    screen_space_query: Query<(&Bounds), (With<ScreenSpace>, With<Interactable>)>, map: ResMut<InteractableMap>){
    
    // Sort bounds by distance to camera, with further bounds being applied first
    // New bounds overwrite old bounds because they're "on top"

    // Apply the world-space bounds first

    // Screen-space bounds overwrite any world-space bounds
}

// We need one of these systems for each input event type we want to support
// This uses NYI trait queries; other more elegant designs are welcome
fn handle_coordinate_input<E>(query: Query<(&mut impl EventHandler<E>, impl WidgetEvents) With<Interactable>>, input_events: EventReader<E>,
    map: Res<InteractableMap>){

    for input_event in input_events.iter(){

    }

    for event_handler, mut widget_input_events in query.iter_mut(){
        // This method is customized to each widget / widget type and converts input events into a common widget-logic format
        let new_events = event_handler.handle();

        // These events are later processed in widget-specific systems
        widget_input_events.push(new_events);
    }
}

```

### Customizing interaction behavior

Control iteration order.
Binding to hotkeys.
Multi-select.

## Drawbacks

1. An ECS-first UI is largely unexplored territory (but see [OrbTk](https://github.com/redox-os/orbtk)); we may encounter serious unknown-unknowns.
2. Debugging inter-widget communication may be complex. This is true regardless of paradigm though.
3. Solving the "event propagation" problem within the ECS will force us to build out new abstractions or improve existing ones.
4. While change detection + query caching severely reduces the performance cost of our classical "polling" (as opposed to "event-driven") UI approach, we create many, many systems, most of which will do nothing in most passes. We must be careful not to heavily regress the base cost of running systems as a result.
5. A good implementation of the signal-connection relies on relations.
6. Using Bevy-events as part of this pattern will interact poorly with pausing due to lost events until [bevy #1776](https://github.com/bevyengine/bevy/pull/1776) or a similar approach is merged.

## Rationale and alternatives

### Why do we want to be able to control UI behavior with systems?

UI widgets should be entities with various components (see #1 and **Motivation**).
With that established, we need to be able to easily extract data from the ECS, and operate on it in complex, user-extensible ways.
In Bevy, this implies the use of systems.

By automatically handling widget-logic in modular systems, we:

1. Make sure it's easy to extend and customize widgets without losing functionality.
2. Enable a data-driven workflow: adding the correct components to your widgets will cause it to automatically work.
3. Have great performance when operating over large numbers of similar widgets at once.
4. Avoid complex widget hierarchies by using composition over inheritance.

### Why is using a polling model okay?

TODO: flesh out.

Less indirection.

Low overhead.

Automatic parallelization.

Change detection.

### Why not use callbacks?

TODO: write.

## Unresolved questions

1. What are the details for input dispatching?
2. Can we unify our input event streams more elegantly?
3. Do circular system dependencies actually exist in real UI use-cases?

## Future possibilities

1. Archetype invariants #5 will be very useful to reduce bugs while building functional widgets, to ensure that all of the required systems will run correctly on them.
2. Relations will significantly improve the ergonomics of many of these patterns where we point to / store specific widget entities.

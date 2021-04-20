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

Widgets, as previously discussed in [UI Building Blocks and Styles](https://github.com/bevyengine/rfcs/pull/1) are simply entities with various components on them.
Rather than merely controlling the cosmetic style of our widgets, we can attach **functional components** to our widgets as well.

Just like when we're using the ECS to control our game's logic, these components both store data and control behavior, by determining which systems apply to each particular `Widget` entity.
Suppose we want to create a radio-button widget.
Rather than trying to create an object with a `RadioButtonWidget` type, like we might in an object-oriented UI framework,
we create an entity with all of the components required to accomplish the desired behavior.
Think of components as behaving in an analogous fashion to traits, with the radio button behavior having trait bounds for each of the required components.

Breaking it down into the constituents, we want our radio button to BEHAVIORS.

Turning this into code, we get a `RadioButton` component and a couple of supporting systems.

```rust
struct RadioButton {
    // These can't be pub fields, and instead must rely on accessor methods
    // to ensure internal invariants hold
    options: Vec<Label>, // TODO: do we need to define what exactly a label would look like here?
    state: Label,
}

// TODO: complete me
fn render_radio_buttons() {}

// TODO: complete me
fn handle_radio_button_input() {}
```

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

TODO: complete me.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

1. An ECS-first UI is unexplored territory; we may encounter serious unknown-unknowns.
2. Debugging inter-widget communication may be complex. This is true regardless of paradigm though.
3. Solving the "event propagation" problem within the ECS will force us to build out new abstractions or improve existing ones.
4. While change detection + query caching severely reduces the performance cost of our classical "polling" (as opposed to "event-driven") UI approach, we create many, many systems, most of which will do nothing in most passes. We must be careful not to heavily regress the base cost of running systems as a result.
5. A good implementation of the signal-connection relies on relations.
6. Using Bevy-events as part of this pattern will interact poorly with pausing due to lost events until https://github.com/bevyengine/bevy/pull/1776 or a similar approach is merged.

## Rationale and alternatives

### Why do we want to be able to control UI behavior with systems?

UI widgets should be entities with various components (see #1 and **Motivation**).
With that established, we need to be able to easily extract data from the ECS, and operate on it in complex, user-extensible ways.
In Bevy, this implies the use of systems.

When implementing UI behavior in the ECS, the system resolution and widget wiring problems are far and away the most complex challenges to be solved.
Unfortunately, they are from unique to the UI problem domain, even if they rarely appear in trivial demo games.
We need good, ergonomic solutions to them (whether these are features or design patterns) in any case.
UI is a great proving ground for these solutions due to its high complexity, tight constraints and shared requirements across many projects.

### Widget wiring

TODO: complete me.

## Unresolved questions

1. What does input handling look like? How do we ensure it's robust to multiple input paradigms?
2. Are there compelling use cases for the components-as-event-channels pattern?
3. Do circular system dependencies actually exist in real UI use-cases?

## Future possibilities

1. Archetype invariants #5 will be very useful to reduce bugs while building functional widgets, to ensure that all of the required systems will run correctly on them.
2. The widget wiring solution is much better handled with relations.

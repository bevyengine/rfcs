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
  options: Vec<Label> // TODO: do we need to define what exactly a label would look like here?
  state: Label,
}

// TODO: complete me
fn render_radio_buttons(){

}

// TODO: complete me
fn handle_radio_button_input(){

}
```

Widgets can be wired to each other and the game state in two common ways:

1. Directly reading the state of our widget's components.
2. Using the **components-as-event-channel** pattern to listen to events emitted by specific widget components.
By storing `Events` in our components, we can mimic the idea of "event channels",
allowing users to quickly differentiate between various events of the same type based on where they came from.

Read the component's state when you care about its current state, use event channels when you want to detect events being fired off.
This is clearer with an example:

```rust
// TODO: complete me
```

## Reference-level explanation

TODO: complete me.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

1. An ECS-first UI is unexplored territory; we may encounter serious unknown-unknowns.
2. Debugging inter-widget communication may be complex. This is true in virtually all cases though.
3. Solving the "event propagation" problem within the ECS will force us to build out new abstractions or improve existing ones.
4. This pattern creates many, many systems, most of which will do nothing in most passes. This constraints
5. A good implementation of the signal-connection relies on relations.

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
2. What exact API and semantics do we want for the components-as-event-channels pattern?
3. Do circular system dependencies actually exist in real UI use-cases?

## Future possibilities

1. Archetype invariants #5 will be very useful to reduce bugs while building functional widgets, to ensure that all of the required systems will run correctly on them.
2. The widget wiring solution is much better handled with relations.

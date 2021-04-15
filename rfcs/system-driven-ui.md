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

### Widget wiring

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

### Strategies for system resolution

When implementing your own systems that control the UI, you need to be mindful that all UI logic is concluded (**at rest**, in contrast to **in motion**)
before advancing to other parts of the game logic or rendering.

Failing to do so can result in strange bugs, unpleasant input delays and flickering UIs (see the corresponding section on system resolution below).

TODO: describe solution.

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

### Why ECS-powered UI?

TODO: complete me.

### Widget wiring

TODO: complete me.

### Widget behavior

TODO: complete me.

### System resolution

When attempting to control UI behavior through our ECS, we are forced to confront the challenge of **system resolution** head-on.
This, in a nutshell, is the notion that we want our relevant collection of systems to be "at rest" before advancing on to the next part of our data pipeline.

While this problem is not *unique* to UI (complex event-based gameplay logic can encounter it too),
it is particularly prevalent in UI due to the large number of heterogenous parts communicating in unpredictable ways.

At first, one may be determined to resolve this through careful system ordering.
Simply trace out the pathways your data could flow, and make sure those systems resolve before their dependencies.
While this may get unwieldy in complex UIs, you can solve the problem without any complex data structures.

But cyclic dependencies, which are reasonably common in any complex enough system, foil our plans.
Suppose we have one widget which when interacted with talks to a second widget.
The second widget then (perhaps through some complex chain of events) modifies the first widget again before eventually the chain terminates.
No matter the order of our systems, we cannot fully resolve this in a single pass of each system.

Presented with such a conundrum, we might throw up our hands and ask "Why not just move on, and render things in motion? What's the worst that could happen?"
Unfortunately, a wide range of unpleasant things:

1. Invalid state. TODO: expand.
2. Responsiveness issues. TODO: expand.
3. Flickering graphical representations. TODO: expand.

Accepting that these are dire enough that we should avoid them, what does a good solution look like, and what are our options to get there?

TODO: discuss potential solutions

## Unresolved questions

1. What does input handling look like? How do we ensure it's robust to multiple input paradigms?
2. How do we ensure that signals are fully propagated before advancing?

## Future possibilities

1. Archetype invariants #5 will be very useful to reduce bugs while building functional widgets, to ensure that all of the required systems will run correctly on them.
2. The widget wiring solution is much better handled with relations.

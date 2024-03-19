# Feature Name: `dev-tools-abstraction`

Key questions:

- what are the defining features of a dev tool?
  - eases Bevy development experience, typically be presenting information about the app or its function
  - not intended for end-user use
    - must be able to be disabled at compile time
  - contextually useful based on encountered weirdness in the app
    - must be able to be toggled on and off at runtime without a reboot
  - should not interfere with normal operation of the app
  - can be customized to meet the needs of both the app and the developer
- what are clear examples of dev tools that we might want?
  - system graph visualizer
  - bevy_inspector_egui
  - fly camera
  - UI node visualizer
  - FPS display
  - resetting a level
  - toggling god mode
- who creates dev tools?
  - Bevy itself: FPS display
  - third-party crates: bevy_rapier collider overlay
  - end users: toggle god mode
- who creates dev tool consumers?
  - Bevy itself: scene editor
  - third-party crates: dev console
  - end users: custom level editors
- how might dev tools be toggled and consumed?
  - Quake-style dev console
  - unified set of hot keys
  - menu bar from scene editor
- how might dev tools be displayed?
  - screen overlay
  - pop-out window
  - embedded panel widget
  - cli
- how might dev tools be configured?
  - color palettes
  - fonts
  - font size
  - screen location
  - dev tool specific features: camera speed, entities to ignore etc
- why do dev tools need to be aware of each other?
  - overlays can clash: toggle off current overlay before enabling next
- how can we make it easy for third-party consumers and producers to play nice together?
  - Cart's suggestion: let each consumer decide which tools it wants to use
    - creates a quadratic problem: must wire up plumbing separately for each tool and consumer
  - define a standard API for available dev tools and essential operations
    - existing tools will have to conform to this in some way

Out of scope:

- [where should dev tools live](https://github.com/bevyengine/bevy/pull/12354)?
  - where the primitives used are defined? e.g. bevy_gizmos
  - where they're used for debugging? e.g. bevy_sprite
  - in bevy_dev_tools?
  - in dedicated crates, saving bevy_dev_tools for higher level abstractions?

## Summary

One paragraph explanation of the feature.

## Motivation

[`bevy_dev_tools`](gh link) was recently added to Bevy, giving us a home for first-party tools to ease the developer experience.
However, since its inception, it has been plagued by debate over "what is a dev tool?"
and every **tool** within both Bevy and its ecosystem follows its own ad hoc conventions about how it might be enabled and configured.

This is frustrating to navigate as an end user, but more critically,
makes the creation of **toolboxes**, interfaces designed to collect multiple dev tools in a single place (such as a Quake-style dev console)
needlessly painful, requiring manual glue work for every new tool that's added to it.

In some cases, tools can actively interfere with the function of other tools.
A prime example of this are text-based overlays, like you might see in an FPS meter.
Toggling one tool might completely cover (or distort the layout) of another, and so some level of coordination is required for a smooth user experience.

## User-facing explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.

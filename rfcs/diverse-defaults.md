# Feature Name: Different plugin groups to support different use cases (diverse-defaults for short)

## Summary

There are more than half a dozen different catagories of bevy 'Apps'. Each 'use case' has a sufficiently different 'common sense' default configuration to warrant different use case specific default plugins. The catagories I have been able to come up with @alice-i-cecile's help are:

- minimal (current bevy DefaultPlugins)
- game
 - 2D
 - 3D
- TUI
- application
 - 2D
 - 3D 
- simulation

Each of which may have some combination of the following divergent platform configuration needs:

- desktop
 - native
 - web
- tablet
 - ios native
 - android native
 - web
- phone
 - ios native
 - android native
 - web


Each of these use cases can be target in a custom manner by the user. This is fine for advanced users, but is less than an ideal user story for bevy beginners who just want to get 'something' working as fast as possible lest they decide to try a different easier to use engine/framework. Were bevy to provide sane defaults for each use case new users would be able to get soemthing working in non-standard (i.e. minimial DefaultPlugins) very easily. This can also be highlighted by different book chapters per use case.

THis also involves not-yet-implemented but planned bevy features that will have different implementations per use case.

For example work-minimizing scheduling for low-power consumption web and mobile applications, or desktop productivity apps are a very different configuration from high performance gaming and high performance scientific simulation.

2D vs 3D will have different built-in bevy selection methods/mechanisms and different cameras... Anything else?

Building for touch vs mouse vs gamepad or other midi/advanced input controller are yet more very different use cases with very different requirements.

I have run out of ideas? Any suggestions?

## Motivation

Bevy would be doing this to support easier an faster getting started in the major catagories of different use cases for bevy. It is particularly helpful to beginniers but would be a nice ergonomics/UX win for experienced bevy devs to not have to configure every minute detail manually when working in multiple 'use case' domains on different projects. Anything else you can think of?

## User-facing explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
    - DefaultPlugins can become an enum with each usecase as a different enumerant?
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
    - Small book chapters would probably be the way to go?
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
    - What I would put here is identical to what I have written in the motivation section, is that ok?
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
    - DefaultPlugins for the current minimal config would still work the same, IIUC this would be a purely additive feature.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.
    - Currently users can create these configs manually, the change would be bevy providing more standard getting started configs/DefaultPlugins out of the box.

## Implementation strategy

I need others help/input with this.

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

I need others help/input with this.

Why should we *not* do this?

## Rationale and alternatives

I need others help/input with this.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

I need others help/input with this.

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

I need others help/input with this.

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

I need others help/input with this.

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

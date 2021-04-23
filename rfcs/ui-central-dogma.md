# Feature Name: (fill me in with a unique ident, `ui-central-dogma`)

## Summary

`bevy_ui` is a Rust-only, data-driven, ECS-powered UI framework that is tightly integrated with the rest of Bevy.
This RFC lays out the motivation for this approach, describes the core data flow, connects the high-level components we'll need, and shows what complete vertical slices of complex UI will look like.

## Motivation

Bevy, like all other game engines, needs a UI solution.
By establishing a common, idiomatic approach to UI we can ensure that the Bevy experience remains accessible for developers of all skill levels and is easy to interact with,
regardless of how blurred the line between gameplay and interface becomes.

Bevy's existing UI framework (as of 0.5), is unequipped to handle complex designs due to the excessive boilerplate required and unopinionated dataflow,
doesn't fit well into a data-driven paradigm, and is generally short on features.

Designing this from the perspective of an advanced user using paper prototypes ensures that it's up to the primary challenge of UI: complexity management.

## Guide-level explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Reference-level explanation

Many of the technical details will be buried in corresponding RFCs to implement various parts.

Despite that, it's valuable to examine, in uncompromising detail, exactly what our final API will look like for complex, vertically sliced examples.


This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
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

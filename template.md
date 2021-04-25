# Feature Name: (fill me in with a unique ident, `my_awesome_feature`)

## Summary

One paragraph explanation of the feature.

## Motivation

Why are we doing this? What use cases does it support?

## User-facing explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## \[Optional\] Examples

Lay out the examples that Bevy should add to either the examples folders or the doc comments to help new users discover and learn about this feature.
These can either be completely new, or simply changes to existing examples.
When completing this section, you can but do not have to write all of these examples yourself.
Describing them in sufficient detail that another developer can do so is perfectly fine.
If your examples are particularly long, please provide a brief description and link to a [Gist](https://gist.github.com/) for each.

Most examples in Bevy aim to clearly demonstrate a single feature, group of closely related small features,
or show how to accomplish a particular task (such as asset loading, creating a custom shader or how to test your app).
In rare cases, creating new "game" examples is justified to demonstrate new features that open a complex class of functionality in a way that's hard to showcase in isolation or requires additional integration testing.

Examples in Bevy should be:

1. Working. They must compile and run, and should fail in obvious ways.
2. Clear. They must use descriptive variable names, have reasonable code-quality, be formatted, and be appropriately commented.
3. Relevant. They should use game-relevant fluff and explain why what they're demonstrating is useful.
4. Minimal. They should be no larger or more complex than is needed to meet their other goals.
This reduces maintenance burden and improves clarity when used as a reference.

More complex demonstrations of functionality are also welcome, but for now belong in community tutorials or template games.

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section, and explain more the details that make those examples work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

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

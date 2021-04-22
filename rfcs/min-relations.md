# Feature Name: `min-relations`

## Summary

Relations connect entities to each other, storing data in these edges. This RFC covers the minimal version of this feature, allowing for basic entity grouping and simple game logic extensions.

## Motivation

Entities often need to be aware of each other in complex ways: an attack that targets another unit, groups of entities that are controlled en-masse, or complex parent-child hierarchies that need to move together.

We *could* simply store an `Entity` in a component, then use `query.get(my_entity)` to access the relevant data.
But this quickly starts to balloon in complexity.

- How do we ensure that these components are cleaned up when the entity they're pointing to is?
- How do we handle pointing to multiple entities in similar ways?
- How do we look for all entities that point to a particular entity of interest?
- How do we despawn an entity and all of its children (with many types of child-like entities)?
- How do we quickly access data on the entity that we're attached to?
- How do we traverse this graph of entities?
- How do we ensure that our graph is acylic?

We *could* force our users to solve these problems, over and over again in each individual game.

This powerful and common pattern deserves a proper abstraction that we can make ergonomic, performant and *fearless*.

## Guide-level explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Reference-level explanation

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

1. GRAPH
2. Kinded entities
3. Use with UI.
4. Frustum culling.
5. Replace parent-child.
6. Reverse relations.

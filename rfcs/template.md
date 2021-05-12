# Feature Name: 'quick-start-book'

## Summary

The [Bevy book](https://bevyengine.org/learn/book/introduction/) will be extended by the community, and supplemented by a Quick Start guide.

## Motivation

The Bevy book has fallen behind the pace of Bevy's features, leaving newcomers a bit baffled and worried about the state of the documentation.
Examples, in-line documentation and community tutorials are great, but aren't as visible, and do a poor job at providing high-level explanations of how all of the pieces of Bevy fit together.

New users encountering Bevy tend to fall into one of two learning styles:

- **Example-first:** These users want to dive right in, see everything in action and get a working game as quickly as possible.
- **Definition-first:** These users want to carefully build up a mental model of Bevy, thoroughly understanding each new concept before moving on.

Each of these requires their own complementary learning paths that should branch as soon as they get to the [Learn page](https://bevyengine.org/learn/) to ensure that the first experience that they have with Bevy matches what they need.

## User-facing explanation

So, you want to help write some learning material for the [Bevy website](https://bevyengine.org/)?

### Example-first path

### Definition-first path

### Contributor's style guide

## Implementation strategy

1. The guide-level explanation above should live in `CONTRIBUTING.md` of the [`bevy-website` repo](https://github.com/bevyengine/bevy-website).
2. This should be linked to from the website, likely under a "Contributing" tab.
3. Per [bevy-website #143](https://github.com/bevyengine/bevy-website/issues/143),the website will need to be revamped somewhat.
4. Automatic compilation of code snippets is the critical feature from the above, ensuring that we can keep the examples current in a reliable way.
5. Once this RFC is merged, work should begin on a separate branch of `bevy-website` repo to create a new book.
6. For the launch of 0.6 (or perhaps sooner), this branch is merged and becomes the new `main`.
7. A persistent `bevy-main` branch of the repo is maintained and with a dependency on the `main` branch of `bevy` to ensure that we can test out new changes and refine the code before new features and breaking changes go live and reduce release crunch.

## Drawbacks

1. Delegating much of this work to the community risks creating an inconsistent editorial voice for the book.
2. Documentation created here *absolutely* must remain correct and current or we will lose new users.
3. Synchronizing the examples against main while in a separate repo is somewhat frustrating.

## Rationale and alternatives

### Why can't this live in community-provided documentation?

### Why do we need two paths?

## Unresolved questions

1. Should we aim to have one continuous example for the Book, or several smaller ones?
2. Which examples are we using?
3. Which features must be taught in an MVP?
4. Which order do we want to teach this in?
5. Which style rules do we want to use to ensure a consistent tone?

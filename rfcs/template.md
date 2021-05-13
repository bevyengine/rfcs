# Feature Name: 'quick-start-book'

## Summary

The [Bevy book](https://bevyengine.org/learn/book/introduction/) will be extended by the community, and supplemented by a Quick Start guide.

## Motivation

The Bevy book has fallen behind the pace of Bevy's features, leaving newcomers a bit baffled and worried about the state of the documentation.
Examples, in-line documentation and community tutorials are great, but:

- aren't as visible
- aren't as approachable
- vary significantly in quality and freshness
- do a poor job at providing high-level explanations of how all of the pieces of Bevy fit together

## User-facing explanation

So, you want to help write some learning material for the [Bevy website](https://bevyengine.org/)?

As you probably noticed, our introductory learning material is split into two main sections:

1. **Bevy Quick Start:** "Get started making your first game now!"
2. **Bevy Book:** "Understand how Bevy works, and how you can use it"

This is intended to cater to two different types of learners, without compromising the experience for either:

- **Example-first:** These users want to dive right in, see everything in action and get a working game as quickly as possible.
These users often have an idea in their mind that they want to start prototyping as quickly as possible.
- **Definition-first:** These users want to carefully build up a mental model of Bevy, thoroughly understanding each new concept before moving on.
These users tend to be driven by curiosity, or are aiming to carefully develop a new skill.

Crucially, these paths are independent of the experience levels of the learner!
Bevy intentionally aims to be inclusive of both complete beginners who have never programmed before, and professionals coming from other engines.

|                      | **Beginner**                                                       | **Professional**                                                     |
| -------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **Example-first**    | Enthusiastic, wants to create a new version of the game they love. | Exploratory, wants to dive in and see how Bevy holds up in practice. |
| **Definition-first** | Curious, wants to understand how making games works.               | Critical, wants to understand Bevy's unique design choices.          |

Each of these requires their own complementary learning paths that branch as soon as they get to the [Learn page](https://bevyengine.org/learn/) to ensure that the first experience that they have with Bevy matches what they need.

### Bevy Quick Start: the example-first path

Users following the example-first path will tend to take the following route:

1. Encounter the Bevy homepage due to social media or word of mouth.
2. Navigate to the Learn page.
3. Choose one of the most relevant **quick start games**.
4. Complete that tutorial.
5. Dive into making the game they have in mind, accessing the following resources as needed when they encounter road-blocks:
   1. Official Examples.
   2. The Bevy book.
   3. Community tutorials and template games.
   4. Various community support forums.

This path should prioritize:

1. Rapid time-to-fun.
2. Functional, good-enough explanations that are tied to the code in front of them.
3. Relevance of quick-start game to the genre of game they want to make.
4. High asset quality.
5. Ease of extending the quick-start game with their own tweaks.
6. Explaining how to get unstuck, through documentation, community help and filing issues.

### The Bevy Book: the definition-first path

Users following the definition-first path will tend to take the following route:

1. Encounter the Bevy homepage due to social media or word of mouth.
2. Navigate to the Learn page.
3. Select the **Bevy book**.
4. Read through the book, largely in order.
5. Once they feel they have a good enough understanding of the engine, they will begin to make their own games, typically by jumping over onto the example-first path.
6. As they explore, they will also browse:
   1. The source code.
   2. [docs.rs](https://docs.rs/bevy/)
   3. CONTRIBUTING.md, GitHub issues and pull requests.
   4. Release notes.
   5. The engine development channels on Discord.

This path should prioritize:

1. Clear, thorough explanations.
2. Carefully introducing one concept at a time in an organized fashion.
3. Connecting concepts to each other in context.
4. Explaining the technical details of how things work, but only in clearly marked asides.
5. Communicating all of the supporting development practices that make Bevy productive:
   1. How to set up your dev environment.
   2. Code organization.
   3. Design patterns and best practices.
   4. Testing, benchmarking and debugging.
   5. Contributing to Bevy itself.
6. Linking to further reading: official examples, `docs.rs` and (very sparingly) source code links.

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

This distinction is not new, but is quite striking within Bevy's community (see [this user report](https://github.com/bevyengine/bevy/issues/2109)).

The types of explanation that one group prefers serve only to frustrate the other group: we can't effectively tackle both use cases at once without painful sacrifices.
Instead, by clearly sign-posting two equally-valid paths to get started, we can trust users to filter into the path that works for them, and consult the other path's learning material when it's particularly relevant.

### Why do we need more than one example game in the Bevy Quick Start?

As discussed above, example-first users often want to jump straight into making their game.
This is great, but Bevy can cover a huge number of use cases, each of which demand very different strategies.

A user trying to adapt a chess tutorial to a first-person shooter will be just as frustrated as one trying to turn an arcade game into a scientific simulation or card game.

We *cannot* tell these users to "just read the docs" and build up their game from first-principles: this isn't how they want to learn, and they generally won't have strong enough conceptual models to do so effectively!
The closer the first example they explore is to the genre they have in mind, the better their experience will be (as long as we are careful to avoid overwhelming them with similar options).

## Unresolved questions

1. Should we aim to have one continuous example for the Book, or several smaller ones?
2. Which examples are we using?
3. Which features must be taught in an MVP?
4. Which order do we want to teach this in?
5. Which style rules do we want to use to ensure a consistent tone?

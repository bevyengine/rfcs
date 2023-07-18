# Feature Name: `book_evolution_in_main`

## Summary

Reduce the cost of starting the book, and of ongoing book evolution with new versions of Bevy.

## Motivation

I want the Bevy Book to get better faster. We've tried to make it evolve in bevy-website repository, but getting to a publishable state means getting the content, the CI and the formatting ready all at once and lead to a long lived `new-book` branch that is in limbo. The current approach would also mean that for a new version of Bevy, we would have to pull a new branch, make it use the next version of Bevy, update everything, then merge it back on the website for publication. I fear this will make the release process longer.

I'm opening this as a RFC as it has moving parts across 2 repositories, and it's an evolution on [RFC #23: Quick Start Book](https://github.com/bevyengine/rfcs/blob/main/rfcs/23-quick_start_book.md).

## User-facing explanation

As a user wanting to contribute to the book, I can do so on the Bevy main repository.

## Implementation strategy

- Add a new folder in Bevy main repository for the book
- Start accepting contributions for it, with the layout / rules from [RFC #23: Quick Start Book](https://github.com/bevyengine/rfcs/blob/main/rfcs/23-quick_start_book.md)
- Set up main repository CI to check that the book content is up to date with the code
  - For new book PRs, this should be blocking
  - For new code PRs, this should be a non blocking CI. Writing or updating the book can be delegated to another issue that will be added to the next milestone
- Set up bevy-website repository CI to pull the book content from the latest tag and publish it

## Drawbacks

🤷

More non code commits in the main repo maybe? But I think that's good as it gives better visibility to book contributions. The main repo will also gets larger which can slow down checkout / clones.

## Rationale and alternatives

- It allows us to start writing the book without worrying it about being public directly, or working in a long lived branch
- This keeps the book versioning in check with Bevy

## Future possibilities

- Publish the book content as part of the rust doc (a `bevy_book` crate?)
- Publish the book content with versioning
- Remove `/docs/` in the main repo and migrate all of that content to the Bevy book

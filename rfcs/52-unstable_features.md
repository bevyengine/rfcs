# Feature Name: `unstable_features`

## Summary

Accepting PRs for new features that may not have reach the high bar we set for Bevy, and disable their changes with a feature gate.

## Motivation

Some features would benefit from iterative work in Bevy, but that may be hard to do when a PRs need a high level of quality to be merged in.

In this RFC, "feature" means the new feature or change under development, and "feature gate" the conditional compilation flag. "unstable" means a feature that is subject to change, not something that will break Bevy.

## User-facing explanation

So you want to contribute a new feature to Bevy! You have a few paths available to you, depending on the scope of the feature:
- A small feature with a clear implementation? Go ahead and open that PR!
- A large feature with an open design space that could use some discussion about how to implement it? A RFC would be good.
- Something in between that would benefit from iterative work? Here come the unstable feature workflow for you!

You will need to open a tracking issue to explain the new feature you want to add. If it looks good, you'll be ready to open that PR with a feature gate for your changes!

## Implementation strategy

The new workflow for this kind of feature would be:
- Submit an issue explaining the feature you want to add under a feature gate. This will be the tracking issue for stabilization once approved. This issue gets a new label `S-Stabilization`.
- If approved, submit a PR with the new feature and the feature gate
- This PR can be merged by someone with the merge rights on the related part of Bevy as soon as it's `S-Ready-For-Final-Review`, without an approval by the main Bevy maintainers.
- Subsequent PRs on the feature must be linked to the tracking issue and follow the same process.
- Once the feature is finished, enter the stabilization period. Open a PR that will remove the feature gate. This PR need to be approved by the main Bevy maintainers.
The feature gate should follow the following convention for its name: `unstable-<#tracking-issue-number>-feature-name`.


An existing PR can be retroffited in this process on the suggestion of Bevy community members, and if the PR author agrees.

The RFCs and the unstable features are closely related. An RFC can be implemented as an unstable feature to help resolve issues that need more experimentation. An unstable feature could require an RFC to reach stabilization.

Each PR will still need to be approved by Bevy community members, and would still need to have the same level of quality as other PRs merged.

The PRs for the initial feature, various changes during its finalization and for the stabilization can be opened by different persons.

The feature gate should not be exposed on the main `bevy` crate, but only on the subcrates where it is relevant. It can be enabled as a user by adding dependencies directly on the subcrates and enabling the feature.

## Drawbacks

This will increase complexity in Bevy code.

It will also increase complexity of a contribution that would like to use this process, but it is opt in and not mandatory. Think of it as a fast track for unstable features.

## Rationale and alternatives

- Improve the process to get new features in Bevy that would benefir from iterative work.
- Alternatives:
    - Continue as it is now, with semi large PRs that bitrot and RFCs that don't often move forward
    - Work harder to implement a new feature as a third party crate (this is not always possible)
    - Fork Bevy to merge unnaproved PRs and check how they work / interact


## Prior art

[The Rust process](https://rustc-dev-guide.rust-lang.org/implementing_new_features.html)

## Unresolved questions

- Do we want to limit the number of feature gates in Bevy to reduce complexity?
    - this could help control the increased complexity in Bevy
- Do we want to limit the maximum age of a unstabilized feature?
    - this could help control the increased complexity in Bevy, and requires being ready to cut a feature off if it's not getting the use / stabilization expected
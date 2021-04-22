# Bevy RFCs

Thank you for contributing to Bevy! If you've been asked to make an RFC, it's because your contribution is significant enough that it warrants careful thought from the Bevy community.

## What is an RFC?

RFCs (request for comments) provide a way for Bevy contributors to propose designs for specific features in a [structured way](template.md). Creating a new RFC starts a collaborative process where the Bevy community (and [@cart](https://github.com/cart), the Bevy project lead) can review your design and suggest changes. If an RFC is accepted, this indicates that it is in line with our vision for Bevy's future. Bevy contributors can implement accepted RFCs comforted by the knowledge that the design has already been "approved".

* RFCs are for large features, breaking into new design areas, major breaking changes, or significant changes to Bevy App development.
* The majority of Bevy changes (bug fixes, small tweaks, and iterative improvements) _should not_ go through the RFC process. Just use the normal [contributing process](https://bevyengine.org/learn/book/contributing/code/) in the [main Bevy repo](https://github.com/bevyengine/bevy).
* RFCs are _not_ feature requests. They are for developers who already have a specific design in mind (including technical details) and want feedback on it. If you want to request a feature, please [create an issue in the main Bevy repo](https://github.com/bevyengine/bevy/issues/new?assignees=&labels=enhancement%2C+needs-triage&template=feature_request.md&title=).
* RFCs should be scoped. Try to avoid creating RFCs for huge design spaces that spans many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
* RFCs should avoid ambiguity. Two developers implementing the same RFC should come up with nearly identical implementations.
* RFCs should be standalone and "implementable". Merged RFCs should only depend on other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.
* Avoid creating RFCs before you have a design in mind. If you want to explore design spaces with the Bevy community, consider finding or creating a [Github Discussion in the main Bevy repo](https://github.com/bevyengine/bevy/discussions). If at any point during the discussion you discover a design you believe in enough to bring to completion, create an RFC. You don't need to have _all_ of the details sorted out ahead of time (RFCs are a collaborative process), but an initial draft RFC should have enough detail to significantly narrow the scope of the design.

If you are uncertain if you should create an RFC for your change, don't hesitate to ask us in the `#dev-general` channel in the official [Bevy Discord](https://discord.com/invite/bevy).

## Why create an RFC?

**RFCs are intended to be a tool for collaboration, not a burden for contributors.**

RFCs protect Bevy contributors from wasting time implementing features that never had a chance of getting merged. This could be due to things like misalignment with project vision, missing a key requirement, forgetting a technical detail, or failing to consider alternative designs.

RFCs also serve as a form of documentation. They describe how a feature should work and why it should work that way. The accompanying pull request(s) record how the Bevy community came to that conclusion.

They don't need to be perfect, complete, or even very good when you submit them. The goal is to move the discussion into a format where we can give each part of the design the focus it deserves in a collaborative fashion.

## The Process

1. Fork this repository and create a new branch for your new RFC.
2. Copy `template.md` into the `rfcs` folder and rename it to `my_feature.md`, where `my_feature` is a unique identifier for your feature.
3. Fill out the RFC template with your vision for `my_feature`.
4. [Create a pull request](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request) in this repo. The first comment should include:
   1. A one-sentence description of what the RFC is about.
   2. A link to the "rendered" form of the `my_feature.md` file. To do so, link directly to the file on your own branch, so then the link stays up to date as the file's contents changes. See #1 for an example of what this looks like.
5. Help us discuss and refine the RFC. Bevy contributors and @cart (Bevy's project lead) will leave comments and suggestions. Ideally at some point relative consensus will be reached. Your RFC is "accepted" if your pull request is merged. If your RFC is accepted, move on to step 6. A closed RFC indicates that the design cannot be accepted in its current form.
6. Bevy contributors are now free to implement (or resume implementing) the RFC in a PR in the [main Bevy repo](https://github.com/bevyengine/bevy), and an associated tracking issue is created in that repo. You are _not_ required to provide an implementation for your RFC, nor are you _entitled_ to be the one that implements the RFC.

## Collaborating

First, make sure you always abide by the [Bevy Code of Conduct](https://github.com/bevyengine/bevy/blob/main/CODE_OF_CONDUCT.md) when participating in the RFC process.

Additionally, here are some suggestions to help make collaborating on RFCs easier:

* The [insert a suggestion](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/commenting-on-a-pull-request#adding-line-comments-to-a-pull-request) feature of GitHub is extremely convenient for making and accepting quick changes.
* If you want to make significant changes to someone else's RFC, consider creating a pull request in their fork/branch. This gives them a chance to review the changes. If they merge them, your commits will show up in the original RFC PR (and they will retain your authorship).
* Try to have design discussions inside the RFC PR. If any significant discussion happens in other repositories or communities, leave a comment with a link to it in the RFC PR (ideally with a summary of the discussion).

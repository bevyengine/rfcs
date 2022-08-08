# Feature Name: `bevy-nursery`

## Summary

Work done within the [broader Bevy ecosystem](https://bevyengine.org/assets/) is sometimes a good fit as part of the core engine.
These crates should live in a **nursery** owned by the Bevy org for a period of time,
where they can be polished and refined by the broader community,
before eventually being merged in completely.
When this occurs, nursery crate maintainers will be granted scoped merge rights to the main engine.

## Motivation

Third-party crates, commonly plugins, are often used to extend the engine in ways that are broadly applicable.
Polished crates that are a good fit for the engine should eventually be merged in wholesale,
rather than reimplemented, to avoid the [second system effect](https://en.wikipedia.org/wiki/Second-system_effect)
and minimize breakage for their users.

However, merging in these crates directly will result in very large, unreviewable PRs,
and may compromise Bevy standards for quality.
But if we delay, critical ecosystem crates may go stale, pointlessly splitting efforts, or making it hard for beginners to discover the de-facto standards.

## User-facing explanation

In addition to the core crates included within `bevy` itself, there are a number of **nursery** crates,
which are being prepared for inclusion in the engine.

These crates follow exactly the same technical rules as any other third-party crate, but:

1. Live in the [`bevyengine` organization](https://github.com/bevyengine) on GitHub.
2. Follow Bevy's standards for code quality and project management.
3. Explicitly welcome community contributions to help get them ready for release.
4. Will have a branch prepared for the newest release of Bevy before we release a new major version.

You can find these crates by looking for the special `Nursery` tag on [Bevy Assets](https://bevyengine.org/assets/),
or by browsing the [`bevyengine` organization](https://github.com/bevyengine)'s list of repositories.

### Submitting a crate to the nursery

If you would like to submit a crate for inclusion in the nursery, please open an issue in the `bevy-nursery` repository.
While you do not need to be the crate's maintainer to do so, the following criteria must be met before it will be accepted to the nursery:

1. The existing code must be MIT + Apache 2 licensed.
2. The crate maintainers must consent.
   1. While this is not strictly necessary due to licenses, buy-in and consent is important to us.
3. The Bevy maintainers must collectively agree to accept the crate as a nursery crate.

A crate is a good fit for the nursery if:

1. The crate covers functionality that we eventually want to include in the engine itself. Factors include:
   1. Broadly useful to engine users.
   2. A direct replacement or improvement for an existing engine feature.
2. The crate is reasonably mature.
3. The crate is well-loved.
4. The crate follows a high standard for testing, code quality and documentation.

Very small or uncontroversial crates may be suitable for a direct PR via the ordinary mechanism.

### Setting up a nursery crate

Once accepted, the repository will be transferred to the Bevy organization under the same name and `crates.io` identity.

The existing crate maintainers will have complete admin permissions over the new crate,
and any other Bevy maintainers will be granted write permissions (or the `bors` equivalent).

A new milestone will be created, called `upstreaming`, which will track the work that needs to be completed before the crate can be submitted as a PR to `bevy` itself.

Finally, the issues and PRs from that crate will be added to the `#bevy-nursery` channel on Discord to improve discoverability.

### Preparing a crate to be upstreamed

Before a nursery crate can be accepted to Bevy itself, the following conditions must be met:

1. All items in the `upstreaming` milestone must be complete.
2. A final release must be published, encompassing all PRs merged to date.
3. The Bevy maintainers must collectively agree to accept the crate upstream in its current form.

This checklist may be helpful when considering issues for the `upstreaming` milestone:

- [ ] are the examples and documentation helpful and complete?
- [ ] do all features work as intended?
- [ ] is the code quality acceptable?
- [ ] is there any urgent highly-experimental work that should be completed before attempting to integrate more closely with the rest of the engine?

Crates **do not have to be complete** to be merged into Bevy itself: merely polished and clearly useful.

The PR to merge the crate into Bevy itself should be very simple, and contain only the bare minimum change set required to enable interoperability.
Further integration work should be done in follow-up PRs.
If you are an interested community member, raise issues and submit fixes on the nursery repository **before** the crate is being upstreamed.

Once the upstreaming PR is merged, the nursery crate's repo will be archived and any still-relevant issues will be migrated to the main Bevy repo.

### Maintaining upstreamed crates

When a nursery crate is upstreamed into the main engine,
the core maintainer team of the nursery crate will be offered (but do not need to accept)
**scoped merge rights** for the Bevy project.

Nursery crate maintainers have shown themselves to have the judgement and reliability needed to manage part of the core engine:
upstreaming a crate should not impose any additional barriers to the crate's development and maintenance.
Following the standard rules for community reviews, they will be able to merge PRs that affect their nursery crate and directly related areas of the engine.
Controversial PRs in their area will require either the sign-off of the project lead *or* the nursery crate maintainer, although like always, consulting experts is always wise!

Like always, the project lead may choose to hand out (or revoke) additional permissions on a case-by-case basis where warranted, but the expansion of merge rights outlined above is the default process when a nursery crate is upstreamed.

## Drawbacks

1. Clearly picks winners within the crate ecosystem.
   1. These winners typically already exist, but it's hard for beginners to discover that critical information.
2. Hands out additional merge rights to the engine.
   1. May increase chaos and dilute product direction.
   2. Tracking areas of expertise may get onerous.
   3. Marginally increases security risks (2FA + only write permissions) reduce this risk substantially.
3. Likely to increase the scope of the core engine.
   1. Feature flags, template games and plugin sets help ensure users only compile and ship the features they care about.
   2. Adding maintainers ensures that the workload does not increase significantly.

## Rationale and alternatives

### Why should we upstream crates?

In other modular ecosystems, it's very common to have critical functionality outside of the core project.
This is unfortunate because:

1. It's very challenging for beginners to discover these essential tools.
2. These tools cannot readily be used in official documentation.
3. These tools may suddenly go unmaintained.
4. Work may be divided across competitors (including work done by the core).
5. Integration with the core may be poor, or suddenly break.

By upstreaming essential tools, we can avoid these problems, and gracefully scale the core organization.

### Why do we need a "nursery" stage?

Ecosystem crates are huge, complex endeavours.
We want to ensure that the appropriate quality bar is met, but still signal early once a clear candidate has been identified.

The nursery stage serves as a distributed PR review process for the entire crate, breaking down the necessary changes into manageable units of work.
The added attention helps ensure that community stakeholders have a chance to weigh in before the crate itself is upstreamed.

Finally, as Bevy stabilizes, nursery crates can be rapidly iterated on without following Bevy's stability guarantees.

### Why shouldn't we rename the crate to `bevy_nursery_*` for improved discoverability?

This results in significant bureaucratic work (transferring issues and PRs), risks user confusion and adds pointless migration pain.
The discoverability issue can be tackled via the Discord channel and the prominent label on Bevy Assets.

### Why do we need to grant merge rights to nursery crate maintainers?

If we do not:

1. Maintainers of popular ecosystem crates are reluctant to submit their crates to the nursery due to the reduced ability to iterate on and fix their projects.
2. Maintainers of the core engine may become overloaded with work as the scope of the engine grows.

The maintainers of nursery crates have shown their ability to identify, create and lead a high-value project: they're natural candidates for additional responsibility in the core engine.

### Why not use GitHub's Code Owners feature to help scope merge rights?

While [bors has code owners integration](https://bors.tech/rfcs/0357-bors-support-for-codeowners.html),
it's not clear that it's a useful tool over simple social contract.

Setting up (and maintaining) these lists is tedious bureaucracy.
This approach would also encourage a less-integrated architecture for organizational reasons, rather than technical ones.

## Prior art

The [rust-nursery](https://github.com/rust-lang-nursery) exists for similar reasons, although there is a greater emphasis on stability.

[Druid](https://github.com/linebender/druid) has a similar [widget nursery](https://github.com/linebender/druid-widget-nursery), and explicitly follows a policy of [optimistic merging](http://hintjens.com/blog:106) within that repo.

Unity has a long history of buying out important ecosystem tools,
which are then often left to rot.
By bringing on the existing maintainer team and allowing for direct community contributions we can hopefully avoid this fate.

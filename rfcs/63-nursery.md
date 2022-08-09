# Feature Name: `bevy-nursery`

## Summary

Work done within the [broader Bevy ecosystem](https://bevyengine.org/assets/) is sometimes a good fit as part of the core engine.
These crates should live in a **nursery** owned by the Bevy org for a period of time,
where they can be polished and refined by the broader community,
before eventually being merged in completely.

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
   3. Nursery crates should never expand the planned scope; the choice is between upstreaming and rewriting, not upstreaming vs. leaving in the third party ecosystem forever.
2. The crate is reasonably mature.
3. The crate is well-loved.
4. The crate follows a high standard for testing, code quality and documentation.

Very small or uncontroversial crates may be suitable for a direct PR via the ordinary mechanism.

### Setting up a nursery crate

Once accepted, the repository will be transferred to the Bevy organization.
The name, version and `crates.io` identity may change at this time on a case-by-case basis.

There are three strategies here:

1. Keep the existing crate name and versioning, then synchronize at the final step.
   1. Avoids breaking end users, very simple.
   2. Well-suited to crates that are likely to be absorbed into other existing `bevy` crates during the integration process to avoid any renaming at all.
2. Change to a temporary `bevy_nursery_*` name, retaining the original versioning and then synchronize at the final step.
   1. Ensures consistent branding and communicates intent to users.
   2. Allows for breaking changes without being synchronized with Bevy's own release schedule.
3. Change directly to the final name and synchronize with Bevy's latest version.
   1. This will limit breaking changes to Bevy's release schedule.
   2. Avoids another rename.
   3. Should only be used for extremely mature crates.

In this repo, the existing crate maintainers will have complete admin permissions over the new crate,
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
- [ ] do the decisions that the crate has made align with [Bevy's general philosophy](https://github.com/bevyengine/bevy/blob/main/CONTRIBUTING.md#what-were-trying-to-build)?
- [ ] if we were to start from scratch, what would we have changed?
  - [ ] can we get there without requiring a full rewrite?
- [ ] are there any dependencies to third-party plugins?
- [ ] can we reduce our dependency tree by standardizing with `bevy` itself or rewriting small dependencies?
- [ ] does the crate contain any features that we have explicitly relegated to the third-party ecosystem?
- [ ] can niche functionality be split out into ecosystem crates that builds on the foundation established by the new core engine crate?
- [ ] does the crate need to be renamed / re-versioned as part of the integration process?

Crates **do not have to be complete** to be merged into Bevy itself: merely polished and clearly useful.

The PR to merge the crate into Bevy itself should be very simple, and contain only the bare minimum change set required to enable interoperability.
Further integration work should be done in follow-up PRs.
If you are an interested community member, raise issues and submit fixes on the nursery repository **before** the crate is being upstreamed.

Once the upstreaming PR is merged, the nursery crate's repo will be archived and any still-relevant issues will be migrated to the main Bevy repo.

## Drawbacks

1. Clearly picks winners within the crate ecosystem.
   1. These winners typically already exist, but it's hard for beginners to discover that critical information.
   2. Competing crates may end up being merged as part of the nursery process.
2. Might increase the scope of the core engine.
   1. Nursery crates should never expand the planned scope of the engine; merely avoid reimplementation of planned work.
   2. Feature flags, template games and plugin sets help ensure users only compile and ship the features they care about.
3. Upstreaming additional parts of the engine without adding more maintainers will increase the workload and may slow down progress.
   1. This can be addressed independently.
   2. PR contributors and nursery contributors should be treated equally.
4. The authors of upstreamed crates will lose some creative control over their crates.
   1. They will still be able to make PRs to the core engine, and their opinion will be weighed heavily.
   2. Nursery crate authors may eventually become maintainers in their own right, in no small part due to their work creating and refining the nursery crate.
   3. This is ultimately no worse than a hard fork or reimplementation, which is the direct alternative.

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

## Prior art

The [rust-nursery](https://github.com/rust-lang-nursery) exists for similar reasons, although there is a greater emphasis on stability.

[Druid](https://github.com/linebender/druid) has a similar [widget nursery](https://github.com/linebender/druid-widget-nursery), and explicitly follows a policy of [optimistic merging](http://hintjens.com/blog:106) within that repo.

Unity has a long history of buying out important ecosystem tools,
which are then often left to rot.
By bringing on the existing maintainer team and allowing for direct community contributions we can hopefully avoid this fate.

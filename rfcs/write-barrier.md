# RFC: Write Barriers for ECS Data

- Feature Name: `write_barrier`
- Start Date: 2026-05-20
- RFC PR: TBD
- Bevy Issue: TBD

## TLDR

A deterministic-snapshot bug we shipped traces to "the schedule has no way to say no one else writes `T` between A and B". This RFC adds that primitive, scoped to a single schedule for v1: a static, schedule-build check that errors if any system in a declared window writes a chosen type. The cost is forced ordering edges on every writer of `T` in range; the benefit is a bug class that can't be added by a new contributor who doesn't know about the snapshot. Cross-schedule and cross-subapp barriers are deferred to Future possibilities once the diagnostic and semantics questions are resolved.

## Summary

A schedule-level primitive that locks writes to a chosen component type between a producer point and a consumer point, with a static check at schedule construction that no other system in that window mutates it. Resources are covered by the same API because as of 0.19 they are components on singleton entities.

## Motivation

System ordering already lets you say "A runs before B". What it doesn't let you say is "between A and B, nothing else is allowed to touch T".

In a deterministic multiplayer game this matters. Our P2P state hash is captured at FormationReveal. The hash is queued by an observer on `BoardSeededEvent` and flushed in `PostUpdate` so commander passives have time to apply. Between the queue and the flush, an unrelated system (the auto-player) ran in the same frame and played a card, mutating `RoundState`. The two peers hashed at different logical moments and desynced.

We can patch this by adding `.before(...)` constraints to every current mutator. The problem is the next mutator. Anyone adding a new system in the BoardSeeded-to-PostUpdate window has to know about the hash. Nothing in the type system or the scheduler tells them. The bug is silent until production.

The need is generic. Any time you want to capture a consistent snapshot of data across an arbitrary number of unrelated systems, you have the same problem. Save systems, replay recorders, network synchronization, undo stacks.

## Worked example: the desync that motivated this RFC

This is the bug in code form, drawn from a 1v1 LAN session in a deterministic multiplayer Bevy project (regicide, vendoring bevy 0.18.1 with the observer-ordering branch). It is the smallest faithful reproduction of the production race that exits a session into infinite state-sync recovery and is killed by the hard timeout.

### Before: the race

The phase hash is queued by an observer on `BoardSeededEvent` and flushed in `PostUpdate` so commander-passive observers have time to mutate the board first.

```rust
// crates/net/net/src/p2p/hash/phase.rs
fn on_board_seeded_queue_phase_hash(
    _trigger: On<BoardSeededEvent>,
    mut pending: ResMut<PendingBoardSeededHash>,
) {
    pending.phase_hash = true;
}

fn flush_pending_board_seeded_hashes(
    mut pending: ResMut<PendingBoardSeededHash>,
    game_query: Query<(&UnitPositions, &RoundState, &NetworkMode), With<Room>>,
    // ...
) {
    if !pending.phase_hash { return; }
    pending.phase_hash = false;
    send_phase_hash_at(PhaseHashPoint::FormationReveal, /* reads game_query */);
}

app.add_systems(PostUpdate, flush_pending_board_seeded_hashes);
```

The auto-player lives in the same schedule and acts on the first turn:

```rust
fn auto_play_card(
    mut round: ResMut<RoundState>,
    mut hand: ResMut<PlayerHand>,
    // ...
) {
    // Picks the first playable card and applies its effect.
    // Plague's effect mutates RoundState; the FormationReveal hash reads RoundState.
}

app.add_systems(PostUpdate, auto_play_card);
```

On the acting peer (player 0), `auto_play_card` and `flush_pending_board_seeded_hashes` are both in `PostUpdate` with no ordering edge. The executor picks an order; in our crash log the order was *auto-play, then flush*. The non-acting peer has not received the card play yet, so its flush observes pre-card `RoundState`. Both peers report computing "the FormationReveal hash"; the inputs disagree by one mutation; recovery never converges.

The patch today is `.before(FlushPhaseHashSet)` on `auto_play_card`. The patch forever is that no one has to remember.

### After: the barrier

```rust
// crates/net/net/src/p2p/hash/phase.rs
#[derive(SystemSet, Hash, Eq, PartialEq, Debug, Clone)]
struct BoardSeededAnchorSet;

#[derive(SystemSet, Hash, Eq, PartialEq, Debug, Clone)]
struct FlushPhaseHashSet;

app.configure_sets(PostUpdate, (BoardSeededAnchorSet, FlushPhaseHashSet).chain());
app.add_systems(
    PostUpdate,
    flush_pending_board_seeded_hashes.in_set(FlushPhaseHashSet),
);

app.add_barrier::<RoundState>(WriteBarrier::new()
    .open_at(BoardSeededAnchorSet)
    .close_at(FlushPhaseHashSet)
    .schedule(PostUpdate));
```

`auto_play_card` carries no ordering edge against either anchor. Schedule construction fails with the diagnostic shape introduced in *Implementation strategy*:

```text
error[B0801]: write to `RoundState` is locked by a barrier in this window
  --> crates/app/auto_player/src/lib.rs:34:32
   |
34 | fn auto_play_card(mut round: ResMut<RoundState>, ...)
   |                              ^^^^^^^^^^^^^^^^^^ writes RoundState
   |
note: barrier declared here
  --> crates/net/net/src/p2p/hash/phase.rs:212:5
   |
212|     app.add_barrier::<RoundState>(WriteBarrier::new()
   |
   = note: window: from `BoardSeededAnchorSet` to `FlushPhaseHashSet` in `PostUpdate`
   = help: order `auto_play_card` outside the window:
            .add_systems(PostUpdate, auto_play_card.after(FlushPhaseHashSet))
```

The fix is the line the diagnostic suggests:

```rust
app.add_systems(PostUpdate, auto_play_card.after(FlushPhaseHashSet));
```

What changed structurally: the contributor who added `auto_play_card` did not need to know about the phase-hash race, did not need to grep for hash-related sets, and did not need to read the multiplayer crate at all. The schedule refused to build. The race could not ship.

The same shape extends without API change:

- An observer on `BattlePlayCardEvent` that inserts a `RoundState`-affecting marker via `Commands` declares the write with `.may_insert::<RoundState>()` at attach time, and the analyzer treats it as a writer for window analysis.
- A `commands.delayed().secs(0.5).insert(round_state_patch)` call is treated as a frame-agnostic writer per *Writes through `Commands`*, so it must be ordered outside every `RoundState` barrier window.
- A second, unrelated `BattleStart` hash with its own anchor set composes by intersection (see *Multiple barriers on the same type*): a writer must be outside both windows.

The diagnostic is the load-bearing artifact. Everything else in the RFC exists to make the schedule emit it on the right system.

## User-facing explanation

**v1 scope: single schedule.** A barrier is declared in one schedule and analyzes only systems in that schedule. Cross-schedule and cross-subapp variants are deferred to Future possibilities.

You declare a barrier on a type, anchor it to a producer set and a consumer set, and add it to a schedule. The example below matches the motivating bug, where `RoundState` is the data that diverged:

```rust
app.add_barrier::<RoundState>(WriteBarrier::new()
    .open_at(BoardSeededSet)
    .close_at(PhaseHashFlushSet)
    .schedule(PostUpdate));
```

While the barrier is open, any system in the same schedule that has `Query<&mut RoundState>`, `ResMut<RoundState>`, or modifies that data via `Commands` (see below) fails schedule construction with an error pointing at the offending system and the barrier that excludes it.

Reads are fine. The barrier is one-directional: it blocks writes, not reads.

The barrier closes when its `close_at` set finishes. After that, the schedule resumes normally.

Open and close are set labels, not system labels. The barrier anchors at the set's logical position derived from its `.before`/`.after` edges, not at any specific member system. Empty sets are valid if they participate in at least one ordering edge that places them in the graph. An empty set with no ordering edges fails barrier construction with a "cannot resolve anchor position" error.

The same `WriteBarrier` API covers components and resources; the analyzer dispatches on the access kind. Since 0.19, resources are components on singleton entities, so this is no longer a separate code path internally.

If a barrier is declared but its open and close points aren't reachable from one another in the schedule, that's a schedule build error too.

### What counts as "in the window" under parallel execution

Bevy's executor schedules systems in parallel when no ordering edge or data-access conflict forces serialization. A system that has no ordering relation to either anchor *could* run inside the window on one frame and outside it on another. To deliver a real safety guarantee, write barriers take the conservative reading: any system that is not strictly ordered after `close_at`, or strictly ordered before `open_at`, is treated as in the window.

A system gated by a run condition counts as in-window regardless of whether the condition would fire on a given frame. Run conditions resolve at runtime, after schedule build, so the analyzer must assume the system can run. Conditional opt-out is not a way to silence the check.

The practical consequence: declaring a write barrier on `T` forces every writer of `T` in the same schedule to acquire an explicit ordering edge that places it outside the window. This is a real scheduling cost. It is the cost of the safety claim. Authors should weigh it against the alternatives.

### Writes through `Commands`

The hard case. Observers and deferred-mutation systems modify data via `Commands::insert`, `Commands::remove`, `Commands::spawn`, and entity commands. These writes apply during command-queue drain and are invisible to the standard access analyzer.

The default story is a *write manifest*. Systems that may write `T` through commands declare it on the system builder, the same way exclusive access is declared today:

```rust
app.add_systems(PostUpdate, my_observer.may_insert::<RoundState>());
```

The schedule builder treats a declared command write the same as a direct `Query<&mut RoundState>` access for the purposes of barrier analysis. Undeclared command writes that hit `T` are an error in strict mode and a warning in lax mode. This RFC recommends strict as the default with lax for prototyping (lax is permissive about undeclared command writes, so the original race can ship: prototype with lax, release with strict), but the final default is open and tracked in Unresolved questions.

`Commands::spawn` is treated as a write because it changes the set of `T` instances visible to consumers that snapshot the whole population. The manifest can suppress this for consumers that only read a fixed entity set, via a builder method that narrows the manifest to `insert`/`remove` on declared entities. Default-pessimistic for safety.

Observers attached to events are handled the same way: an observer that may write `T` through commands declares it at attach time, and the barrier analyzer treats the observer's effective access set as if it were a system in the schedule.

User `SystemBuffer` impls (post-0.19, where `queue()` over `DeferredWorld` replaces `apply()`, PR #22832) declare their writes via the same `.may_insert::<T>()` builder on the system that owns the buffer. The buffer's effective access is folded into the owning system's manifest.

**Delayed commands** (Bevy 0.19, e.g. `commands.delayed().secs(1.0).spawn(T)` or `commands.delayed().secs(1.0).entity(e).insert(T)`, PR #23090) are a wrinkle: the write applies on a future frame whose schedule position is not the issuing system's. The manifest treats a system that issues delayed writes to `T` as a writer on every frame, since the analyzer cannot know which frame the delayed apply will land on. This is conservative-to-a-fault: a system that occasionally schedules a delayed write to `T` whose delay rules out collision in practice still has to be ordered outside every barrier window for `T`. Per-call-site temporal analysis is out of scope for v1; authors who need finer control can lift the delayed payload into an event and write `T` from an explicit, declared observer.

This is the actual story for catching the motivating bug. Blanket-blocking the command queue drain in the window was considered and rejected: it would freeze every unrelated `Commands::insert` in the window, breaking systems that have nothing to do with `T`.

### Multiple barriers on the same type

Multiple barriers on the same type compose. Phrased as permitted positions for a writer of `T`, the per-barrier constraints intersect: the writer must be outside *every* declared window. Phrased as forbidden regions, the windows union: any frame position covered by at least one barrier is forbidden. Both phrasings describe the same set; the document uses "intersection of permitted positions" as the canonical framing.

This is what users will reach for when, for example, snapshot and network protocols both want a coherent view of the same data on different anchors.

## Implementation strategy

A barrier is metadata on the schedule graph. The schedule builder already walks systems and computes data access through `FilteredAccess`, and 0.19 expanded `Access::archetypal` to return a structured `&ComponentIdSet` (PR #23384). That is the substrate the analyzer reads from. The change is:

1. Track barriers as `(locked_component, open_set, close_set, schedule)` tuples. Resolve set positions via the sets' existing ordering edges.
2. For every system in the schedule, decide its position relative to the barrier. Conservatively: any system not strictly ordered after the close anchor and not strictly ordered before the open anchor is "potentially in window".
3. For each potentially-in-window system, intersect its `FilteredAccess` and its declared command-write manifest with the barrier's locked component. If the intersection includes write access, error.
4. Multiple barriers per type compose by intersection of permitted writer positions (see above).
5. The check runs once at schedule build, so the runtime cost is zero.

The error is structured and includes: the locked type, the violating system, the barrier site (file:line of `add_barrier`), the open and close anchors, the schedule, and a suggested fix (`.after(BarrierClose)` or `.before(BarrierOpen)`). A representative diagnostic:

```text
error[B0801]: write to `RoundState` is locked by a barrier in this window
  --> src/auto_player.rs:142:36
   |
142| fn auto_play_card(mut round: ResMut<RoundState>, ...)
   |                              ^^^^^^^^^^^^^^^^^^ writes RoundState
   |
note: barrier declared here
  --> src/p2p/hash/phase.rs:54:5
   |
 54|     app.add_barrier::<RoundState>(WriteBarrier::new()
   |
   = note: window: from `BoardSeededSet` to `PhaseHashFlushSet` in `PostUpdate`
   = help: order `auto_play_card` outside the window:
            .add_systems(PostUpdate, auto_play_card.after(PhaseHashFlushSet))
```

`B0801` is illustrative; the final error-code scheme follows whatever Bevy's diagnostic conventions land on (see #15036). The shape (locked type, violating site, barrier site, window, suggested fix) is the load-bearing part.

Resources do not need a separate code path. Since 0.19 (PRs #20934 and #22910, the latter removed resources from `Access` entirely), the analyzer treats `ResMut<T>` the same as a write to the singleton-entity component `T`. The implementation depends on the full chain #20934 / #22910 / #22911 / #22919 / #22930 / #23616 / #23716 / #24077 / #24164 having landed; that chain is fully merged in 0.19 as of 2026-05-08. If a future regression splits the path, the implementation needs a transitional `ResMut` route, but the public API stays unchanged.

The existing `bevy_ecs::schedule::tests` infrastructure can be extended with barrier-violation test cases covering: writers in window, command-mediated writes via the manifest, undeclared command writes (strict vs lax), conditionally-running writers, multiple barriers composing on one type, and the empty-set "cannot resolve anchor position" error. Each case asserts the expected `Result` from `App::build` (or panic, depending on the resolution of the violation-reporting unresolved question).

## Drawbacks

It's another concept. Bevy's schedule already has system sets, ordering, run conditions, ambiguity detection, and exclusive systems. Adding barriers grows the surface.

Some users will reach for barriers instead of just ordering, when ordering would do. The docs need to be honest about when each is right.

Because the "in window" check is conservative, declaring a barrier can force ordering edges that the executor would otherwise have skipped. Heavy barrier use on hot types can flatten the schedule's parallelism.

The error at schedule build is at a distance from the bug. The user adds a system somewhere, then a different file fails to build a schedule. We mitigate with structured diagnostics that name the barrier site and suggest the missing edge, same shape as the existing ambiguity diagnostic.

Command-write manifests rely on authors declaring their writes honestly. An undeclared command write to `T` slips through if strict mode is off. This is a real correctness gap, the same shape as `unsafe` opting out of borrow checks, and it's the price of not blanket-blocking `apply_deferred`.

Barriers are a public API contract. A barrier declared in plugin A on type `T` forces every downstream plugin that writes `T` in the same schedule to acquire an ordering edge against A's anchors. Plugin authors should think of `add_barrier::<T>` the way they think of exposing a public set label: it's a stable surface that downstream consumers depend on, and changing the anchors is a breaking change. This is the real social cost of barriers in a plugin ecosystem.

Late barrier registration is constrained. The shipped behavior depends on what Bevy supports at the time: today, Bevy does not have first-class partial schedule rebuilds, so the v1 contract is "barriers must be registered before the first schedule run, and registering one afterward panics". If a future PR adds first-class partial rebuild support (this is a real prerequisite, not assumed available), the contract can relax to "post-build registration triggers a rebuild on the affected schedules". Plugins should register barriers in their `Plugin::build` either way. Naming the rebuild dependency keeps it from being silently pulled into the scope of this RFC.

## Rationale and alternatives

**Just use system ordering.** Works for the systems you know about. Doesn't compose. Every new mutator needs a `.before(PhaseHashFlush)` that nobody remembers to add. That's how the bug got shipped.

**Use an exclusive system to flush the hash.** Too coarse. Exclusive systems take `&mut World`, which serializes everything. Barriers are per-type.

**Snapshot the data at the open point and read the snapshot at the close point.** What the application code can already do today, and what we'll do as a stopgap. It works but it's manual, easy to forget, and you pay for the clone every time. A barrier lets you keep using the live data.

**Use change detection.** `Changed<T>` lets a consumer check whether `T` was mutated since a marker tick. This is post-hoc detection: you find out after the race happened, on the next frame. Barriers are pre-hoc prevention: the offending system can't be scheduled in the window in the first place.

**Push state into a separate "frozen" world.** Heavy. We don't need an entire shadow world to express "no writes to this type for the next 5 systems".

**Make it a runtime check.** Wrap the locked type and panic on writes during the window. Simpler to implement but you only find out at runtime, and the panic is per-frame.

## Prior art

- Rust's borrow checker is the obvious shape: exclusive write access for a region of code. Barriers extend that to a region of a schedule.
- Specs had `BarrierWrite` and `Barrier` for similar reasons in a different ECS shape.
- Database transactions and `BEGIN; ... COMMIT;` blocks are the same idea at runtime.
- Bevy's existing ambiguity detection is a close cousin: it already walks the schedule and reports unordered systems that touch the same data. Barriers are stricter and intentional rather than diagnostic.

## Related Bevy work

Things already proposed, shipped, or in flight that overlap with this RFC. We should not duplicate them, and the API should compose with them where it makes sense.

### Open RFCs

- [bevyengine/rfcs#46 Atomic schedule groups](https://github.com/bevyengine/rfcs/pull/46). Ensures a group of systems is evaluated "together" so that other parallel systems can't break the group's internal state mid-execution. Closest sibling. Difference: atomic groups protect the group's state during its own execution, while write barriers protect a chosen type during a window defined by two outside anchor points, against any writer anywhere in the schedule. Both could coexist.
- [bevyengine/rfcs#16 Bevy Subworld RFC](https://github.com/bevyengine/rfcs/pull/16). The "shadow world" alternative listed above. Heavier than barriers but solves a wider problem (full isolation, not just one type).
- [bevyengine/rfcs#36 Encoding Side Effect Lifecycles as Subgraphs](https://github.com/bevyengine/rfcs/pull/36). Different angle on the same underlying problem: making the points where side effects become visible explicit in the schedule graph.
- [bevyengine/rfcs#47 Relaxed (if-needed) ordering constraints](https://github.com/bevyengine/rfcs/pull/47). The complement of barriers. If-needed ordering relaxes constraints that have no real effect. Write barriers tighten them when ordering alone isn't enough.

### Shipping or shipped in 0.19

These changed the substrate the RFC builds on and should be acknowledged in any review.

- [bevyengine/bevy#20934 + follow-ups Resources as Components](https://github.com/bevyengine/bevy/pull/20934). Resources are now components on singleton entities. Eliminates the "component vs resource dispatch" concern and lets the barrier API use one code path.
- [bevyengine/bevy#22144 Render Graph as Systems](https://github.com/bevyengine/bevy/pull/22144). Render passes are now systems in `Core3d`/`Core2d` schedules. Out of scope for v1 (single-schedule), but the change means a future cross-schedule extension can target render pipelines without a special-case "render graph node" path.
- [bevyengine/bevy#22602 Observer Run Conditions](https://github.com/bevyengine/bevy/pull/22602). Observers now accept run conditions. Strengthens the case for treating conditionally-running writers as in-window (a writer guarded by `run_if` is still a potential writer).
- [bevyengine/bevy#23090 Delayed Commands](https://github.com/bevyengine/bevy/pull/23090). Issuing a delayed write to `T` schedules it for a future frame. Addressed in the commands subsection: a system that issues delayed writes is treated as a writer on every frame.
- [bevyengine/bevy#23384 Expanded `FilteredAccess` data and `Access::archetypal` return type](https://github.com/bevyengine/bevy/pull/23384). The richer access metadata is the analysis substrate the barrier implementation depends on.
- [bevyengine/bevy#23414 `Schedule::set_executor`](https://github.com/bevyengine/bevy/pull/23414). Pluggable executors; barriers must be a property of the schedule graph, not the executor, so any executor honors them.
- [bevyengine/bevy#22832 `SystemBuffer::queue()` over `apply()`](https://github.com/bevyengine/bevy/pull/22832). Any custom `SystemBuffer` is a candidate for declaring command writes; the manifest design must extend to user `SystemBuffer` impls.

### Outstanding issues

- [bevyengine/bevy#4918 Selective command application parallelization](https://github.com/bevyengine/bevy/issues/4918). Directly relevant to the command-write manifest. If commands gain per-type access analysis, barriers can drop the manifest opt-in for declared types.
- [bevyengine/bevy#15036 Replace "system order ambiguity" terminology to make it more clear it's a race condition](https://github.com/bevyengine/bevy/issues/15036). Confirms the underlying problem is widely understood as a race-condition class. The diagnostic shape we want for barrier violations should match whatever lands here.
- [bevyengine/bevy#14951 System ambiguity CI is too verbose](https://github.com/bevyengine/bevy/issues/14951). Cautions us against producing noisy diagnostics. Barrier errors should fire only on real violations, never as ambient warnings.
- [bevyengine/bevy#18310 Being able to visualize the exact schedule order being used](https://github.com/bevyengine/bevy/issues/18310). If schedule visualization lands, barriers should be rendered as a marked region on the graph.
- [bevyengine/bevy#23057 Potential for UB in `AssetChanged` access propagation](https://github.com/bevyengine/bevy/issues/23057). Confirms that derived access (`AssetChanged`-style) can fail to propagate. The barrier manifest needs to declare derived writes; this issue is precedent for why.

## Unresolved questions

### Must answer before merging

These shape the v1 surface and need resolution before implementation lands.

- API name. `WriteBarrier`, `WriteLock`, `MutationWindow`. None are perfect, and "write barrier" already has informal usage in the Bevy community to refer to command-queue drain / sync points. The final name needs to avoid that collision.
- How strict should command-mediated writes be by default. Strict is safe, lax is convenient. Plausible answer: strict in CI, lax in dev builds.
- Whether barrier violations should be `Result`-returning at schedule build for tooling consumers, or panic-only. `Result` is friendlier to editor integrations and `bevy_dev_tools`; panic matches the rest of `App::build`. The choice is hard to reverse after release.

### Future-direction signal

These are not v1 blockers but reviewers may want to see them acknowledged.

- Whether the open and close points should be expressible as run conditions rather than sets, for barriers that gate on data rather than scheduling. Observer run conditions (PR #22602) are the recent precedent for this composition.
- Whether to surface a runtime trace event when a barrier closes, for debug.
- Whether the schedule-build error should suggest an auto-fix (`.after(BarrierClose)`), in a format consumable by `cargo fix` or rust-analyzer code actions. Clippy's machine-applicable hints are the model.

## Future possibilities

- **Cross-schedule barriers.** Open in one schedule, close in another (e.g. `PreUpdate` to `PostUpdate`). The shape is the same as the single-schedule case, but two problems were called out in review and need explicit answers before this lands:
  - Bevy schedules are separate graphs. A system in `Update` cannot `.before(SetInPostUpdate)` directly. The diagnostic for a between-schedules writer needs to suggest something actionable: move the system to a schedule outside the window, wrap the offending set in a system-set that *is* visible across schedules, or use the snapshot alternative for that case. Without that diagnostic story, a between-schedules violation has no in-API fix.
  - Schedule order resolution leans on `MainScheduleOrder` for built-in schedules. User schedules outside that order need a declared position or the analyzer must refuse to analyze. v1 should pick.
- **Cross-subapp barriers.** Anchor in one sub-app, close in another (canonical case: main world to `ExtractSchedule`). Depends on cross-schedule. Open question from review: does `across_subapps` mirror the lock across both worlds (so `T` in render world is also locked), or only lock the open-side world? The mirroring semantics drive whether the explicit form is useful for the motivating Extract case or whether a dedicated `.close_at_extract::<T>()` helper is the right shape.
- Read barriers. "No reads of T are allowed between A and B." Useful for systems that need to invalidate cached state.
- Cross-frame barriers, paired with frame markers. Useful for replay systems that want to capture a consistent multi-frame window.
- Per-archetype barriers. Lock writes to T but only on entities with marker C. Probably not worth the complexity for v1.
- Dynamic schedule support. Schedules invoked from inside a system (`world.run_schedule(X)`) are not statically analyzable today. A future extension could lift this with declared call sites; only relevant once cross-schedule lands.

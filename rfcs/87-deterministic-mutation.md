Feature Name: `deterministic_mutation_hooks`

# Deterministic Mutation Hooks for ECS

## Summary

This RFC proposes two small [`bevy_ecs`][bevy-ecs] primitives for deterministic mutation
pipelines:

1. [`ObserverSet`](#observerset) and ordered [observer][observer-module] dispatch.

   Observers get the familiar ordering vocabulary that [systems][systemset] already use:
   `.in_set(...)`, `.before(...)`, `.after(...)`, and `.chain()`.

   Ordering is deterministic, works across the different ways observers are
   attached (global, entity, and component) for the same event, and preserves
   the existing observer model: observers are still observers, not scheduled
   systems.

2. [Restricted component access](#restricted-component-access) through
   `#[component(restricted)]` and [`RestrictedMut<T>`](#restrictedmutt).

   A component can opt into restricted mutable access. Once it does, ordinary
   safe in-place mutable ECS access no longer compiles or returns a mutable
   pointer for that component. Authorized in-place mutation goes through
   `RestrictedMut<T>`, which registers the same ECS write access as a normal
   mutable query and therefore preserves Bevy's existing aliasing and
   schedule-conflict rules.

Together, these primitives let libraries build deterministic mutation pipelines
without asking Bevy to own the policy layer. Bevy provides ordering and
safe-access enforcement. Libraries provide logs, replication, save/load, undo,
replay, auditing, or domain-specific validation.

Bevy has two related gaps for users building event-driven deterministic state
pipelines.

First, [observers][observer-module] are useful for colocating domain reactions with [events][event] and
entities, but observer execution order is not currently expressed with the same
scheduling vocabulary as systems. If multiple observers respond to the same
event, users need a deterministic way to say "validate before apply", "apply
before notify", or "all score updates before win checks", without collapsing the
whole pipeline into one observer.

Second, [component hooks][component-hooks] can observe structural changes such as add, insert,
discard, remove, and despawn, but they do not run for every in-place mutation
made through safe mutable ECS access. A downstream library can ask users to
mutate through a mediated API, but ordinary safe mutable access remains an easy
accidental bypass unless the component itself can opt into a restricted write
path.

The RFC deliberately does not propose any built-in `MutationLog`, `SyncBarrier`,
networking system, save/load system, undo system, replay system, audit log, or
replication policy. Those remain library concerns built on top of the two
proposed engine-level hooks.

### Why a single RFC for two primitives

The two primitives are independent: neither depends on the other, and either
could land first. They are presented together because they share one motivation
(making deterministic mutation pipelines practical), are designed to compose,
and are each individually small. Because they are independent, bundling them
imposes no merge-order constraint between implementation PRs. An author who
preferred could split them into two cross-linked standalone RFCs without
changing their substance; they are kept together here because they are easier to
evaluate as two halves of the same problem.

### AI assistance

AI was used to make holistic design decisions.

## Motivation

### Ordered observers

Observers are a natural fit for ECS event reactions:

- They can be attached globally.
- They can be attached to an entity.
- They can target component lifecycle events.
- They run at the point where the event is triggered.
- They let plugins define behavior close to the event they care about.

That shape is useful for mutation pipelines. A game might trigger `DamageEvent`,
then run observers that validate the hit, apply damage, update derived indices,
emit secondary effects, and notify UI or networking code.

The problem is that ordering is part of the semantics of many such pipelines:

```text
validate input
then apply mutation
then update derived state
then emit consequences
```

For [systems][systemset], Bevy already has a well-understood vocabulary for this:
`.before(...)`, `.after(...)`, `.in_set(...)`, and `.chain()`. Users should not
need to learn a separate ordering model for observers.

### Restricted in-place mutation

Component hooks are good at structural lifecycle events:

```text
add
insert
discard
remove
despawn
```

They are not a complete answer for mediated mutation, because ordinary in-place
mutation does not necessarily pass through a domain-specific API:

```rust
fn accidental_bypass(mut query: Query<&mut AccountBalance>) {
    for mut balance in &mut query {
        balance.cents += 100;
    }
}
```

For many Bevy applications this is fine. Direct mutable access is one of the main
strengths of ECS.

For some components, however, the component owner wants stronger discipline:

- A save system may need to mark data dirty whenever it is edited.
- An audit system may need all balance changes to pass through a checked path.
- A replay system may need to record semantic operations.
- A deterministic networking crate may need domain state mutations to occur at
  known points and through known APIs.
- A debugging tool may need time-travel checkpoints around specific mutations.

Those policies are too application-specific for Bevy to own. But Bevy can provide
the type-level enforcement hook that prevents the most common accidental bypass:
ordinary safe in-place mutable ECS access.

These requirements come from building a deterministic peer-to-peer replication
layer, where every peer must converge on identical state. Each peer must apply
the same event reactions in the same order, and every mutation to replicated
state must pass through the layer's capture path. A single observer running in an
undefined order, or one stray `Query<&mut T>` editing replicated state outside
that path, silently diverges peers — and a wrapper that can only request the
mediated path by convention cannot prove the path was not bypassed. These two
primitives turn both requirements into engine-enforced guarantees rather than
conventions.

### Goals

This RFC aims to provide:

1. Deterministic observer ordering.

   Observers for the same event can be ordered using the same vocabulary as
   systems.

2. Cross-bucket observer ordering.

   A global observer can be ordered relative to an entity observer or component
   observer for the same event.

3. Stable behavior when no explicit observer ordering is supplied.

   Users who do not opt into `ObserverSet` still get a deterministic dispatch
   order (free of hash-map and bucket iteration artifacts) for a fixed
   registration order, rather than order that depends on hash-map or bucket
   iteration internals.

4. A zero-cost path for ordinary components.

   Components that do not opt into restricted access continue to use Bevy's
   existing mutable component path.

5. A safe write gate for restricted components.

   Components that opt into restricted access cannot be mutated through ordinary
   safe in-place mutable ECS access such as [`Query`][query]`<&mut T>`,
   [`Query`][query]`<`[`Mut`][mut]`<T>>`,
   typed world/entity mutable access, or safe untyped mutable access by
   [`ComponentId`][component-id].

6. Preservation of Bevy's existing aliasing and conflict model.

   [`RestrictedMut<T>`](#restrictedmutt) must register the same component write access as `&mut T`
   would have registered. It must not create a side channel around schedule
   conflict detection.

7. A narrow engine surface that libraries can build on.

   Bevy provides ordering and access control. Library policy remains downstream.

### Scope (out of scope)

This RFC does not propose:

- `MutationLog`
- `SyncBarrier`
- a network replication framework
- a save/load framework
- undo or redo
- deterministic replay
- audit policy
- rollback netcode
- a schedule-level write barrier
- cross-schedule mutation windows
- cross-app or sub-app mutation synchronization
- parallel observer execution
- `ambiguous_with` for observers
- a security sandbox
- a capability system that prevents malicious code from mutating state
- a lint that detects every possible semantic mutation

The restricted-access primitive is a safe API discipline tool, not a security
boundary. [Unsafe world access][unsafe-world-cell] remains an escape hatch, as it does for many other
ECS invariants.

## User-facing explanation

### Ordered observer pipeline

A plugin can define observer sets:

```rust
use bevy_ecs::prelude::*;

#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct ValidateMove;

#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct ApplyMove;

#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct EmitConsequences;
```

Then configure their order:

```rust
app.configure_observer_sets((ValidateMove, ApplyMove, EmitConsequences).chain());
```

Observers can be added to those sets:

```rust
app.add_observers((
    validate_move.in_set(ValidateMove),
    apply_move.in_set(ApplyMove),
    update_derived_state.after(ApplyMove).before(EmitConsequences),
    emit_consequences.in_set(EmitConsequences),
));
```

This says:

```text
validate_move
then apply_move
then update_derived_state
then emit_consequences
```

The same ordering vocabulary works at entity observer entry points:

```rust
commands
    .entity(piece)
    .observe(recompute_legal_moves.after(ApplyMove));
```

If `recompute_legal_moves` and `apply_move` both observe the same event, their
relative order is respected even if one is an entity observer and the other is a
global observer.

### Restricted component mutation

A component can opt into restricted mutable access:

```rust
use bevy_ecs::prelude::*;

#[derive(Component)]
#[component(restricted)]
struct Health {
    current: u16,
    max: u16,
}
```

Safe reads are still ordinary reads:

```rust
fn read_health(query: Query<&Health>) {
    for health in &query {
        info!("{} / {}", health.current, health.max);
    }
}
```

Plain in-place mutable access does not compile:

```rust
fn illegal_mutation(mut query: Query<&mut Health>) {
    for mut health in &mut query {
        health.current = health.current.saturating_sub(1);
    }
}
```

The authorized path is `RestrictedMut<T>`:

```rust
fn apply_damage(
    damaged: Query<(Entity, &IncomingDamage)>,
    mut health: RestrictedMut<Health>,
) {
    for (entity, damage) in &damaged {
        let _ = health.modify(entity, |health| {
            health.current = health.current.saturating_sub(damage.amount);
        });
    }
}
```

`RestrictedMut<Health>` registers a normal write access to `Health`. It therefore
conflicts with other readers and writers exactly as `Query<&mut Health>` would.

A downstream library can wrap this further. In the example below `MutationLog` and
`HealthChanged` are illustrative application-owned types, not Bevy APIs:

```rust
fn apply_logged_damage(
    damaged: Query<(Entity, &IncomingDamage)>,
    mut health: RestrictedMut<Health>,
    mut log: ResMut<MutationLog>,
) {
    for (entity, damage) in &damaged {
        let result = health.modify(entity, |health| {
            let before = health.current;
            health.current = health.current.saturating_sub(damage.amount);
            before
        });

        if let Ok(before) = result {
            log.push(HealthChanged {
                entity,
                before,
                after_damage: damage.amount,
            });
        }
    }
}
```

The logging policy is not part of Bevy. The engine-level guarantee is that a
component that opts into restricted access cannot be accidentally edited through
ordinary safe in-place mutable access.

### Composing the two primitives

This RFC is motivated by deterministic mutation pipelines, but the primitives are
independent.

`ObserverSet` solves ordering:

```text
when event E is triggered, run these observers in this declared order
```

Restricted component access solves safe in-place write discipline:

```text
this component cannot be edited through ordinary safe mutable ECS access
```

A downstream library can combine them:

```rust
#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct ValidateCommand;

#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct ApplyCommand;

#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct RecordCommand;

#[derive(Component)]
#[component(restricted)]
struct RoundState {
    turn: u32,
}

app.configure_observer_sets((ValidateCommand, ApplyCommand, RecordCommand).chain());

app.add_observers((
    validate_command.in_set(ValidateCommand),
    apply_command.in_set(ApplyCommand),
    record_command.in_set(RecordCommand),
));
```

The apply observer mutates through `RestrictedMut<RoundState>`:

```rust
fn apply_command(
    commands_to_apply: Query<(Entity, &ValidatedCommand)>,
    mut round_state: RestrictedMut<RoundState>,
) {
    for (entity, command) in &commands_to_apply {
        let _ = round_state.modify(entity, |state| {
            state.turn += command.turn_delta;
        });
    }
}
```

A networking, replay, save, or audit crate can then add its own policy around
that path. This RFC does not define the policy.

### Compatibility and migration

#### Existing components

Existing components are unchanged. A component only participates in restricted
access if it explicitly opts in.

#### Existing observers

Observer dispatch order becomes deterministic even for users who do not define
observer sets. This is a global, non-opt-in behavioral change, so it deserves a
short migration note:

- **What changes.** When several observers respond to the same event, their
  relative order — and therefore the order of their side effects — may differ
  from what a given build happened to produce before.
- **How to pin behavior.** Express the intended order explicitly with
  `.before(...)`, `.after(...)`, or `.chain()`, or group observers with
  `.in_set(...)`.
- **What the baseline actually was.** Today observer order is documented as
  arbitrary and is build- and run-dependent. The new insertion-order tie-break
  therefore does not reproduce any single prior order, and there is no de-facto
  order to preserve: code that relied on the old ordering was already relying on
  an unspecified contract.
- **Communication.** Because the change is global rather than opt-in, it should
  ship with a migration-guide / release-notes entry.

Plugins that own sensitive components may optionally mark those components as
restricted. Doing so is a source-breaking change for downstream code that
currently uses ordinary safe in-place mutable access on those components. That
break is intentional and opt-in.

#### Existing plugins

Plugins that expose observers may optionally add observer sets to document their
extension points.

#### Internal observer APIs

An implementation may need to change internal observer cache shapes and
diagnostic accessors. Public APIs used by ordinary applications should be
preserved where possible. If public observer-inspection APIs change, the
migration path should be documented for debugging tools.

## Implementation strategy

This is the technical portion of the RFC. The two primitives are described
independently, followed by a testing plan.

Prototype and implementation references:

- [Observer ordering implementation][bevy-pr-24328]
- [Restricted component access prototype][bevy-pr-24370]

[`bevy#24370`][bevy-pr-24370] is currently closed and unmerged; the design below describes the
intended behavior rather than asserting a merged implementation. See
[Unresolved questions](#unresolved-questions) for the contribution-policy note
that applies to any reopened or successor implementation PR.

### ObserverSet

#### Public API shape

This RFC adds an `ObserverSet` label trait and derive macro, modeled after
[`SystemSet`][systemset]:

```rust
#[derive(ObserverSet, Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct ApplyDamage;
```

Observer configs support the familiar system-ordering vocabulary:

```rust
app.add_observer(observer.in_set(ApplyDamage));
app.add_observer(observer.before(ApplyDamage));
app.add_observer(observer.after(ApplyDamage));
app.add_observers((a, b, c).chain());
```

Observer sets can be configured directly:

```rust
app.configure_observer_sets((Validate, Apply, Notify).chain());
```

The intended public shape mirrors `SystemSet` where practical:

- `.in_set(set)`
- `.before(target)`
- `.after(target)`
- `.chain()`
- tuple configs
- nested sets
- derived labels
- named observers for diagnostics

The exact internal trait names are implementation details, but the public API
should feel like Bevy's existing system scheduling API.

#### Event-local ordering graph

Observer order is resolved per [event][event] key.

Here "event key" is Bevy's existing [`EventKey`][event-key] type
(`bevy_ecs::event::EventKey`, a newtype over the [`ComponentId`][component-id] that identifies an
event type, looked up via [`World::event_key`][world]). It covers user [`Event`][event] types, the
built-in component-lifecycle events (`Add`, `Insert`, `Discard`, `Remove`,
`Despawn`), and dynamic events. Ordering, the ordering graph, and the
insertion-order tie-break are all scoped to a single `EventKey`; observers are
never ordered across distinct `EventKey`s. An observer registered for several
event types participates in the graph of each one independently.

Each registered `(observer, EventKey)` pair becomes an observer-ordering node.
The node records:

- observer entity
- event key
- target kind
- observer runner
- explicit before/after edges
- observer-set membership
- stable insertion order
- optional diagnostic name

For each event key, Bevy builds one ordering graph containing all observers that
can run for that event. The graph includes global observers, entity observers,
component observers, and entity-component observers.

The graph is topologically sorted. If two observers have no ordering relation,
their relative order is deterministic and stable. The recommended tie-break is
observer insertion order for that event key.

#### Cross-bucket ordering

Observer dispatch currently has different lookup paths depending on how the
observer is attached. Conceptually, those dispatch buckets include:

- global event observers
- entity-targeted observers
- component-targeted lifecycle observers
- entity-component targeted lifecycle observers

This RFC does not remove those buckets. They are still useful lookup indices.

Instead, the event-local ordering graph provides one canonical order across all
buckets for the same event. Dispatch gathers the bucket streams that are active
for the trigger and runs their union according to the event-local order.

This is what enables:

```rust
app.add_observer(global_score_update.in_set(UpdateScore));

commands
    .entity(player)
    .observe(entity_win_check.after(UpdateScore));
```

If both observers are active for the same event, the entity observer runs after
the global observer even though they came from different lookup buckets.

An ordering edge against a target — observer, set, or bucket — that is not active
for a given trigger has no effect for that trigger: the dependent observer still
runs in its sorted position. Edges constrain relative order when both endpoints
run; they never block an observer whose predecessor is inactive. This is the same
model as a `.run_if`-skipped observer: order is a graph property, not a runtime
gate.

#### Set targets

Ordering against a set expands to ordering against all observer nodes in that set
for the same event key.

For example:

```rust
app.add_observer(score.in_set(UpdateScore));
app.add_observer(announce.after(UpdateScore));
```

For an event where `score` observes the event and belongs to `UpdateScore`,
`announce` runs after `score`.

If a set has no matching observers for a particular event key, the edge has no
effect for that event key. More generally — as noted above for cross-bucket
edges — an edge against any target that is not active for a given trigger is a
no-op for that trigger, never a block. This matches the ergonomic expectation
that plugins can configure sets before all observers are registered.

#### `.chain()`

`.chain()` creates consecutive ordering edges across the configured observers or
sets.

```rust
app.add_observers((a, b, c).chain());
```

is equivalent to:

```text
a before b
b before c
```

Tuple-wide modifiers compose with `.chain()`:

```rust
app.add_observers((a, b, c).chain().in_set(MyPipeline));
```

This places all three observers in `MyPipeline` and also preserves the chain
edges among them.

#### Run conditions

Observer run conditions do not change the ordering graph.

If an observer is skipped by `.run_if(...)`, dispatch continues to later observers
in the sorted order. A skipped observer does not block its neighbors and does not
remove ordering edges from the graph.

This is the same mental model users already have for scheduled systems: order is
a graph property, while run conditions are runtime execution filters.

#### Propagation

This RFC does not change traversal semantics.

If an event propagates across multiple entities or relationship hops, the traversal
still decides hop order. Within each hop, active observers are dispatched according
to the observer order for that event key.

In other words:

```text
Traversal owns which targets are visited and in what target order.
ObserverSet owns the observer order at each visited target.
```

#### Cycles

Observer-ordering cycles must be diagnosed.

For example:

```rust
app.add_observer(a.in_set(A).before(B));
app.add_observer(b.in_set(B).before(A));
```

creates a cycle: `a` is ordered before set `B` (which contains `b`), and `b` is
ordered before set `A` (which contains `a`), so the expanded edges `a -> b` and
`b -> a` form the cycle `a -> b -> a`.

The diagnostic should identify:

- the event key, where possible
- the observer names or entities involved
- the observer sets involved
- the conflicting edges

The final error/warning mechanism should follow Bevy's [schedule][schedule] diagnostics
conventions. The exact severity — hard error, schedule-build error, registration
panic, or a diagnostic that preserves a deterministic fallback order (e.g.
per-event-key insertion order) — is left open; see
[Unresolved questions](#unresolved-questions). Whichever is chosen, the invariant
is that a cycle is never silently accepted and dispatch never becomes
nondeterministic.

#### Determinism without explicit ordering

Even if users never call `.in_set(...)`, `.before(...)`, `.after(...)`, or
`.chain()`, observer dispatch order should be deterministic.

Concretely, determinism here means: for a fixed set of observers registered in a
fixed order, dispatch order is identical across runs and does not depend on
hash-map or bucket iteration. The tie-break for otherwise-unrelated observers is
the per-event-key order in which they were registered. This is not a canonical
order independent of how the app is assembled: if plugins register observers in a
different order, the tie-break among unrelated observers changes accordingly.
Users who need a stronger guarantee should express it with `.before(...)`,
`.after(...)`, or `.in_set(...)`.

This does not mean Bevy promises a semantic ordering between unrelated observers.
It means the order should not depend on hash-map iteration artifacts or other
unstable implementation details.

#### Performance expectations

The design should preserve the existing fast paths as much as possible:

- Build or rebuild the ordering graph when observer registration or observer-set
  configuration changes.
- Store a sorted order per event key.
- Keep bucket indices for efficient trigger lookup.
- Avoid materializing a merged observer list on the common single-stream and
  global+entity dispatch paths; the multi-component lifecycle path may pre-merge
  its variable per-component buckets into a small fixed set of streams before the
  ordered walk.
- Preserve the existing archetype-flag fast skip for lifecycle component
  observers.

A good implementation strategy is:

1. Store a dense node table per event key.
2. Store one topologically sorted `order` per event key.
3. Store bucket indices as sorted node-id slices.
4. At dispatch, pass active bucket streams to a shared `run_ordered` helper.
5. For one active stream, use a direct sorted slice walk.
6. For lifecycle/component events that fire for several components at once, the
   active per-component buckets are a variable-length set of streams. Because the
   ordered-walk helper takes a fixed number of streams, first collapse the
   per-component buckets into a small fixed number of pre-merged streams (for
   example, one component-global stream and one entity-component stream) using a
   sorted merge of already-sorted slices — not a full re-sort — then run the
   ordered walk over those plus the global and entity streams.
7. For multiple active streams, walk the event-local order and run nodes present
   in the active streams.

The exact data structures are implementation details, but the observable behavior
must be deterministic and must respect cross-bucket edges.

#### Implementation plan

A plausible implementation plan:

1. Add `ObserverSet` derive support alongside `SystemSet`.

2. Add observer config types mirroring system configs:
   - `IntoObserverConfigs`
   - `IntoObserverSetConfigs`
   - ordering target conversion traits
   - tuple support
   - `.in_set`
   - `.before`
   - `.after`
   - `.chain`

3. Store observer-set identities as interned labels.

4. Replace per-bucket ordering with an event-local node table:

   ```text
   EventKey -> CachedObservers {
       nodes: Vec<ObserverNode>,
       order: Vec<NodeId>,
       global_index: Vec<NodeId>,
       entity_index: EntityHashMap<Entity, Vec<NodeId>>,
       component_index: ...
   }
   ```

5. Build ordering edges from:
   - direct observer-before-observer constraints
   - observer-before-set constraints
   - observer-after-set constraints
   - set-before-set constraints
   - tuple `.chain()` constraints
   - nested set membership

6. Topologically sort once per event key after registration/configuration changes.

7. Preserve stable insertion order as the tie-break for unrelated nodes.

8. Dispatch through a shared ordered runner.

9. Keep diagnostic accessors for tests and debugging, but document if they allocate
   and should not be used in hot paths.

### Restricted component access

#### Public API shape

This RFC recommends adding restricted access as a component derive attribute:

```rust
#[derive(Component)]
#[component(restricted)]
struct AccountBalance {
    cents: i64,
}
```

This spelling keeps [component mutability][component-mutability] policy under
the existing [`Component`][component] derive, alongside
`#[component(immutable)]`.

The already-prototyped implementation uses a dedicated derive:

```rust
#[derive(RestrictedAccess)]
struct AccountBalance {
    cents: i64,
}
```

That spelling is concise, but this RFC recommends `#[component(restricted)]` as
the canonical API before stabilization. The semantic requirement is the same in
either spelling:

```text
AccountBalance is a component.
AccountBalance is mutable.
AccountBalance cannot be accessed through ordinary safe in-place mutable ECS APIs.
AccountBalance can be accessed through RestrictedMut<AccountBalance>.
```

The implementation may still expose a marker trait for bounds:

```rust
pub trait RestrictedAccess: Component<Mutability = RestrictedMutable> {
    // Marker trait. The `Mutability = RestrictedMutable` bound is load-bearing:
    // it is what makes `&mut T` and friends (bounded on `Mutability = Mutable`)
    // refuse to apply to restricted components, and what makes the
    // immutable + restricted combination unrepresentable.
}
```

#### `RestrictedMut<T>`

`RestrictedMut<T>` is the authorized safe in-place mutation path:

```rust
pub struct RestrictedMut<'w, 's, T: RestrictedAccess> {
    // implementation detail
}
```

Its v1 API should make closure-style mutation the primary path, with two
direct-handle forms (`get_mut` for a single entity, `iter_mut` for a batch) that
hand out raw `Mut<T>`:

```rust
impl<T: RestrictedAccess> RestrictedMut<'_, '_, T> {
    pub fn modify<R>(
        &mut self,
        entity: Entity,
        f: impl FnOnce(&mut T) -> R,
    ) -> Result<R, QueryEntityError>;

    pub fn get_mut(&mut self, entity: Entity) -> Result<Mut<'_, T>, QueryEntityError>;

    pub fn iter_mut(&mut self) -> impl Iterator<Item = Mut<'_, T>> + '_;
}
```

`modify` imposes the closure shape, which gives downstream crates a natural point
to wrap mutations with logging, validation, or domain events. `get_mut` and
`iter_mut` are the lower-level direct-handle forms inside the restricted safe API.
Examples and docs should lead with `modify`, because the closure boundary is the
natural place for policy; `get_mut` and `iter_mut` exist for parity with Bevy's
existing ECS ergonomics, but they weaken the teaching story if they are the first
APIs users see.

Bevy itself does not attach logging, validation, undo, save, or network policy to
any of these methods.

#### Enforcement model

For a restricted [component][component] `T`, ordinary safe in-place mutable ECS access must be
rejected or withheld.

The v1 implementation should gate these typed query-data paths:

- [`Query`][query]`<&mut T>`
- [`Query`][query]`<`[`Mut`][mut]`<T>>`
- derived [`QueryData`][query-data] structs or tuples containing `&mut T` or [`Mut<T>`][mut]
- `Option<&mut T>`, `Option<Mut<T>>`, `AnyOf<...>`, and other query-data wrappers
  whose mutability is derived from their subqueries
- contiguous mutable query access for `&mut T` and `Mut<T>`
- [query lens][query-lens] or transmute APIs that would produce `&mut T` or [`Mut<T>`][mut] for a
  restricted component

The current implementation strategy is to make ordinary mutable query data require:

```rust
T: Component<Mutability = Mutable>
```

Restricted components use a different mutability marker, so these query-data impls
do not apply.

The v1 implementation should also gate these typed safe [world][world] and
[entity][entity-mut] paths:

- [`World::get_mut<T>`][world]
- [`DeferredWorld::get_mut<T>`][deferred-world]
- [`EntityMut::get_mut<T>`][entity-mut] and `EntityMut::into_mut<T>`
- [`EntityWorldMut::get_mut<T>`][entity-world-mut] and `EntityWorldMut::into_mut<T>`
- [`FilteredEntityMut::get_mut<T>`][filtered-entity-mut] and `FilteredEntityMut::into_mut<T>`
- `ComponentEntry<T>::and_modify`
- `OccupiedComponentEntry<T>::get_mut` and `OccupiedComponentEntry<T>::into_mut`

These are already naturally expressible by requiring:

```rust
T: Component<Mutability = Mutable>
```

The v1 implementation must additionally handle safe untyped mutable access by
[`ComponentId`][component-id]. This is the easiest accidental dynamic bypass if it only checks
that the component is "mutable" in the broad sense. The implementation should
reject restricted components from all `DynamicComponentFetch`-based by-id mutable
accessors, including:

- `World::get_mut_by_id`
- `DeferredWorld::get_mut_by_id`
- `EntityMut::get_mut_by_id`
- `EntityWorldMut::get_mut_by_id` and `EntityWorldMut::into_mut_by_id`
- `FilteredEntityMut::get_mut_by_id`
- [`EntityMutExcept::get_mut_by_id`][entity-mut-except]
- the `DynamicComponentFetch` paths used by those APIs

([`EntityMutExcept::get_mut_by_id`][entity-mut-except] is the one type-erased mutable surface in the
`*Except` family that is not type-gated; its typed `get_mut<C>` already is.
[`EntityRefExcept`][entity-ref-except] is read-only and needs no gating.)

The implementation can do this by exposing component metadata that distinguishes
mutability states (see [Type-erased and dynamic access](#type-erased-and-dynamic-access))
and by treating restricted mutable components as unavailable to ordinary safe
untyped mutation APIs.

The v1 implementation must also gate safe reflection-based mutation. The
[`ReflectComponent`][reflect-component] mutation methods — `reflect_mut`, `apply`,
`apply_or_insert_mapped`, and `reflect_unchecked_mut` — currently gate only on the
value-level `C::Mutability::MUTABLE` const and then call
`get_mut_assume_mutable::<C>()`. Because a restricted component is still mutable
(`MUTABLE == true`), reflection is a fully safe mutable path that bypasses
`RestrictedMut<T>` entirely. These methods must therefore additionally consult the
restricted-mutability marker and reject restricted components, consistent with how
they already reject immutable components. A dedicated restricted reflection API can
be added later if reflection-driven tooling has a clear use case; see
[Reflection and scenes](#reflection-and-scenes) for the scene-loading implication.

[Unsafe APIs][unsafe-world-cell] remain unsafe escape hatches.

#### Access registration

`RestrictedMut<T>` must register a write access to `T`.

This is essential. Restricted access must not become a way to bypass Bevy's
existing aliasing model.

The following should conflict exactly as ordinary mutable access would:

```rust
fn conflict(
    read: Query<&Health>,
    mut write: RestrictedMut<Health>,
) {
    // This system has both read and write access to Health.
}
```

The [scheduler][schedule] and [system-param][system-param] validation should treat `RestrictedMut<T>` like a
normal write to `T`.

#### Reads

Restricted access only affects in-place mutable access.

These remain allowed:

```rust
Query<&T>
Query<Ref<T>>
Has<T>
With<T>
Without<T>
Changed<T>
Added<T>
```

Read-only access should not require the restricted path.

#### Change detection

`RestrictedMut<T>` should preserve Bevy's [change-detection][change-detection] behavior.

Mutating through `RestrictedMut<T>` should update the same component [ticks][tick] that
would be updated by ordinary `Mut<T>` access. APIs that expose changed-by caller
information should continue to work where Bevy supports that information.

Note a nuance worth documenting: `modify` passes `&mut T` to its closure
unconditionally, so it marks the component changed even for a no-op closure —
identical to dereferencing `Mut<T>` mutably. `get_mut` and `iter_mut` only mark
changed when the returned `Mut<T>` is actually dereferenced mutably. Downstream
policy that wants change-only-on-real-mutation should compare before/after itself.

#### Structural changes

This RFC does not gate all ways that a component's value can change.

Structural operations remain in the existing [component lifecycle][lifecycle] model:

```text
spawn
insert
discard
remove
despawn
take
```

A restricted component may be a [required component][required-components] (`#[require]`). The
require-insert path constructs and inserts the value structurally, so it is
governed by the structural-change model and is not subject to the in-place
mutation gate — the same as any other insert.

Typed and untyped `modify_component` APIs also remain structural in this RFC's
model because they temporarily remove the component, run lifecycle hooks, and
reinsert or discard through the existing component-hook path.

This distinction is intentional:

- restricted access gates ordinary safe in-place mutation;
- component hooks observe structural mutation;
- library policy can combine both.

A future RFC could propose stronger restrictions around insert, discard, or
structural `modify_component` APIs if a clear use case and ergonomic design
emerges. That is not part of this RFC.

#### Relationship and immutable components

A restricted component must still be mutable through the restricted path.

Therefore, a component should not be both:

```text
restricted
immutable
```

The `RestrictedAccess: Component<Mutability = RestrictedMutable>` bound makes this
combination unrepresentable through the recommended derive: a component has a
single `Mutability` marker, so it cannot be both `RestrictedMutable` and
`Immutable`. If a chosen implementation spelling allows such a combination
syntactically, it should produce a compile-time error.

Relationship components that are treated as immutable by Bevy should likewise be
excluded unless Bevy later changes their mutability model.

#### Type-erased and dynamic access

[`ComponentInfo`][component-info] already exposes two independent boolean flags rather than three
mutually exclusive states:

```rust
component_info.mutable()           // true for ordinary AND restricted components
component_info.restricted_access() // true only for restricted components
```

A restricted component keeps `mutable() == true`, preserving the existing meaning
of that flag — the component is still mutable; only the in-place mutation path is
constrained. Restriction is an additional, independent flag. There is no separate
`immutable()` accessor; immutable simply means `!mutable()`.

This independence is the crux of the dynamic gate. Safe untyped/dynamic APIs must
reject any component with `restricted_access() == true`; they must not key off
`mutable()` alone, because `mutable()` is also true for restricted components.
Gating with `if mutable()` would let restricted components straight through — the
exact bypass this section guards against. Names are illustrative; the requirement
is that safe dynamic mutable APIs can distinguish restricted components instead of
gating only on `mutable()`.

Safe dynamic access that currently returns [`MutUntyped`][mut-untyped] should either:

1. reject restricted components, or
2. be paired with an explicit restricted dynamic mutation API.

This RFC recommends rejection for v1. A dedicated restricted dynamic API can be
added later if reflection-driven tooling has a clear use case.

#### Reflection and scenes

Reflection deserves an explicit note because it is the substrate for scene
loading and editor inspectors.

The typed `ReflectComponent` mutation methods route through the typed
assume-mutable escape hatch (`get_mut_assume_mutable`) and gate only on
`Mutability::MUTABLE`, which is `true` for restricted components. Per the
[enforcement model](#enforcement-model), v1 rejects restricted components from
these methods, so a restricted component cannot be silently mutated through safe
reflection. The manual dynamic accessor [`World::get_reflect_mut`][world-reflect] routes through
`get_mut_by_id` / [`MutUntyped`][mut-untyped] and is therefore already covered by the untyped
by-id rejection rule.

The practical consequence is that, for v1, a restricted component is not freely
mutable through `ReflectComponent`, and tooling that needs to edit one must go
through `RestrictedMut<T>` or a future dedicated restricted reflection API.
Ordinary reflection of _non-restricted_ components — including scene load/save
round-trips — is unaffected.

#### Generic systems

A generic system using ordinary mutable access:

```rust
fn generic_mut<T: Component>(mut query: Query<&mut T>) {
    // ...
}
```

should not accept restricted components.

A generic system that wants to support restricted components should ask for the
restricted path explicitly:

```rust
fn generic_restricted_mut<T: RestrictedAccess>(mut access: RestrictedMut<T>) {
    // ...
}
```

This is a feature, not a bug: the type parameter communicates which mutation
discipline the system participates in.

#### Implementation plan

A plausible implementation plan:

1. Add a restricted mutability marker beside Bevy's existing mutable and immutable
   component mutability markers.

   Conceptually:

   ```rust
   pub struct RestrictedMutable;
   ```

2. Make `#[component(restricted)]` components use that mutability marker and
   implement `RestrictedAccess`.

3. Ensure ordinary mutable query data is only implemented for ordinary mutable
   components, not restricted mutable components.

   Conceptually:

   ```rust
   impl<T: Component<Mutability = Mutable>> QueryData for &mut T {
       // existing mutable path
   }
   ```

   Restricted components should fail this bound.

4. Add a hidden query-data implementation used by `RestrictedMut<T>`.

   Conceptually:

   ```rust
   #[doc(hidden)]
   pub struct RestrictedWrite<T: RestrictedAccess>(PhantomData<T>);
   ```

   It should fetch `Mut<T>` and register write access exactly like `&mut T`.

5. Implement `RestrictedMut<T>` as a `SystemParam` backed by the hidden restricted
   write query.

6. Add `ComponentInfo` mutability metadata that distinguishes restricted mutable
   from ordinary mutable.

7. Gate typed safe in-place mutable access with `T: Component<Mutability = Mutable>`.

8. Gate safe untyped `ComponentId` mutable access, and the `ReflectComponent`
   mutation methods, by checking the component's mutability metadata and rejecting
   restricted components.

9. Add compile-fail tests for unauthorized mutable access.

10. Add examples showing non-networking use cases, such as audit logging and
    dirty save tracking, without upstreaming those policies into Bevy.

### Testing

#### Observer ordering tests

The observer-ordering implementation should include coverage for:

- `ObserverSet` derive.
- Observer set membership.
- `.before(...)` by observer entity.
- `.after(...)` by observer entity.
- `.before(...)` by observer set.
- `.after(...)` by observer set.
- tuple `.chain()`.
- tuple-wide `.in_set(...)`.
- tuple-wide `.before(...)` and `.after(...)`.
- nested observer sets.
- multiple set membership.
- empty set targets.
- cycle diagnostics.
- deterministic tie-breaking for unrelated observers.
- unregistering observers while preserving order among remaining observers.
- observers added after observer-set configuration.
- cross-bucket ordering:
  - global vs entity
  - global vs component
  - entity vs entity-component
- propagation using sorted dispatch per hop.
- dynamic trigger helpers using the ordered dispatch path.
- `.run_if(...)` interaction with ordered observers.
- diagnostic names in cycle reports.

#### Restricted access tests

The restricted-access implementation should include coverage for:

- restricted component attribute compiles.
- restricted components are registered as components.
- restricted components are mutable through `RestrictedMut<T>`.
- `Query<&mut T>` fails to compile for restricted components.
- `Query<Mut<T>>` fails to compile for restricted components.
- ordinary `Query<&T>` compiles for restricted components.
- `Changed<T>` and other read-side filters continue to work.
- `RestrictedMut<T>` registers write access.
- `RestrictedMut<T>` conflicts with simultaneous reads/writes as expected.
- change ticks update through `RestrictedMut<T>`.
- generic `Query<&mut T>` systems reject restricted components.
- typed `World`, `DeferredWorld`, `EntityMut`, `EntityWorldMut`,
  `FilteredEntityMut`, and component-entry mutable accessors reject restricted
  components.
- safe untyped `ComponentId` mutable accessors, including
  `EntityMutExcept::get_mut_by_id`, reject restricted components.
- `ReflectComponent::reflect_mut`, `apply`, `apply_or_insert_mapped`, and
  `reflect_unchecked_mut` reject restricted components.
- unsafe assume-mutable APIs remain unsafe escape hatches.
- immutable and restricted markers cannot be combined.
- relationship components cannot accidentally derive restricted mutable access if
  their mutability model forbids it.
- component metadata reports the restricted marker.
- structural insert/remove/discard/take/modify behavior remains documented and
  tested.

#### Examples

Examples should demonstrate policies without making them engine features:

- `restricted_access_audit.rs`
  - shows an application-owned audit log around `RestrictedMut<T>`
- `restricted_access_save.rs`
  - shows dirty tracking or save extraction through a mediated mutation path

The examples should avoid implying that Bevy owns audit or save semantics.

## Drawbacks

### More ECS surface area

This adds new labels, config traits, derive or attribute support, and a new system
parameter. That increases the size of `bevy_ecs`.

### Observer ordering complexity

Cross-bucket ordering requires a more sophisticated cache than independent bucket
iteration. The implementation must be careful not to regress observer dispatch
performance.

### Restricted access is coarse-grained

The restriction applies at the component type level, not the field level.

If only one field needs mediation, users must either split the component or accept
that all in-place mutation of the component goes through `RestrictedMut<T>`.

### Restricted access is not a security boundary

A determined user can still use unsafe APIs, structural replacement, or
application-specific escape hatches.

The feature is meant to prevent accidental safe API bypasses, not malicious code.

### Generic mutable systems need adaptation

Generic systems written against `Query<&mut T>` will not work for restricted
components. They must either remain ordinary-mutable-only or add a separate
restricted path.

### Diagnostics may be rough initially

Rust trait-bound errors for `Query<&mut RestrictedComponent>` may not be as clear
as a custom diagnostic. The implementation should add docs and compile-fail tests,
and should improve error messages where feasible.

## Rationale and alternatives

The core design choice is to add the smallest engine-level surface that makes
deterministic mutation pipelines enforceable, using vocabulary users already know,
and to leave all policy to libraries. Two questions deserve direct answers before
the alternatives: why this belongs in the engine, and why `ObserverSet` is a
distinct label rather than a reuse of `SystemSet`.

### Why in-engine, not an ecosystem crate

The enforcement cannot be expressed from a downstream crate.

Restricted access is genuinely engine-only. A crate cannot remove or override
`impl QueryData for &mut T` for a chosen component, cannot introduce a new
`ComponentMutability` marker that gates every safe typed and untyped mutable path
(the marker is sealed), and cannot make the existing reflection and dynamic
`ComponentId` paths reject one specific component. These are engine-internal and
only enforceable upstream.

Observer ordering is the weaker engine-only case. A crate could only order within
a single existing dispatch bucket; it could not guarantee deterministic
global/entity/component interleaving for one event, because the buckets are
dispatched by internal Bevy machinery. Even if a narrower observer-ordering surface
could ever live in a crate, the in-engine justification falls back to cohesion: the
feature reuses Bevy's existing `.in_set` / `.before` / `.after` / `.chain`
vocabulary and the existing schedule topological-sort machinery, and it pairs
naturally with restricted access.

The boundary is deliberate: the engine owns the enforcement hook (ordering and
access control); the crate owns policy (logs, deltas, undo, replay, audit). The
impact of not doing this is that downstream determinism crates can only ask, by
convention, that users mutate through a mediated path and never reorder observer
reactions — they cannot prove a path was not bypassed, and a single
`Query<&mut T>` or hash-map-ordered observer silently breaks determinism.

This design is the best in the space because it adds no new mental model: it
reuses the scheduling vocabulary for ordering and the existing component-mutability
machinery for access control, rather than introducing a parallel policy system.

### `ObserverSet` as a distinct trait vs. reusing `SystemSet`

There are three obvious options for the label type: make `ObserverSet` an alias of
`SystemSet`, share a common `ScheduleLabel`-style base, or make it a fully distinct
trait. This RFC chooses the distinct trait.

A label that is valid for observer ordering is not generally valid for system
scheduling and vice versa. Observer order resolves per event key against an
event-local graph; system sets order against a schedule graph with run conditions,
ambiguity detection, and exclusive-access semantics that do not apply to observers.
A distinct type keeps the type system honest about where a label can be used and
prevents meaningless cross-domain constraints such as
`observer.before(some_system_set)`. The cost is a doubled derive and config
surface that closely mirrors the system equivalent. Whether the two label systems
should later be unified behind a shared base is a reasonable community question;
see [Unresolved questions](#unresolved-questions).

### Do nothing

Users can manually combine observers, use registration order, or move logic into
systems.

This works for small examples but does not scale well to plugin ecosystems where
multiple crates contribute observers to the same event pipeline.

It also does nothing for accidental mutable-access bypasses of mediated mutation
paths.

### Use systems instead of observers

Systems already have ordering.

However, observers are useful precisely because they are event-local, can be
entity-targeted, can participate in lifecycle events, and run at trigger sites.
Forcing users to translate observer pipelines into schedules loses those benefits.

### Registration-order-only observers

Bevy could promise that observers run in registration order.

That is deterministic but insufficient. It does not let independent plugins
declare semantic constraints without coordinating registration order globally.

`.before(...)`, `.after(...)`, and `.in_set(...)` are a better fit because they
describe relationships, not incidental insertion order.

### Component hooks only

Component hooks observe structural changes, but they do not cover every in-place
mutation through safe mutable ECS access.

Hooks and restricted access solve different halves of the problem:

```text
hooks: observe structural lifecycle changes
restricted access: prevent accidental direct in-place mutation
```

### Broad schedule-level write barriers

A previous RFC explored a broader schedule-level write barrier
([bevyengine/rfcs#86][rfc-86]). That design tries to say "no
system writes T in this schedule window."

This RFC intentionally does not propose that.

Schedule-level barriers may still be useful, but they are larger and harder to
specify. They raise questions about commands, delayed commands, cross-schedule
windows, diagnostics, and schedule graph analysis. The two primitives in this RFC
are smaller and independently useful.

### Engine-owned mutation logs

Bevy could provide a first-party mutation log.

This RFC rejects that direction for v1. Different users want different semantics:

- full value snapshots
- semantic operations
- compact network deltas
- audit entries
- undo patches
- replay commands
- dirty flags
- save chunks

Bevy should not choose one policy for all of those use cases here.

### Private component fields

Users can make component fields private and expose methods.

This is useful and should continue to be encouraged where appropriate. However,
private fields do not by themselves integrate with ECS access analysis, observer
ordering, dynamic tools, or generic mutation paths. They are also not always
sufficient for components shared across crates.

Restricted access complements Rust visibility; it does not replace it.

### Dedicated `RestrictedAccess` derive

The prototype uses:

```rust
#[derive(RestrictedAccess)]
struct T;
```

This is concise and keeps examples short. The drawback is that it makes restricted
access look like a second component derive rather than a component mutability
choice. Since restricted access is fundamentally a component mutability policy,
this RFC recommends `#[derive(Component)] #[component(restricted)]` instead.

## Prior art

The two primitives are best understood against how today's deterministic-state
crates approximate them, and against access-discipline mechanisms in other engines.

### How current Bevy crates work around the absence of these hooks

- **[`bevy_replicon`][bevy-replicon]** exposes no traits to implement against by design; integration
  happens by resource and queue convention. It can be wrapped, but it cannot
  _enforce_ that a replicated component is only mutated through the replication
  path — an ordinary `Query<&mut T>` bypasses it silently.
- **[Lightyear][lightyear]** is split into `lightyear_link` / `transport` / `replication` /
  `prediction`, demonstrating that several netcode models can share one core. It
  still has no engine-level way to prevent direct mutation or to order the
  observers that react to a replicated change.
- **[`naia-bevy-server`][naia-bevy-server]** bundles its own transport and replication model. Like the
  others, it relies on convention for "mutate through our API".
- **[`bevy_ggrs`][bevy-ggrs]** owns the frame clock through `Save` / `Load` / `AdvanceFrame` and
  demands whole-simulation determinism. It shows the cost of _not_ having a
  scoped enforcement hook: determinism is an all-or-nothing property of the whole
  app rather than an opt-in property of specific components and pipelines.
- **[`bevy_save`][bevy-save]** approximates dirty-tracking and snapshotting but cannot guarantee
  that every edit to a saved component passes through a checked path.

The common lesson is that each of these crates can _ask_ for ordered reactions and
mediated mutation, but none can prove the path was not bypassed, because the two
engine hooks this RFC proposes do not exist yet.

### Access discipline in other engines

- **[Unity DOTS][unity-dots]** annotates system component access as read-only or read-write and
  uses that information for scheduling and safety. This is precedent for
  type-level write discipline that the engine — not user convention — enforces.
- **[Unreal `UPROPERTY(SaveGame)`][unreal-property-specifiers]** and **[Godot `@export`][godot-export]** are field-level
  annotations that mediate which data participates in save/serialization. They are
  a useful analogue, with one important difference: this RFC's restriction is
  type-level, not field-level (see the "Restricted access is coarse-grained"
  drawback). Field-level mediation is a plausible future direction but is out of
  scope here.

These precedents motivate the _shape_ of the feature — engine-enforced write
discipline — without, on their own, motivating the specific API; the motivation
for that comes from the Bevy-specific gaps described above.

## Unresolved questions

### Exact restricted component spelling

This RFC recommends:

```rust
#[derive(Component)]
#[component(restricted)]
struct T;
```

The prior prototype used:

```rust
#[derive(RestrictedAccess)]
struct T;
```

Review should confirm whether the restricted flag belongs under the `Component`
derive before implementation resumes.

### Direct `get_mut` versus closure-first mutation

`RestrictedMut<T>::get_mut(entity)` is maximally ergonomic and closely matches
existing ECS APIs.

`RestrictedMut<T>::modify(entity, |value| ...)` gives downstream code a cleaner
shape for wrapping mutations with logs or validation.

This RFC allows both, but recommends closure-first docs and examples. Reviewers
may prefer a smaller v1 API with `modify` plus iteration only — noting that
`iter_mut` is itself a direct-handle escape (it hands out raw `Mut<T>` for batch
traversal, and is not more closure-disciplined than `get_mut`).

### `ObserverSet` / `SystemSet` factoring

Should `ObserverSet` remain a fully distinct trait (this RFC's choice), or should
it eventually share a base with `SystemSet`? The distinct trait is chosen here for
type-safety reasons (see Rationale and alternatives), but the near-duplication of
the derive and config surface is a reasonable thing to revisit with the community.

### Dynamic restricted mutation API

V1 should reject restricted components from ordinary safe untyped `ComponentId`
mutation APIs and from `ReflectComponent` mutation methods. Should Bevy also add an
explicit restricted dynamic / reflection API in the first PR, or should that wait
for a later use case?

### Observer cycle severity

Should observer ordering cycles be hard errors, schedule-build errors,
registration panics, or diagnostics that preserve a fallback deterministic order?

The RFC requires cycles to be diagnosed and not silently accepted. The final
severity should match Bevy's broader scheduling diagnostics.

### Empty observer-set targets

If an observer orders itself relative to a set that has no matching observers for
the event key, should Bevy silently treat that as a no-op, emit a diagnostic, or
offer a debug-only warning?

This RFC recommends no-op behavior for ergonomics, but this should be confirmed
against Bevy's existing set-ordering conventions.

### Contribution and AI policy for the implementation PR

[`bevy#24370`][bevy-pr-24370] (the restricted-access prototype) is currently closed and unmerged,
and its discussion history includes [Bevy AI-policy][bevy-ai-policy] concerns. This RFC is intended
to discuss the ECS design only. Any reopened or successor implementation PR must
independently satisfy Bevy's contribution and AI policy; this RFC does not bypass
those requirements. (Process details belong in the RFC PR description rather than
the design body.)

## Future possibilities

The following are intentionally outside this RFC, but the proposed primitives
would support them:

- parallel observer execution using observer `SystemParam` access analysis
- observer ambiguity diagnostics
- `ambiguous_with` for observers
- observer graph visualization
- richer observer cycle diagnostics
- dynamic restricted-mutation APIs
- field-level restricted access
- schedule-level write barriers
- command write manifests
- downstream mutation logs
- downstream save/load frameworks
- downstream deterministic replay frameworks
- downstream replication frameworks
- downstream audit frameworks

None of these future features are required for the two primitives proposed here.

`ObserverSet` lets users describe the order of event reactions. Restricted
component access lets component owners prevent accidental direct mutable access.
Together, they close two important gaps without committing Bevy to a networking,
save/load, replay, undo, or audit design.

[bevy-ecs]: https://docs.rs/bevy_ecs/latest/bevy_ecs/
[observer-module]: https://docs.rs/bevy_ecs/latest/bevy_ecs/observer/index.html
[systemset]: https://docs.rs/bevy_ecs/latest/bevy_ecs/schedule/trait.SystemSet.html
[schedule]: https://docs.rs/bevy_ecs/latest/bevy_ecs/schedule/index.html
[system-param]: https://docs.rs/bevy_ecs/latest/bevy_ecs/system/trait.SystemParam.html
[query]: https://docs.rs/bevy_ecs/latest/bevy_ecs/system/struct.Query.html
[query-data]: https://docs.rs/bevy_ecs/latest/bevy_ecs/query/trait.QueryData.html
[query-lens]: https://docs.rs/bevy_ecs/latest/bevy_ecs/system/struct.QueryLens.html
[component]: https://docs.rs/bevy_ecs/latest/bevy_ecs/component/trait.Component.html
[component-info]: https://docs.rs/bevy_ecs/latest/bevy_ecs/component/struct.ComponentInfo.html
[component-id]: https://docs.rs/bevy_ecs/latest/bevy_ecs/component/struct.ComponentId.html
[component-hooks]: https://docs.rs/bevy_ecs/latest/bevy_ecs/lifecycle/struct.ComponentHooks.html
[lifecycle]: https://docs.rs/bevy_ecs/latest/bevy_ecs/lifecycle/index.html
[required-components]: https://docs.rs/bevy_ecs/latest/bevy_ecs/component/struct.RequiredComponent.html
[component-mutability]: https://docs.rs/bevy_ecs/latest/bevy_ecs/component/trait.ComponentMutability.html
[event]: https://docs.rs/bevy_ecs/latest/bevy_ecs/event/trait.Event.html
[event-key]: https://docs.rs/bevy_ecs/latest/bevy_ecs/event/struct.EventKey.html
[world]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.World.html
[deferred-world]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.DeferredWorld.html
[entity-mut]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.EntityMut.html
[entity-world-mut]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.EntityWorldMut.html
[filtered-entity-mut]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.FilteredEntityMut.html
[entity-mut-except]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.EntityMutExcept.html
[entity-ref-except]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.EntityRefExcept.html
[unsafe-world-cell]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/unsafe_world_cell/struct.UnsafeWorldCell.html
[mut]: https://docs.rs/bevy_ecs/latest/bevy_ecs/change_detection/struct.Mut.html
[mut-untyped]: https://docs.rs/bevy_ecs/latest/bevy_ecs/change_detection/struct.MutUntyped.html
[change-detection]: https://docs.rs/bevy_ecs/latest/bevy_ecs/change_detection/index.html
[tick]: https://docs.rs/bevy_ecs/latest/bevy_ecs/change_detection/struct.Tick.html
[reflect-component]: https://docs.rs/bevy_ecs/latest/bevy_ecs/reflect/struct.ReflectComponent.html
[world-reflect]: https://docs.rs/bevy_ecs/latest/bevy_ecs/world/struct.World.html#method.get_reflect_mut
[bevy-pr-24328]: https://github.com/bevyengine/bevy/pull/24328
[bevy-pr-24370]: https://github.com/bevyengine/bevy/pull/24370
[rfc-86]: https://github.com/bevyengine/rfcs/pull/86
[bevy-ai-policy]: https://bevy.org/learn/contribute/policies/ai/
[bevy-replicon]: https://docs.rs/bevy_replicon/latest/bevy_replicon/
[lightyear]: https://docs.rs/lightyear/latest/lightyear/
[naia-bevy-server]: https://docs.rs/crate/naia-bevy-server/latest
[bevy-ggrs]: https://docs.rs/bevy_ggrs/latest/bevy_ggrs/
[bevy-save]: https://docs.rs/bevy_save/latest/bevy_save/
[unity-dots]: https://docs.unity.cn/Packages/com.unity.entities%401.0/manual/iterating-data-ijobentity.html
[unreal-property-specifiers]: https://dev.epicgames.com/documentation/en-us/unreal-engine/property-specifiers
[godot-export]: https://docs.godotengine.org/en/latest/tutorials/scripting/gdscript/gdscript_exports.html

# Feature Name: `commands-are-dead`

## Summary

By adding a fourth, maximally privileged tier of data access (addition and removal > mutation > read-only > detection), we can move virtually all common functionality out of `Commands`.
This allows us to completely deprecate commands (and `FromWorld`) in favor of scoped data access plus exclusive systems which watch for specific events, simplifying the code base and API and significantly improving both performance and usability.

## Motivation

Commands allow users to mutate the world in arbitrary ways, so long as they wait for the next hard-sync point to do so.
But this unscoped power has several serious implications for performance Commands have several serious issues:

- their behavior is unpredictable and overly broad, as they have the ability to mutate the world arbitrarily
- because of this unscoped behavior, commands can only be applied at the next hard sync point
  - this delay makes prototyping harder, forces many more hard syncs than otherwise would be required, and consistently confuses new users
- because of this unscoped behavior, commands can only be applied one at a time, in slow, unbatched sequence

Solves [Bevy #1613](https://github.com/bevyengine/bevy/issues/1613).

## User-facing explanation

Bevy's ability to automatically parallelize your systems comes from the carefully scoped data access specified in system parameters.
There are four tiers of data access, presented from most powerful to least powerful:

1. **Forge:** Add or remove data structures of this type to the world.
   1. Entity spawning and despawning.
   2. Component addition and removal.
   3. Resource addition and removal.
2. **Mutate:** Change the values of data of this type.
   1. Component modification.
   2. Resource modification.
   3. Event writing.
3. **Read:** Read the values of data of this type.
   1. Component reading.
   2. Resource reading.
   3. Event reading.
   4. `Changed` query filters.
4. **Detect:** Detect whether data of this type exists.
   1. `With` and `Without` query filters.
   2. `Exists<R>` system parameters to detect the presence or absence of resources.
   3. `Added` query filters.

Each tier implies the permissions required by all lower tiers; you can convert from a higher tier into an access of any lower tier using the appropriate `Into` trait.
This structure holds for both resources and components: `ResForge<R>` is to `ResMut<R>` is to `Res<R>` as `Forge<C>` is to `Mut<C>` is to `&C` for components.

A system can only run, if, for all pieces of data that they access, the following rules are upheld:

1. No other system can access data that is being forged in any way.
2. No other system can forge or read data that is being mutated.
3. No other system can forge or mutate data that can be read.
4. No other system can forge data that is being detected.

This information can be determined statically based on the system parameters, at the time of schedule creation, allowing us to store a [**topologically-sorted**](https://www.geeksforgeeks.org/topological-sorting/) graph of system dependencies that combines these rules with explicit system dependencies (as might be created with `.before`) to cache and quickly retrieve the results of applying these rules.

### Forging in practice

Let's start with the simplest case: creating and destroying resources.
By adding `ResForge<R>` to our system parameters, we can ensure that no other system is attempting to read our resource at the time it is being created (or destroyed):

```rust
struct Score(u8);

fn init_score_system(score: ResForge<Score>){
	score.create(Score(0));
}
```

Note that, as these resources are created in the context of a system, you can pull in other data from the `World` to perform more complex initialization.
In simple cases though, you'll want to use the convenience methods provided on `App`, `.init_resource` and `.insert_resource`, which create a startup system; running a single time at app initialization to generate the supplied resources.

Spawning entities is similar, but requires a `Foundry` system parameter to ensure data access is appropriately scoped:

```rust
fn spawn_entity(foundry: Foundry<Entity>>){
	// Creates a single entity.
	// Because we only requested Entity, we cannot add any components to this entity.
	foundry.spawn();
}

#[derive(Component)]
struct Player;

fn spawn_player(foundry: Foundry<Entity, &Player>>){
	// We can choose whether or not to add components
	// on a per-entity basis
	foundry.spawn().insert(Player);
}

// We need to explicitly use AsBundle here
// as we have no way to forbid the use of bundles as components
fn spawn_sprite(foundry: Foundry<(Entity, AsBundle<&SpriteBundle>)>){
	foundry.spawn().insert_bundle(SpriteBundle::default());
}

// You can access specific entities using Forge::entity()
fn remove_honored(foundry: Foundry<&Honored>, player_entity: Res<PlayerEntity>){
	// Removing components works the same way as inserting them;
	// just use the .remove method
	foundry.entity(player_entity.0).remove::<Honored>();
}
```

You can also forge entities from within queries, allowing you to quickly access their data and then decide whether or not to add or remove components to them on that basis.

```rust
#[derive(Component)]
struct Life(i8);

#[derive(Component)]
struct Dead;

fn check_for_death(query: Query<(&Life, Forge<&Dead>)>){
	for life, forge_dead in query.iter_mut(){
		if life.0 <= 0 {
			forge_dead.insert(Dead);
		}
	}
}
```

Completely despawning entities is more involved, and requires explicit opt-in via a special `Despawner` query parameter due to the extremely delocalized effects.
When we're merely removing specific components, we can be assured that systems which do not access those particular components will never conflict.
But when we're despawning entire entities, any systems that access even one potential shared archetype are in conflict, as we're not allowed to delete data others are using!

```rust
// This will block the execution of any system that accesses entity-component data
fn despawn_anything(query: Query<Despawner>, entity_to_despawn: Res<EntityToDespawn>){	
	query.get_mut(entity_to_despawn.0).despawn();
}

// This will only block systems that could access the player entity's archetype
// We need to include `Entity` in our first type-parameter to ensure that th
fn despawn_player(query: Query<Despawner, With<Player>>){
	query.single_mut().despawn();
}

// As before, we can combine this with data access to check if an entity should be despawned
fn despawn_if_dead(query: Query<(&Life, Despawner)>){
	// The Despawner type is specific to a particular entity;
	// by iterating over our query we can despawn the corresponding entity
	for life, despawner in query.iter_mut(){
		if life.0 <= 0 {
			despawner.despawn();
		}
	}
}
```

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

## Drawbacks

1. This is a serious breaking change, and will require significant rewriting both within the engine and for our end users.
2. Certain tasks that genuinely require exclusive world access (such as saving and loading games or modifying the schedule) will have to use an events + exclusive system pattern instead of the current `Commands` strategy.
3. Updating tables / archetypes under this model becomes much more complex as we're not allowed to block on component insertion / entity creation; the implementation may be quite challenging.
4. Spawning entities with a dynamic set of components becomes more complex: requiring either extensive query parameters or, in the case of truly dynamic sets, an exclusive system.
5. Despawning is complex; the proposed approach is sound but despawning will be very blocking without archetype invariants or very carefully constructed queries across the entire app.
6. System ordering can become more important, as entity spawning and despawning are no longer comfortably delayed.

## Rationale and alternatives

### Why remove `Commands`?

We should avoid offering users inferior tools to do the same job.

The large majority of command use cases are completely covered with just component, entity and resource insertion and removal.
Even complex custom commands tend to only involve the chaining of these operations.
The remaining use cases are app-specific and rather involved: moving them to event-listening exclusive systems improves clarity and allows us to fully deprecate the API.

The serious drawbacks of commands means that we should completely remove them: providing a single clear, fast and usable approach to these tasks.

### Why remove `FromWorld`?

It's redundant.

The dominant advantage of `FromWorld` vs. `commands.insert_resource` was that it did not take delayed effect.
This advantage is now removed, allowing us to use the much more comfortable (and parallelizable) data access of systems, rather than relying on exclusive world access.

### Why implement resource initialization as startup systems?

API consistency and simplicity mostly.
This also allows explicit ordering and parallel resource initialization.
Note that these should run in an earlier stage than ordinary startup systems to reduce system ordering footguns.

The alternative would be to special-case resource initialization, and always resolve data-less resource initialization as direct `World` operations.

## Unresolved questions

- How 

## \[Optional\] Future possibilities

1. The blocking nature of entity despawning will be improved by [archetype invariants](https://github.com/bevyengine/bevy/issues/1481), as users will be able to narrow the space of possible archetypes to be much closer to the set of archetypes that actually exist if they care about the system-parallelization performance.

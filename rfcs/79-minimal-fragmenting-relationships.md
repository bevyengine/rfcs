# Feature Name: `minimal-fragmenting-relationships`

## Summary

The central idea behind fragmenting relationships is the ability to express a connection between two `Entity`s by adding a pair of `(ComponentId, Entity)` as if it were a component. For example if one had several entities representing different kinds of swords one could add the pair `(Equipped, sword_entity)` to the player to signify which is currently equipped. 

Of course you could also just have created an `Equipped` component with an `Entity` field member to represent the same relationship. The advantage to "fragmenting" relationships is each unique pair is treated as if it were a unique component id, therefore they can appear in queries, use the existing insert and get APIs as well as form thier own archetypes. For example we could add `(Equipped, shield_entity)` to the same player and query for `(Equipped, *)`to find all equipped pairs.

Another analogy when considering the ECS as an in memory database is that relationships act as foreign keys. This unlocks a potential future where we can express more complex queries matching across multiple entities and relationship pairs. There are a myriad of additional features that can be built on top of the initial implementation described below. The main focus in this RFC will be the blocking issues that must be addressed before the initial implementation is achievable.
## Motivation

Relationships have been a long desired bevy feature. The ability to express and traverse hierarchies and graphs within the ECS unlocks a huge amount of potential. Features such as breadth-first query traversal, immediate hierarchy clean-up, component inheritance, multi-target queries etc. all rely on or are simplified by the existence of relationships as first class ECS primitive. 
## User-facing explanation

Relationship pairs can be inserted and removed like any other component. There are 3 ids associated with any relationship pair instance:
1. The source entity that the pair is being added to
2. The relationship component
3. The target entity

In the minimal implementation all relationships will take the form of `(ZST Component, entity)` pairs so the API surface will look quite familiar. All naming is subject to bikeshedding:
```rust
#[derive(Component)]
struct Relationship;

world.entity_mut(source_entity_a)
	.insert_pair::<Relationship>(target_entity_b)
	.remove_pair::<Relationship>(target_entity_b);

command.entity(source_entity_a)
	.insert_pair::<Relationship>(target_entity_b)
	.remove_pair::<Relationship>(target_entity_b);
```

Since each pair is a unique component you can also add multiple target entities with the same relationship type, as well as use APIs to access and query for them:
```rust
#[derive(Component)]
struct Eats;

let alice_mut = world.entity_mut(alice);

alice_mut.insert_pair::<Eats>(apple)
	.insert_pair::<Eats>(banana);

alice_mut.has_pair<Eats>(apple); // == true

for target in alice_mut.targets<Eats>() {
	// target == apple
	// target == banana
}

let query = world.query<(Entity, Targets<Eats>)>();
for entity, targets in query.iter(&world) {
	// entity == alice
	for target in targets {
		// target == apple
		// target == banana
	}
}

let query = world.query<Entity, With<Targets<Eats>>>();
for entity in query.iter(&world) {
	// entity == alice
}

```

Dynamic queries would also be able to match on specific pairs:
```rust
let query = QueryBuilder::new::<Entity>(&mut World)
	.with_pair::<Eats>(apple)
	.build();

query.single(&world); // == alice
```
As relationship pairs are effectively regular components this doesn't require any changes to the more complex query iteration APIs such as `par_iter`.

## Implementation strategy

There are a number of technical blockers before even a minimal version of relationships could be implemented in bevy. They're individually broken down below:
#### Archetype Cleanup
In order to fit each pair into a single `Identifier` we lose the entity generation (see this [article](https://ajmmertens.medium.com/doing-a-lot-with-a-little-ecs-identifiers-25a72bd2647) for more details). As a consequence when an entity is despawned we need to ensure that all the archetypes that included that entity as a target are also destroyed so that a future entity re-using that id is not misinterpreted as being the old target. This is non-trivial.

`bevy_ecs` currently operates under the assumption that archetype and component ids are dense and strictly increasing. In order to break that invariant we need to address several issues:
##### Query and System Caches
Query's caches of matching tables and archetypes are updated each time the query is accessed by iterating through the archetypes that have been created since it was last accessed. As the number of archetypes in bevy is expected to stabilize over the runtime of a program this isn't currently an issue. However with the increased archetype fragmentation caused by fragmenting relationships and the new wrinkle of archetype deletion the updating of the query caches is poised to become a performance bottleneck.

Luckily observers [#10839](https://github.com/bevyengine/bevy/pull/10839) can help solve this problem. By sending ECS events each time an archetype is created and creating an observer for each query we can target the updates to query caches so that we no longer need to iterate every new archetype. Similarly we can fire archetype deletion events to cause queries to remove the archetype from their cache.

In order to implement such a mechanism we need some way for the observer to have access to the query state in order to update it. This can be done by moving the query state into a component on the associated entity (WIP branch here: [queries-as-entities](https://github.com/james-j-obrien/bevy/tree/queries-as-entities)) and will require further refactors to the cache. For example, currently the collection of tables and archetypes is a vector but that doesn't provide an efficient method for removal without scanning the entirety of the vector so this data structure will have to be replaced with one that performs reasonably for both iteration and insertion/removal. `QueryState` would then be taken private in order to prevent unsafe accesses and users would refer to queries either by their type signature or entity id. 

However moving ECS structures into entities is largely blocked by the deletion of entities in the render world tracked here: [#12144](https://github.com/bevyengine/bevy/issues/12144).

Similarly systems cache their `ArchetypeComponentId` accesses for the purposes of multithreading, we also need to update theses caches to remove deleted ids presenting many of the same problems. 
##### Access Bitsets and Component SparseSets
The `FixedBitSet` implementation used by `Access` inside of systems, system params, queries etc. relies on dense component ids to avoid using a large amount of memory. If we were to start inserting ids with arbitrarily large entity targets as well as component ids shifted into the upper bits each `FixedBitSet` would allocate non-trivial amounts of memory to have flags for all the intervening ids. In order to minimize the affects of this we could consider all `(R, *)` pairs as one id in accesses, reducing the amount we need to track at the expense of some theoretical ability to parallelise.

Similarly the `SparseSet` implementation used in both table and sparse component storage also operate under the assumption that component ids will remain low and would allocate an even larger amount of memory storing indexes for all the intervening ids when storing relationship pairs on an entity. In order to circumvent this we can implement a "component index", this index would track for each component the tables and archetypes it belongs to as well as the column in that table the component inhabits. This removes the need for the sparse lookup as well as providing opportunities for other optimisations related to creation of observers and queries.

### Minimal Implementation
The component index could be implemented at any time, however the other pieces require some ordering to satisfy dependencies or prevent half implemented features.
Implementing the remainder in an appropriate order of dependance would work as follows:
- Merge observers [#10839](https://github.com/bevyengine/bevy/pull/10839)
- Resolve [#12144](https://github.com/bevyengine/bevy/issues/12144) so entities can be persisted in the render world.
- Refactor queries to become entities with caches updated via observers
- Implement archetype deletion and the associated clean-up for systems and queries, this is the only change that pays a complexity cost for no user benefit until relationships are realised

After all the issues above are addressed the actual implementation of the feature can begin. This requires changes in 3 main areas:
- Implementation of `WorldData` and `WorldFilter` implementations for accessing relationship targets
- Transition to new `Identifier` implemented in [#9797](https://github.com/bevyengine/bevy/pull/9797) instead of `ComponentId` in all places where pairs could be expected: tables, archetypes etc. 
- New methods for `EntityCommands`, `EntityMut`/`Ref`/`WorldMut` and `QueryBuilder`
## Drawbacks
- Increased archetype fragmentation when used, this can have impacts on memory fragmentation and query iteration performance, however the impact is non-trivial and varies from case to case
- Increase in internal and API complexity
## Prior art
- Prior [RFC](https://github.com/BoxyUwU/rfcs/blob/min-relations/rfcs/min-relations.md)
- [flecs](https://github.com/SanderMertens/flecs/blob/master/docs/Relationships.md)
- Partial draft implementation: [#9123](https://github.com/bevyengine/bevy/pull/9123)
## Unresolved questions
- How to address [#12144](https://github.com/bevyengine/bevy/issues/12144)
## Future possibilities
- Data on relationship pairs is a trivial extension:
  ```rust
  world.entity_mut(source).insert_pair(Relationship {}, target);
  ```
- Clean-up policies to allow specifying recursive despawning, this is important to allow porting `bevy_hierarchy` to use relationships
- More advanced traversal methods: up, BFS down, etc.
- More expressive query types: multi-target, grouping, sorted etc.
- With component as entities we unlock full `(entity, entity)` relationships

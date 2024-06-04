# Feature Name: `minimal-fragmenting-relationships`

## Summary
Users often run into difficulty while trying to model entity hierarchies in Bevy. The tools we currently have are not expressive enough to meet their needs. See this issue for more discussion [#3742](https://github.com/bevyengine/bevy/issues/3742).

This RFC proposes a minimal design that would unblock a large amount of users while also leaving avenues for the engine to further expand these features. In short:
- Define a new variant of `Identifier` that combines a `ComponentId` (the relationship type) and an `Entity` (the target) that can be added to an entity (the source) to model a relationship of that type between the source and the target.
- Implement command, world and query APIs to support adding, removing and reading these new relationships.

A large amount of internal refactoring has to be done in order to support this API so a significant focus will be on eliminating these technical blockers. For example:
- Replacing internal `SparseSet` usage where `Identifier` would be used as keys as they are no longer dense.
- Allowing archetypes to be deleted when an entity that was used as part of a pair in that archetype is despawned. This involves updating several internal caches that are used for correctness and/or safety.

## Motivation
Creating, traversing and maintaining hierarchies in Bevy has been a long standing UX problem, it can result in large amounts of code with poor performance characteristics that users will often bend their designs in order to avoid. Users should be able to many types of relationships between two entities and we should offer the tools to manage those relationships in an ergonomic and performant way.

There are many possible implementations of relationships in an ECS, some of which have been proposed for bevy before. The main factor separating them is how deeply integrated they are into the ECS itself. The central idea behind fragmenting relationships is the ability to express a connection between two `Entity`s by adding a combined id of `(ComponentId, Entity)` as if it were a single component id. This is advantageous compared to other designs as it makes it trivial to query for these relationships and thus expose advanced features such as breadth-first query traversal, immediate hierarchy clean-up, component inheritance, multi-target queries and more.

## Terminology

- **[Identifier](https://docs.rs/bevy_ecs/latest/bevy_ecs/identifier/struct.Identifier.html)**: A globally unique ECS id represented as a `u64`.
- **[Entity](https://docs.rs/bevy_ecs/latest/bevy_ecs/entity/struct.Entity.html)**: A variant of `Identifier` representing an entity.
- **Entity Id**: The lower 32 bits of an `Entity`, no two living entities should share the same id.
- **Generation**: The upper 32 bits of an `Entity`, represents how many times the id has been re-used, important to prevent interpreting a recycled `Entity` as the original.
- **Relationship**: A `Component` type used to model a relationship between two entities i.e. `Eats`, takes up the upper 32 bits of a `Pair`
- **Target**: An entity used as a target of a relationship, it's id takes up the lower 32 bits of a `Pair`
- **Pair**: A variant of `Identifier` representing a relationship to a target entity i.e. `(Eats, apple)` where `apple` is the target entity
- **Source**: The entity a `Pair` is added to as a component i.e. if `(Eats, apple)` is added to entity `alice`, `alice` would be the source of that relationship.
- **ZST**: Zero-sized type, these are types with `std::mem::size_of::<T>() == 0` and hence carry no data.
- **Wildcard**: A special id used to match all any id, similar to a glob i.e. `(Eats, *)` would match all ids where the relationship type was `Eats`
## User-facing explanation

Pairs can be inserted and removed like any other component. For simplicity the initial implementation will have all relationships take the form of `(ZST Component, entity)` pairs. All naming is subject to bikeshedding:
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

Since each pair is a unique id you can also add multiple target entities with the same relationship type, as well as use APIs to access and query for them:
```rust
#[derive(Component)]
struct Eats;

let alice_mut = world.entity_mut(alice);

alice_mut.insert_pair::<Eats>(apple)
	.insert_pair::<Eats>(banana);

alice_mut.has_pair<Eats>(apple); // == true

// Finds all (Eats, *) pairs and iterates the targets
for target in alice_mut.targets<Eats>() {
	// target == apple
	// target == banana
}

// Matches all entities with at least one (Eats, *) pair and exposes the targets
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
let query = QueryBuilder::<Entity>::new(&mut World)
	.with_pair::<Eats>(apple)
	.build();

query.single(&world); // == alice
```

As relationship pairs are treated internally as regular components this doesn't require any changes to the more complex query iteration APIs such as `par_iter`.
## Implementation strategy

There are a number of technical blockers before even a minimal version of relationships could be implemented in Bevy. They're individually broken down below:
#### Archetype Cleanup
In order to fit each pair into a single `Identifier` we lose the entity generation (see this [article](https://ajmmertens.medium.com/doing-a-lot-with-a-little-ecs-identifiers-25a72bd2647) for more details). As a consequence when an entity is despawned we need to ensure that all the archetypes that included that entity as a target are also destroyed so that a future entity re-using that id is not misinterpreted as being the old target. This is non-trivial.

`bevy_ecs` currently operates under the assumption that archetype and component ids are dense and strictly increasing. In order to break that invariant we need to address several issues:
##### Query and System Caches
Query's caches of matching tables and archetypes are updated each time the query is accessed by iterating through the archetypes that have been created since it was last accessed. As the number of archetypes in bevy is expected to stabilize over the runtime of a program this isn't currently an issue. However with the increased archetype fragmentation caused by fragmenting relationships and the new wrinkle of archetype deletion the updating of the query caches is poised to become a performance bottleneck.

An appropriate solution would allow us to expose new/deleted archetypes to only the queries that may be interested in iterating them. Luckily observers [#10839](https://github.com/bevyengine/bevy/pull/10839) could cleanly model this pattern. By sending ECS triggers each time an archetype is created and creating an observer for each query we can target the updates so that we no longer need to iterate every new archetype. Similarly we can fire archetype deletion events to cause queries to remove the archetype from their cache.

In order to implement such a mechanism we need some way for the observer to have access to the query state. Currently `QueryState` is stored in the system state, so in order to update it we would need to expose several additional APIs, even then it's quite difficult to reach into the system state tuple to access the correct query without some macros to handle tuple indexing. An alternative would be to move the query state into a component on an associated entity (WIP branch here: [queries-as-entities](https://github.com/james-j-obrien/bevy/tree/queries-as-entities)) this provides additional benefits in making it easier manage dynamic queries, but they are largely tangential to this RFC. 

Regardless this will also require further refactors to the query cache implementation. For example, currently the collection of tables and archetypes is a vector but that doesn't provide an efficient method for removal without an O(n) scan, so this data structure will have to be replaced with one that performs reasonably for both iteration and insertion/removal. Similarly systems cache their `ArchetypeComponentId` accesses for the purposes of multithreading, we also need to update theses caches to remove deleted ids presenting many of the same problems. 

Using observers globally or moving queries to entities are both blocked by the deletion of entities in the render world tracked here: [#12144](https://github.com/bevyengine/bevy/issues/12144).
##### Access Bitsets and Component SparseSets
The `FixedBitSet` implementation used by `Access` inside of systems, system params, queries etc. relies on dense ids to avoid allocating a large amount of memory. Since pairs have component ids in the upper bits the `u64` they are represented as is extremely large, each `FixedBitSet` would allocate non-trivial amounts of memory to have flags for all the intervening ids. In order to minimize the affects of this we could consider all `(R, *)` pairs as one id in accesses, reducing the amount we need to track at the expense of some theoretical ability to parallelise.

Similarly the `SparseSet` implementation used in both table and sparse component storage also operate under the assumption that ids will remain low and would allocate an even larger amount of memory storing pairs on an entity. In order to circumvent this we can implement a "component index", this index would track for each `Identifier` the tables and archetypes it belongs to as well as the column in that table the id inhabits (if any). This removes the need for the sparse lookup as well as providing opportunities for other optimizations related to creation of observers and queries.

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
- How to address [the render world blocking ECS developments](https://github.com/bevyengine/bevy/issues/12144)
## Future possibilities
- Data on relationship pairs is a trivial extension:
  ```rust
  world.entity_mut(source).insert_pair(Relationship {}, target);
  ```
- Clean-up policies to allow specifying recursive despawning, this is important to allow porting `bevy_hierarchy` to use relationships
- More advanced traversal methods: up, BFS down, etc.
- More expressive query types: multi-target, grouping, sorted etc.
- With component as entities we unlock the ability `(entity, entity)` pairs, where either entity could also be a component. This makes it much easier to express dynamic relationships where you don't necessarily want to define your relationship type statically or go through the trouble of creating a dynamic component, as well as cases where you want a component as the target.
  
There is also a hackmd that acts as a living design document and covers the implementation in more detail: https://hackmd.io/iNAekW8qRtqEW_BkcHDtgQ

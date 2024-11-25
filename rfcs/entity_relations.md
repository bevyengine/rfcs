# Feature Name: `min-relations`

## Summary

Relations allow users to create logical connection between entities. Each source entity can have multiple kinds of relations, each with their own data, and multiple relations of the same kind pointing to a different target entity.
Queries are expanded to allow efficient filtering of entities by the relations they have or the targets of these relations.

This RFC covers the minimal version of this feature, allowing for basic dynamic entity grouping and simple game logic extensions.

## Motivation

Entities often need to be aware of each other in complex ways: an attack that targets another unit, groups of entities that are controlled en-masse, or complex parent-child hierarchies that need to move together.

We *could* simply store an `Entity` in a component, then use `query.get(my_entity)` to access the relevant data.
(In this RFC, we call this the **Entity-in-component pattern**.)
But this quickly starts to balloon in complexity.

- How do we ensure that these components are cleaned up when the entity they're pointing to is?
- How do we handle pointing to multiple entities in similar ways?
- How do we look for all entities that point to a particular entity of interest?
- How do we despawn an entity and all of its children (with many types of child-like entities)?
- How do we quickly access data on the entity that we're attached to?
- How do we traverse this graph of entities?
- How do we ensure that our graph is acylic?

We *could* force our users to solve these problems, over and over again in each individual game.

This powerful and common pattern deserves a proper abstraction that we can make ergonomic, performant and *fearless*.
Eventually, at least ;)

## Guide-level explanation

**Relations** are similar to components that connect entities together, pointing from the entity they are stored on to the entity that they target.
Just like components, they can store effectively arbitrary data, or they can be data-less unit structs that serve as **markers**. Unlike components, they don't store the target entity directly, as that connection is managed by the engine.

Each relation has three parts:

1. The **source entity** that it is stored on.
2. The **target entity** (or simply **target**) that it points to.
3. The **relation kind** that defines the data that is stored on it.

Making your first connection is easy, and mirrors the API that you're familiar with from components:

```rust
// Relations might just be markers
struct FriendsWith;

// But they might store data instead
struct Owes(i32);

// Let's make some entities
let world = World::new();
let alice = world.spawn().id();
let boxy = world.spawn().id();

// You can make relations with commands!
commands.entity(alice).insert_relation(FriendsWith, boxy); // uwu:3

// You can remove them with commands!
commands.entity(alice).remove_relation(FriendsWith, boxy); // not uwu :(

// You can use direct World access!
world.entity_mut(alice).insert_relation(FriendsWith, boxy); // uwu:3

// Relations are directed; a relation in one direction does not imply the same in the other direction
commands.entity(boxy).insert_relation(FriendsWith, alice); // one way friendship sad :(

// You can mix different relation kinds on the same entity!
commands.entity(alice).insert_relation(Owes(9999), boxy); // :)))))

// You can add mulitple relations of the same kind to a single source entity!
let cart = world.spawn().id();
commands.entity(alice).insert_relation(FriendsWith, cart); // Hi!
```

Once your relations are in place, you'll want to access them in some form.
Just like with ordinary components, you can request them in your queries:

```rust
// Relations can be queried for immutably
fn friendship_is_magic(mut query: Query<(&mut Magic, &Relation<FriendsWith>), With<Pony>>) {
    // Each entity can have their own friends
    for (mut magic, friends: RelationIter<FriendsWith>) in query.iter_mut() {
        // Relations return an iterator over the target and data of each relation on an entity
        magic.power += friends.count();
    }
}

// Or you can request mutable access to modify the relation's data
fn interest(query: Query<&mut Relation<Owes>>) {
    for debts in query.iter() {
        for (owed_to: Entity, amount_owed: &mut Owes) in debts {
            println!("we owe {} some money", owed_to);
            amount_owed *= 1.05;
        }
    }
}

// Query filters work too!
fn not_alone(
    mut commands: Commands,
    query: Query<Entity, Without<Relation<FriendsWith>>>,
    puppy: Res<Puppy>,
) {
    for lonely_entity in query.iter() {
        commands.insert_relation(FriendsWith, puppy.entity);
    }
}

// So does change detection with Added and Changed! You can use `Relation<T>` 
// instead of a component type in all of the query filters!
fn new_friends(query: Query<&mut Excitement, Added<Relation<FriendsWith>>>) {
    for (excitement, new_friends) in query {
        excitement.value += 10 * new_friends.count();
    }
}

// You can look for relations with a specific kind, source and target
fn unrequited_love(query: Query<(Entity, &Relation<Loves>)>) {
    for (lover, crushes) in query.iter() {
        for (crush, _) in crushes {
            // The `get_relation` method on Query is used just like `get_component`
            // except you have to specify the target of the relation you want to get!
            let reciprocal_crush = query.get_relation::<Loves>(crush, lover);
            reciprocal_crush.expect("Surely they must feel the same way! u-uwu");
        }
    }
}

// You can query for entities that target a specific entity
fn friend_of_dorothy(query: Query<Entity, With<Relation<FriendsWith>>>, dorothy: Res<Dorothy>) {
    let filtered = query
        // Relation filters make the `Relation<T>` in our query type
        // only be for the given targets specified- by default `Relation<T>`
        // will apply to any target
        .filter_relation(
            RelationFilter::<FriendsWith>::target(dorothy.entity)
        )
        // You must call .apply_filters() before you can access the query again
        .apply_filters();

    for source_entity in filtered.iter() {
        println!("{} is friends with Dorothy!", source_entity);
    }
}

// Or even look for some combination of targets!
fn caught_in_the_middle(
    mut query: Query<&mut Stress, With<Relation<FriendsWith>>>,
    dorothy: Res<Dorothy>,
    elphaba: Res<Elphaba>,
) {
    query
        // Note that we can set relation filters even if the relation is in a query filter
        .filter_relation(
            RelationFilter::<FriendsWith>::all_of()
                .target(dorothy.entity)
                .target(elphaba.entity),
        )
        .apply_filters()
        // apply_filters() returns `&mut Query` so we can just iterate straight away
        .for_each(|mut stress| {
            println!("{} is friends with Dorothy and Elphaba!", source_entity);
            stress.val = 1000;
        })
}

```


```rust
// You can use the `Entity` target returned by your relations to fetch data from the 
// target's components by combining it with `query::get()` or `query::get_component<C>()`.
fn debts_come_due(
    mut debt_query: Query<(Entity, &Relation<Owes>)>,
    mut money_query: Query<&mut Money>,
) {
    for (debtor, debt) in debt_query.iter() {
        for (lender, amount_owed) in debt {
            *money_query
                .get(debtor)
                .unwrap()
                .checked_sub(amount_owed)
                .expect("Nice kneecaps you got there...");
            // getting the money on the target of our `Owes` relation!
            *money_query.get(lender).unwrap() += amount_owed;
        }
    }
}

```

### Nuances of relations

Relations between entities are automatically cleaned up when either the source or target entity is removed.
Just like components, you can view which relations were removed in the last frame through the `RemovedRelations<T>` system parameter, which also returns their data.

Each relation kind + target combination is treated like a unique type for the purposes of archetypes.
This makes can be relevant to the performance of your game; accelerating target entity filtering but creating overhead if you have a very large number of distinct targets.
As a result, changing the source or target of a relation can only be done while the app has exclusive access to the world, via `Commands` or in exclusive systems.
Here's an example of how you might do so:

```rust
fn love_potion(
    mut commands: Commands,
    query: Query<Entity, &Relation<Loves>, With<Cute>>,
    player_query: Query<Entity, With<Player>>,
) {
    let player = player.single().unwrap();

    for (victim, former_love) in query.iter() {
        // Only one relation of a given kind can exist between a source and a target
        // (although the opposite direction can exist on the target entity)
        // like always, new relations overwrite existing relations
        // with the same source, kind and target
        for (_, former_lover) in former_love {
            commands
                .entity(victim)
                .change_target::<Loves>(former_lover, player); // This is unethical!!
        }
    }
}

/*

FIXME: This example is a bit borked but if we fix it the we no longer need `move_relation`

fn adoption(
    mut commands: Commands,
    // As with components, you can query for relations that may or may not exist
    query: Query<(Entity, &NewOwner, Option<&Relation<Owns>>), With<Kitten>>,
) {
    for (kitten, new_owner, previous_ownership) in query.iter() {
        // Changing the target or the source will fail silently if no appropriate relation exists
        match previous_ownership {
            // We can change sources by controlling which entity owns the relation
            // move_relation is directly analagous to move_component
            Some(old_owner, _) => commands
                .entity(old_owner)
                .move_relation::<Owns>(new_owner, kitten), // uwu :3
            None => commands
                .entity(new_owner)
                .insert_relation(Owns::default(), kitten), // uwu!!
        }
    }
}
*/

// `change_target` and `move_relation` preserve the relation's data
// This system removes all springs attached to mass 1, and adds them to mass 2 instead
fn reattach_springs(
    mut commands: Commands,
    selected_mass1: Res<FirstSelected>,
    selected_mass2: Res<SecondSelected>,
    query: Query<Relation<Spring>>,
) {
    let m1 = selelected_mass1.entity;
    let m2 = selelected_mass2.entity;
    let springs = query.get(m1);

    // Spring relations are symmetric; each spring is defined by
    // two identical relations that point opposite direction.
    // We need to preserve both during this operation
    for (target_mass, _) in springs {
        // When mass 1 is the source, we must change the source to mass 2
        commands.entity(m1).move_relation::<Spring>(m2, target_mass);

        // When mass 1 is the target, we must change the target to mass 2
        commands.entity(target_mass).change_target::<Spring>(m1, m2);
    }
}

```

### Advanced relation filters

The examples shown above show of the basics of relation filtering, but that's not all they can do.
The most important extension is the ability to filter for multiple entities at once from a `Vec<Entity>`.
In concert with this, `any_of` and `all_of` can be used to collect groups of entities into a single filter.
`any_of` uses **or semantics**, returning any entity if any of the filters are met,
while `all_of` uses **and semantics**, rejecting entities who do not meet every specified filter.

```rust
// In this example, possible moves are stored on each piece
// as a relation to a specific board position
fn next_move(
    mut commands: Commands,
    board_state: Res<BoardState>,
    query: Query<(Entity, &Relation<Move>), With<Piece>>,
) {
    let valid_positions: Vec<Entity> = board_state.compute_valid_positions();

    let filtered = query
        .filter_relation(
            RelationFilter::<Action>::any_of().targets(valid_positions)
        )
        .apply_filters();

    let best_move = valid_moves
        .max_by_key(|(potential_position, _)| potential_position.compute_heuristic());

    if let Some((_, best_move)) = best_move {
        commands.entity(piece).insert_relation::<NextMove>(best_move);
    }
}

// We can use this with either all_of or any_of to get the effect we need
fn all_roads_lead_to_rome(
    mut commands: Commands,
    cities_query: Query<Entity, With<&City>>,
    roads_query: Query<Entity, (With<City>, With<Relation<Road>>)>,
) {
    let all_cities: Vec<Entity> = cities_query.iter().collect();
    roads_query
        .filter_relation(
            RelationFilter::<Road>::all_of().targets(all_cities)
        )
        .apply_filters()
        // The cities still left are connected to all other cities by roads
        // We're tagging them with a Hub marker component so we can find them again quickly
        .map(|city| commands.entity(city).insert(Hub));
}

```

As your filters grow in complexity, it can be useful to filter by the source entity, rather than target entity.
Note that you can get the same sort of effect by using `Query::get`;
this functionality just makes it more ergonomic to specify certain complex relation filters.

```rust
fn paths_to_choose(location: Res<PlayerLocation>, mut query: Query<&mut Relation<Path>>) {
    // We only care about paths that lead away from our current position
    query
        .filter_relation(RelationFilter::<Path>::source(location.entity))
        .apply_filters()
        .map(|_, mut path| {
            path.highlight();
        });

    // You can accomplish the same thing by subsetting your query
    // using query.get, query.single or branching on Entity
    // This is equivalent because queries are composed of a collection of source entities
    let mut paths_from_player_location = query.get_mut(location.entity).unwrap();
    for (path, _) in paths_from_player_location {
        println!(
            "It will take {} minutes to get to {} from here.",
            path.travel_time, path.destination_name
        );
    }
}

// With more complex logic that blends sources and targets,
// the value of this approach is more evident
fn sever_groups(
    mut commands: Commands,
    selection_query: Query<Entity, &Selection>,
    mass_query: Query<Entity, (With<Mass>, With<Relation<Spring>>)>,
) {
    // Our goal is to remove all of the springs
    // connecting these two groups of point masses
    let mut group_1 = Vec::<Entity>::new();
    let mut group_2 = Vec::<Entity>::new();

    for (entity, selection) in selection_query.iter() {
        match selection {
            Selection::Group1 => group_1.push(entity),
            Selection::Group2 => group_2.push(entity),
        }
    }

    let one_to_two = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_1).targets(group_2))
        .apply_filters();

    let two_to_one = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_2).targets(group_1))
        .apply_filters();

    // First we remove the springs that point in one direction
    for (source, springs) in one_to_two.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }

    // And then, because the relation is symmetric, the other
    for (source, springs) in two_to_one.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }
}

```

From time-to-time, we may care about *excluding* entities who have relations to certain targets.
To do so, we set our relation filter on a `Without<Relation<R>>` query parameter.

```rust
// Fraternizing with communists is a strict disqualification!
fn purity_testing(
    mut commands: Commands,
    candidate_query: Query<Entity, (With<Candidate>, Without<Relation<FriendsWith>>)>,
    communist_query: Query<Entity, With<Communist>>,
) {
    let communists = communist_query.iter().collect();

    // Normally, Without relation filters disqualify all entities with that relation.
    // But by narrowing them to only affect certain sources or targets,
    // we can control which entities are affected!
    candidate_query
        .filter_relation(
            // Even a single communist friend is disqualifying!
            RelationFilter::<FriendsWith>::any_of().targets(communists),
        )
        .apply_filters()
        // Candidates who are not friends with any communists have been filtered
        // Leaving only suspect candidates to be stripped of their candidacy
        .for_each(|candidate| commands.entity(candidate).remove::<Candidate>());

    // Note that candidates with no friends at all are always excluded from the query,
    // sparing them from our arbitrary interrogation
}

```

We can filter on multiple types of relations at once by chaining together our `.filter_relation` methods before we call `.apply_filters()`.

```rust
fn frenemies(
    query: Query<Entity, (With<Relation<FriendsWith>>, With<Relation<EnemiesWith>>)>,
    player: Res<Player>,
) {
    query
        .filter_relation(RelationFilter::<FriendsWith>::target(player.entity))
        .filter_relation(RelationFilter::<EnemiesWith>::target(player.entity))
        .for_each(|entity| println!("{} is frenemies with the player!", entity));
}
```

Filters chained in this way will operate on the restricted list produces by the previous filter (following "and semantics").
We can even combine positive and negative filters by including multiple copies of the same relation in our query, and then specifying which relation we're filtering using the second type parameter.
This advanced technique is helpful when you really need to be specific, as shown in these examples:

```rust
fn herbivores(
    mut commands: Commands,
    plants_query: Query<Entity, With<Plant>>,
    animals_query: Query<Enity, With<Animal>>,
    species_query: Query<Entity, (With<Relation<Consumes>>, Without<Relation<Consumes>>)>,
) {
    let plants = plants_query.iter().collect();
    let animals = animals_query.iter().collect();

    // The generics on `filter_relation` are normally inferred but we can manually
    // specify them to disambiguate which `Relation<T>` we are filtering in the query
    species_query
        // The second type parameter controls which relation filter of that type is being modified
        .filter_relation::<Consumes, InFilter<{ 0 }>>(RelationFilter::any_of().targets(plants))
        // Because this is a without filter, this means that we exclude all entities
        // with even one Consumes relation that targets an animal
        .filter_relation::<Consumes, InFilter<{ 1 }>>(RelationFilter::any_of().targets(animals))
        .for_each(|entity| commands.entity(entity).insert(Herbivore));
}

```

### Grouping entities

By targeting a common entity, relations work fantastically as an ergonomic way to group entities.
This pattern acts as a nice complement to marker components when either:

1. You want to define the *kind* of group, rather than just membership.
2. The number of different groups you need is unknowable at compile time.

Here are some concrete examples where this pattern works well:

- you're storing multiple parts (such as meshes or sprites with their own `Transform`s) of the same in-game-object across multiple entities
- you want to dynamically group units together, like in a real-time-strategy game
- the effects created by a single entity, like the bullets they're spewing out
- you want a way to mark the controller of a unit in a queryable way, but the number of possible controllers is unknown at compile time
- you have multiple cameras, and want to mark which frustum each entity is in so they can be culled independently

### Entity graphs

Unsurprisingly, you can build all manner of complex entity graphs using relations.
Before you begin to rip out your resources with graph data structures storing `Entity` keys,
pause to consider that this is, in fact, `*min*-relations`.
Performance will be limited, APIs will be lacking,
and you will bemoan your inability to specify information about the global properties of the graph.

However, even relations do have three critical advantages over the **external graph data structure pattern**:

1. You can readily store data in each edge of your directed graph.
2. You can quickly filter for the source or target entities (nodes) of each relation (edge).
3. You don't have to think about synchronization.

These advantages make even `min-relations` a compelling alternative to both the Entity-in-component and external graph data structure patterns if:

1. You don't care about performance.
2. You don't need to perform complex traversals.
3. You're operating on general graphs that can contain cycles and self-referential edges.
4. You don't want to worry about synchronization bugs.

Let's take a look at how you could use relations to build an API for a tree-shaped graph that looks suspiciously like Bevy 0.5's parent-child hierarchy.

## Reference-level explanation

TODO: Boxy explains the magic.

### Data Storage Model

### The Second Relation Filter Type Parameter

## Drawbacks

1. Introduces another core abstraction to users that they will have to learn.
Rewriting our parent-child code in terms of relations is important, but will introduce the concept to users quite quickly.
2. Heavy use of relations can seriously fragment archetypes. We need to be careful that this doesn't badly impact our performance.
3. `move_component`, `move_relation` and `change_target` only work on `Clone` components / relations due to the need to temporarily be able to access removed component data.

## Rationale and alternatives

### Why do relations fragment archetypes?

TODO: Boxy explains her decisions.

### Why are relations stored in the ECS?

TODO: Boxy help!

### Why aren't relations described as a special type of components?

They don't behave in exactly the same ways,
and describing them as such is confusing to beginners.

Under the hood, the existing implementation models

TODO: add examples of the differences

## Why do we need the builder pattern for filtering relations?

TODO: Boxy explains perf implications

### Why don't we just store Entities in components?

See [Motivation].

### Why don't we just store graph data structures in a Resource?

See [Motivation] *and* [Entity Graphs].

### Why do we need arbitrarily complex relation filters??

Relation filters take advantage of archetypes to be much faster and more ergonomic than doing the same operation by naive iteration.
In order to ensure that it can always be used effectively, we need a genuinely complete DSL,
in addition to the convenient methods discussed first.

The API for compound relation filters is simple to implement and easily ignored.

### Why is my favorite feature relegated to Future Work?

It's already many thousands of words long, and the associated PR has thousands of lines of code.
Done is better than perfect, and we had to draw the line somewhere.
As with all things, we can gradually improve this over time.

The set of features expressed here is useful enough to be a clear improvement over the existing patterns in at least some use cases.
There's a lot of value in getting this feature in front of users to see the response at scale.

### Why is the implementation for this so *slow*?

Many uses of relations are not performance-critical,
and the value of a nice API and automatic handling of data synchronization far outweighs any performance cost for those users.

We can fix the perf once we have a better understanding of the APIs we need to support,
and a tangible implementation to benchmark against.
There's also nothing stopping you from doing things the old ways in your elaborate graph-traversing systems.

See also: the question directly above.

## Prior art

[`Flecs`](https://github.com/SanderMertens/flecs), an advanced C++ ECS framework, has a similar feature, which they call "relationships".
These are somewhat different, they use an elaborate query domain-specific language along with being more feature rich.
You can read more about them in the [docs](https://github.com/SanderMertens/flecs/blob/master/docs/Queries.md) or [corresponding PR](https://github.com/SanderMertens/flecs/pull/358).

You can, of course, build similar data structures using the ECS itself.
Here's a look at the complexities involved in doing so in [EnTT](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/).

## Unresolved questions

1. How do we put relations in Bundles?
2. What do we actually call [`Noitaler`](#bikeshed)? Candidates so far are:
   1. `ReverseRelation`
   2. `Inverse`
   3. `CoRelation`
   4. `TargetOf`
3. Do we want a macro to make complex calls easier?
4. Bikeshed: what argument order do we want for `change_target` and `move_relation`? Currently:
   1. `commands.entity(source).change_target(old_target, new_target)`
   2. `commands.entity(old_source).change_target(new_source, target)`

```rust
macro_rules Relation {
  (this $kind:ty *) => Relation<$kind>,
  (* $kind:ty this) => Noitaler<$kind>,
}
```

## Future possibilities

Relation enhancements beyond the scope of `min-relations`:

1. Convenience functions of all sorts:
    1. `remove_relations::<T>`, for removing a `Vec<Entity>` of relations at once.
    2. `clear_relation::<T>`, for removing all relations of that type at once.
    3. Generalized `despawn_recursive::<T>`, parameterizing by relation kind.
2. Sugar for accessing data on the target entity in a single query.
Proposed API: `Relation<T: Component, Q: WorldQuery>`
3. Graph shape guarantees (e.g tree, acyclic, 1-depth, non-self-referential).
Likely implemented using archetype invariants.
4. Graph traversals API: breadth-first, depth-first, root of tree etc.
5. Arbitrary target types instead of just `Entity` combined with `KindedEntity` proposal to ensure your targets are valid.
6. Automatically symmetric (or anti-symmetric) relations to model undirected edges.
7. `Noitaler`*, for relations that point in the opposite direction.
8. Streaming iters to allow for queries like: `Query<&mut Money, Relation<Owes, &mut Money>>` and potentially use invariants on graph shape to allow for skipping soundness checks for the aliasing `&mut Money`'s
9. Assorted performance optimizations. For example:
   1. Reducing the cost of having many archetypes.
   2. Relation storage type options to cause archetype fragmentation based on whether *any* relations of that type are present.
   3. Index-backed relations look-up.
10. Relative ordering between relations of the same kind on the same entity.
This would enable the `Styles` proposal from #1 to use relations.
11. Compound relation filters, letting you nest logic arbitrarily deep. These are just sugar / perf for very advanced users, and can wait for a while.
12. \[Controversial\] A full graph constraint solver DSL ala [Flecs](https://github.com/SanderMertens/flecs) for advanced querying.
13. Entity-to-entity event channels using relations (blocked on [bevy #2116](https://github.com/bevyengine/bevy/pull/2116)

Relation applications in the engine:

1. Replace existing parent-child API.
2. Applications for UI: styling, widget wiring etc.
3. Frustum culling with multiple cameras.

<b id="bikeshed">AUTHOR'S NOTE</b>: Much like `yeet`, `Noitaler` is the bikeshed-avoidance name for the inverse of `Relation`, and will never be used.

EDITOR's NOTE: `Noitaler` may very much be used :3# Feature Name: `min-relations`

## Summary

Relations allow users to create logical connection between entities. Each source entity can have multiple kinds of relations, each with their own data, and multiple relations of the same kind pointing to a different target entity.
Queries are expanded to allow efficient filtering of entities by the relations they have or the targets of these relations.

This RFC covers the minimal version of this feature, allowing for basic dynamic entity grouping and simple game logic extensions.

## Motivation

Entities often need to be aware of each other in complex ways: an attack that targets another unit, groups of entities that are controlled en-masse, or complex parent-child hierarchies that need to move together.

We *could* simply store an `Entity` in a component, then use `query.get(my_entity)` to access the relevant data.
(In this RFC, we call this the **Entity-in-component pattern**.)
But this quickly starts to balloon in complexity.

- How do we ensure that these components are cleaned up when the entity they're pointing to is?
- How do we handle pointing to multiple entities in similar ways?
- How do we look for all entities that point to a particular entity of interest?
- How do we despawn an entity and all of its children (with many types of child-like entities)?
- How do we quickly access data on the entity that we're attached to?
- How do we traverse this graph of entities?
- How do we ensure that our graph is acylic?

We *could* force our users to solve these problems, over and over again in each individual game.

This powerful and common pattern deserves a proper abstraction that we can make ergonomic, performant and *fearless*.
Eventually, at least ;)

## Guide-level explanation

**Relations** are similar to components that connect entities together, pointing from the entity they are stored on to the entity that they target.
Just like components, they can store effectively arbitrary data, or they can be data-less unit structs that serve as **markers**. Unlike components, they don't store the target entity directly, as that connection is managed by the engine.

Each relation has three parts:

1. The **source entity** that it is stored on.
2. The **target entity** (or simply **target**) that it points to.
3. The **relation kind** that defines the data that is stored on it.

Making your first connection is easy, and mirrors the API that you're familiar with from components:

```rust
// Relations might just be markers
struct FriendsWith;

// But they might store data instead
struct Owes(i32);

// Let's make some entities
let world = World::new();
let alice = world.spawn().id();
let boxy = world.spawn().id();

// You can make relations with commands!
commands.entity(alice).insert_relation(FriendsWith, boxy); // uwu:3

// You can remove them with commands!
commands.entity(alice).remove_relation(FriendsWith, boxy); // not uwu :(

// You can use direct World access!
world.entity_mut(alice).insert_relation(FriendsWith, boxy); // uwu:3

// Relations are directed; a relation in one direction does not imply the same in the other direction
commands.entity(boxy).insert_relation(FriendsWith, alice); // one way friendship sad :(

// You can mix different relation kinds on the same entity!
commands.entity(alice).insert_relation(Owes(9999), boxy); // :)))))

// You can add mulitple relations of the same kind to a single source entity!
let cart = world.spawn().id();
commands.entity(alice).insert_relation(FriendsWith, cart); // Hi!
```

Once your relations are in place, you'll want to access them in some form.
Just like with ordinary components, you can request them in your queries:

```rust
// Relations can be queried for immutably
fn friendship_is_magic(mut query: Query<(&mut Magic, &Relation<FriendsWith>), With<Pony>>) {
    // Each entity can have their own friends
    for (mut magic, friends: RelationIter<FriendsWith>) in query.iter_mut() {
        // Relations return an iterator over the target and data of each relation on an entity
        magic.power += friends.count();
    }
}

// Or you can request mutable access to modify the relation's data
fn interest(query: Query<&mut Relation<Owes>>) {
    for debts in query.iter() {
        for (owed_to: Entity, amount_owed: &mut Owes) in debts {
            println!("we owe {} some money", owed_to);
            amount_owed *= 1.05;
        }
    }
}

// Query filters work too!
fn not_alone(
    mut commands: Commands,
    query: Query<Entity, Without<Relation<FriendsWith>>>,
    puppy: Res<Puppy>,
) {
    for lonely_entity in query.iter() {
        commands.insert_relation(FriendsWith, puppy.entity);
    }
}

// So does change detection with Added and Changed! You can use `Relation<T>` 
// instead of a component type in all of the query filters!
fn new_friends(query: Query<&mut Excitement, Added<Relation<FriendsWith>>>) {
    for (excitement, new_friends) in query {
        excitement.value += 10 * new_friends.count();
    }
}

// You can look for relations with a specific kind, source and target
fn unrequited_love(query: Query<(Entity, &Relation<Loves>)>) {
    for (lover, crushes) in query.iter() {
        for (crush, _) in crushes {
            // The `get_relation` method on Query is used just like `get_component`
            // except you have to specify the target of the relation you want to get!
            let reciprocal_crush = query.get_relation::<Loves>(crush, lover);
            reciprocal_crush.expect("Surely they must feel the same way! u-uwu");
        }
    }
}

// You can query for entities that target a specific entity
fn friend_of_dorothy(query: Query<Entity, With<Relation<FriendsWith>>>, dorothy: Res<Dorothy>) {
    let filtered = query
        // Relation filters make the `Relation<T>` in our query type
        // only be for the given targets specified- by default `Relation<T>`
        // will apply to any target
        .filter_relation(
            RelationFilter::<FriendsWith>::target(dorothy.entity)
        )
        // You must call .apply_filters() before you can access the query again
        .apply_filters();

    for source_entity in filtered.iter() {
        println!("{} is friends with Dorothy!", source_entity);
    }
}

// Or even look for some combination of targets!
fn caught_in_the_middle(
    mut query: Query<&mut Stress, With<Relation<FriendsWith>>>,
    dorothy: Res<Dorothy>,
    elphaba: Res<Elphaba>,
) {
    query
        // Note that we can set relation filters even if the relation is in a query filter
        .filter_relation(
            RelationFilter::<FriendsWith>::all_of()
                .target(dorothy.entity)
                .target(elphaba.entity),
        )
        .apply_filters()
        // apply_filters() returns `&mut Query` so we can just iterate straight away
        .for_each(|mut stress| {
            println!("{} is friends with Dorothy and Elphaba!", source_entity);
            stress.val = 1000;
        })
}

```


```rust
// You can use the `Entity` target returned by your relations to fetch data from the # Feature Name: `min-relations`

## Summary

Relations allow users to create logical connection between entities. Each source entity can have multiple kinds of relations, each with their own data, and multiple relations of the same kind pointing to a different target entity.
Queries are expanded to allow efficient filtering of entities by the relations they have or the targets of these relations.

This RFC covers the minimal version of this feature, allowing for basic dynamic entity grouping and simple game logic extensions.

## Motivation

Entities often need to be aware of each other in complex ways: an attack that targets another unit, groups of entities that are controlled en-masse, or complex parent-child hierarchies that need to move together.

We *could* simply store an `Entity` in a component, then use `query.get(my_entity)` to access the relevant data.
(In this RFC, we call this the **Entity-in-component pattern**.)
But this quickly starts to balloon in complexity.

- How do we ensure that these components are cleaned up when the entity they're pointing to is?
- How do we handle pointing to multiple entities in similar ways?
- How do we look for all entities that point to a particular entity of interest?
- How do we despawn an entity and all of its children (with many types of child-like entities)?
- How do we quickly access data on the entity that we're attached to?
- How do we traverse this graph of entities?
- How do we ensure that our graph is acylic?

We *could* force our users to solve these problems, over and over again in each individual game.

This powerful and common pattern deserves a proper abstraction that we can make ergonomic, performant and *fearless*.
Eventually, at least ;)

## Guide-level explanation

**Relations** are similar to components that connect entities together, pointing from the entity they are stored on to the entity that they target.
Just like components, they can store effectively arbitrary data, or they can be data-less unit structs that serve as **markers**. Unlike components, they don't store the target entity directly, as that connection is managed by the engine.

Each relation has three parts:

1. The **source entity** that it is stored on.
2. The **target entity** (or simply **target**) that it points to.
3. The **relation kind** that defines the data that is stored on it.

Making your first connection is easy, and mirrors the API that you're familiar with from components:

```rust
// Relations might just be markers
struct FriendsWith;

// But they might store data instead
struct Owes(i32);

// Let's make some entities
let world = World::new();
let alice = world.spawn().id();
let boxy = world.spawn().id();

// You can make relations with commands!
commands.entity(alice).insert_relation(FriendsWith, boxy); // uwu:3

// You can remove them with commands!
commands.entity(alice).remove_relation(FriendsWith, boxy); // not uwu :(

// You can use direct World access!
world.entity_mut(alice).insert_relation(FriendsWith, boxy); // uwu:3

// Relations are directed; a relation in one direction does not imply the same in the other direction
commands.entity(boxy).insert_relation(FriendsWith, alice); // one way friendship sad :(

// You can mix different relation kinds on the same entity!
commands.entity(alice).insert_relation(Owes(9999), boxy); // :)))))

// You can add mulitple relations of the same kind to a single source entity!
let cart = world.spawn().id();
commands.entity(alice).insert_relation(FriendsWith, cart); // Hi!
```

Once your relations are in place, you'll want to access them in some form.
Just like with ordinary components, you can request them in your queries:

```rust
// Relations can be queried for immutably
fn friendship_is_magic(mut query: Query<(&mut Magic, &Relation<FriendsWith>), With<Pony>>) {
    // Each entity can have their own friends
    for (mut magic, friends: RelationIter<FriendsWith>) in query.iter_mut() {
        // Relations return an iterator over the target and data of each relation on an entity
        magic.power += friends.count();
    }
}

// Or you can request mutable access to modify the relation's data
fn interest(query: Query<&mut Relation<Owes>>) {
    for debts in query.iter() {
        for (owed_to: Entity, amount_owed: &mut Owes) in debts {
            println!("we owe {} some money", owed_to);
            amount_owed *= 1.05;
        }
    }
}

// Query filters work too!
fn not_alone(
    mut commands: Commands,
    query: Query<Entity, Without<Relation<FriendsWith>>>,
    puppy: Res<Puppy>,
) {
    for lonely_entity in query.iter() {
        commands.insert_relation(FriendsWith, puppy.entity);
    }
}

// So does change detection with Added and Changed! You can use `Relation<T>` 
// instead of a component type in all of the query filters!
fn new_friends(query: Query<&mut Excitement, Added<Relation<FriendsWith>>>) {
    for (excitement, new_friends) in query {
        excitement.value += 10 * new_friends.count();
    }
}

// You can look for relations with a specific kind, source and target
fn unrequited_love(query: Query<(Entity, &Relation<Loves>)>) {
    for (lover, crushes) in query.iter() {
        for (crush, _) in crushes {
            // The `get_relation` method on Query is used just like `get_component`
            // except you have to specify the target of the relation you want to get!
            let reciprocal_crush = query.get_relation::<Loves>(crush, lover);
            reciprocal_crush.expect("Surely they must feel the same way! u-uwu");
        }
    }
}

// You can query for entities that target a specific entity
fn friend_of_dorothy(query: Query<Entity, With<Relation<FriendsWith>>>, dorothy: Res<Dorothy>) {
    let filtered = query
        // Relation filters make the `Relation<T>` in our query type
        // only be for the given targets specified- by default `Relation<T>`
        // will apply to any target
        .filter_relation(
            RelationFilter::<FriendsWith>::target(dorothy.entity)
        )
        // You must call .apply_filters() before you can access the query again
        .apply_filters();

    for source_entity in filtered.iter() {
        println!("{} is friends with Dorothy!", source_entity);
    }
}

// Or even look for some combination of targets!
fn caught_in_the_middle(
    mut query: Query<&mut Stress, With<Relation<FriendsWith>>>,
    dorothy: Res<Dorothy>,
    elphaba: Res<Elphaba>,
) {
    query
        // Note that we can set relation filters even if the relation is in a query filter
        .filter_relation(
            RelationFilter::<FriendsWith>::all_of()
                .target(dorothy.entity)
                .target(elphaba.entity),
        )
        .apply_filters()
        // apply_filters() returns `&mut Query` so we can just iterate straight away
        .for_each(|mut stress| {
            println!("{} is friends with Dorothy and Elphaba!", source_entity);
            stress.val = 1000;
        })
}

```


```rust
// You can use the `Entity` target returned by your relations to fetch data from the 
// target's components by combining it with `query::get()` or `query::get_component<C>()`.
fn debts_come_due(
    mut debt_query: Query<(Entity, &Relation<Owes>)>,
    mut money_query: Query<&mut Money>,
) {
    for (debtor, debt) in debt_query.iter() {
        for (lender, amount_owed) in debt {
            *money_query
                .get(debtor)
                .unwrap()
                .checked_sub(amount_owed)
                .expect("Nice kneecaps you got there...");
            // getting the money on the target of our `Owes` relation!
            *money_query.get(lender).unwrap() += amount_owed;
        }
    }
}

```

### Nuances of relations

Relations between entities are automatically cleaned up when either the source or target entity is removed.
Just like components, you can view which relations were removed in the last frame through the `RemovedRelations<T>` system parameter, which also returns their data.

Each relation kind + target combination is treated like a unique type for the purposes of archetypes.
This makes can be relevant to the performance of your game; accelerating target entity filtering but creating overhead if you have a very large number of distinct targets.
As a result, changing the source or target of a relation can only be done while the app has exclusive access to the world, via `Commands` or in exclusive systems.
Here's an example of how you might do so:

```rust
fn love_potion(
    mut commands: Commands,
    query: Query<Entity, &Relation<Loves>, With<Cute>>,
    player_query: Query<Entity, With<Player>>,
) {
    let player = player.single().unwrap();

    for (victim, former_love) in query.iter() {
        // Only one relation of a given kind can exist between a source and a target
        // (although the opposite direction can exist on the target entity)
        // like always, new relations overwrite existing relations
        // with the same source, kind and target
        for (_, former_lover) in former_love {
            commands
                .entity(victim)
                .change_target::<Loves>(former_lover, player); // This is unethical!!
        }
    }
}

/*

FIXME: This example is a bit borked but if we fix it the we no longer need `move_relation`

fn adoption(
    mut commands: Commands,
    // As with components, you can query for relations that may or may not exist
    query: Query<(Entity, &NewOwner, Option<&Relation<Owns>>), With<Kitten>>,
) {
    for (kitten, new_owner, previous_ownership) in query.iter() {
        // Changing the target or the source will fail silently if no appropriate relation exists
        match previous_ownership {
            // We can change sources by controlling which entity owns the relation
            // move_relation is directly analagous to move_component
            Some(old_owner, _) => commands
                .entity(old_owner)
                .move_relation::<Owns>(new_owner, kitten), // uwu :3
            None => commands
                .entity(new_owner)
                .insert_relation(Owns::default(), kitten), // uwu!!
        }
    }
}
*/

// `change_target` and `move_relation` preserve the relation's data
// This system removes all springs attached to mass 1, and adds them to mass 2 instead
fn reattach_springs(
    mut commands: Commands,
    selected_mass1: Res<FirstSelected>,
    selected_mass2: Res<SecondSelected>,
    query: Query<Relation<Spring>>,
) {
    let m1 = selelected_mass1.entity;
    let m2 = selelected_mass2.entity;
    let springs = query.get(m1);

    // Spring relations are symmetric; each spring is defined by
    // two identical relations that point opposite direction.
    // We need to preserve both during this operation
    for (target_mass, _) in springs {
        // When mass 1 is the source, we must change the source to mass 2
        commands.entity(m1).move_relation::<Spring>(m2, target_mass);

        // When mass 1 is the target, we must change the target to mass 2
        commands.entity(target_mass).change_target::<Spring>(m1, m2);
    }
}

```

### Advanced relation filters

The examples shown above show of the basics of relation filtering, but that's not all they can do.
The most important extension is the ability to filter for multiple entities at once from a `Vec<Entity>`.
In concert with this, `any_of` and `all_of` can be used to collect groups of entities into a single filter.
`any_of` uses **or semantics**, returning any entity if any of the filters are met,
while `all_of` uses **and semantics**, rejecting entities who do not meet every specified filter.

```rust
// In this example, possible moves are stored on each piece
// as a relation to a specific board position
fn next_move(
    mut commands: Commands,
    board_state: Res<BoardState>,
    query: Query<(Entity, &Relation<Move>), With<Piece>>,
) {
    let valid_positions: Vec<Entity> = board_state.compute_valid_positions();

    let filtered = query
        .filter_relation(
            RelationFilter::<Action>::any_of().targets(valid_positions)
        )
        .apply_filters();

    let best_move = valid_moves
        .max_by_key(|(potential_position, _)| potential_position.compute_heuristic());

    if let Some((_, best_move)) = best_move {
        commands.entity(piece).insert_relation::<NextMove>(best_move);
    }
}

// We can use this with either all_of or any_of to get the effect we need
fn all_roads_lead_to_rome(
    mut commands: Commands,
    cities_query: Query<Entity, With<&City>>,
    roads_query: Query<Entity, (With<City>, With<Relation<Road>>)>,
) {
    let all_cities: Vec<Entity> = cities_query.iter().collect();
    roads_query
        .filter_relation(
            RelationFilter::<Road>::all_of().targets(all_cities)
        )
        .apply_filters()
        // The cities still left are connected to all other cities by roads
        // We're tagging them with a Hub marker component so we can find them again quickly
        .map(|city| commands.entity(city).insert(Hub));
}

```

As your filters grow in complexity, it can be useful to filter by the source entity, rather than target entity.
Note that you can get the same sort of effect by using `Query::get`;
this functionality just makes it more ergonomic to specify certain complex relation filters.

```rust
fn paths_to_choose(location: Res<PlayerLocation>, mut query: Query<&mut Relation<Path>>) {
    // We only care about paths that lead away from our current position
    query
        .filter_relation(RelationFilter::<Path>::source(location.entity))
        .apply_filters()
        .map(|_, mut path| {
            path.highlight();
        });

    // You can accomplish the same thing by subsetting your query
    // using query.get, query.single or branching on Entity
    // This is equivalent because queries are composed of a collection of source entities
    let mut paths_from_player_location = query.get_mut(location.entity).unwrap();
    for (path, _) in paths_from_player_location {
        println!(
            "It will take {} minutes to get to {} from here.",
            path.travel_time, path.destination_name
        );
    }
}

// With more complex logic that blends sources and targets,
// the value of this approach is more evident
fn sever_groups(
    mut commands: Commands,
    selection_query: Query<Entity, &Selection>,
    mass_query: Query<Entity, (With<Mass>, With<Relation<Spring>>)>,
) {
    // Our goal is to remove all of the springs
    // connecting these two groups of point masses
    let mut group_1 = Vec::<Entity>::new();
    let mut group_2 = Vec::<Entity>::new();

    for (entity, selection) in selection_query.iter() {
        match selection {
            Selection::Group1 => group_1.push(entity),
            Selection::Group2 => group_2.push(entity),
        }
    }

    let one_to_two = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_1).targets(group_2))
        .apply_filters();

    let two_to_one = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_2).targets(group_1))
        .apply_filters();

    // First we remove the springs that point in one direction
    for (source, springs) in one_to_two.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }

    // And then, because the relation is symmetric, the other
    for (source, springs) in two_to_one.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }
}

```

From time-to-time, we may care about *excluding* entities who have relations to certain targets.
To do so, we set our relation filter on a `Without<Relation<R>>` query parameter.

```rust
// Fraternizing with communists is a strict disqualification!
fn purity_testing(
    mut commands: Commands,
    candidate_query: Query<Entity, (With<Candidate>, Without<Relation<FriendsWith>>)>,
    communist_query: Query<Entity, With<Communist>>,
) {
    let communists = communist_query.iter().collect();

    // Normally, Without relation filters disqualify all entities with that relation.
    // But by narrowing them to only affect certain sources or targets,
    // we can control which entities are affected!
    candidate_query
        .filter_relation(
            // Even a single communist friend is disqualifying!
            RelationFilter::<FriendsWith>::any_of().targets(communists),
        )
        .apply_filters()
        // Candidates who are not friends with any communists have been filtered
        // Leaving only suspect candidates to be stripped of their candidacy
        .for_each(|candidate| commands.entity(candidate).remove::<Candidate>());

    // Note that candidates with no friends at all are always excluded from the query,
    // sparing them from our arbitrary interrogation
}

```

We can filter on multiple types of relations at once by chaining together our `.filter_relation` methods before we call `.apply_filters()`.

```rust
fn frenemies(
    query: Query<Entity, (With<Relation<FriendsWith>>, With<Relation<EnemiesWith>>)>,
    player: Res<Player>,
) {
    query
        .filter_relation(RelationFilter::<FriendsWith>::target(player.entity))
        .filter_relation(RelationFilter::<EnemiesWith>::target(player.entity))
        .for_each(|entity| println!("{} is frenemies with the player!", entity));
}
```

Filters chained in this way will operate on the restricted list produces by the previous filter (following "and semantics").
We can even combine positive and negative filters by including multiple copies of the same relation in our query, and then specifying which relation we're filtering using the second type parameter.
This advanced technique is helpful when you really need to be specific, as shown in these examples:

```rust
fn herbivores(
    mut commands: Commands,
    plants_query: Query<Entity, With<Plant>>,
    animals_query: Query<Enity, With<Animal>>,
    species_query: Query<Entity, (With<Relation<Consumes>>, Without<Relation<Consumes>>)>,
) {
    let plants = plants_query.iter().collect();
    let animals = animals_query.iter().collect();

    // The generics on `filter_relation` are normally inferred but we can manually
    // specify them to disambiguate which `Relation<T>` we are filtering in the query
    species_query
        // The second type parameter controls which relation filter of that type is being modified
        .filter_relation::<Consumes, InFilter<{ 0 }>>(RelationFilter::any_of().targets(plants))
        // Because this is a without filter, this means that we exclude all entities
        // with even one Consumes relation that targets an animal
        .filter_relation::<Consumes, InFilter<{ 1 }>>(RelationFilter::any_of().targets(animals))
        .for_each(|entity| commands.entity(entity).insert(Herbivore));
}

```

### Grouping entities

By targeting a common entity, relations work fantastically as an ergonomic way to group entities.
This pattern acts as a nice complement to marker components when either:

1. You want to define the *kind* of group, rather than just membership.
2. The number of different groups you need is unknowable at compile time.

Here are some concrete examples where this pattern works well:

- you're storing multiple parts (such as meshes or sprites with their own `Transform`s) of the same in-game-object across multiple entities
- you want to dynamically group units together, like in a real-time-strategy game
- the effects created by a single entity, like the bullets they're spewing out
- you want a way to mark the controller of a unit in a queryable way, but the number of possible controllers is unknown at compile time
- you have multiple cameras, and want to mark which frustum each entity is in so they can be culled independently

### Entity graphs

Unsurprisingly, you can build all manner of complex entity graphs using relations.
Before you begin to rip out your resources with graph data structures storing `Entity` keys,
pause to consider that this is, in fact, `*min*-relations`.
Performance will be limited, APIs will be lacking,
and you will bemoan your inability to specify information about the global properties of the graph.

However, even relations do have three critical advantages over the **external graph data structure pattern**:

1. You can readily store data in each edge of your directed graph.
2. You can quickly filter for the source or target entities (nodes) of each relation (edge).
3. You don't have to think about synchronization.

These advantages make even `min-relations` a compelling alternative to both the Entity-in-component and external graph data structure patterns if:

1. You don't care about performance.
2. You don't need to perform complex traversals.
3. You're operating on general graphs that can contain cycles and self-referential edges.
4. You don't want to worry about synchronization bugs.

Let's take a look at how you could use relations to build an API for a tree-shaped graph that looks suspiciously like Bevy 0.5's parent-child hierarchy.

## Reference-level explanation

TODO: Boxy explains the magic.

### Data Storage Model

### The Second Relation Filter Type Parameter

## Drawbacks

1. Introduces another core abstraction to users that they will have to learn.
Rewriting our parent-child code in terms of relations is important, but will introduce the concept to users quite quickly.
2. Heavy use of relations can seriously fragment archetypes. We need to be careful that this doesn't badly impact our performance.
3. `move_component`, `move_relation` and `change_target` only work on `Clone` components / relations due to the need to temporarily be able to access removed component data.

## Rationale and alternatives

### Why do relations fragment archetypes?

TODO: Boxy explains her decisions.

### Why are relations stored in the ECS?

TODO: Boxy help!

### Why aren't relations described as a special type of components?

They don't behave in exactly the same ways,
and describing them as such is confusing to beginners.

Under the hood, the existing implementation models

TODO: add examples of the differences

## Why do we need the builder pattern for filtering relations?

TODO: Boxy explains perf implications

### Why don't we just store Entities in components?

See [Motivation].

### Why don't we just store graph data structures in a Resource?

See [Motivation] *and* [Entity Graphs].

### Why do we need arbitrarily complex relation filters??

Relation filters take advantage of archetypes to be much faster and more ergonomic than doing the same operation by naive iteration.
In order to ensure that it can always be used effectively, we need a genuinely complete DSL,
in addition to the convenient methods discussed first.

The API for compound relation filters is simple to implement and easily ignored.

### Why is my favorite feature relegated to Future Work?

It's already many thousands of words long, and the associated PR has thousands of lines of code.
Done is better than perfect, and we had to draw the line somewhere.
As with all things, we can gradually improve this over time.

The set of features expressed here is useful enough to be a clear improvement over the existing patterns in at least some use cases.
There's a lot of value in getting this feature in front of users to see the response at scale.

### Why is the implementation for this so *slow*?

Many uses of relations are not performance-critical,
and the value of a nice API and automatic handling of data synchronization far outweighs any performance cost for those users.

We can fix the perf once we have a better understanding of the APIs we need to support,
and a tangible implementation to benchmark against.
There's also nothing stopping you from doing things the old ways in your elaborate graph-traversing systems.

See also: the question directly above.

## Prior art

[`Flecs`](https://github.com/SanderMertens/flecs), an advanced C++ ECS framework, has a similar feature, which they call "relationships".
These are somewhat different, they use an elaborate query domain-specific language along with being more feature rich.
You can read more about them in the [docs](https://github.com/SanderMertens/flecs/blob/master/docs/Queries.md) or [corresponding PR](https://github.com/SanderMertens/flecs/pull/358).

You can, of course, build similar data structures using the ECS itself.
Here's a look at the complexities involved in doing so in [EnTT](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/).

## Unresolved questions

1. How do we put relations in Bundles?
2. What do we actually call [`Noitaler`](#bikeshed)? Candidates so far are:
   1. `ReverseRelation`
   2. `Inverse`
   3. `CoRelation`
   4. `TargetOf`
3. Do we want a macro to make complex calls easier?
4. Bikeshed: what argument order do we want for `change_target` and `move_relation`? Currently:
   1. `commands.entity(source).change_target(old_target, new_target)`
   2. `commands.entity(old_source).change_target(new_source, target)`

```rust
macro_rules Relation {
  (this $kind:ty *) => Relation<$kind>,
  (* $kind:ty this) => Noitaler<$kind>,
}
```

## Future possibilities

Relation enhancements beyond the scope of `min-relations`:

1. Convenience functions of all sorts:
    1. `remove_relations::<T>`, for removing a `Vec<Entity>` of relations at once.
    2. `clear_relation::<T>`, for removing all relations of that type at once.
    3. Generalized `despawn_recursive::<T>`, parameterizing by relation kind.
2. Sugar for accessing data on the target entity in a single query.
Proposed API: `Relation<T: Component, Q: WorldQuery>`
3. Graph shape guarantees (e.g tree, acyclic, 1-depth, non-self-referential).
Likely implemented using archetype invariants.
4. Graph traversals API: breadth-first, depth-first, root of tree etc.
5. Arbitrary target types instead of just `Entity` combined with `KindedEntity` proposal to ensure your targets are valid.
6. Automatically symmetric (or anti-symmetric) relations to model undirected edges.
7. `Noitaler`*, for relations that point in the opposite direction.
8. Streaming iters to allow for queries like: `Query<&mut Money, Relation<Owes, &mut Money>>` and potentially use invariants on graph shape to allow for skipping soundness checks for the aliasing `&mut Money`'s
9. Assorted performance optimizations. For example:
   1. Reducing the cost of having many archetypes.
   2. Relation storage type options to cause archetype fragmentation based on whether *any* relations of that type are present.
   3. Index-backed relations look-up.
10. Relative ordering between relations of the same kind on the same entity.
This would enable the `Styles` proposal from #1 to use relations.
11. Compound relation filters, letting you nest logic arbitrarily deep. These are just sugar / perf for very advanced users, and can wait for a while.
12. \[Controversial\] A full graph constraint solver DSL ala [Flecs](https://github.com/SanderMertens/flecs) for advanced querying.
13. Entity-to-entity event channels using relations (blocked on [bevy #2116](https://github.com/bevyengine/bevy/pull/2116)

Relation applications in the engine:

1. Replace existing parent-child API.
2. Applications for UI: styling, widget wiring etc.
3. Frustum culling with multiple cameras.

<b id="bikeshed">AUTHOR'S NOTE</b>: Much like `yeet`, `Noitaler` is the bikeshed-avoidance name for the inverse of `Relation`, and will never be used.

EDITOR's NOTE: `Noitaler` may very much be used :3
// target's components by combining it with `query::get()` or `query::get_component<C>()`.
fn debts_come_due(
    mut debt_query: Query<(Entity, &Relation<Owes>)>,
    mut money_query: Query<&mut Money>,
) {
    for (debtor, debt) in debt_query.iter() {
        for (lender, amount_owed) in debt {
            *money_query
                .get(debtor)
                .unwrap()
                .checked_sub(amount_owed)
                .expect("Nice kneecaps you got there...");
            // getting the money on the target of our `Owes` relation!
            *money_query.get(lender).unwrap() += amount_owed;
        }
    }
}

```

### Nuances of relations

Relations between entities are automatically cleaned up when either the source or target entity is removed.
Just like components, you can view which relations were removed in the last frame through the `RemovedRelations<T>` system parameter, which also returns their data.

Each relation kind + target combination is treated like a unique type for the purposes of archetypes.
This makes can be relevant to the performance of your game; accelerating target entity filtering but creating overhead if you have a very large number of distinct targets.
As a result, changing the source or target of a relation can only be done while the app has exclusive access to the world, via `Commands` or in exclusive systems.
Here's an example of how you might do so:

```rust
fn love_potion(
    mut commands: Commands,
    query: Query<Entity, &Relation<Loves>, With<Cute>>,
    player_query: Query<Entity, With<Player>>,
) {
    let player = player.single().unwrap();

    for (victim, former_love) in query.iter() {
        // Only one relation of a given kind can exist between a source and a target
        // (although the opposite direction can exist on the target entity)
        // like always, new relations overwrite existing relations
        // with the same source, kind and target
        for (_, former_lover) in former_love {
            commands
                .entity(victim)
                .change_target::<Loves>(former_lover, player); // This is unethical!!
        }
    }
}

/*

FIXME: This example is a bit borked but if we fix it the we no longer need `move_relation`

fn adoption(
    mut commands: Commands,
    // As with components, you can query for relations that may or may not exist
    query: Query<(Entity, &NewOwner, Option<&Relation<Owns>>), With<Kitten>>,
) {
    for (kitten, new_owner, previous_ownership) in query.iter() {
        // Changing the target or the source will fail silently if no appropriate relation exists
        match previous_ownership {
            // We can change sources by controlling which entity owns the relation
            // move_relation is directly analagous to move_component
            Some(old_owner, _) => commands
                .entity(old_owner)
                .move_relation::<Owns>(new_owner, kitten), // uwu :3
            None => commands
                .entity(new_owner)
                .insert_relation(Owns::default(), kitten), // uwu!!
        }
    }
}
*/

// `change_target` and `move_relation` preserve the relation's data
// This system removes all springs attached to mass 1, and adds them to mass 2 instead
fn reattach_springs(
    mut commands: Commands,
    selected_mass1: Res<FirstSelected>,
    selected_mass2: Res<SecondSelected>,
    query: Query<Relation<Spring>>,
) {
    let m1 = selelected_mass1.entity;
    let m2 = selelected_mass2.entity;
    let springs = query.get(m1);

    // Spring relations are symmetric; each spring is defined by
    // two identical relations that point opposite direction.
    // We need to preserve both during this operation
    for (target_mass, _) in springs {
        // When mass 1 is the source, we must change the source to mass 2
        commands.entity(m1).move_relation::<Spring>(m2, target_mass);

        // When mass 1 is the target, we must change the target to mass 2
        commands.entity(target_mass).change_target::<Spring>(m1, m2);
    }
}

```

### Advanced relation filters

The examples shown above show of the basics of relation filtering, but that's not all they can do.
The most important extension is the ability to filter for multiple entities at once from a `Vec<Entity>`.
In concert with this, `any_of` and `all_of` can be used to collect groups of entities into a single filter.
`any_of` uses **or semantics**, returning any entity if any of the filters are met,
while `all_of` uses **and semantics**, rejecting entities who do not meet every specified filter.

```rust
// In this example, possible moves are stored on each piece
// as a relation to a specific board position
fn next_move(
    mut commands: Commands,
    board_state: Res<BoardState>,
    query: Query<(Entity, &Relation<Move>), With<Piece>>,
) {
    let valid_positions: Vec<Entity> = board_state.compute_valid_positions();

    let filtered = query
        .filter_relation(
            RelationFilter::<Action>::any_of().targets(valid_positions)
        )
        .apply_filters();

    let best_move = valid_moves
        .max_by_key(|(potential_position, _)| potential_position.compute_heuristic());

    if let Some((_, best_move)) = best_move {
        commands.entity(piece).insert_relation::<NextMove>(best_move);
    }
}

// We can use this with either all_of or any_of to get the effect we need
fn all_roads_lead_to_rome(
    mut commands: Commands,
    cities_query: Query<Entity, With<&City>>,
    roads_query: Query<Entity, (With<City>, With<Relation<Road>>)>,
) {
    let all_cities: Vec<Entity> = cities_query.iter().collect();
    roads_query
        .filter_relation(
            RelationFilter::<Road>::all_of().targets(all_cities)
        )
        .apply_filters()
        // The cities still left are connected to all other cities by roads
        // We're tagging them with a Hub marker component so we can find them again quickly
        .map(|city| commands.entity(city).insert(Hub));
}

```

As your filters grow in complexity, it can be useful to filter by the source entity, rather than target entity.
Note that you can get the same sort of effect by using `Query::get`;
this functionality just makes it more ergonomic to specify certain complex relation filters.

```rust
fn paths_to_choose(location: Res<PlayerLocation>, mut query: Query<&mut Relation<Path>>) {
    // We only care about paths that lead away from our current position
    query
        .filter_relation(RelationFilter::<Path>::source(location.entity))
        .apply_filters()
        .map(|_, mut path| {
            path.highlight();
        });

    // You can accomplish the same thing by subsetting your query
    // using query.get, query.single or branching on Entity
    // This is equivalent because queries are composed of a collection of source entities
    let mut paths_from_player_location = query.get_mut(location.entity).unwrap();
    for (path, _) in paths_from_player_location {
        println!(
            "It will take {} minutes to get to {} from here.",
            path.travel_time, path.destination_name
        );
    }
}

// With more complex logic that blends sources and targets,
// the value of this approach is more evident
fn sever_groups(
    mut commands: Commands,
    selection_query: Query<Entity, &Selection>,
    mass_query: Query<Entity, (With<Mass>, With<Relation<Spring>>)>,
) {
    // Our goal is to remove all of the springs
    // connecting these two groups of point masses
    let mut group_1 = Vec::<Entity>::new();
    let mut group_2 = Vec::<Entity>::new();

    for (entity, selection) in selection_query.iter() {
        match selection {
            Selection::Group1 => group_1.push(entity),
            Selection::Group2 => group_2.push(entity),
        }
    }

    let one_to_two = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_1).targets(group_2))
        .apply_filters();

    let two_to_one = mass_query
        .filter_relation(RelationFilter::<Spring>::any_of().sources(group_2).targets(group_1))
        .apply_filters();

    // First we remove the springs that point in one direction
    for (source, springs) in one_to_two.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }

    // And then, because the relation is symmetric, the other
    for (source, springs) in two_to_one.iter() {
        for (target, _) in springs {
            commands.entity(source).remove_relation::<Spring>(target)
        }
    }
}

```

From time-to-time, we may care about *excluding* entities who have relations to certain targets.
To do so, we set our relation filter on a `Without<Relation<R>>` query parameter.

```rust
// Fraternizing with communists is a strict disqualification!
fn purity_testing(
    mut commands: Commands,
    candidate_query: Query<Entity, (With<Candidate>, Without<Relation<FriendsWith>>)>,
    communist_query: Query<Entity, With<Communist>>,
) {
    let communists = communist_query.iter().collect();

    // Normally, Without relation filters disqualify all entities with that relation.
    // But by narrowing them to only affect certain sources or targets,
    // we can control which entities are affected!
    candidate_query
        .filter_relation(
            // Even a single communist friend is disqualifying!
            RelationFilter::<FriendsWith>::any_of().targets(communists),
        )
        .apply_filters()
        // Candidates who are not friends with any communists have been filtered
        // Leaving only suspect candidates to be stripped of their candidacy
        .for_each(|candidate| commands.entity(candidate).remove::<Candidate>());

    // Note that candidates with no friends at all are always excluded from the query,
    // sparing them from our arbitrary interrogation
}

```

We can filter on multiple types of relations at once by chaining together our `.filter_relation` methods before we call `.apply_filters()`.

```rust
fn frenemies(
    query: Query<Entity, (With<Relation<FriendsWith>>, With<Relation<EnemiesWith>>)>,
    player: Res<Player>,
) {
    query
        .filter_relation(RelationFilter::<FriendsWith>::target(player.entity))
        .filter_relation(RelationFilter::<EnemiesWith>::target(player.entity))
        .for_each(|entity| println!("{} is frenemies with the player!", entity));
}
```

Filters chained in this way will operate on the restricted list produces by the previous filter (following "and semantics").
We can even combine positive and negative filters by including multiple copies of the same relation in our query, and then specifying which relation we're filtering using the second type parameter.
This advanced technique is helpful when you really need to be specific, as shown in these examples:

```rust
fn herbivores(
    mut commands: Commands,
    plants_query: Query<Entity, With<Plant>>,
    animals_query: Query<Enity, With<Animal>>,
    species_query: Query<Entity, (With<Relation<Consumes>>, Without<Relation<Consumes>>)>,
) {
    let plants = plants_query.iter().collect();
    let animals = animals_query.iter().collect();

    // The generics on `filter_relation` are normally inferred but we can manually
    // specify them to disambiguate which `Relation<T>` we are filtering in the query
    species_query
        // The second type parameter controls which relation filter of that type is being modified
        .filter_relation::<Consumes, InFilter<{ 0 }>>(RelationFilter::any_of().targets(plants))
        // Because this is a without filter, this means that we exclude all entities
        // with even one Consumes relation that targets an animal
        .filter_relation::<Consumes, InFilter<{ 1 }>>(RelationFilter::any_of().targets(animals))
        .for_each(|entity| commands.entity(entity).insert(Herbivore));
}

```

### Grouping entities

By targeting a common entity, relations work fantastically as an ergonomic way to group entities.
This pattern acts as a nice complement to marker components when either:

1. You want to define the *kind* of group, rather than just membership.
2. The number of different groups you need is unknowable at compile time.

Here are some concrete examples where this pattern works well:

- you're storing multiple parts (such as meshes or sprites with their own `Transform`s) of the same in-game-object across multiple entities
- you want to dynamically group units together, like in a real-time-strategy game
- the effects created by a single entity, like the bullets they're spewing out
- you want a way to mark the controller of a unit in a queryable way, but the number of possible controllers is unknown at compile time
- you have multiple cameras, and want to mark which frustum each entity is in so they can be culled independently

### Entity graphs

Unsurprisingly, you can build all manner of complex entity graphs using relations.
Before you begin to rip out your resources with graph data structures storing `Entity` keys,
pause to consider that this is, in fact, `*min*-relations`.
Performance will be limited, APIs will be lacking,
and you will bemoan your inability to specify information about the global properties of the graph.

However, even relations do have three critical advantages over the **external graph data structure pattern**:

1. You can readily store data in each edge of your directed graph.
2. You can quickly filter for the source or target entities (nodes) of each relation (edge).
3. You don't have to think about synchronization.

These advantages make even `min-relations` a compelling alternative to both the Entity-in-component and external graph data structure patterns if:

1. You don't care about performance.
2. You don't need to perform complex traversals.
3. You're operating on general graphs that can contain cycles and self-referential edges.
4. You don't want to worry about synchronization bugs.

Let's take a look at how you could use relations to build an API for a tree-shaped graph that looks suspiciously like Bevy 0.5's parent-child hierarchy.

## Reference-level explanation

TODO: Boxy explains the magic.

### Data Storage Model

### The Second Relation Filter Type Parameter

## Drawbacks

1. Introduces another core abstraction to users that they will have to learn.
Rewriting our parent-child code in terms of relations is important, but will introduce the concept to users quite quickly.
2. Heavy use of relations can seriously fragment archetypes. We need to be careful that this doesn't badly impact our performance.
3. `move_component`, `move_relation` and `change_target` only work on `Clone` components / relations due to the need to temporarily be able to access removed component data.

## Rationale and alternatives

### Why do relations fragment archetypes?

TODO: Boxy explains her decisions.

### Why are relations stored in the ECS?

TODO: Boxy help!

### Why aren't relations described as a special type of components?

They don't behave in exactly the same ways,
and describing them as such is confusing to beginners.

Under the hood, the existing implementation models

TODO: add examples of the differences

## Why do we need the builder pattern for filtering relations?

TODO: Boxy explains perf implications

### Why don't we just store Entities in components?

See [Motivation].

### Why don't we just store graph data structures in a Resource?

See [Motivation] *and* [Entity Graphs].

### Why do we need arbitrarily complex relation filters??

Relation filters take advantage of archetypes to be much faster and more ergonomic than doing the same operation by naive iteration.
In order to ensure that it can always be used effectively, we need a genuinely complete DSL,
in addition to the convenient methods discussed first.

The API for compound relation filters is simple to implement and easily ignored.

### Why is my favorite feature relegated to Future Work?

It's already many thousands of words long, and the associated PR has thousands of lines of code.
Done is better than perfect, and we had to draw the line somewhere.
As with all things, we can gradually improve this over time.

The set of features expressed here is useful enough to be a clear improvement over the existing patterns in at least some use cases.
There's a lot of value in getting this feature in front of users to see the response at scale.

### Why is the implementation for this so *slow*?

Many uses of relations are not performance-critical,
and the value of a nice API and automatic handling of data synchronization far outweighs any performance cost for those users.

We can fix the perf once we have a better understanding of the APIs we need to support,
and a tangible implementation to benchmark against.
There's also nothing stopping you from doing things the old ways in your elaborate graph-traversing systems.

See also: the question directly above.

## Prior art

[`Flecs`](https://github.com/SanderMertens/flecs), an advanced C++ ECS framework, has a similar feature, which they call "relationships".
These are somewhat different, they use an elaborate query domain-specific language along with being more feature rich.
You can read more about them in the [docs](https://github.com/SanderMertens/flecs/blob/master/docs/Queries.md) or [corresponding PR](https://github.com/SanderMertens/flecs/pull/358).

You can, of course, build similar data structures using the ECS itself.
Here's a look at the complexities involved in doing so in [EnTT](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/).

## Unresolved questions

1. How do we put relations in Bundles?
2. What do we actually call [`Noitaler`](#bikeshed)? Candidates so far are:
   1. `ReverseRelation`
   2. `Inverse`
   3. `CoRelation`
   4. `TargetOf`
3. Do we want a macro to make complex calls easier?
4. Bikeshed: what argument order do we want for `change_target` and `move_relation`? Currently:
   1. `commands.entity(source).change_target(old_target, new_target)`
   2. `commands.entity(old_source).change_target(new_source, target)`

```rust
macro_rules Relation {
  (this $kind:ty *) => Relation<$kind>,
  (* $kind:ty this) => Noitaler<$kind>,
}
```

## Future possibilities

Relation enhancements beyond the scope of `min-relations`:

1. Convenience functions of all sorts:
    1. `remove_relations::<T>`, for removing a `Vec<Entity>` of relations at once.
    2. `clear_relation::<T>`, for removing all relations of that type at once.
    3. Generalized `despawn_recursive::<T>`, parameterizing by relation kind.
2. Sugar for accessing data on the target entity in a single query.
Proposed API: `Relation<T: Component, Q: WorldQuery>`
3. Graph shape guarantees (e.g tree, acyclic, 1-depth, non-self-referential).
Likely implemented using archetype invariants.
4. Graph traversals API: breadth-first, depth-first, root of tree etc.
5. Arbitrary target types instead of just `Entity` combined with `KindedEntity` proposal to ensure your targets are valid.
6. Automatically symmetric (or anti-symmetric) relations to model undirected edges.
7. `Noitaler`*, for relations that point in the opposite direction.
8. Streaming iters to allow for queries like: `Query<&mut Money, Relation<Owes, &mut Money>>` and potentially use invariants on graph shape to allow for skipping soundness checks for the aliasing `&mut Money`'s
9. Assorted performance optimizations. For example:
   1. Reducing the cost of having many archetypes.
   2. Relation storage type options to cause archetype fragmentation based on whether *any* relations of that type are present.
   3. Index-backed relations look-up.
10. Relative ordering between relations of the same kind on the same entity.
This would enable the `Styles` proposal from #1 to use relations.
11. Compound relation filters, letting you nest logic arbitrarily deep. These are just sugar / perf for very advanced users, and can wait for a while.
12. \[Controversial\] A full graph constraint solver DSL ala [Flecs](https://github.com/SanderMertens/flecs) for advanced querying.
13. Entity-to-entity event channels using relations (blocked on [bevy #2116](https://github.com/bevyengine/bevy/pull/2116)

Relation applications in the engine:

1. Replace existing parent-child API.
2. Applications for UI: styling, widget wiring etc.
3. Frustum culling with multiple cameras.

<b id="bikeshed">AUTHOR'S NOTE</b>: Much like `yeet`, `Noitaler` is the bikeshed-avoidance name for the inverse of `Relation`, and will never be used.

EDITOR's NOTE: `Noitaler` may very much be used :3

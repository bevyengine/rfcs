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
commands.entity(alice).remove_relation(FriendsWith, boxy) // not uwu :(

// You can use direct World access!
world.entity_mut(alice).insert_relation(FriendsWith, boxy); // uwu:3

// Relations are directed; a relation in one direction does not imply the same in the other direction
commands.entity(boxy).insert_relation(FriendsWith, alice); // one way friendship sad :(

// You can mix different relation kinds on the same entity!
commands.entity(alice).insert_relation(Owes(9999), boxy); // :)))))

// You can add mulitple relations of the same kind to a single source entity!
let cart  = world.spawn().id();
commands.entity(alice).insert_relation(FriendsWith, cart); // Hi!
```

Once your relations are in place, you'll want to access them in some form.
Just like with ordinary components, you can request them in your queries:

```rust
// Relations can be queried for immutably
fn friendship_is_magic(mut query: Query<(&mut Magic, &Relation<FriendsWith>), With<Pony>>) {
    // Each entity can have their own friends
    for (mut magic, friends) in query.iter_mut(){
        // Relations return an iterator when unpacked
        magic.power += friends.count();
    }
}

// Or you can request mutable access to modify the relation's data
fn interest(query: Query<&mut Relation<Owes>>){
    for debts in query.iter(){
        for (owed_to, amount_owed) in debts {
            println!("we owe {} some money", owed_to);
            amount_owed *= 1.05; 
        }
    }
}

// You can look for relations with a specific kind, source and target
fn unrequited_love(query: Query<(Entity, &Relation<Loves>)>){
    for (lover, crushes) in query.iter(){
        for (crush, _) in crushes {
            let reciprocal_crush = query.get_relation::<Loves>(crush, lover);
            reciprocal_crush.expect("Surely they must feel the same way!");
        }
    }
}

// You can query for entities that target a specific entity
fn friend_of_dorothy(
    query: Query<Entity, With<Relation<FriendsWith>>>, 
    dorothy: Res<Dorothy>,
) {
    let filtered = query.filter_relation::<FriendsWith, _>(
        RelationFilter::target(dorothy.entity)
    // .build() causes the relation filters to be applied
    ).build();

    for source_entity in filtered.iter(){
        println!("{} is friends with Dorothy!", source_entity);
    }
}

// Or even look for some combination of targets!
fn caught_in_the_middle(mut query: Query<&mut Stress, With<Relation<FriendsWith>>>,
    dorothy: Res<Dorothy>,
    elphaba: Res<Elphaba>){

    // Note that we can set relation filters even if the relation is in a query filter
    query.filter_relation::<FriendsWith, _>(
        RelationFilter::all_of()
        .target(dorothy.entity)
        .target(elphaba.entity)
    // You can directly chain this as filter_relations returns &mut Query
    ).build().for_each(|mut stress| {
        println!("{} is friends with Dorothy and Elphaba!", source_entity);
        stress.val = 1000; 
    })
}

// Query filters work too!
fn not_alone(
    commands: mut Commands, 
    query: Query<Entity, Without<Relation<FriendsWith>>>,
    puppy: Res<Puppy>
) {
    for lonely_entity in query.iter(){
        commands.insert_relation(FriendsWith, puppy.entity);
    }
}

// So does change detection with Added and Changed!
fn new_friends(query: Query<&mut Excitement, Added<Relation<FriendsWith>>>){
    query.for_each(|(mut excitement, new_friends)| {
        excitement.value += 10 * new_friends.count();
    });
}

```

You can use the `Entity` returned by your relations to fetch data from the target's components by combining it with `query::get()` or `query::get_component<C>()`.

```rust
fn debts_come_due(
    mut debt_query: Query<(Entity, &Relation<Owes>)>, 
    mut money_query: Query<&mut Money>,
) {
    for (debtor, debt) in debt.query() {
        for (lender, amount_owed) in debt {
            *money_query.get(debtor).unwrap()
                .checked_sub(amount_owed)
                .expect("Nice kneecaps you got there...");
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
    player_query: Query<Entity, With<Player>>){
    
    let player = player.single().unwrap();

    for victim, former_love in query.iter(){
        // Only one relation of a given kind can exist between a source and a target;
        // like always, new relations overwrite existing relations
        // with the same source, kind and target
        for (_, former_lover) in former_love {
            commands.entity(victim).change_target::<Loves>(former_lover, player); // This is unethical!!
        }
    }
}

fn adoption(
    mut commands: Commands,
    // As with components, you can query for relations that may or may not exist
    query: Query<(Entity, &NewOwner, Option<&Relation<Owns>>), With<Kitten>>,
) {
   for (kitten, new_owner, previous_ownership) in query.iter(){
       // Changing the target or the source will fail silently if no appropriate relation exists
       match previous_ownership {
           // We can change sources by controlling which entity owns the relation
           // move_relation is directly analagous to move_component
           Some(old_owner, _) => commands.entity(old_owner).move_relation::<Owns>(new_owner, kitten); // uwu :3
           None => commands.entity(new_owner).insert_relation(Owns::default(), kitten); // uwu!!
       }
   } 
}

// `change_target` and `move_relation` preserve the relation's data
// This system removes all springs attached to mass 1, and adds them to mass 2 instead
fn reattach_springs(mut commands: Commands,
                   selected_mass1: Res<FirstSelected>,
                   selected_mass2: Res<SecondSelected>,
                   query: Query<Relation<Spring>>
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

TODO: Complete examples in this section

For those cases where previous examples weren't enough, there are several additional API extensions that you might want to use. Using those, Relation filters can be combined in arbitrarily complex ways.

The first of these is the ability to filter by the source entity, rather than target entity.
Note that you can get the same sort of effect by using `Query::get`;
this functionality just makes it more ergonomic to specify complex relation filters.

```rust
fn 

```

Next, you may wish to operate over entire groups of entities at once:

```rust
fn 

```

From time-to-time, we may care about *excluding* entities who have relations to certain targets.
To do so, we set our relation filter on a `Without<Relation<R>>` query parameter.

```rust
fn blackball(){

}

```

We can even combine positive and negative filters by including multiple copies

`any_of` and `all_of` can be used to collect groups of entities into a single filter.
`any_of` uses **or semantics**, returning any entity if any of the filters are met,
while `all_of` uses **and semantics**, rejecting entities who do not meet every specified filter.

```rust
fn 

```

We can filter on multiple types of relations at once by chaining together our `.filter_relation` methods before we call `.build()`.

```rust
fn 

```

Filters chained in this way will operate on the restricted list produces by the previous filter (following "and semantics").

Finally, for when you have *truly* complex relation filtering needs, you can turn to **compound relation filters**.
`RelationFilter::any_of` and `RelationFilter::all_of` can combine *other* relation filters,
allowing you to nest your logic arbitrarily deep.
Let's look at a relatively simple example of that.

```rust

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

Let's examine the real-time strategy group example hands-on:

```rust

```

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
You read more about them in the [corresponding PR](https://github.com/SanderMertens/flecs/pull/358).

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

1. Sugar for accessing data on the target entity in a single query.
Proposed API: `Relation<T: Component, Q: WorldQuery>`
2. Graph shape guarantees (e.g tree, acyclic, 1-depth, non-self-referential).
Likely implemented using archetype invariants.
3. Graph traversals API: breadth-first, depth-first, root of tree etc.
4. Arbitrary target types instead of just `Entity` combined with `KindedEntity` proposal to ensure your targets are valid.
5. Automatically symmetric (or anti-symmetric) relations to model undirected edges.
6. `Noitaler`*, for relations that point in the opposite direction.
7. Streaming iters to allow for queries like: `Query<&mut Money, Relation<Owes, &mut Money>>` and potentially use invariants on graph shape to allow for skipping soundness checks for the aliasing `&mut Money`'s
8. Assorted performance optimizations. For example:
   1. Reducing the cost of having many archetypes.
   2. Relation storage type options to cause archetype fragmentation based on whether *any* relations of that type are present.
   3. Index-backed relations look-up.
9. Relative ordering between relations of the same kind on the same entity.
This would enable the `Styles` proposal from #1 to use relations.
10. Generalized `despawn_recursive` by parameterizing on relation type.
11. \[Controversial\] A full graph constraint solver DSL ala [Flecs](https://github.com/SanderMertens/flecs) for advanced querying.

Relation applications in the engine:

1. Replace existing parent-child API.
2. Applications for UI: styling, widget wiring etc.
3. Frustum culling with multiple cameras.

<b id="bikeshed">AUTHOR'S NOTE</b>: Much like `yeet`, `Noitaler` is the bikeshed-avoidance name for the inverse of `Relation`, and will never be used.

EDITOR's NOTE: `Noitaler` may very much be used :3

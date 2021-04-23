# Feature Name: `min-relations`

// FIXME
// Make sure to talk about these:
// - Reverse relations.
// - Or support for add_target_filter.

## Summary

Relations connect entities to each other, storing data in these edges. This RFC covers the minimal version of this feature, allowing for basic entity grouping and simple game logic extensions.

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

## Guide-level explanation

**Relations** are components that connect entities together, pointing from the entity they are stored on to the entity that they target.
Just like other components, they can store effectively arbitrary data, or they can be data-less unit structs that serve as **markers**.

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

// Relations are directed
commands.entity(boxy).insert_relation(FriendsWith, alice); // one way friendship sad oh wait you reinserted your one nvm rip my joke :(

// You can mix different relation kinds on the same entity!
world.commands.entity(alice).insert_relation(Owes(9999), boxy); // :)))))

// You can add mulitple relations of the same kind to a single source entity!
let cart  = world.spawn().id();
world.commands.entity(alice).insert_relation(FriendsWith, cart); // Hi!
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

// Or you can request mutable access to modify their data
fn interest(query: Query<&mut Relation<Owes>>){
	for debts in query.iter(){
		for (owed_to, amount_owed) in debts {
			println!("we owe {} some money", owed_to);
            amount_owed *= 1.05; 
		}
	}
}

// You can query for entities that target a specific entity
fn friend_of_dorothy(
    mut query: Query<Entity, With<Relation<FriendsWith>>>, 
    dorothy: Res<Dorothy>,
) {
	// Setting relation filters requires that the query is mutable
	query.set_relation_filters(
        RelationFilters::new()
            .add_target_filter::<FriendsWith, _>(dorothy.entity)
    );

	for source_entity in query.iter(){
		println!("{} is friends with Dorothy!", source_entity);
	}

    // RelationFilters are reset at the end of the system, make sure to set them every time your system runs!
	// You can also reset this manually by applying a new `RelationFilters` to the query
	// They override existing relation filters, rather than adding to them
}

// Or even look for some combination of targets!
fn caught_in_the_middle(mut query: Query<&mut Stress, With<Relation<FriendsWith>>>,
	dorothy: Res<Dorothy>,
	elphaba: Res<Elphaba>){
	
	// You can directly chain this as set_relation_filters returns &mut Query
	query.set_relation_filters(
        RelationFilters::new()
            .add_target_filter::<FriendsWith, _>(dorothy.entity)
			.add_target_filter::<FriendsWith, _>(elphaba.entity)
    ).for_each(|mut stress| {
		stress.val = 1000; 
	})
}

/// If we want to combine targets in an "or" fashion, we need a slightly different syntax
TODO: write example

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
	query.for_each(|mut excitement|{excitement.value += 10});
}

```

Sometimes, we may instead want to look for "entities that are targets of some relation".
We can do so by iterating over all entities with that relation, and then filtering by target.

```rust
fn all_children(
    player_query: Query<Entity, With<Player>>,
    child_query: Query<&Name, (With<Relation<ChildOf>, Without<Player>)>>,
) {
	let player_entity = player_query.single.unwrap();
	
    child_query.set_relation_filters(
        RelationFilters::new()
            .add_target_filter::<ChildOf, _>(player_entity)
    );

	for name in child_query.iter() {
        println!("{} is one of my children.", name) 			
    }
}
```

Finally, you can use the `Entity` returned by your relations to fetch data from the target's components by combining it with `query::get()` or `query::get_component<C>()`.

```rust
// We need to use a marker component to avoid having multiple mutable references
fn debts_come_due(
    mut debtor_query: Query<(&mut Money, Relation<Owes>), Without<Shark>>, 
    mut lender_query: Query<&mut Money, With<Shark>>,
) {
	for (mut debtor_money, debt) in debtor_query.iter(){
		for (lender, amount_owed) in debt {
			let mut lender_money = lender_query.get().unwrap();
			if debtor_money >= amount_owed {
				debtor_money.dollars -= amount_owed.dollars;
				lender_money.dollar += amount_owed.dollar;
			} else {
				panic!("Nice kneecaps you got there...");
			}
		}
	}
}
```

### Grouping entities

### Entity hierarchies

## Reference-level explanation

TODO: Boxy explains the magic.

## Drawbacks

1. Introduces another core abstraction to users that they will want to learn.
Rewriting our parent-child code in terms of relations is important, but will introduce the concept to users quite quickly.
2. Even if relations are more ergonomic than the Entity-in-component approach to accomplish the same tasks, relations will be held to a higher bar for quality.
3. Heavy use of relations can seriously fragment archetypes. We need to be careful that this doesn't badly impact our performance.

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## Prior art

[`Flecs`](https://github.com/SanderMertens/flecs), an advanced C++ ECS framework, has a similar feature, which they call "relationships".
These are somewhat different, they use an elaborate query domain-specific language along with being more feature rich.
You read more about them in the [corresponding PR](https://github.com/SanderMertens/flecs/pull/358).

## Unresolved questions

1. How do we put relations in Bundles?
2. What do we actually call `Noitaler`? Candidates so far are:
   1. `ReverseRelation`
   2. `Inverse`
   3. `CoRelation`
   4. `TargetOf`
3. Do we need a full graph constraint solver in our queries to handle things like "check for unrequited love"?
4. Do we want a macro to make complex calls easier?

```rust
macro_rules Relation {
  (this $kind:ty *) => Relation<$kind>,
  (* $kind:ty this) => Noitaler<$kind>,
}
```

## Future possibilities

Relation enhancements:

1. Sugar for accessing data on the target entity in a single query. `Relation<T: Component, Q: WorldQuery>`
2. Graph traversals.
3. Arbitrary target types instead of just `Entity` combined with `KindedEntity` proposal to ensure your targets are valid.
4. Automatically symmetric relations.
5. `Noitaler`*
6. Graph shape guarantees using archetype invariants.
7. Streaming iters to allow for queries like: `Query<&mut Money, Relation<Owes, &mut Money>>` and potentially use invariants on graph shape to allow for skipping soundness checks for the aliasing `&mut Money`'s
8. Assorted Performance optimizations.
9. Relation orders to enable things like `Styles`.
10. Generalized `despawn_recursive`.

Relation applications in the engine:

1. Replace existing parent-child.
2. Applications for UI: styling, widget wiring etc.
3. Frustum culling.

AUTHOR'S NOTE: Much like `yeet`, `Noitaler` is the bikeshed-avoidance name for the inverse of `Relation`, and will never be used.
EDITOR's NOTE: `Noitaler` may very much be used :3
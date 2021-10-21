# Feature Name: `trait-queries`

## Summary

Trait queries allow users to query for components that implement a given trait.

## Motivation

Abstracting over multiple similar component typically involves manually keeping track of a list of all of the relevant components that match your criteria, then updating various call sites.

This tends to occur when writing queries for extensible gameplay systems, where we want to operate over *all* components that have a particular behavior, or entities that have exactly one component that has a particular behavior.
The workarounds for this often involve a huge number of tiny, specialized systems that are very challenging to maintain and reason about.

Rust **traits** are the standard, idiomatic way to handle these types of abstraction.
However, as Rust's reflection story is very limited, Bevy does not currently support the uses of traits that many users expect from within the ECS.  

By improving our ability to access the trait information of our component and resource types at runtime, we can enable these use cases, allowing users to write safer, clearer and more extensible code.

## User-facing explanation

In addition to querying for specific concrete component types, you can request any (or exactly one) component that meets a given trait bound.

For example, suppose you're designing an action RPG and have a number of different ways a critical hit could be modified.
The traditional approach to this would be to use events.
When a critical hit occurs, emit a `CriticalHit` event (perhaps stored on the entity to avoid repeated scans), then, for each critical hit modifying effect, add a custom system to read the event and trigger downstream events as appropriate.

However, this indirection massively clutters our schedule and splits our logic across dozens of functions, making it harder to reason about, visualize and maintain.
Trait queries allow us to instead create an `OnCrit` trait, add components that modify the critical hit behavior of the entity, and then apply the effects of each of those components.
Let's see how that might look.

```rust
// This trait can only be added to components,
// which allows the compiler to catch silly mistakes for us
trait OnCrit: Component {
  // Common behavior can directly work with or mutate the provided components 
  fn modify_damage(damage: &mut Damage){}

  // More complex side effects can be generated and deferred using commands
  fn command(critting_entity: Entity) -> impl Command{}
}

// This query returns any entity that just crit, and has at list one component with the `OnCrit` trait
fn crit_system(query: Query<(Entity, &mut Damage, AnyOf<&(dyn OnCrit)>), With<Crit>>, mut commands: Commands){
 for (entity, mut damage, on_crit_effects) in query.iter_mut(){
  // `on_crit_effects` is an iterator of components, all of whom are guaranteed to have the `OnCrit` trait
  // This allows us to call methods defined in the trait on each item
  for effect in on_crit_effects {
   effect.modify_damage(damage);
   commands.add(effect.command(entity));
  }
 }
}
```

If we wanted to ensure that there was only ever one handling method available, we could use `OneOf<&(dyn OnCrit)>` instead, which will only return entities that have exactly one component with that particular trait.

You can also use these `dyn Trait` types in query filters: allowing you to search for entities with a component of a particular trait, without a component of that trait, with a changed component of that trait and so on.
These all operate on an "at least one" basis: `Without<&dyn OnCrit>` will filter out any entities that have one or more components with the `OnCrit` trait.

## Implementation strategy

TODO: write me.

## Drawbacks

1. Both trait queries and trait-universal systems will make implementing a trait a breaking change, contrary to Rust's standard semantic versioning rules.
   1. Adding traits which enable trait queries or trait-universal systems by accident should be extremely rare: these will generally be clearly marked, game / plugin-specific and serve no other useful purpose.

## Rationale and alternatives

### Why don't components that store a boxed trait objects work for trait queries?

This pattern does not interact smoothly with other existing Bevy code, as the internal type will be hidden to the ECS.
There are also some other ergonomic drawbacks:

- Trait objects can only contain one object with a given trait, making it unsuitable for queries where we want to fetch *all* components with a given trait.
- This can be bypassed with the use of `Vec<Box<dyn MyTrait>>`, but this is even more bulky and unergonomic.
- Boxed trait objects must be heap-allocated* due to their variable size, incurring a serious performance hit.
- Not all traits are object safe: the trait queries design has no such limitation.

*Some complex ecosystem crates seem to offer workarounds for this, but most users will not discover or use these.

### Why can't we just use `AnyOf` in place of trait queries?

`AnyOf` queries (introduced in [Bevy #2889](https://github.com/bevyengine/bevy/pull/2889)), along with a hypothetical `OneOf` counterpart, can mostly capture the proposed trait query logic.

With a type alias, it's even fairly ergonomic!
However, this has three serious drawbacks:

1. This list must be manually maintained and updated: catching that an extra component type has been added or removed is going to be a frustrating and tedious debugging experience.
2. Other users and modules cannot extend this list. This is particularly severe when looking at plugins, rather than games, where the author of the system and the user of the system are not the same. This is a serious limitation in UI use cases in particular.
3. Because the Rust compiler does not *know* that all elements of this tuple type share a trait (or several), we cannot call trait methods on the elements in an automatic, unified fashion.

## Prior art

@Davier has a [working proof of concept](https://github.com/bevyengine/bevy/compare/main...Davier:trait_query) of trait queries that is worth investigating and building upon for implementation details.

## Unresolved questions

1. How cam we bridge the gap between the prototype API and the dream API laid out in the user-facing explanation?
2. How can we determine an order to the components returned by trait queries?

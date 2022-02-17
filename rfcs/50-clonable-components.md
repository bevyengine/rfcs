# Feature Name: `clonable-components`

## Summary

The `Component` trait requires the `Clone` trait to be implemented.
This allows us to clone entities and entire worlds quickly and ergonomically.

## Motivation

Cloning the components of entities is generally useful for:

- rollback networking
- simple gameplay code
- complex entity-driven workflows for generating new entities

## Background

Open issues:

- [Clonable entities](https://github.com/bevyengine/bevy/issues/1515)
- [Clonable worlds](https://github.com/bevyengine/bevy/issues/3703)

Open PRs:

- [Add CloneBundle command](https://github.com/bevyengine/bevy/pull/3820)

## User-facing explanation

The `Component` trait indicates that a struct (or enum) can be used as a component in Bevy.
Because `Component: Send + Sync + 'static + Clone`, every component can be cloned.

We can clone specific components using (for example) `commands.entity(target_entity).clone_from::<Transform>(source_entity)` or `commands.entity(target_entity).clone_bundle_from::<SpriteBundle>(source_entity)`, or the `clone_to` equivalent.

Specific components can be cloned onto a new entity by calling this methods immediately after `commands.spawn()`:

```rust
fn mirror_image(mut commands: Commands, player_query: Query<Entity, With<Player>>){
 let player_entity = player_query.single();

 for _ in 0..8 {
  commands.spawn().insert(MirrorImage).clone_bundle_from::<PlayerBundle>(player_entity);
 }
}
```

However, we may not know at compile time which components our player has that we wish to copy!
Suppose, in classic roguelike fashion, our player can have many, many effects: they can be buffed, they can be poisoned, they can be in the middle of casting a spell, they could have an enemy's debuff on them...

Each of these effects is naturally (and efficiently) represented as a component, so simply cloning the `PlayerBundle` will make it immediately clear which player is real!

If we're insistent that we must carefully enumerate the possible component types to clone, we could attempt to create a `PossiblePlayerComponentsBundle`, which lists all of these possible effects.
Then:

```rust
#[derive(Bundle)]
enum PossiblePlayerComponentsBundle {
  #[bundle]
  player_bundle: PlayerBundle,
  thorns_buff: Buff<Thorns>,
  spell_power_buff: Buff<SpellPower>,
  poison: Poison,
  rooted_debuff: Debuff<Rooted>,
  // Etc etc
}

fn mirror_image(mut commands: Commands, player_query: Query<Entity, With<Player>>){
 let player_entity = player_query.single();

 for _ in 0..8 {
  commands.spawn().insert(MirrorImage).clone_bundle_from_infallible::<PossiblePlayerComponentsBundle>(player_entity);
 }
}
```

Now this is much harder to detect, if somewhat slower due to the repeated checks.
Hopefully we never forget to add any of the many possible components to this list...

Instead, we should probably use the `clone_entity` command, which clones all components of the given entity onto a freshly spawned entity:

```rust
fn mirror_image(mut commands: Commands, player_query: Query<Entity, With<Player>>){
 let player_entity = player_query.single();

 for _ in 0..8 {
  commands.clone_entity(player_entity).insert(MirrorImage);
 }
}
```

No boilerplate, and impossible to mess up!

If there are a few components that we *don't* want to copy onto our new entity, we can use `remove_bundle` (or more likely, `remove_bundle_infallible`) to be rid of them.

```rust
fn mirror_image(mut commands: Commands, player_query: Query<Entity, With<Player>>){
 let player_entity = player_query.single();

 for _ in 0..8 {
  commands.clone_entity(player_entity).insert(MirrorImage).remove_bundle_infallible::<DoNotClonePlayerBundle>();
 }
}
```

## Implementation strategy

1. Add a `Clone` trait bound to `Component`.
2. Add the `Clone` derive to all built-in components that do not currently have one.

## Drawbacks

1. Adds another trait that users must derive for each component.
2. Forbids the use of un-cloneable components.
   1. Despite significant investigation, no examples of components that should never be cloned have been found.
   2. Every commericial game engine investigated permits this functionality,
3. Cloning entire entities does not cover multiple-entity assemblages.
4. Users may be irrationally suspicious that we're cloning their components as part of the internal ECS mechanisms.

## Rationale and alternatives

### Why do we need clone_entity at all?

As shown in the user-facing example, cloning only specific enumerated types forces a very "class-based" and global perspective.
This is very heavy on boilerplate, and results in a requirement to synchronize the list of components that could be added to an entity.
If a component is missed, no warning will be emitted to the user, even if it is added to the entity to be cloned.

In addition, the checks required are likely to be meaningfully slower than simply cloning all of the comnponents.
Because of the relatively core nature of this operation, this is likely to have a meaningful impact in games that make heavy use of this feature.

### Alternative: `Clone` trait registration

Generally speaking, information about whether or not a type implements a trait is not available at run-time.
As a result, naive implementations of `clone_entity` simply do not work without a trait bound, as we cannot clone each component.

However, through the magic of type reflection, we can get around this.
Whenever a component type is registered, we can record whether or not it implements `Clone`, by storing that information in a method on `Component`.
Then, when `commands.clone_entity(source_entity)` is called, we can iterate over the component types of `source_entity`, using the `Component::clone(source_entity: Entity, target_entity: Entity, world: &mut World)` method to clone the components (if the object derives `Clone`).

While this does not force a `Clone` trait bound, it has several disadvantages:

- dramatically higher technical complexity
- significantly more indirection and branching, likely resulting in measurably worse performance
- much harder to safely parallelize, resulting in more technical complexity and worse performance
- cloning entities is now fallible, which must be handled in some form or another
  - existing tools for command error-handling are very poor
  - users can forget to derive `Clone`, resulting in run-time logic errors, rather than compilation errors
- may not work correctly with manual `Clone` impls
  - these are generally quite rare, but are important
- not all components are guaranteed be cloned, which may have impacts on determinism in multi-world applications of this feature

### Alternative: Scene-powered cloning

We could create a secene out of an entity or group of entities, clone the scene and then rehydrate the scene.

This is painfully indirect and confusing, but would handle multiple entity use cases much better.

This also still requires a `Reflect` derive, and the same issues of inefficiency and error-risk arise around whether or not `Component` should require `Reflect`.
However, while virtually all structs that should be stored in components are `Clone`, the same is not true for `Reflect`, making this tool much harder to use with, for example, exotic data structures.

## Prior art

The following game engines allow users to clone entire entities (or their analog):

- [Unity](https://docs.unity3d.com/Packages/com.unity.entities@0.3/manual/ecs_entities.html): using the `Enitity.Instantiate` method, after significant [user frustration](https://forum.unity.com/threads/pure-ecs-how-to-copy-all-components-from-one-entity-to-another.638308/)
- [Unreal](https://ikrima.dev/ue4guide/engine-programming/uobjects/duplicate-or-copy-object/)
- [Godot](https://www.reddit.com/r/godot/comments/8z84aq/how_to_duplicate_a_node_with_codes/): yes, but buggy, due to the existence of multiple-node objects
- [RPGMaker](https://forums.rpgmakerweb.com/index.php?threads/cloning-an-object-without-references.143725/)
- [Phaser](https://newdocs.phaser.io/docs/3.55.2/focus/Phaser.Utils.Objects.DeepCopy)
- [Lumberyard](https://github.com/awsdocs/amazon-lumberyard-user-guide/blob/master/doc_source/lumberyard-editor-menus.md)
- [CryEngine](https://docs.cryengine.com/display/CEMANUAL/Create+and+Edit+Objects#:~:text=Duplicate%20a%20selected%20object%20by,Terrain%2C%20and%20Snap%20to%20Geometry.)
- [GameMaker Studio](https://www.reddit.com/r/gamemaker/comments/6ekd82/making_a_clone_of_an_object/)\
- [Ren'Py](https://lemmasoft.renai.us/forums/viewtopic.php?t=35831)
- [Twine](http://twinery.org/questions/13878/want-javascript-objects-to-be-copied-as-references)
- [Flecs](https://discord.com/channels/691052431525675048/742569353878437978/943951423534674022)
- [Love2D](https://love2d.org/wiki/Data:clone)

The following game engines do not support this functionality:

- [Fyrox]: No trait bound on [GameState](https://docs.rs/fyrox/latest/fyrox/engine/framework/trait.GameState.html), and there does not appear to be an equivalent central data-store.

Generally speaking, other Rust ECS's do not force this trait bound, and have no way to copy entire entities.

## Unresolved questions

- Can or should this be used for the current render-extraction step?
- How much slower is the reflection-powered approach?

## Future possibilities

### Cloning worlds

With proper multiple world support, ([RFC #43](https://github.com/bevyengine/rfcs/pull/43), we could use this feature to clone entire worlds.

This allows us to take snapshots of the entire world, which is essential for:

- in-memory checkpoints for scientific computation
- counterfactual scenario exploration for AI
- certain rollback networking structures
- undo-and-redo functionality, including for the editor

To do so, we would need to move resources out of the `World` into their own storage, as `Clone` is too strict a bound for `Resource` and particularly `NonSend`.

### Query cloning

Being able to clone all entities (or a fixed set of their components) that match a query would be particularly useful for efficiently working with multiple worlds, as it would allow users to restrict the entities of interest.

This may also be useful to avoid cloning metadata out from the world, reducing overhead in use cases like pipelined-rendering and undo-redo.

### Multi-entity copying

If and when [relations](https://github.com/bevyengine/bevy/issues/3742) and [schematics](https://github.com/bevyengine/bevy/issues/3877) are introduced, it would be helpful to have tools to copy a whole "logically connected set of entities".

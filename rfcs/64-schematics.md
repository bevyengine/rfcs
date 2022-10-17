# Feature Name: `schematics`

## Summary

The goal of this RFC is to facilitate and standardize two-way conversion and synchronization between ECS worlds.
The approach described here aims to be more user friendly than just handcoded synchronization systems, but more flexible than a prefab system.
This design is buitl on top of the [many worlds RFC][many_worlds].

## Motivation

When devolping an editor for bevy, the information visible should not necessarily be in the same format that most runtime systems operate on.
The runtime representation is often optimized for efficiency and use in those systems, while the information displayed in the editor should be optimized for
easy understanding and stability agains changes.
We propose to address this by adding another default world to the ones described in the [many worlds RFC][many_worlds] that we call the "schematic world",
and which contains the information used in the editor and its scene file formats.

The main problem that arises from this approach is the synchronisation between the main and schematic worlds, so the RFC focuses on this aspect.
Similar problems, where data needs to kept synchronized between worlds, might also appear in other areas.
One example for this is the "render world", for which the extract phase could be adapted to use schematics.
Another example could be a "server world" in a multiplayer game, that only contains information which is shared with other clients and the server.
Finally the approach chosen here would help in implementing a "prefab" system.

## User-facing explanation

### The `CoreWorld::Schematic`

### The `SchematicQuery`

The main interface added by this RFC is the `SchematicQuery`, which you will usually only use in systems that run on `CoreWorld::Schematic`.
We call such systems "schematics".
A `SchematicQuery` works almost the same as a normal `Query`, but when iterating over components will additionaly return `SchematicCommands` for the component queried.
This can be used to insert and modify components on the corresponding entity on `CoreWorld::Main` as well as spawn new entities there.
The entities spawned in this way can not be modified by `SchematicCommands` from other systems, but will be remembered for this system similar to `Local` system parameters.

A typical schematic will look like this:

```rust
#[derive(Component)]
struct SchematicA(u32, Entity);

#[derive(Component)]
struct MainA(u32);

#[derive(Component)]
struct MainAChild(Entity);

/// Translates a `SchematicA` to a `MainA` as well as a child entity that has a `MainAChild`.
/// The system can contain any other parameter besides the schematic query
fn schematic_for_a(query: SchematicQuery<A, With<Marker>>, mut some_resource: ResMut<R>) {
    for (a, commands) in query {
        some_resource.do_something_with(a);
        // You can modify components
        commands.insert_or_update(MainA(a.0));
        // And spawn other entities.
        // This will only spawn an entity on the first execution. It will remember the entity
        // by the label "child" so `child_commands` will operate on the same entity the next
        // time this system is run.
        commands.spawn_or_select_child("child", |child_commands| {
            // Entity references within the schematic world need to be mapped in this way.
            // (This might not be needed depending on the implementation of many worlds)
            let entity = child_commands.map_entity(a.1);
            child_commands.insert_or_update(MainAChild(entity));
        });
    }
}
```

The `SchematicQuery` will automatically only iterate over components that were changed since the system last ran.
Other than a normal query, the first generic argument can only be a component, so tuples or `Entity` are not allowed as the first argument.

The main methods of `SchematicCommands` are:
* `insert_or_update<B: Bundle>(bundle: B)`
* `spawn_or_select_child<L: SchematicLabel, F: FnOnce(SchematicCommands)>(label: L, F)`
* `spawn_or_select<L: SchematicLabel, F: FnOnce(SchematicCommands)>(label: L, F)`
* `despawn_if_exists<L: SchematicLabel>(label: L)`
* `remove_if_exists<B: Bundle>()`

### The `default_schematic` system

Since with many components you want to just copy the same component from the schematic to the main world, a `default_schematic` is provided.
The implementation of this is just:

```rust
fn default_schematic<A: Clone>(query: SchematicQuery<A>) {
    for (a, commands) in query {
        commands.insert_or_update(a.clone());
    }
}
```

### Adding schematics to your app

`Schematic`s can be added to schedules like any other system
```rust
schedule.add_system(schematic_for_a);
```

Usually you will want to syncronize between the app's `SchematicWorld` and `CoreWorld::Main`.
In this case you can use

```rust
app.add_schematic(default_schematic::<A>);
```

## Implementation strategy

* `SchematicQuery<C, F>` is basically `(Query<C, (F, Changed<C>)>, Local<HashMap<Entity, SchematicData>>, AppCommands)` where `SchematicData` is something like
  ```rust
      struct SchematicData {
          corresponding_entities: HashMap<SchematicLabelId, Entity>
      }
  ```
* `SchematicCommands` is something like
  ```rust
      struct SchematicCommands<'a> {
          app_commands: &'a mut AppCommands,
          schematic_data: &'a mut SchematicData,
          current_label: Option<SchematicLabelId>,
      }
  ```

## Drawbacks

* Adds another data model that programmers need to think about.
* The design may not be very performant and use a lot of extra memory
* Need to add additional code to pretty much any component, even if it is just `app.add_schematic(default_schematic::<C>)`
* It is not possible to check invariants, e.g. synchronization should be idempotent, schmatics shouldn't touch same component, ...

## Rationale and alternatives

This design
* Integrates well into ECS architecture
* Is neutral with regard to usage
* Can use existing code to achive parallel execution
* Can live alongside synchronization systems that don't use `SchematicQuery`

The problem could also be solved by "prefabs", i.e. scenes that might expose certain fields of the components they contain.
But this would be a lot more restrictive the "schematics" described here.
What might make more sense is to implement prefabs atop schematics.

This design does not allow the schematic world to behave sensibly for use in the editor by itself.
It would need something like [archetype invariants](https://github.com/bevyengine/bevy/issues/1481) additionally.

## Prior art

*TODO: Compare with Unity's system of converting game objects to ECS components*

See https://github.com/bevyengine/bevy/issues/3877

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.

[many_worlds]: https://github.com/bevyengine/rfcs/pull/43

# Feature Name: `schematics`

## Summary

The goal of this RFC is to facilitate and standardize two-way conversion and synchronization between ECS worlds.
The approach described here aims to be more user friendly than just handcoded synchronization systems, but more flexible than a prefab system.
This design is built on top of the [many worlds RFC][many_worlds], but the implementation can be easily adapted to work with sub apps instead.

## Motivation

When eventually devolping an editor for bevy, the information visible should not necessarily be in the same format that the runtime operates on.
The runtime representation is often optimized for efficiency and use in logic systems, while the information displayed in the editor should be optimized for
easy understanding and stability.
We propose to address this by adding another default world to the ones described in the [many worlds RFC][many_worlds] that we call the "schematic world".
The entities and components in this world are the ones shown to users in the editor.

A big emphasis on the design is also for the schematic representation to be able to be stable against major changes to the runtime representation.
This not only enables us to use the schematic world for a stable scene format inside one project, but it also helps with separating internal implentation changes in both official
and unofficial plugins from what users, especially non-technical users, see in the editor.

The main problem that arises from this approach is the synchronisation between the main and schematic worlds, so the RFC focuses on this aspect.
Similar problems, where data needs to kept synchronized between worlds, might also appear in other areas.
One example for this is the "render world", for which the extract phase could be adapted to use schematics.
Another example could be a "server world" in a multiplayer game, that only contains information which is shared with other clients and the server.
Finally the approach chosen here would help in implementing a "prefab" system.

## User-facing explanation

### The `CoreWorld::Schematic`

We propose adding another world to the `CoreWorld` enum, namely `CoreWorld::Schematic`.
This world is meant to be an representation of the runtime world which is to be used in the editor and for serialization formats.
As such it should group components from the runtime world, if they can only show up together, and avoid duplicate data such as `Transform` and `GlobalTransform`.
The purpose of this RFC is to facilitate the synchronization between the runtime world and the schematic world.

### Schematics

To synchronize between the main and schematic worlds, for every component in the schematic world you need to add a `Schematic` to your app.
A `Schematic` is just a collection of systems used for this synchronization process.

* A system to convert components in the schematic world to components in the runtime world
* A system to convert components in the runtime world to components in the schematic world

Only the first those two is mandatory.
Usually though, you will not need to write these algorithms yourself - you can use `#[derive(DefaultSchematic)]` in most cases.

### Deriving `DefaultSchematic`

You can add `#[derive(DefaultSchematic)]` to any struct or enum that is a component and that implements `Default`.

If you derive `DefaultSchematic` for a struct, every untagged field needs to implement `Component` and `Clone`.
Otherwise it either needs to implement `Default` and be tagged it with `#[schematic_ignore]`, or you need to specify another `Component`, such that the field has `From` and `Into` instances for this component.
In the latter case you need to tag the field with `#[schematic_into(OtherComponent)]`

```rust
#[derive(Component, Default, DefaultSchematic)]
struct MeshRenderer {
    mesh: Handle<Mesh>,
    material: Handle<Mesh>,
    #[schematic_into(Name)]
    name: String,
    #[schematic_ignore]
    thumbnail: Handle<Image>,
}
```

If you derive `DefaultSchematic` for an enum
* Every unit variant nees to be tagged with `#[schematic_marker(MarkerComponent)]`. The given component needs to implement `Default`.
* Every tuple variant can only have one field which needs to implement `Component` and `Clone` or be tagged with `#[schematic_into(OtherComponent)]`.
* For struct variants the same rules as for structs apply. Struct variants need to have at least one field that is not marked with `#[schematic_ignore]`.
Alternatively every variant can also be tagged with `#[schematic_ignore]`

```rust
#[derive(Component)]
struct Walking {
    speed: f32,
}

impl From<f32> for Walking {
    // ...
}

#[derive(Component, Default)]
struct Jumping;

#[derive(Component, Clone)]
struct Attacking {
    damage: f32,
    weapon_type: WeaponType,
}

#[derive(Component, Default, DefaultSchematic)]
enum AnimationState {
    Walking {
        #[schematic_into(Walking)]
        speed: f32
    }
    #[schematic_marker(Jumping)]
    Jumping,
    Attacking(Attacking),
}
```

The `DefaultSchematic` trait has only one function
```rust
fn default_schematic() -> Schematic
```

The derived implementation contains conversions both directions.

### Adding schematics to your app

`Schematic`s can be added to the app using
```rust
app.add_schematic(AnimationState::default_schematic());
```

Since with many components you want to just copy the same component from the schematic to the main world, a `CloneSchematic<A: Clone>` is provided.

```rust
app.add_schematic(CloneSchematic::<Visibility>::default());
```

### Creating `Schematic` manually

```rust
fn new<S>(system: S) -> Schematic
where
    S: IntoSchematicConversion

fn add_inference<S>(self, system: S) -> Schematic
where
    S: IntoSchematicInference

fn add_inference_for_entity<S>(self, entity_label: impl SchematicLabel, system: S) -> Schematic
where
    S: IntoSchematicInference
        
fn add_discovery<S>(self, system: S) -> Schematic
where
    S: IntoSchematicDiscovery
```

#### Example for struct

```rust
#[derive(Component, Default)]
struct MeshRenderer {
    mesh: Handle<Mesh>,
    material: Handle<Mesh>,
    name: String,
    thumbnail: Handle<Image>,
}

fn mesh_renderer_schematic(query: SchematicQuery<MeshRenderer>) {
    for (mesh_renderer, commands) in query {
        commands.require(mesh_renderer.mesh.clone());
        commands.require(mesh_renderer.material.clone());
        commands.require(mesh_renderer.name.into());
    }
}

fn infer_mesh(
    query: InferenceQuery<&Handle<Mesh>, MeshRenderer>,
) {
    for (mesh, commands) in query {
        commands.infer(|mesh_renderer| mesh_renderer.mesh = mesh.clone());
    }
}

fn infer_material(
    query: InferenceQuery<&Handle<Material>, MeshRenderer>,
) {
    for (material, commands) in query {
        commands.infer(|mesh_renderer| mesh_renderer.material = material.clone());
    }
}

fn infer_name(
    query: InferenceQuery<&Handle<Name>, MeshRenderer>,
) {
    for (name, commands) in query {
        commands.infer(|mesh_renderer| mesh_renderer.name = name.into());
    }
}

fn discover_mesh_renderer(
    query: Query<Entity, (With<Handle<Mesh>>, With<Handle<Material>>, With<Name>)>,
    commands: DiscoveryCommands<AnimationState>,
) {
    for entity in query {
        comands.discover(entity);
    }
}

impl DefaultSchematic for MeshRenderer {
    fn default_schematic() -> Schematic {
        Schematic::new(mesh_renderer_schematic)
            .add_inference(infer_mesh)
            .add_inference(infer_material)
            .add_inference(infer_name)
            .add_discovery(discover_mesh_renderer)
    }
}
```

#### Example for enum

```rust
#[derive(Component)]
struct Walking {
    speed: f32,
}

#[derive(Component)]
struct Jumping;

#[derive(Component, Clone)]
struct Attacking {
    damage: f32,
    weapon_type: WeaponType,
}

#[derive(Component, Default)]
enum AnimationState {
    Walking {
        speed: f32
    }
    Jumping,
    Attacking(Attacking),
}

fn animation_state_schematic(query: SchematicQuery<AnimationState>) {
    for (animation_state, commands) in query {
        match animation_state {
            AnimationState::Walking { speed } => commands.require(Walking { speed }),
            AnimationState::Jumping => commands.require(Jumping),
            AnimationState::Attacking(attacking) => commands.require(attacking.clone()),
        }
    }
}

fn infer_walking(
    query: InferenceQuery<&Walking, AnimationState>,
) {
    for (walking, commands) in query {
        commands.infer(|animation_state| *animation_state = AnimationState::Walking { speed });
    }
}

fn infer_jumping(
    query: InferenceQuery<(), AnimationState, With<Jumping>>,
) {
    for (_, commands) in query {
        commands.infer(|animation_state| *animation_state = AnimationState::Jumping);
    }
}

fn infer_attacking(
    query: InferenceQuery<&Attacking, AnimationState>,
) {
    for (attacking, commands) in query {
        commands.infer(|animation_state| *animation_state = AnimationState::Attacking(attacking.clone()));
    }
}

fn discover_animation_state(
    query: Query<Entity, Or<(With<Walking>, With<Jumping>, With<Attacking>)>,
    commands: DiscoveryCommands<AnimationState>,
) {
    for entity in query {
        commands.discover(entity);
    }
}

impl DefaultSchematic for AnimationState {
    fn default_schematic() -> Schematic {
        Schematic::new(animation_state_schematic)
            .add_inference(infer_walking)
            .add_inference(infer_jumping)
            .add_inference(infer_attacking)
            .add_discovery(discover_animation_state)
    }
}
```

#### Example with related entities

```rust
#[derive(Component)]
struct SchematicA(u32, Entity);

#[derive(Component)]
struct MainA(u32);

#[derive(Component)]
struct MainAChild(Entity);

/// Translates a `SchematicA` to a `MainA` as well as a child entity that has a `MainAChild`.
/// The system can contain any other parameter besides the schematic query
fn schematic_a(query: SchematicQuery<A>>, mut some_resource: ResMut<R>) {
    for (a, commands) in query {
        some_resource.do_something_with(a);
        // You can modify components
        commands.require(MainA(a.0));
        // And spawn other entities.
        // This will only spawn an entity on the first execution. It will remember the entity
        // by the label "child" so `child_commands` will operate on the same entity the next
        // time this system is run.
        commands.require_child("child", |child_commands| {
            // Entity references within the schematic world need to be mapped in this way.
            // (This might not be needed depending on the implementation of many worlds)
            let entity = child_commands.map_entity(a.1);
            child_commands.require_component(MainAChild(entity));
        });
    }
}

fn inference_for_main_a(
    query: InferenceQuery<&MainA, SchematicA>,
) {
    for (a, inference_commands) in query {
        inference_commands.infer(|schematic_a| {
            schematic_a.0 = a.0;
        });
    }
}

fn inference_for_main_a_child(
    query: InferenceQuery<&MainAChild, SchematicA>,
) {
    for (a_child, inference_commands) in query {
        let entity = inference_commands.map_entity(a_child.0);
        inference_commands.infer(|schematic_a| {
            schematic_a.1 = entity;
        });
    }
}

fn discover_schematic_a(
    query: Query<(Entity, &Children), With<MainA>>,
    child_query: Query<Entity, With<MainAChild>>,
    commands: DiscoveryCommands,
) {
    for (entity, children) in query {
        let child = child_query.get_many(children).filter_map(|result| result.ok()).next();
        if Some(child) = child {
            commands
                .discover(entity)
                .with_entity("child", child);
        }
    }
}

impl DefaultSchematic for SchematicA {
    fn default_schematic() -> Schematic {
        Schematic::new(schematic_a)
            .add_inference(inference_for_main_a)
            .add_inference_for_entity("child", inference_for_main_a_child)
            .add_discovery(discover_schematic_a)
    }
}
```

The `SchematicQuery` will automatically only iterate over components that were changed since the system last ran.
Other than a normal query, the first generic argument can only be a component, so tuples or `Entity` are not allowed as the first argument.
It provides read-only access to that component.
Systems that mutate data in the schematic world should usually be separate from schematics.

The main methods of `SchematicCommands` are:
* `require<B: Bundle>(bundle: B)`
* `require_child<L: SchematicLabel, F: FnOnce(SchematicCommands)>(label: L, F)`
* `require_entity<L: SchematicLabel, F: FnOnce(SchematicCommands)>(label: L, F)`
* `require_despawned<L: SchematicLabel>(label: L)`
* `require_deleted<B: Bundle>()`

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
* `Schematic` is
  ```rust
  struct Schematic {
      conversion: Box<dyn SchematicConversion>,
      inference: Option<Box<dyn SchematicInference>>,
  }
  ```
* `SchematicConversion` and `SchematicInference` are basically just `System` with a restriction on the parameters

## Drawbacks

* Adds another data model that programmers need to think about.
* The design may not be very performant and use a lot of extra memory
* Need to add additional code to pretty much any component, even if it is just `app.add_schematic(CopySchematic::<C>::default())`
* It is not possible to check invariants, e.g. synchronization should be idempotent, schmatics shouldn't touch same component, ...

## Rationale and alternatives

This design
* Integrates well into ECS architecture
* Is neutral with regard to usage
* Can use existing code to achive parallel execution
* Can live alongside synchronization systems that are not built using `Schematic`

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

* Checking that every component is read during conversion and similar things

## Future possibilities

*TODO: Write how this would fit into an editor UI*
*TODO: Write how it's ok that not every schematic can be converted back*

[many_worlds]: https://github.com/bevyengine/rfcs/pull/43

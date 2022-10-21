# Feature Name: `schematics`

## Summary

The goal of this RFC is to facilitate and standardize two-way conversion and synchronization between ECS worlds.
The approach described here aims to be more user friendly than just handcoded synchronization systems, but more flexible than a prefab system.
This design is built on top of the [many worlds RFC][many_worlds], but the implementation can be easily adapted to work with sub apps instead.

## Motivation

When devolping an editor for bevy, the information visible should not necessarily be in the same format that the runtime operates on.
The runtime representation is often optimized for efficiency and use in logic systems, while the information displayed in the editor should be optimized for
easy understanding and stability.
We propose to address this by adding another default world to the ones described in the [many worlds RFC][many_worlds] that we call the "schematic world".
The entities and components in this world are the ones shown to users in the editor.

A big emphasis on the design is also for the schematic representation to be able to be stable against major changes to the runime representation.
This not only enables us to use the schematic world for a stable scene format inside one project, but it also helps with separating internal implentation changes in both official
and unofficial plugins from what users, especially non-technical users.

The main problem that arises from this approach is the synchronisation between the main and schematic worlds, so the RFC focuses on this aspect.
Similar problems, where data needs to kept synchronized between worlds, might also appear in other areas.
One example for this is the "render world", for which the extract phase could be adapted to use schematics.
Another example could be a "server world" in a multiplayer game, that only contains information which is shared with other clients and the server.
Finally the approach chosen here would help in implementing a "prefab" system.

We optionally suggest the "collect-resolve-apply" approach to solve the problem of keeping and resolving invariants between components like "every `ParticleSystem` needs a `Transform`".
This could also be solved using archetype invariants in the schematic world.
Some work on archetype invariants is  already in progress (see https://github.com/bevyengine/bevy/pull/5121).
The design does not depend on either of these approaches though.

## User-facing explanation

### The `CoreWorld::Schematic`

We propose adding another world to the `CoreWorld` enum, namely `CoreWorld::Schematic`.
This world is meant to be an representation of the runtime world which is to be used in the editor and for scene formats.
As such it should group components from the runtime world, if they can only show up together, and avoid duplicate data such as `Transform` and `GlobalTransform`.
The purpose of this RFC is to facilitate the synchronization between the runtime world and the schematic world.

For this up to 3 algorithms can be defined for every schematic component:
* Conversion from the schematic world to the runtime world
* Conversion from the runtime world to the schematic world (optional)
* Update the schematic world with changed data from the runtime world (optional)

Usually you will not need to write these algorithms yourself - you can use `#[derive(Schematic)]` in most cases.

### Deriving `Schematic`

You can add `#[derive(Schematic)]` to any struct or enum that is a component.

If you derive `Schematic` for a struct, every field needs to implement `Component` and `Clone`.
Otherwise it either needs to implement `Default` and be tagged it with `#[schematic_ignore]`, or you need to specify another `Component`, such that the field has `From` and `Into` instances for this component.
In the latter case you need to tag the field with `#[schematic_into(OtherComponent)]`

```rust
#[derive(Component, Schematic)]
struct MeshRenderer {
    mesh: Handle<Mesh>,
    material: Handle<Mesh>,
    #[schematic_into(Name)]
    name: String,
    #[schematic_ignore]
    thumbnail: Handle<Image>,
}
```

If you derive `Schematic` for an enum
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

#[derive(Component, Schematic)]
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

The derived implementation contains conversions both direction as well as updates.

### Schematic combinators

Every schematic that implements `SimpleSchematic` has the following functions which can be used to create new schematics:
* `map<C, F, G>(into: F, from: G) -> impl SimpleSchematic<Component = C> where F: Fn(&Self::Component) -> C, G: Fn(C) -> Self::Component`
* `add<S: SimpleSchematic>() -> impl SimpleSchematic<Component = (Self::Component, S::Component)>`
* `alternative<S: SimpleSchematic>() -> impl SimpleSchematic<Component = Result<Self::Component, S::Component>>`
* `filter<F: Fn(&Self::Component) -> bool>() -> impl SimpleSchematic<Component = Self::Component>`

(`SimpleSchematic` is a schematic that only handles one schematic component. All derived schematics are also simple schematics)

### Writing `Schematic` manually

#### Conversion from schematic to runtime world

Conversions from schematic to runtime world are just normal systems that run on the schematic world and have a `SchematicQuery` parameter.
A `SchematicQuery` works almost the same as a normal `Query`, but when iterating over components will additionaly return `SchematicCommands` for the component queried.
This can be used to insert and modify components on the corresponding entity on `CoreWorld::Main` as well as spawn new entities there.
The entities spawned in this way can not be modified by `SchematicCommands` from other systems, but will be remembered for this system similar to `Local` system parameters.

A typical conversion will look like this:

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

#### Conversion from runtime to schematic world

You can also optionally write conversions from the runtime world to the schematic world.

```rust
fn inference_for_a(
    query: InferenceQuery<(&MainA, &Children), SchematicA>,
    child_query: InferenceQuery<(Entity, &MainAChild)>,
) {
    for ((a, children), inference_commands) in query {
        let result = children
            .iter_many(children)
            .filter_map(|result| result.ok())
            .next();
        let (child, a_child) = match result {
            Some(value) => value,
            None => continue,
        };
        let entity = inference_commands.map_entity(a_child.0);
        inference_commands.infer(SchematicA(a.0, entity));
        // This is important so that the schematic world knows about this relation
        inference_commands.set_entity("child", child);
    }
}
```

#### Update schematic components

For more efficient tracking you should also add update systems

```rust
fn update_for_a(
    schematic_a: &mut SchematicA,
    main_a: &MainA,
    main_a_child: SchematicUpdate<&MainAChild, "child">,
) -> bool {
    schematic_a.0 = main_a.0;
    // TODO: Introduce entity mapping here
    schematic_a.1 = main_a_child.1;
    // return true, as this schematic still exists
    true
}
```

Example for "enum schematic"

```rust
#[derive(Component)]
enum AnimationState {
    Idle,
    Walking,
    Jumping,
}

#[derive(Component)]
struct Idle;

#[derive(Component)]
struct Walking;

#[derive(Component)]
struct Jumping;

fn update_animation_state(
    animation_state: &mut AnimationState, 
    idle: &Option<Idle>,
    walking: &Option<Walking>,
    jumping: &Option<Jumping>
) -> bool {
    match (idle, walking, jumping) {
        (Some(_), _, _) => *animation_state = AnimationState::Idle,
        (_, Some(_), _) => *animation_state = AnimationState::Walking,
        (_, _, Some(_)) => *animation_state = AnimationState::Jumping,
        _ => return false,
    }
    true
}
```

* If for a schematic there is only an update system but no inference, then no new instances of this schematic can be found during runtime.
* If for a schematic there is only an inference system but no update, then the schematic gets replaced by a new one every time the schmatic world is updated.
* If for a schematic there is both, then inference works as expected (schematics are updated if they still exist, new ones are created).
* If for a schematic there is neither, then no inference is done (schematics are not updated, no new ones are created).

#### Putting the algorithms together

```rust
trait Schematic {
    type InterpretParams;
    type InferParams;

    fn interpret() -> Option<Box<dyn IntoSystemDescriptor<InterpretParams>>;
    fn infer() -> Option<Box<dyn IntoSystemDescriptor<InferParams>>>;
    fn updates() -> Vec<Box<dyn IntoSchematicUpdate>>;
}

trait SchematicUpdate {
    type Component: Component;
    type Params: SchematicUpdateParams;

    fn update(component: &mut Component, params: Params) -> bool;
}
```

Notice that an implementation of `Schematic` is not tied to any component.
The trait can also be implemented for any unit struct unrelated to any components similar to `Plugin`.
As such you can write your own implementations for existing schematic components.

### Adding schematics to your app

`Schematic`s can be added to the app using
```rust
app.add_schematic::<AnimationState>();
```

Since with many components you want to just copy the same component from the schematic to the main world, a `DefaultSchematic<A: Clone>` is provided.

```rust
app.add_schematic::<DefaultSchematic<Visibility>>();
```

### \[Optional\] Collect-Resolve-Apply

@SamPruden you can add your stuff here

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

* Conversion from the runtime to schematic world
* Conflict resolution in the schematic world vs during conversion
* "Function style" vs "system" schematics
* Handle removal of components in the schematic world
* Checking that every component is read during conversion and similar things

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

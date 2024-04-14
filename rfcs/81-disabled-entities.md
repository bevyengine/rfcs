# Feature Name: `disabled_entities`

## Summary

Disabled entities are hidden from most queries, the app should behave as if these entities are no longer there. This could be achieved using a special `Disabled` marker component.

## Motivation

For one of various reasons we might need to temporarily stop an entity from executing any of its behavior. This should be done in a simple and performant manner. To allow for this bevy needs to introduce a feature to disable entities.

Use-cases that would heavily rely on `Disabled`:

- Rollback networking:
  - When rolling back to an earlier frame, there is a very big chance that some entities did not exist yet, if these entities stay around they would cause resimulation to be incorrect. These entities could instead be `Disabled`, and either have a marker attached to them for when they get enabled again, or get re-enabled if they get spawned again.
  - For similar reasons entities cannot be immediately despawned, instead they need to be disabled until they can no longer be rolled back to.
- Networking interpolation. Data received from the network is frequently interpolated to avoid jitter caused by variation in latency. When data for a new entity is received that should not be shown yet, it can be spawned but kept in a `Disabled` state.
- Background loading. Scenes could be loaded in a `Disabled` state and finish up entirely in-place, once the whole scene is ready can the old scene be despawned and the new one enabled.
- Switching scenes. Currently switching scenes involves spawning and despawning lots of entities. This can be very heavy when frequently switching scenes, instead all inactive scenes could be kept in a `Disabled` state.

Use-cases that could be simplified using `Disabled`:

- Toggling effects:
  - A sci-fi game might have force field barriers that can be toggled on or off, the barrier entity could simply have `Disabled` added or removed. Hiding the visuals and allowing characters to pass trough.
  - A laser turret has a laser which is not active when it has no target, the laser entity could be kept around and be `Disabled` while there is no target. Hiding the visuals and preventing it from triggering collision events.
  - Despawning a particle effect could cause the handle and all relevant information to be dropped, causing all partciles to disappear immediately. Working around this would mean storing lots of data in resources. If they are instead `Disabled` these particles have time to disappear before despawning the entity entirely.
- Effect pools. Some games need to frequently spawn lots of identical effects, and the spawning becomes the bottleneck. Keeping `Disabled` versions of these entities around would simplify reusing entities when such optimizations are necessary.

## User-facing explanation

The `Disabled` component temporarily removes entities from all queries that do not mention `Disabled` in either their query data or filter.
Systems should function in such a way that `Disabled` entities do not influence the app's behavior. Since they are still accessible when requesting with `With<Disabled>`, systems to manage when these entities should be re-enabled or despawned permanently can still use the ECS mostly like usual.

Ignoring `Disabled` entities is not always the desired behavior for any given system. Some systems will want to access disabled entities for their influence on the hierarchy, or to synchronize or save that state.

## Implementation strategy

### The component

Define a `Disabled` component. Check if a query references `Disabled` in any way, if it doesn't add a `Without<Disabled>` filter.

### Handling of the component

Not all systems will handle `Disabled` entities correctly out of the box. For some systems, which don't contribute directly to the behavior of the app, it doesn't make sense to handle them either. Most systems that wouldn't handle `Disabled` correctly respond only to changes. These systems need to pick up disabled entities as if they are gone, this could be done trough an event stream with disabled entities, or trough observers. When these entities get enabled again all components could be marked as changed to make sure it gets picked up.

## Drawbacks

Filtering every query in the app adds a tiny bit of overhead. `Disabled` is one extra concept to keep in mind while writing plugins.

## Rationale and alternatives

Disabling entities in-place would be the ideal mix between simplicity and efficiency. Having a universal way to disable is the simplest way to support usecases that are not aware of all the details of your app all the details of your app.

There are two main alternative approaches:

- Using multiple individual marker components. This approach would create incredble amounts of complexity for anyone trying to disable enttiies, and expecting every crate in the eco system to have marker components is not realistic.
- Despawning the entities and respawning them later. This approach breaks references to the existing entity as the new entity id would always be different, it would also be far less efficient.

Not supporting this feature would heavily restrict the approaches users can take when the need to disable an entity arises.

One viable slight alteration to this approach would be to allow the registration of components that filter entities by default. Follow the same rule as `Disabled`, for each of them individually, but also add a general way to opt-out of all of them. This way entities could be disabled by multiple separate systems for different reasons at the same time.

## Prior art

Many game engines (including the big three) already have functionality to disable specific entities/objects/nodes.

Flecs features a similar `Disabled` component design.

The concept of keeping disabled versions of entities around instead of despawning and spawning new ones is common when it comes to rollback networking.

## Unresolved questions

- How exactly do we expect rendering to handle this?
- Would this get in the way of any current plans?
- Should `Disabled` propagate?
- Is `Disabled` the right name?

## Future possibilities

If systems handle entire entities being disabled correctly, disabling individual components would be easy to support in a similar way.

Support for disabled entities could allow crates for complex topics like networking and rollback to function.

Having this concept around could simplify similar designs:
- A marker component to hide entities that users shouldn't mess with. For example: `SystemId` entities, `Asset`s as Entities, `Component`s as Entities, entities in a entity-based schedule, etc.
- A marker for "prefab" entities, this could work similar to the background loading use case and get similar benefits, except the entities are copied into a new entity.

# Feature Name: `removed-filter`

## Summary

`Removed<C>` can be used as a query filter, with reliable change detection behavior.
Users must opt-in to enabling this by configuring the `Component` trait for `C`.

## Motivation

Removal detection is a powerful tool for ensuring the validity of game state, but the existing `RemovedComponents` system parameter is unwieldy (see the motivating [bevy #2148](https://github.com/bevyengine/bevy/issues/2148)).

It cannot be easily composed with other queries, has an unexpected API, and only persists for two frames, unlike reliable change detection.

## User-facing explanation

In addition to `Changed` and `Added` query filters, you can also use `Removed` query filters.
As you may expect, a query filter for `Removed<Asleep>` will filter the list of entities returned by your query to only include entities which have had a component of the type `Asleep` removed from them since the last time this system ran.

For performance reasons, this behavior is opt-in and attempts to use removal detection without having opted-in will result in a run-time panic.
You can enable it by configuring the `Component` trait.
In practice, this will generally be done like so:

```rust
#[derive(Component)]
#[component(removal_detection = "true")]
struct Asleep;
```

However, be mindful that queries can only return entities which currently exist.
If you need to respond to component removal of any kind, including that caused by entity despawning (as you might if you were attempting to maintain a valid entity graph), use the `RemovedComponents` system parameter instead.

Note that this system parameter returns a list of entities which must be manually handled, and only tracks removals that occur over the past two game loops.
As a result, you should prefer the `Removed` query filter where possible.

## Implementation strategy

Steps:

- add a `LastRemoved<C: Component>` marker component, which stores the `change_tick` of the world at the time of entity removal
- add an associated `RemovalDetection` bool to the component trait
- if `RemovalDetection == true`, the `remove` method on `World` also inserts an appropriate `LastRemoved<C>` component to that entity
- modify `RemovedComponents` to only update when this value is modified
- add a `Removed<C: Component>` query filter using the `WorldQuery` trait with `Fetch: FilterFetch`, which fetches the value of `LastRemoved` and appropriately filters entities using the same logic as `Changed` or `Added`

## Drawbacks

1. Users must opt-in to removal detection.

## Rationale and alternatives

### Why do we want opt-in removal detection?

1. Removal detection is much rarer than change detection, at least in existing code bases.
2. More data must be stored: each component that *was* present must store its own `u32`.
3. Removal detection pollutes the component list of entities in user-visible ways if the type is `pub`.

### Why can't we simply use a `Removed<C>` component?

1. Conflicting trait implementations for `WorldQuery`, since query filters use the same trait.
2. Strange and confusing results when users accidentally use `Removed<C>` in the wrong type position.

### Why can't we remove the `RemovedComponents` system parameter?

Unfortunately, you can only query for entities which *currently exist*.
One of the important use cases of `RemovedComponents` is checking for entity despawning.

We can't fully replace this functionality, so the old API must stay.

### Why not address "removed component data access" at the same time?

While this is another [commonly desired feature](https://github.com/bevyengine/bevy/issues/1655), it has substantially more implementation challenges, and is beyond the scope of this small RFC.

It should also be configured separately, due to the increased data storage demands.

## Unresolved questions

1. Should removal detection be opt-in, always enabled, or automatically discovered?
2. Should the `LastRemoved` type be `pub`?
   1. Users can manually mess things up if it is `pub`.
   2. Users can only clean up unneeded `LastRemoved` components if the component type is `pub`.
3. Can we move the "failed to opt-in to removal detection" check to compile time?
4. Is this the correct way to specialize `remove`?

## Future work

1. Removal detection (and change detection) could somehow be automatically detected and enabled if systems are present in the schedule that require it.
2. The hypothetical (and trivial to implement) `Not` query filter modifier will be more useful with the addition of the `Removed` query filter.

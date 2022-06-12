# Feature Name: `dynamic_query`

## Summary

Expand the query interface of `World` to support queries with parameters which are determined at runtime, not compile time, to enable future scripting and hot reloading.

## Motivation

At present, Bevy leverages Rust's powerful type system to query entities within its ECS architecture. Static 
type-based queries are a defining element of Bevy's ergonomic approach to ECS. However, static queries' 
reliance on Rusts type system limits the types of queries which can be performed in `bevy_ecs`, as queries 
must be defined at compile time. The `dynamic_query` RFC provides an alternative interface for performing 
queries using `ComponentId`s at runtime, as the next step towards exposing the full power of bevy to scripting 
and other types of dynamic plugin loading.

## User-facing explanation

As a bevy developer with a single vision who wants the best performance availabe, dynamic queries are strictly 
inferior to their statically typed counterparts. However, developers who want to add support for scripting and mods 
need a method of interacting with the Entity-Component-System at runtime, which is a niche filled by the 
`DynamicQuery` type.

Using static typing, you would perform a query like this:

```rust
let query_state = world.query::<(Entity, &Velocity, &mut Position, Option<&Layer>), Without<Frozen>>();
```

While the dynamic alternative looks like the following:

```rust
let dynamic_query = DynamicQuery::new()
                        .entity()
                        .component(velocity_component_id)
                        .mut_component(position_component_id)
                        .optional_component(layer_component_id)
                        .without(frozen_component_id)
                        .build();

let query_state = world.query_dynamic(&query);
```

A static query returns the various components it has retrieved in a tuple. This isn't an option for a Dynamic Query
however, since the number of components being returned isn't known at compile time. Instead, a Dynamic Query returns 
a `DynamicQueryEntity` struct, which is both indexable and iterable to access the individual `DynamicItem`s it
contains.

A `DynamicItem` is an enum, which can represent the different types returned in a query result. Here are its
variants:
```rust
pub enum DynamicItem {
    Entity(Entity),
    Component(DynamicComponentReference),
    MutableComponent(DynamicMutComponentReference),
    ComponentNotPresent
}
```
The two inner types, `DynamicComponentReference` and `DynamicMutComponentReference` provide safe access to 
`&`/`&mut` and change tracking functionality while also enabling users to have access to the underlying pointer for 
more advanced usage.

```rust
let dynamic_query = DynamicQuery::new()
                        .entity()
                        .component(velocity_component_id)
                        .mut_component(position_component_id)
                        .without(frozen_component_id)
                        .build();

for result in world.query_dynamic(&query).iter_mut() {
    info!("Updating the position of Entity {}", result[0].unwrap_entity().id())
    result[2].unwrap_mut_component().downcast_mut::<Position>().unwrap() += result[1].unwrap_component().downcast_mut::<Velocity>.unwrap();
}
```


## Implementation strategy

One of the key objectives of any implementation of Dynamic Queries into Bevy must be to minimize its footprint 
within the code base. Significant sacrifices shouldn't be made for the static queries in order to improve dynamic 
support. With that in mind, a dynamic query implementation must use as much of the existing architecture of 
`bevy_ecs` as possible, maintaining two separate `QueryIter` implementations would be frustrating and fragile.

This RFC proposes the following implementation:

A new internal extension trait is introduced, `DynamicQueryState`, with the following definition:
```rust
trait DynamicQueryState {
    fn new_dynamic(world: &mut World, query: &DynamicQuery) -> QueryState<DynamicQuery, DynamicFilterQuery>;
}
```
The trait is only implemented for `QueryState<DynamicQuery, DynamicFilterQuery>` type, providing a method for 
creating `QueryState`s which derive their internal `fetch_state` and `filter_state` fields from a function parameter 
instead of their type parameters. A corresponding `query_dynamic` function is added to `World`. `DynamicQuery` 
defines `fetch` and `filter` functions which initialize `FetchState`s for the fetch and filter components of the
query.

From here, the remaining implementation is fairly straightforward, with `DynamicFetchState`, `DynamicFilterState` 
which match on an enum to mimic the behaviour of their static countrerparts, along with accompanying `DynamicSet`
equivalents which combine the fetched items into a single return value.

Since the recent addition of derive `Component`, `is_dense` has gone from a `fn` to a `const` within the `Fetch`
trait. This represents a problem for dynamic queries, as the storage types of the query's components aren't known
at compile time. As a result, all dynamic queries will be sparse archetype queries instead of table queries.

## Drawbacks

Dynamic queries offer a competing interface through which users will be able to query for entity data, but one that
is inferior to the static interface in all but a very specific subset of cases. While this is a concern, it isn't
one which will have much of an impact as long as documentation guides users towards the standard static queries, and 
makes it clear that the primary audience of dynamic queries is plugin authors. The interface has been designed with 
ergonomic builders, but it's unlikely that the intended consumers of the dynamic query API will ever user it in this 
fashion.

## Rationale and alternatives

While the solution to the dynamic query problem presented in this RFC does not necessarily represent the best 
possible implementation of the internal dynamic query mechanics, it offers an ergonomic interface which can meet the
needs of any consumer of the dynamic query API, while requiring as few changes as possible to the implementation of 
`QueryState` and `QueryIter`, keeping the maintenance burden of offering the dynamic query interface as small as
possible. 

It's likely that implementing dynamic queries in this way will not have optimal performance since it's
forcing a type-erased interface into a strongly typed hole, but the much lower labour requirements for implementation
and continued upkeep justify this downside. In addition, it isn't strictly necessary to achieve the best possible
performance. In the cases where dynamic queries are required, it's extremely unlikely for the query performance to 
be the bottleneck when compared with the performance overhead of scripting interop.

Enabling scripting is extremely important for Bevy though, even as a second class citizen within the
ecosystem. It is inevitable that as the engine grows, games developed with the engine will find the need to include
robust modding support into their titles, which would be dramatically more difficult without support directly
integrated into the engine. Additionally, Bevy projects which reach a certain size will likely require some method
of game programming for less technical designers and artists which would find Rust concepts intimidating to learn.
Dynamic queries will enable future engine plugins to accommodate these users in an officially supported manner
which can be reused across many different scripting solutions.


## Prior art

Back when bevy's ECS was still based on hecs, zicklag took a swing at implementing both dynamic queries and
components in bevy PR [#632](https://github.com/bevyengine/bevy/pull/623). The underlying ECS has changed
radically since then, so the only element which could be compared would be the interfaces for building dynamic
queries. In that one small way, the builder pattern employed by this RFC offers a more ergonomic method for
constructing queries, though the merits of those ergonomics for a feature only itended for advanced usage is
questionable.

## Unresolved questions

These questions remain unresolved:
- How much of the type's information should the `DynamicComponentReference` types carry with them (Layout?)?
- How can change tracking information be captured in dynamic queries?
- Should dynamic queries be hidden behind a feature?
- Which traits should the `DynamicItem`/`DynamicItemSet` types implemented for maximum convenience?
- Do we even want maximum convenience?

## Future possibilities
Dynamic queries lay the groundwork for future expansion of the ECS to fully support runtime behaviour changes. 
Plugins could dynamically define systems at runtime based on configuration files or scripts, since they now have the 
tools to interact with the Entity Component System freely. This addition of this single RFC would still leave Bevy a
long way from full dynamic support though. Future RFCs should set up implementations for inserting new systems and 
component types at runtime to further expand possible mod/script support.

Another important engine feature that could build on top of dynamic queries is hot reloading. Individual rust 
modules could be compiled to WASM and loaded by a dedicated plugin to add systems which could be hot reloaded at 
runtime. Allowing Bevy developers to skip potentially expensive recompilation and reloading by hot swapping code 
while the game runs would dramatically improve Bevy's inner loop and enable developers to be more productive with 
the engine. This may not be the best possible solution for hot reloading, but it is an option which dynamic queries 
enable.

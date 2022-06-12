# Feature Name: `world_query_filter`

## Summary

Disallow query filter types that don't make sense as filters, e.g. `&MyComponent`.

## Motivation

Currently it is possible to create `Query<Entity, &MyComponent>` with `&MyComponent` as a filter. Currently `&MyComponent` is equivalent to `With<MyComponent>`, but it is not clear from the code and is error prone. Disallowing passing component names to the query filter parameter eliminates this class of programming mistakes, thus improving ECS API ergonomics.

## User-facing explanation

No extra explanation is needed. Current documentation already suggests using `With<MyComponent>` for filtering on component existence. This proposal does not affect already unambiguous code.

A migration path of replacing `&MyComponent` with `With<MyComponent>` should be proposed in th change logs for the few users who have ambiguous code.

## Implementation strategy

1. Introduce a new `trait WorldQueryFilter: WorldQuery {}` with empty interface
2. Implement it for:
    * `With`
    * `Without`
    * `Added`
    * `Changed`
    * `Or`
    * Tuples of the above
3. Replace the trait bound in the `Query`:
    * Before: `struct Query<'world, 'state, Q: WorldQuery, F: WorldQuery = ()>`
    * After:  `struct Query<'world, 'state, Q: WorldQuery, F: WorldQueryFilter = ()>`
4. Let the Rust type system handle the validation via usual type checking

## Drawbacks

- The proposed change may break user code, if it is not canonical, but the migration is mechanical

## Rationale and alternatives

The main alternative design is to make `WorldQueryFilter` a fully functional trait with:

1. It's own `*Gats` trait
2. It's own `FiterState` trait
3. It's own `Fiter` trait
4. Moved `*_filter_fetch` methods out of `Fetch`
5. etc

This design is more difficult to implement yet it does not provide any additional benefits to Bevy users. It may provide benefits to Bevy developers by making the code cleaner. However all that work would be wasted if the further proposal discussed in the "Future possibilities" is implemented as that one would yet again require redesigning the trait system for `Fetch` / `Filter`. For that reason the simpler design is chosen.

## \[Optional\] Prior art

N/A.

## Unresolved questions

N/A.

## \[Optional\] Future possibility: implementing the reverse restriction

Just like not allowing `&MyComponent` in the filter, it also makes sense to not allow filter types like `With<MyComponent>` in the main query. Currently `Query<With<MyComponent>>` is allowed and produces a stream of empty tuples `()`. By the same reasoning as in the current proposal, this is error prone and making such code a compile-time error would improve ECS API ergonomics.

This idea is harder to implement as it requires changes to existing traits. The detailed design for that feature is left for a future RFC, conditioned on this one being merged and implemented.

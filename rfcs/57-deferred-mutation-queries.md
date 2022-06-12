# Feature Name:
Deferred Mutation Queries

## Summary

Easy and predictable way of deferring component mutations to increase parallelity.

## Motivation

Accessing components mutably is costly, because we can only run one mutable query at a time.
On the other hand, references are extremely cheap and we should abuse using them when possible.

The problem is that we cannot simply get rid of mutation, it has to happen at some point.
That's why simply putting off the mutation does not fix anything.

This RFC targets the use cases, where we borrow components mutably, despite not mutating many of them or often.
Those cases are often external events, such as user input or networking.

## User-facing explanation

### When to use

It's important to know, that this feature is not a magical optimizer.
It's meant to be used in very specific use cases and will require benchmarking to determine whether it should be used.

The specific set of rules where this would be applicable is:
- The mutated component has to implement `Clone`.
- No system before next sync point depends (changes it's behavior) on the values of mutated component.
- There are other systems that also want to query this component in parallel (otherwise we gain no benefits).

Most fit for this are external inputs, such as user input and networking, but also arbitrary mutations from randomness.
Physic simulation could also benefit from this, after all, many components are static or frozen and their counts are low.

### How to use

The API shouldn't be much different to a normal query with mutability.
Ideally this would be a change from `Query<&mut Foo>` to `Query<DelayMut<Foo>>`.
The body of the system shouldn't see any changes.

It may be that implementation requirements bleed into `Query` itself, transforming it into some special `DelayMutQuery`.

It would look something like this:
```rs
fn foo(
    mut bars: DelayMutQuery<DelayMut<Bar>>,
) {
    for bar in bars.iter() {
        if arbitrary == condition {
            bar = new_bar;
        }
    }
}
```
or with a different approach:
```rs
fn foo(
    mut bars: DelayMutQuery<DelayMut<Bar>>,
    targets: Res<SomeEntityList>,
) {
    for entity in targets {
        bars.get_mut(entity) = new_bar;
    }
}
```

## Implementation strategy

A naive implementation can be done with the following code:
```rs
fn foo(
    // We need to `Entity` to apply mutation to the right entity
    query: Query<(Entity, &Bar)>,
    // We need `Commands` to buffer the mutation for future application
    mut commands: Commands,
) {
    for (entity, bar) in bars.iter() {
        if arbitrary == condition {
            // Component has to implement `Clone`
            let mut mutated_bar = bar.clone();
            // To show the need for `Clone` in this example I only mutate a field
            mutated_bar.0 = value;
            // Use `insert` like an `upsert`, not too obvious
            commands.entity(entity).insert(mutated_bar);
        }
    }
}
```
As you can see it's a lot of additional work that's mostly boilerplate,
but it outlines the necessary components.

The smart copy-on-write-like pointer has to store the `Entity` as well as a mutable reference to the `Commands`.
The component itself has to implement `Clone`.

With this approach the order of mutation will be determined by the order of command applications in the sync points.

## Drawbacks

The use-cases are little to the point it may not be worth implementing.

Speed benefits are also to be tested and can, potentially, spectacularly kill this idea.

## Rationale and alternatives

Staying with `&mut` is always an option.

Resolving mutable access at runtime within systems is a massive **no**.

## Prior art

The topic as far as I'm aware is unexplored to the point it's hard to predict the benefits.
It may be, that the parallelizm of those few use cases simply does not outweight the algorithm behind it.

I've not aware of anyone using the naive implementation. My assumed reason for that is that it's non-intuitive, since it uses `insert` for upsert.

## Unresolved questions

- Because `Commands` are parameter-level and component wrappers aren't, the query signature may require changes like `Query` -> `DelayMutQuery`.

- Order of mutations

## Future possibilities

- Separation from `Commands` would allow for
    - Different sync points from entity spawning and despawning
    - Resolving ordering early as opposed to during application
    - Batching mutations AA

- Lensing would allow to targeting more components.
  Instead of copying everything, only the modified fields can be memorized.
  This means that non-`Copy` components are also usable.
  This also means that large components (memory-wise) can be updated with less memory usage.

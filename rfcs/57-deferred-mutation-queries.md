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

### Naive implementation

can be done by any user with the existing codebase with the following:
```rs
fn foo(
    // We need `Entity` to apply mutation for identification
    query: Query<(Entity, &Bar)>,
    // We need `Commands` to buffer the mutations
    mut commands: Commands,
) {
    for (entity, bar) in bars.iter() {
        if arbitrary == condition {
            // Component has to implement `Clone`
            let mut mutated_bar = bar.clone();
            // To show the need for `Clone` in this example I only mutate a single field (as opposed to replacing the whole component)
            mutated_bar.0 = value;
            // Use `insert` like an `upsert`, not super obvious
            commands.entity(entity).insert(mutated_bar);
        }
    }
}
```

It's a lot of work that's mostly boilerplate, but it outlines the necessary components.

The smart copy-on-write-like pointer has to store the `Entity` as well as a mutable reference to the `Commands`.
The component itself has to implement `Clone`.

With this approach the order of mutation will be determined by the order of command applications in the sync points.

### Dedicated implementation

If naive implementation brings enough performance improvement through benchmarking, a dedicated solution can be made with guaranteed performance benefits.

The biggest optimizations can be achieved through:
- Batching mutations of the same component type in advance.  
  Instead of collecting all mutations right before applying them
  collect them during the registration of mutation
  or somewhere on the way.
- Sorting mutations of the same component type.  
  Will dramatically reduce cache misses.
  Most likely insert sort during batching.
- Applying mutations per component type in parallel.  
  Unlike `Commands` which locks the entire world.

The application of mutations can be done separately from command resolution
furthermore each component can be applied at different moments.

Registration of those systems will need to be explicit like `apply_mutation_system<T: Component + Clone>`,
but convinience `App` methods can be provided to implement them at some default moment.

The query will require a special type, like `DelayMutQuery`, to contain the local buffer (naive implementation's `Commands`).

(I need to read through the command architecture to be sure, but...)
This may require modifications in the scheduler, new type of benchmark will be required to ensure the overhead is minimal as compared to the existing implementation.

## Drawbacks

The use-cases are little to the point it may not be worth implementing. Although it won't stop me from trying! :')

Performance benefits are hard to guess and require benchmarking, even in user code.

## Rationale and alternatives

Just using `&mut`.

~~Resolving mutable access at runtime.~~ (only villains do that)

## Prior art

The topic as far as I'm aware is unexplored.

I have not seen anyone use the naive implementation.
I assume it's non-intuitive, since it uses `insert` for upsert.

## Unresolved questions

- Order of mutations

## Future possibilities

- Lensing would allow for targeting more components at questionably smaller memory cost.
  Instead of copying everything, only the modified fields can be memorized.
  This means that non-`Copy` components are also usable.
  This also means that large components (memory-wise) can be updated with less memory usage.

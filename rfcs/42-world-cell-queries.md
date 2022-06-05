# Feature Name: `world_cell_queries`

## Summary

Allow component queries to be issued through `WorldCell` API in single-threaded systems.

Allow the user to relax usual borrowing semantics in favour of runtime checks. Allow for simultaneous querying
of world data that is hard to prove disjoint at compile time. Rely on runtime assertions to provide safety.

Allow performing world modifications through WorldCell shared access using command API.

Make the world cell queries always reflect the most recent updated world data, without requiring explicit
sync points inside the single-threaded systems.

## Motivation

Bevy ECS is currently quite well suited for high data throughput and relatively low complexity scenarios.
Most focus is being put on general performance and ergonomics of safe multi-threaded query system.
That high performance API imposes some hard limitations on how to structure the data, effectively requiring
every system to be carefully designed around rust borrowing rules, as well as making complex data dependency chains
sometimes very hard to deal with.

While this trade-off might be right for a lot of scenarios, there is currently no way out of it. Most notably, it can
easily get in the way of complex gameplay logic prototyping, where development speed is much more important than raw performance.
Similarly, some systems might not be performance critical, but annoying to write with usual query limitations in place.

The trade-off of performance versus code complexity should be a decision that a developer could make on their own.

## User-facing explanation

### World cell query

When having access to a `WorldCell`, you are free to issue any ECS queries you wish, at any time. Even ones that seemingly break rust principles.

```rust
pub fn single_threded_system(world: &mut World) {
    let mut cell = world.cell();

    // create two seemingly conflicting queries and iterate over them
    let query_ref = cell.query::<(Entity, &Transform)>();
    let query_mut = cell.query::<(Entity, &mut Transform)>();
    let mut iter_ref = query_ref.iter();
    let mut iter_mut = query_mut.iter();

    iter_ref.next(); // offset mutable query by one item to avoid aliasing

    for (previous, next) in iter_mut.zip(iter_ref) {
        // assign a transform from one entity to another
        *previous.1 = next.1.clone();
    }   
}
```

In the above example, the two queries would not be allowed to coexist in a single system. But they are still used in a way that will never lead to mutable aliasing.
This is because one of the queries is being offset during actual iteration. Because the rust aliasing invariant checks are moved to runtime, this code is able to run.

Any unexpected situation when the safety invariant is not upheld is being detected at runtime, at the time of access to given entity.
In above example, removing `iter_mut.next()` call would lead to a runtime panic: 
```
Component 'Transform' of entity 1234v5 already borrowed
```
That panic is being issued from inside the query iterator's `next` method, which is implcitly called by the `for` loop.

Any combination of queries is valid and both can be iterated over at the same time. It's the code author's responsibility to check if mutability rules are not violated.
This is a similar tradeoff to usage of `RefCell` from standard library in cases where rust rules are too restrictive.

### Shared access commands

Any time you have access to a `&WorldCell`, you are able to dispatch commands using API consistent with `Commands` system parameter.
```rust
// spawn new entity
let e = cell.spawn().insert(C).id();
// remove a component from existing entity
cell.entity(e).remove::<B>();
// add a component to existing entity
cell.entity(e).insert(A);
```

This API is similar to `Commands` or `World`, but doesn't require unique world access. That means it can be used during query iteration.

```rust
let query = cell.filtered_query::<Entity, (With<A>, Without<B>)>();
for entity in query.iter() {
    // find every entity with component A and insert a component B
    cell.entity(entity).insert(B);    
}
```

### Immediate application of commands on queries

Any `WorldCell` query iterator is operating on a snapshot of world, as if all issued commands has been applied.
This allows quick prototyping of game systems without having to think too much about structuring your algorithm
to work within the bevy ECS constraints immediately.

```rust
// create world with two entities
let mut world = World::default();
let e1 = world.spawn().insert(A).insert(B).id();
let e2 = world.spawn().insert(A).id();

let mut cell = world.cell();

let query = cell.filtered_query::<Entity, (With<A>, Without<B>)>();

// verify query output before any modifications
assert_eq!(query.iter().collect::<Vec<_>>(), vec![e2]);

// remove C from one entity, insert on another
cell.entity(e1).remove::<B>();
cell.entity(e2).insert(B);

// new query results reflect the changes immediately
assert_eq!(query.iter().collect::<Vec<_>>(), vec![e1]);
```

Only commands issued before `iter()` are taken into account when iterating.
That way the iterator that's already started cannot be influenced by future commands, preventing unpredictible
feedback cycles from happening (e.g. growing an iterator indefinitely with component inserts inside its loop body).

This is mostly a safeguard feature to keep working with iterators as simple as possible.

```rust
let query = cell.query::<(Entity, &A)>();

let iter1 = query.iter();
let inserted = cell.spawn().insert(A).id();
let iter2 = query.iter();

// iterator created before insert doesn't include new entity
assert!(!iter1.any(|(id, _)| id == inserted));
// iterator created after insert does include new entity
assert!(iter2.any(|(id, _)| id == inserted));
```

## Implementation strategy

This API is meant to be permissive and easy to use. Performance would be nice, but it is secondary to actually being able to implement it.
As long as it doesn't significantly affect the performance of the multi-threaded path.

While `WorldCell` is live, the world is necessarily mutably borrowed. That means there is absolutely no interaction with other systems being run at the same time.

That means we have perfect sequential order of thing. We can do all sorts of tricks to "fake" the immediate application of commands on queries, as long as it's guaranteed
that the changes are persisted once the `WorldCell` is dropped. 

Given only [minimal changes](#necessary-changes) to the existing world internals, the whole `WorldCell::query` system can be implemented as a wrapper type around standard queries.
The query iterator would delegate most of the work to the existing query system, and only perform necessary modifications around entities that have been modified through commands.
It also have to figure out which entities to add to the query, based on queued commands. This is essentially an "world with changelog" iterator.

```rust
// simplified view of WorldCell query iterator state
struct CellQueryIter<'w, 's, Q, F> {
    // it queries the world in its original unmodified shape, and overlays changes on top of returned results
    query_iter: QueryIter<'w, 'w, Q, F>,
    // extra query that is able to extract partial component data from the world. Necessary for cases when
    // previously excluded entity would have to be queried due to queued commands. Only accessed through entity IDs.
    partial_query: &'s QueryState<(Q, F)::PartialQuery>,
    // metadata about world snapshot taken at the time of iterator creation
    overlay: WorldOverlay,
    // reference to the command queue. Required to access inserted component data.
    command_queue: &'w CellCommandQueue
}
```

The `partial_query` bit could be replaced with direct access to the ECS data, but that would require an addtional internal API that allows unsafe data access.
Current design proposal tries to minimize number of changes to existing world internals.

Instead of usual component references, queries return all component data through wrappers types. Similar to already existing `Mut` wrapper for `&mut X` world fetches.
The existance of those wrappers allows for both runtime checks and "world snapshots" with all commands applied - the wrapper can reference either world or a command queue data.

The command queue is implemented as an append log - no commands can be removed from the queue, until the whole queue is flushed. The commands can be inserted into the queue
by shared access, without ever triggering backing store reallocation. That guarantees pointer stability of the commands, which is necessary for queries to be able to return
references to the command data. This is necessary to allow new commands to be submitted while holding onto queried data.

### necessary changes

Necessary changes are only related to query initialization - it must be possible to perform with shared world access.

There are workarounds to that, but API would have to be modified in a way that reduces ergonomics. This is also how a proof-of-concept is initially implemented.

Such API requires splitting query creation into two steps - with and without mutable cell access:

```rust
// guarantee that the query is initialized. Requires &mut WorldCell
let token1 = cell.init_query::<A>();
let token2 = cell.init_query::<B>();

// once initialization is done, queries can be accessed by shared reference at any time
let query1 = cell.query(token1);
let query2 = cell.query(token2);
```

## Drawbacks

Code complexity is the biggest one. Adding a system that allows single-threaded queries will necessarily have impact on ECS World structure internals in the future.
Those changes will likely impact some implementation details of multi-threaded part, but likely not by much (see (necessary changes)[#necessary-changes]).
Adding new features will be more complex, as any modification of World data layout will have to stay compatible with both single and multi-threaded query implementations.
This is especially true for new query filters or commands. Any query filter compatible with `WorldCellQuery` will have to implement the data overlaying behaviour. Every command
that modifies the world also has to be correctly applied during that overlaying process. Fortunately, not all commands or filters have to be supported in that case, but we do want
to keep the API as consistent as possible.

## Rationale and alternatives

### permissive querying
The world cell queries could check for possible aliasing much earlier, trying to make conflicting queries mutually exclusive similarily to normal systems.
This unfortunately breaks both the "easy prototyping" usecase, as well as makes the "instant commands application" feature almost (?) impossible to implement.

Allowing to break the usual borrowing rules while keeping safety with runtime check is the obvious missing part of current design. This is analogous to RefCell over arbitrary data,
but specialised for ECS World acccess.

### reflect the most recently updated world data

It does naturally follow from "usability first, performance second" rationale.
Having the issued action be immediately visible leads to simplification of the system code.

An alternative would be to require the "sync points" to be manually inserted by the user.

```rust
// apply all queued commands to the world. Requires `&mut WorldCell`.
cell.maintain();
```

This goes agains the ease of prototyping, but does lead to significantly less complicated implementation of `WorldCell` queries.

## Prior art

- whole existing query system and world access is a sort-of prior art. All proposed changes are heavily based on that API, only extending it.
- RefCell - similar idea applied to arbitrary rust structs, part of standard library. Moves borrow checks from compile time to runtime.

## Unresolved questions

- no questions so far

## Future possibilities

Similar relaxed runtime-checked query API could be exposed in multi-threaded systems, trading performance for simplicity.
Multiple potentially-conflicting queries could be ran concurrently within given system.

# Feature Name:
Deffered Mutation Queries

## Summary

Defering mutation of specific query components to increase parallelity.

## Motivation

This is another iteration of my idea few months back
for parallelizing mutable queries.

I think it'd be nice for
run-time selective entity mutation in small counts
and maybe GUI elements.

## User-facing explanation

From user-sided perspective,
this can be seen as a way of doing
efficient sparse mutable access
to query components.

`DeferMut<T: Component>` a generic type,
will fit into queries much like `Option<T>` does.

From the system perspective,
it will act exactly like a `&mut Component`,
systems can freely mutate it.

The difference will be in scheduling,
where it would be treated as `&Component`,
multiple systems can use it in parallel.

You **will** use this wherever there is little
(usualy optional) mutation over a component,
which also happens to be wanted by other systems,
and the mutation is not based on the previous value
of the `DerefMut` component.

This **will not** use this when explicit ordering matters,
because the actual mutation (hidden from user)
will happen somewhere between the 2 systems,
which means they cannot be parallelized.

## Implementation strategy

The idea is to use copy-on-write together with deferred mutation.
When a `DeferMut` component is about to be modified,
we create a copy of it and store it in the query.

After the system is over, we recover the mutated copies
and create a system that will be only responsible for updating them.

This system should respect all the same ordering restricitons as the original,
but it doesn't have to happen directly after it.

Because the system is small, we'll be mutably locking the entities for much shorter.

## Drawbacks

It's mostly optional,
the only draw back I can think of right now
is that it can add more overhead,
memory or/and calculation,
to the queries even despite not being used.

## Rationale and alternatives

Defering the mutation until the end of frame could be more benefitial,
depends heavily on the use case.

mostly todo

## Prior art

Unaware

## Unresolved questions

None for now

## Future possibilities

Batching defered mutations of components of the same type.
This will clash with the ordering, not sure if it's reasonable.

## Additional context

It's 6 am and I'm tired,
I'm just hoping to see some responses.

There may be some very big logic holes in this,
which could turn this into a complete flop.

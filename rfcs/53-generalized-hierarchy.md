# Feature Name: `generalized-hierarchies`

## Summary
Create a general hierarchy or hierarchies for Bevy worlds that is ergonomic to
use and always globally consistent.

## Motivation
Hierarchies are present in all parts of game development. The one that
immediately comes to mind is the Transform hierarchy, where children exist in the
local space of their parents. However, this concept is can be more general than
transform hierarchies:

 - UI elements have their own parent-sibling-child relationships that are not
   strictly expressible in 3D affine transforms.
 - Animations are usually bound to a bone hierarchy that may or may not involve
   Transforms, and is typically queried based on paths within the hierarchy.
 - Certain attributes intuitively should be inherited from parent to child. For
   example, hiding a parent via the `Visibility` component typically means the
   children of that parent (and all of their descendants) are also hidden. Many
   of these properties are not directly tied to transforms or their propagation.
   However, many of these use cases also share a similar pattern of detecting
   changes and propagating them either down to their descendants or bubbling up a
   message through their ancestors.

All of these cases involve mutating, querying, and traversing these large trees
or forests of linked entities. Throughout each of these operations, it's
imperative that Bevy provides a globally consistent view of the
hierarchy/hiearchies.

Many of these operations are not cheap to do at runtime. Queries and traversals
can be linear in the number of children and involve heavy random access patterns
which can be detrimental to game performance.

## Background
In this RFC, we will be using the following terms repeatedly:

 - **Parent:** this refers to the upper level Entity relative to their children. Each
   Entity can either have zero or one parents, but may have zero or more
   children.
 - **Child:** this refers to a lower level Entity relative to their parent. Any child
   will have exactly one parent. Children can have children of their own.
 - **Sibling:** a child that shares the same parent as another different entity.
 - **Roots:** These are entities without parents. May or may not have children.
 - **Leaves:** These are entities without children. May or may not have a parent.

Currently, there are two components under `bevy_transform` that describe Bevy's
current hierarchy: `Parent` and `Children`. Adding or altering the `Parent`
component will not immediately manifest in an update to `Children`.

A dedicated transform hierarchy maintenance system is run every game tick to
update `Children` to match the actual Entities that point to said Entity as a
Parent.  separate `PreviousParent` component is used to keep track of which
entity was the entity's parent before it's current one. This is created, managed,
and removed by the maintenance system. This introduces state where both
components on different entities are out of sync, and relies on the maintenance
system to make it eventually consistent.

In the meantime, querying for either results in a inconsistent state of the
world, and is the subject of much frustration from those not intimately familiar
with the inner workings of the system.

## User-facing explanation
Instead of using three separate components to manage the hierachy, this proposal
aims to have only one: `Hierarchy`. This hierarchy component provides *read-only*
access into an entity's immediate hierarchial relationships: its parent,
previous sibling, next sibling, and it's first child. All of these may return
`Option<Entity>`, signalling that such relationships do not exist. This creates
a single component that creates a top-down forest of entity trees, where each
entity forms a doubly-linked list of siblings. It may also cache infromation
about the component's position within the hierarchy like it's depth.

Mutations to the hierarchy: de-parenting, re-parenting, re-ordering of children,
etc. can only be done via commands. This enforces a globally consistent view at
any given time. Delaying any modifications to a global synchronization point,
much like component removal. The `Hierarchy` component is not publicly
constructible and must be made via the dedicated hierarchy management commnds.

To make querying for specific parts of the hierarchy easier, ZST marker
components will be added and removed to make querying for specific cases easier,
and will be kept in sync with the hierarchy state: `Parent`, `Child`,
`FirstSibling`, `LastSibling`. These can be used in conjunction with ECS Query
filters to speed up finding targetted use cases. For example, `Without<Child>`
can be used to query for roots, or `With<Parent>` can be used to find all of the
non-leaf entities.

To signal changes in the hierachy, a `HierarchyEvent` event type is registered
and any changes made to the hierarchy is published as an event.

## Implementation strategy
The core of this proposal is the `Hierarchy` component, which roughly looks like
this:

```rust
#[derive(Component)]
pub struct Hierarchy {
  depth: u16,
  parent: Option<Entity>,
  prev: Option<Entity>,
  next: Option<Entity>
  first_child: Option<Entity>,
}
```

This type has no public constructors and has no public APIs that allow for
mutation of any of the internal data, only getters. The consistency between
multiple instances of the component is maintained by a set of dedicated hierachy
management commands. The following should be added as extensions to
`EntityCommands`:

  - `set_parent(Option<Entity>)` - sets the parent of a given entity.
      - If `Hierarchy` is on neither parent or child, it will be added and both
        will be atomically updated to reflect this change.
      - If the parent already had children, the new child will be prepended at
        the start of the linked list, and the sibling pointers of both will be
        updated.
      - If the child already had a parent, that previous parent will be updated
        too.
      - A HierarchyEvent is added to the queue to reflect this update.
  - `move_to_first(Entity)` - Moves one of the children to the front.
  - `move_to_last(Entity)` - Moves one of the children to the end.
  - `disconnect_children()` - Disconnects all children from a parent. All
    children become roots.
      - A `HierarchyEvent` is added to the queue to reflect this update.

All of these commands will also update the four marker components appropriately.
Optionally, we could also mark these components as changed (even though they're
ZSTs) to leverage change detection.

One optional extension to this design is to create dedicated `HierarchyQuery<Q,
F>`, which is a custom `SystemParam` that wraps multiple `Query` objects to simplify
querying for items in the hierarchy:

 - `iter_descendants(_mut)` - returns an iterator over `(parent, child)` results
   from a query parameter. This does a depth-first traversal over the hierarchy
   starting at some entity.
 - `iter_ancestors(_mut)` - returns an iterator over `(parent, child)` results
   from a query parameter. This bubbles up queries from the bottom to the top
   starting at some entity.

As these operations are handled at any command buffer flush. The hierarchy
maintenance system can be removed.

## Benefits
The biggest benefit to this new design is guarenteed global consistency. The
hierarchy cannot be arbitrarily mutated between synchronization points.

The second benefit is that any update to the hierarchy is "atomic" and is `O(1)`
for the majority of the operations handled.

## Drawbacks
There are a notable number of drawbacks to this design:

 - Counting children is now `O(n)`. Finding and editing children order is still
   `O(n)`, that has not changed.
 - The fact that the hierarchy is not optimally stored in memory is still an
   issue, and all hiearchy traversals require heavy random access into memory.
 - Updates are still not immediately visible and deferred until the next command
   buffer flushes.
 - Hierarchy updates are now single threaded.
 - The marker components may cause heavy archetype fragmentation.
 - Some potential implementations of this hierarchy requires normal ECS commands
   to be aware of the hierarchy. For example, despawning a child in the middle of
   the linked list without updating it's immediate siblings will violate several
   invariants of the system.

## Rationale and alternatives
The primary goal of this proposal is to get rid of the hierarchy maintenance
system and the consistency issues that come with it's design, which it delivers
on.

### Keep `Parent`/`Children` as is. Implement Commands/Events.
This simply keeps the components as is but restricts both of them to be
read-only. The commands and events seen in this proposal can also be implemented
on top of them. The internals of `Children` could be replaced with a `HashSet<T>`
to keep operations close to `O(1)`, but a consistent ordering will be lost here.
Marker components will not be needed and normal change detection can be used. The
events can also be implemented as a more consistent replacement to
`PreviousParent`.

#### Benefits
This rocks the boat the least and provides the same benefit.

#### Drawbacks
TODO: Complete this section.

### Type Markers in Components
It may be possible to include type markers as a part of the hierarchy components
to differentiate multiple hierachies from each other. This would allow an entity
to be part of multiple hierarchies at the same time.

```rust
#[derive(Component)]
pub struct Hierarchy<T> {
  depth: u16,
  parent: Option<Entity>,
  prev: Option<Entity>,
  next: Option<Entity>
  first_child: Option<Entity>,
  _marker: PhantomData<T>,
}
```

#### Benefits
This allows us to directly tie hierarchies to the components they're targetting.
Instead of a general `Hierarchy`, a `Hierarchy<Transform>` declares it's meant
for `bevy_transform`'s transform components.

It also allows multiple overlapping hierarchies to be made that may differ from
each other depending on the use case.

#### Drawbacks
This can be quite confusing to show in an editor and is reasonably complex to
understand from a user perspective. A compelling use case might be needed here.

### Hierarchical ECS Storage
This involves directly adding a third ECS storage option. In this solution, a
data-oriented SoA structure of split BlobVec columns is used, virtually identical
to the way Tables are currently stored. However, entities are stored in
depth-first order, and an implicit `Option<Entity>` parent reference is stored
alongisde every Entity. Hierarchy roots are stored at the beginning of the
slices, and leaves near the end. When a entity's components are mutated, it's
marked "dirty" by moving it to the end of the slice. A separate "first dirty"
index is kept to track a list of all dirty hierarchy entities as a form of change
detection. Finally, when iterating over the storage, if a child is found before
it's parent, it's swapped with it's parent to enforce the depth-first invariant.

#### Benefits
This enables linear iteration time over hierarchical components akin to that of a
typical ECS table, allowing depth first iteration without random accesses.

#### Drawbacks
This method is very write heavy. Components are moved whenever there is a
mutation, and shuffled around to ensure that the depth first iteration invariant
is maintained. This may be impractical for hierarchy dependent components that
are very big in size.

The need to mutate and shuffle the storage during iteration also means parallel
iteration cannot be supported on it, so going wide might not be possible. Even
read-only queries against this storage requires exclusive access to the table.

### ECS Relations
ECS relations can be used to represent the exact same links in the hierarchy, and
might be usable as a different backing for the hierarchy, provided the same
consistency guarentees.

#### Benefits
Relations are generic, expressing more than just hierarchical relations, but any
link between multiple entities. The exact benefits and drawbacks relative to this
design is reliant on the exact implementation.

## Prior art
The original design that this proposal is based on is this 2014 article from
[bitsquid
developers](https://bitsquid.blogspot.com/2014/10/building-data-oriented-entity-system.html).
The memory layout sugggested is awfully similar to bevy's ECS tables. However,
components are regularly swapped based on their dirty status. Placing dirty
entities near the end of each contiguous slice. The hierarchical ECS storage
alterantive aims to implement this.

A similar [blog
post](https://blog.molecular-matters.com/2013/02/22/adventures-in-data-oriented-design-part-2-hierarchical-data/)
from 2013 suggests a similar approach.

Unity offers similar APIs to the `HierarchyQuery`, with similar performance
penalties for heavy use.

 - [GameObject.GetComponentsInChildren](https://docs.unity3d.com/ScriptReference/GameObject.GetComponentsInChildren.html)
 - [GameObject.GetComponentsInParent](https://docs.unity3d.com/ScriptReference/GameObject.GetComponentsInParent.html)

## Unresolved questions

 - How do we resolve ordering problems for the same entity? This ties into the
   question of Command determinism.

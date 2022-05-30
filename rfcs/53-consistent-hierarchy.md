# Feature Name: `consistent-hierarchy`

## Summary
The current transform hierarchy operations currently allow for it to remain in a
inconsistent state. This RFC proposes a few changes to make it globally
consistent at any given time, as well as make it easier to support non-Tranform
use cases.

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

Many of these operations are both abundantly common and moderately expensive.
Queries and traversals can be linear in the number of children and involve heavy
random access patterns which can be detrimental to game performance.

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
Both of the current components' public interface will be made *read-only*.
Structural mutations to the hierarchy: de-parenting, re-parenting, and moving
children from one parent to another, etc. can only be done via commands. This
defers all changes to the hierarchy until the next command buffer flush.
This enforces a globally consistent view at any given time. Delaying any
modifications to a global synchronization point, much like component removal.
Both components will not publicly constructible and must be made via the
dedicated hierarchy management commands.

`PreviousParent` will also be removed and replaced with `ChildAdded`,
`ChildRemoved` and `ChildMoved` events instead. These will signal changes in the
hierachy instead of relying on change detection on `Parent` and `PreviousParent`.

## Implementation strategy
This design attempts to minimize the changes made to the current design while
addressing the issues enumerated above. The core changes to be made are as
follows:

 - Make `Parent` and `Children` (mostly) publicly immutable.
 - Make mutations to the hierarchy rely solely on commands.
 - Update hierarchy commands to automatically update both `Parent` and
   `Children` to remain consistent with each other.
 - Remove the `parent_update_system`.
 - Remove `PreviousParent`.
 - Register several hierarchy events that fires whenever time a new child is
   added or removed from a parent.
 - Create a `HiearchyQuery` custom SystemParam for more easily iterating over
   components in the hierarchy.

This change will repurpose the following existing `EntityCommand` extension
functions for this purpose:

 - `add_child`
 - `push_children`
 - `insert_children`
 - `remove_child` (this will be added as a wrapper around `remove_children`)
 - `remove_children`

An additional `set_parent(Option<Entity>)` command should also be added as a
child-side way of altering the hierarchy, defaulting to adding the entity as the
last child to the provided parent if provided, and detaching the child from it's
parent if `None`.

### HierarchyQuery
An extension to this design is to create dedicated `HierarchyQuery<Q, F>`, which
is a custom `SystemParam` that wraps multiple `Query` objects to simplify
querying for items in the hierarchy:

 - `roots(_mut)` - returns an iterator over entities that match the provided
   query parameters that do not have a parent.
 - `parent(_mut)` - returns single result from the parent of an entity.
 - `iter_children(_mut)` - returns an iterator over `(parent, child)` results
   from a query parameter. This does not return Children of Children.
 - `iter_descendants(_mut)` - returns an iterator over `(parent, child)` results
   from a query parameter. This does a depth-first traversal over the hierarchy
   starting at some entity.
 - `iter_ancestors(_mut)` - returns an iterator over `(parent, child)` results
   from a query parameter. This bubbles up queries from the bottom to the top
   starting at some entity.

As suggested by the generic parameters, the query results and filters are
identical to the ones used by a normal Query or QueryState.

The main motivation for this is to make creating systems like
`transform_propagate_system` easier. These queries can be generically used for
any system that needs some form of hierachy traversal. These queries can be quite
expensive to run, so the documentation for them should reflect this and
discourage misuing it.

### HierarchyEvent
One new event should be added here, with enum variants corresponding to a
specific change in the hierarchy:

 - `ChildAdded` - Fired when a previous root entity has been added as a child to
   a entity in the hierarchy.
 - `ChildRemoved` - Fired when an entity is removed from the hierarchy and is now
   a root entity.
 - `ChildMoved` - Fired when an child is moved from one parent in the hierarchy
   to another.

These can be used to detect changes where `PreviousParent` is currently used.
Each event is exclusive, meaning that any change only generates one event. A move
will not fire additional separate `ChildAdded` and `ChildRemoved` events.

Only one event type is used here to ensure that hierarchy alterations are
properly sequenced.

As hierarchy edits typically happen in bursts (i.e. scene loads). There may be
large number of these events generated at once. To reclaim this memory, it may be
necessary to add support for automatic garbage collection to events.

```rust
#[derive(Clone)]
pub enum HierarchyEvent {
  ChildAdded {
    parent: Entity,
    child: Entity,
  },
  ChildRemoved {
    parent: Entity,
    child: Entity,
  },
  ChildMoved {
    parent: Entity,
    previous_parent: Entity,
    new_parent: Entity,
  }
}
```

### `bevy_hierarchy`-`bevy_transform` split.
These hierarchy types should be moved to a new `bevy_hierarchy` crate to avoid
requiring compiling the hierachy alongside the default "f32-based affine
transform system in Euclidean space". This should allow users to create their own
transform systems (i.e. f64-based transforms, fixed-point transforms,
non-Euclidean spaces) while also being able to leverage the same
parent-sibling-child hierarchies currently found in `bevy_transform`. This should
also allow us to internally leverage the same decoupling too for specialized
transforms for 2D and UI.

## Benefits

 - This rocks the boat the least compared to the following alternatives and
   largely provides the same benefits.
 - This design has (nearly) guarenteed global consistency. The hierarchy cannot
   be arbitrarily mutated in inconsistent between synchronization points.
   Changes are deferred to a until the end of the current stage instead of a
   single global maintainence system.
 - The cache locality when iterating over the immediate children of an entity is
   retained. This saves an extra cache miss per entity when traversing the
   hierarchy.
 - The removal of `PreviousParent` should reduce the archetype fragmentation of
   the current solution.
 - Any update to the hierarchy is "atomic" and is `O(n)` in the number of
   children an entity has. For any practical hierarchy, this will typically be
   a very small n.
 - Existing query filters still work as is. For example, `Without<Parent>` can
   still be used to find root entities, and `Changed<Children>` can be used to
   find any entity with either new children, removed children, or reordered
   children.
 - The hierarchy is decoupled from the base transform system, allowing both Bevy
   developer and third-party crates to make their own independent transform
   system without compiling the default transform system.

## Drawbacks

 - The fact that the hierarchy is not optimally stored in memory is still an
   issue, and all hiearchy traversals require heavy random access into memory.
 - Updates are still not immediately visible within a stage and deferred until
   the next command buffer flush.
 - Hierarchy updates are now single threaded. Commands can be generated from
   multiple systems, [or from multiple threads in the same
   system](https://github.com/bevyengine/bevy/pull/4749) at the same time, but
   they now require exclusive access to apply.
 - Each of the changed commands do a lot more work before and are signifgantly
   more branch heavy. This could negatively impact the performance of
   command-heavy workloads (i.e.  scene loading)
 - The commands applied for controlling the hierarchy are computationally more
   expensive, requiring more branching access and more `World::get` calls.
 - Some potential implementations of this hierarchy requires normal ECS commands
   to be aware of the hierarchy. Using `EntityCommands::remove` to remove
   `Children` or `Parent` will break the invariants of the system without some
   hooks to enforce the invariants on removal. This is partially mitigatable via
   documentation until hooks or relations lands.

## Rationale and alternatives
The primary goal of this proposal is to get rid of the hierarchy maintenance
system and the consistency issues that come with it's design, which it delivers
on.

### Alternative: Hierarchy as a Resource
This would store the resource of the entire world as a dedicated Resource
instead of as components in normal ECS storage. A prototypical design for this
resource would be:

```rust
pub struct Hierarchy {
  relations: Vec<(Option<usize>, Vec<usize>)>,
  entities: Vec<Entity>,
  indices: SparseArray<Entity, usize>,
}
```

This would be strikingly similar to the `SparseSet` used in `bevy_ecs`. The main
difference here is that the parallel `Vec`s are roughly kept in topological order.
Inserting should be `O(1)` as it just updates the new parent. Removing should be
`O(n)` with the number of children of that `Entity`. At least once per frame, a
system should run a topological sort over the resource if it's dirty.

An alternative version of this hierarchy resource only stores parents. This may
speed up iteration as individual relations are smaller during swaps, but finding
children in the hierarchy is O(n) over the entire world. If the depth of each
entity within the hierarchy, and the hierarchy is kept sorted in depth order this
can be kept to O(n) in the next depth.

#### Benefits

 - Iteration in hierarchy order (parents first) is entirely linear. This may make
   transform and other parent-reliant propagation signfigantly faster.
 - Finding roots in the sorted hierarchy is both linear and compacted tightly in
   memory.
 - Removes `Parent` and `Children` ECS components entirely. Could further reduce
   archetype fragmentation. The hierarchy is stored densely in a single location
   and is not spread between multiple archetypes/tables.
 - Can still be used with the command-driven interface proposed in this design
   instead of using ECS storage.

#### Drawbacks

 - Merging and splitting hierarchies loaded from scenes can be quite difficult to
   maintain.
 - The non-hierarchy components are still stored in an unordered way in ECS data.
   Iterating over the hierarchy will still incur random access costs for anything
   non-trivial.
 - Requires a dedicated topological sort system. Mitigated by only sorting when
   the hierarchy is dirty, or simply tracking which entities in the hierarchy are
   dirty.
 - Editing the hierarchy requires exclusive access to the resource, which can
   will serialize all systems editing the hierarchy. This also prevents internal
   parallelism (i.e. via `Query::par_for_each`) as `ResMut` cannot be cloned.
   This can be partially mitigated by deferring edits via commands, as proposed
   in this RFC.

### Alternative: Linked List Hierarchy
Instead of using two separate components to manage the hierarchy, this
alternative aims to have only one: `Hierarchy`. This hierarchy component provides
*read-only* access into an entity's immediate hierarchial relationships: its
parent, previous sibling, next sibling, and it's first child. All of these may
return `Option<Entity>`, signalling that such relationships do not exist. This
creates a single component that creates a top-down forest of entity trees, where
each entity forms a doubly-linked list of siblings. It may also cache infromation
about the component's position within the hierarchy like it's depth.

```rust
#[derive(Component)]
pub struct Hierarchy {
  parent: Option<Entity>,
  prev: Option<Entity>,
  next: Option<Entity>
  first_child: Option<Entity>,
}
```

Like the main proposal here, there are no public constructors or mutation APIs,
relying only on commands to mutate the internals.

#### Benefits

 - Hierarchy updates are all `O(1)`, and can be maintained within a system. Does
   not require commands to ensure consistency.
 - The Hierarchy component is smaller in ECS storage than the current
   Parent/Children combination. With Entity-niching, a `Hierarchy` component is
   only 32 bytes in memory, compared to the 64 + 8 used by the current two
   components.
 - Requires less archetype fragmentation due to only requiring one component
   instaed of two.
 - Getting siblings does not require a parent Query::get call.

#### Drawbacks

 - Iterating over children requires `O(n)` Query::get calls.
 - Despawning a child without updating it's sibilings causes all subsequent
   children to be in a broken state. This exacerbates the lack of hooks, and
   would require query access in hooks to fix up.
 - Change detection and With/Without filters no longer work as is. Will need
   dedicated ZST marker components to get the same filtering capacity.

### Alternative: HashSet `Children`
The internals of `Children` could be replaced with a `HashSet<T>`.
operations close to `O(1)`, versus `O(n)` operations against a `SmallVec`.

#### Benefits
This makes all operations against `Children` O(1). Could speed up a number of
hierarchy related commands.

#### Drawbacks

 - We lose any ability to order children. This is critically important for
   certain use cases like UI.
 - We lose cache locality for entities with few children. Adding a single child
   forces an allocation.

### Alternative: Hierarchical ECS Storage
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

This is very similar to the "Hierarchy as a Resource" alternative, but

#### Benefits

 - Iteration in hierarchy order (parents first) is entirely linear. This may make
   transform and other parent-reliant propagation signfigantly faster.
 - Unlike the hierarchy as a resource, this guarentees linearity of all iterated
   components, not just the hierarchy relations itself.

#### Drawbacks
This method is very write heavy. Components are moved whenever there is a
mutation, and shuffled around to ensure that the topologically ordered iteration
invariant is maintained. This may be impractical for hierarchy dependent
components that are very big in size.

The need to mutate and shuffle the storage during iteration also means parallel
iteration cannot be supported on it, so going wide might not be possible. Even
read-only queries against this storage requires exclusive access to the table.

### Alternative: ECS Relations
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

## Future Work
One of the immediate pieces of future work here is to patch up the potential
footgun of using `EntityCommands::remove` on either `Parent` or `Children` via
add/remove/changed hooks. This is currently mitigatable via documentation, but a
complete solution prevents users from breaking the invariant in the first place.

### Dedicated RectTransform and Transform2D systems
Transform/GlobalTransform is not particularly well suited for encoding the
hierarchy of UIs and 2D-only scenes. By splitting out the hierarchy from the
exsisting transform system, dedicated transform types and systems for these two
different types of hierarchies. This could also enable further parallelism by
allowing each transform system to independently handle propagation.

### Multiple Hierarchies via Type Markers
It may be possible to include type markers as a part of the hierarchy components
to differentiate multiple hierachies from each other. This would allow an entity
to be part of multiple hierarchies at the same time.

```rust
#[derive(Component)]
pub struct Parent<T> {
  parent: Entity,
  _marker: PhantomData<T>,
}

#[derive(Component)]
pub struct Children<T> {
  children: SmallVec<[Entity, 8]>,
  _marker: PhantomData<T>,
}
```

This allows us to directly tie hierarchies to the components they're targetting.
Instead of a general `Parent`, a `Parent<Transform>` declares it's meant
for `bevy_transform`'s transform components.

It also allows multiple overlapping hierarchies to be made that may differ from
each other depending on the use case.

However, this may be quite confusing to show in an editor and is reasonably
complex to understand from a user perspective. A compelling use case might be
needed to justify this kind of change.

## Unresolved questions

 - How do we resolve ordering problems for the same entity? This ties into the
   question of Command determinism.

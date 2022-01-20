# Feature Name: (fill me in with a unique ident, `my_awesome_feature`)

## Summary

One paragraph explanation of the feature.

## Motivation

Why are we doing this? What use cases does it support?

## User-facing explanation

**Atomic groups** are collections of systems (and run criteria) that must be executed "together": once the group has been started, no systems can mutate data that they rely on.
Like the atom, they are indivisible: nothing can get between them!
Run criteria are always part of the same atomic group as the systems they control.

**Atomic ordering constraints** are a form of ordering constraints that ensure that two systems are run directly after each other.
At least, as far as "directly after" is a meaningful concept in a parallel, nonlinear ordering.

In addition to their use in run criteria, atomic groups are a useful (if quite advanced) feature for forcing tight interlocking behavior between systems, preserving plugin privacy and avoiding leaking sensitive timing information.
They allow us to ensure that the state read from and written to the earlier system cannot be invalidated.
This functionality is extremely useful when updating indexes (used to look up components by value):
allowing you to ensure that the index is still valid when the system using the index uses it.

The **atomicity** of each label can be controlled using `.configure_label(MySystemLabel::Variant.set_atomicity(AtomicityLevel::Coherent))`:

```rust
enum AtomicityLevel {
   // Read and write-locks are released when a system completes
   None,
   // Read and write-locks are held until they are no longer needed by a member of this atomic group
   Coherent,
   // Read and write-locks are only released when this atomic group is complete
   Isolated,
}
```

Labels are non-atomic by default, and atomic ordering constraints create coherent atomic groups.
Isolated atomic groups are relatively niche, used only when you want to be certain that no information about internal progress can leak out.

Calling `a.atomically_before(b)` will create a strict ordering constraint between `a` and `b`, and add them to the same atomic group (with a uniquely-generated system label).

Of course, atomic groups (and thus atomic ordering constraints) should be used thoughtfully.
As the strictest of the three types of system ordering dependency, they can easily result in unsatisfiable schedules if applied to large groups of systems at once.
For example, consider the simple case, where we have an atomic ordering constraint from system A to system B.
If for any reason commands must be flushed between systems A and system B, we cannot satisfy the schedule.
The reason for this is quite simple: the lock on data from system A will not be released when it completes, but we cannot run exclusive systems unless all locks on the `World's` data are clear.
As we add more of these atomic constraints (particularly if they share a single ancestor), the risk of these problems grows.
In effect, atomically-connected systems look "almost like" a single system from the scheduler's perspective.

On the other hand, atomic ordering constraints can be helpful when attempting to split large complex systems into multiple parts.
By guaranteeing that the initial state of your complex system cannot be altered before the later parts of the system are complete,
you can safely parallelize the rest of the work into separate systems, improving both performance and maintainability.

## Implementation strategy

For all systems that are part of the same atomic system group:

- When a system in an atomic group completes, its write-locks and read-locks are downgraded to **virtual locks**.
- Virtual locks are treated as if they were real locks for all systems outside of the atomic group.
- Virtual locks are ignored by systems in the same atomic group.
- If the atomic group is **coherent**, virtual locks are removed once all systems in the atomic group have finished with it.
- If the atomic group is **isolated**, virtual locks are held until all systems in the atomic group complete.

This can be modelled as:

```rust
enum DataLock{
   Read,
   Write,
   VirtualRead(Box<dyn SystemLabel>),
   VirtualWrite(Box<dyn SystemLabel>),
}
```

Multiple virtual locks on the same data can exist at once, and membership in an atomic group is *not* transitive.
Suppose we have two atomic groups, `GroupX = {A, B}` and `GroupY = {B, C}`.

Systems `A` and `C` do not share an atomic group, and data that is virtually locked by both `GroupX` and `GroupY` can only be accessed by system `B`.

This reduces the risk of an unsatisfiable schedule by allowing locks to be released in a maximally permissive fashion.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- Should we allow overlapping atomic groups that don't have a subset-superset relationship?
  - An atomic group is supposed to make multiple systems look like one system to outside systems.
  - What do overlapping subsets mean conceptually?

## \[Optional\] Future possibilities

- Indexes!
- Relations!
- Automatic flush inference!

# Feature Name: `atomic-groups`

## Summary

Atomic groups ensure that systems are evaluated "together", ensuring that nothing can break their internal state during execution.
While this would be feasible by running systems in strict sequence, atomicity ensures safe "as-if" parallel execution of other systems while a group is being evaluated.

## Motivation

Atomic groups are a basic building block of scheduling, and are essential to support more complex user-facing tools that rely on setting up and then using state.
Critically, indexes (used to efficiently store and then look up components by value) require some form of atomicity to ensure that users are not reading from bad state.

In addition, atomicity can be a powerful tool for plugin authors, allowing them to impose strong guarantees of privacy without disabling configuration.

While system piping creates a simple form of atomicity, atomic groups are substantially more expressive. They:

- allows for systems with conflicting data accesses to co-exist
- allows for nonlinear atomicity

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

- This is a complex feature!
- It could slow down evaluation of the schedule, including in cases where this feature is not used.

## Rationale and alternatives

### Why do we want to distinguish between coherent and isolated groups?

In most cases, coherent groups will be desired.
They ensure correctness while releasing locks sooner.
However, isolated groups have a few advantages:
  
- They are conceptually simpler: isolated atomic groups are run "as-if" they are a single system.
- They preserve privacy more strictly and do not leak timing information about their internal components.
- They work correctly in the face of interior mutability.
- Isolated atomic groups can efficiently be run as part of a single task, without communication upstream.
  - They release their locks all at once, which can have lower overhead for simple systems.

## Unresolved questions

- Should we allow overlapping atomic groups that don't have a subset-superset relationship?
  - An atomic group is supposed to make multiple systems look like one system to outside systems.
  - What do overlapping subsets mean conceptually?

## Future possibilities

- Built-in index primitives
  - Reduces boilerplate
  - Allows the community to reuse optimized, featureful code
- Automatic inference of index rebuilds
  - Increases usability by deduplicating state refreshes without risking stale state

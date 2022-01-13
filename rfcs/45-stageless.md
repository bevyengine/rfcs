# Feature Name: `stageless`

## Summary

The existing stage boundaries result in a large number of complications: preventing us from operating across stages.
Moreover, several related features (run criteria, states and fixed timesteps) are rather fragile and overly complex.
This tangled mess needs to be holistically refactored: simplifying the scheduling model down into a single `Schedule`, moving towards an intent-based configuration style, taking full advantage of labels and leveraging exclusive systems to handle complex logic.

## Motivation

The fundamental challenges with the current stage-driven scheduling abstraction are [numerous](https://github.com/bevyengine/bevy/discussions/2801).
Of particular note:

- Dependencies do not work across stages.
- States do not work across stages.
- Plugins can add new, standalone stages.
- Run criteria (including states!) cannot be composed.
- The stack-driven state model is overly elaborate and does not enable enough use cases to warrant its complexity.
- Fixed timestep run criteria do not behave in a fashion that supports.
- The architecture for turn-based games and other even slightly unusual patterns is not obvious.
- We cannot easily implement a builder-style strategy for configuring systems.

Unfortunately, all of these problems are deeply interwoven. Despite our best efforts, fixing them incrementally at either the design or implementation stage is impossible, as it results in myopic architecture choices and terrible technical debt.

## User-facing explanation

This explanation is, in effect, user-facing documentation for the new design.
In addition to a few new concepts, it throws out much of the current system scheduling design that you may be familiar with.
The following elements are radically reworked:

- schedules (flattened)
- run criteria (can no longer loop, are now systems)
- states (simplified, no longer run-criteria powered)
- fixed time steps (no longer a run criteria)
- exclusive systems (no longer special-cased)
- stages (yeet)

### Scheduling overview and intent-based configuration

Systems in Bevy are stored in a `Schedule`: a collection of **configured systems**.
Each frame, the `App`'s `runner` function will run the schedule,
causing the **scheduler** to run the systems in parallel using a strategy that respects all of the configured constraints (or panic, if it is unsatisfiable).

In the beginning, each `Schedule` is entirely unordered: systems will be selected in an arbitrary order and run if and only if all of the data that it must access is free.
Just like with standard borrow checking, multiple systems can read from the same data at once, but writing to the data requires an exclusive lock.

While this is a good, simple strategy for maximizing parallelism, it comes with some serious drawbacks for reproducibility and correctness.
The relative execution order of systems tends to matter!
In simple projects and toy examples, the solution to this looks simple: let the user specify a single global ordering and use this to determine the order in the case of conflicts!

Unfortunately, this strategy falls apart at scale, particularly when attempting to integrate third-party dependencies (in the form of plugins), who have complex, interacting requirements that must be respected.
While users may be able to find a single working strategy through trial-and-error (and a robust test suite!), this strategy becomes ossified: impossible to safely change to incorporate new features or optimize performance.

The fundamental problem is a mismatch between *how* the schedule is configured and *why* it is configured that way.
Information about "physics must run after input but before rendering" is implicitly encoded into the order of our systems, and changes which violate these constraints will silently fail in far-reaching ways without giving any indication as to what the user did wrong.

TODO: intent-based configuration.

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

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

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.

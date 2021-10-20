# Feature Name: `system-graph-flavors`

## Summary

Several new, higher-level system ordering constraints are introduced to allow users to describe more complex rules that the scheduler must obey.
These constraints limit the set of possible paths through the schedule and serve as initialization-time assertions on the schedule that protect against accidental misconfiguration.

## Motivation

While the existing strict ordering constraints implemented as `.before` and `.after` are an excellent foundation,
more complex constraints are needed to elegantly express and enforce rules about how systems relate to each other.

In order to manage the complexity of code bases with hundreds of systems, these constraints need to be both minimal and local.
Each constraint should have a good, clear reason to exist.

This is particularly critical in the context of [plugin configurability](https://github.com/bevyengine/rfcs/pull/33), where plugin authors need a powerful toolset of constraints in order to ensure that their logic is not broken when reconfigured by the end user.

## User-facing explanation

### Scheduling 101

Bevy schedules are composed of stages, of which there are two varieties: parallel and sequential.
Typically, these will alternate, as adjacent stages of the same type will be collapsed.
Parallel stages will perform easily isolated work, and generate commands.
Then a sequential stage will run, beginning with the standard command-processing exclusive system.
Any adjacent parallel stages are collapsed into one parallel stage to maximize parallelism, and any adjacent sequential stages are concatenated.

During **sequential stages**, each system can access the entire world freely but only one can run at a time.
Exclusive systems can only be added to the sequential phase while parallel systems can be added to either phase.

During **parallel stages**, systems are allowed to operate in parallel, carefully dividing up access to the `World` according to the data accesses requested by their system parameters to avoid undefined behavior.

Without any user-provided ordering constraints, systems within the same parallel phase can be executed in any order so long as Rust's ownership semantics are obeyed.
That means that a **waiting** (scheduled to run during this stage) system cannot be started if any **active** (currently running) systems are **incompatible** (cannot be scheduled at the same time) if they have conflicting data access.
This is helpful for performance reasons, but, as the precise order in which systems start and complete is nondeterministic, can result in logic bugs or inconsistencies due to **system order ambiguities**.

### System ordering constraints

To help you eliminate ambiguities, eliminate delays and enforce **logical invariants** about how your systems work together, Bevy provides a rich toolbox of **system ordering constraints**.

There are several **flavors** of system ordering constraints, each with their own high-level behavior:

- **Strict ordering:** While any systems from set `A` are waiting or active, systems from set `B` cannot be started.
  - Simple and explicit.
  - Use the `before(label: impl SystemLabel)` or `after(label: impl SystemLabel)` methods to set this behavior.
  - If `A` is empty (or if no systems from `A` are enabled), this rule has no effect.
- **If-needed ordering:** While any systems from set `A` are waiting or active, systems from set `B` cannot be started if they are incompatible with any remaining systems in `A`.
  - Usually, if-needed ordering is the correct tool for ordering groups of systems as it avoids unnecessary blocking.
  - If systems in `A` use interior mutability, if-needed ordering may result in non-deterministic outcomes.
  - Use `before_if_needed(label: impl SystemLabel)` or `after_if_needed(label: impl SystemLabel)`.
- **At-least-once separation:** If a type of side effect has been "produced" by any system, at least one "consuming" system must run before any system that "uses" that side effect can run.
  - `Commands` are the most important side effect here, but others (such as indexing) may be added to the engine itself later.
  - This design expands on the API in [RFC 36: Encoding Side Effect Lifecycles as Subgraphs](https://github.com/bevyengine/rfcs/pull/36).
  - At-least-once separation also applies a special "causal tie" (see below): if a pair of `before` and `after` systems are enabled during a schedule pass, and all intervening `between` systems are disabled, the last intervening `between` system is forcibly enabled to ensure that cleanup occurs

System ordering constraints operate across stages, but can only see systems within the same schedule.

A schedule will panic on initialization if it is **unsatisfiable**, that is, if its system ordering constraints cannot all be met at once.
This may occur if:

1. A cyclic dependency exists. For example, `A` must be before `B` and `B` must be before `A` (or some more complex transitive issue exists).
2. Two systems are at-least-once separated and no separating system exists that could be scheduled to satisfy the constraint.

System ordering constraints cannot change which stage systems run in or add systems: you must fix these problems manually in such a way that the schedule becomes satisfiable.

### Causal ties: systems that must run together

Sometimes, the working of a system is tied in an essential way to the operation of another.
Suppose you've created a simple set of physics systems: updating velocity based on acceleration without then applying the velocity to update the position will result in subtle and frustrating bugs.
Rather than relying on fastidious documentation and careful end-users, we can enforce these constraints through the schedule.

These **causal ties** have fairly simple rules:

- causal ties are directional: the **upstream** system(s) determine whether the **downstream** systems must run, but the reverse is not true.
- if an upstream system runs during a pass through the schedule, the downstream system must also run during the same pass through the schedule
  - this overrides any existing run criteria, although any ordering constraints are obeyed as usual
  - causal ties propagate downstream: you could end up enabling a complex chain of systems because a single upstream system needed to run during this schedule pass
- if no suitable downstream system exists in the schedule, the schedule will panic upon initialization
- causal ties are independent of system execution order: upstream systems may be before, after or ambiguous with their downstream systems

Let's take a look at how we would declare these rules for our physics example using the `if_runs_then` system configuration method on our upstream system:

```rust
fn main(){
  App::new()
    .add_system(update_velocity
      .label("update_velocity")
      // If this system is enabled,
      // all systems with the "apply_velocity" label must run this loop.
      // If no systems with that label exist,
      // the schedule will panic.
      .if_runs_then("apply_velocity")
    )
    .add_system(
      apply_velocity
      .label("apply_velocity")
    )
    .run();
}
```

If we only have control over our downstream system, we can use the `run_if` system configuration methods:

```rust
fn main(){
  App::new()
    .add_system(update_velocity
      .label("update_velocity")
    )
    .add_system(
      apply_velocity
      .label("apply_velocity")
      // If any system with the given label is enabled,
      // this system must run this loop.
      .run_if("update_velocity")
    )
    .run();
}
```

`only_run_if` has the same semantics, except that it also adds a run criteria that always returns `ShouldRun::No`, allowing the system to be skipped except where overridden by a causal tie (or other run criteria).

If we want a strict bidirectional dependency, we can use `always_run_together(label: impl SystemLabel)` as a short-hand for a pair of `.if_run_then` and `run_if` constraints.
Bidirectional links of this sort result in an OR-style logic: if either system in the pair is scheduled to run in this schedule pass, both systems must run.

## Implementation strategy

Simple changes:

1. Commands should be processed in a standard exclusive system with the `CoreSystem::ApplyCommands` label.

### Strict ordering

This algorithm is a restatement of the existing approach, as added in [Bevy #1144](https://github.com/bevyengine/bevy/pull/1144).
It is provided here to establish a foundation on which we can build other ordering constraints.

TODO: describe algorithm.

### If-needed ordering

TODO: describe the algorithm.

### At-least-once separation

TODO: describe the algorithm.

### Schedules own systems

Currently, each `Stage` owns the systems in it.
This creates painful divisions that make it challenging or impossible to do things like check dependencies between systems in different stages.

Instead, the `Schedule` should own the systems in it, and constraint-satisfaction logic should be handled at the schedule-level.

### Cross-stage system ordering constraints

When two systems are in separate stages, the exact meaning of ordering constraints gets a bit trickier.

When system `A` is before system `B`:

- if system `A` is in an earlier stage, nothing happens: this is trivially satisfied
- if system `A` is in the same stage, system `B` is held in queue until system `A` completes
- if system `A` is in a later stage, the schedule panics: this is trivially impossible

This check (and panic) should be skipped for if-needed ordering constraints, as they should be minimally impairing since they are only intended to resolve ambiguities, rather than introduce logically-necessary constraints.

### Causal ties

After all `SystemDescriptors` are added to the schedule, any causal ties described within are converted to a single `CausalTies` graph data structure.

This stores directed edges: from one upstream system to the corresponding downstream system.
If a label is applied to more than one system, it will create an edge between each appropriate system.
If the label for the upstream system is empty, it should be silently skipped, but if the label for the downstream system is empty the schedule should panic.
This is because our high-level functionality will continue to behave correctly if a system that creates work to be handled is absent, but will break if a system that cleans up work is missing.

Whenever run criteria are checked, the set of all systems which have `ShouldRun::Yes` during this schedule pass are compared to the set of upstream systems (those which have at least one outgoing edge) in the `CausalTies` struct.
All systems that are immediately downstream of those systems have their `ShouldRun` state set to `Yes` as well.
If the downstream system is not waiting to be scheduled in the *current or future* stages, the executor should panic (likely due to the system being added to an earlier stage).
This process is repeated until no new changes occur.

If this is implemented under the current `RunCriteria` design that includes looping information, that looping information should be propagated to downstream dependencies as well.
In particular, if an upstream system is `YesAndLoop`, the downstream system becomes `YesAndLoop`.
If the upstream system is `NoAndLoop`, the downstream system becomes either `NoAndLoop` or `YesAndLoop`, based on whether or not it was already set to run.
This branching is much more complex to reason about and likely to be substantially slower, and is another reason to simplify the `RunCriteria` enum.

## Drawbacks

1. This adds significant opt-in end-user complexity (especially for plugin authors) over the simple and explicit `.before` and `.after` that already exist or use of stages (or direct linear ordering).
2. Additional constraints will increase the complexity (and performance cost) of the scheduler.

## Rationale and alternatives

### Why do we need to include causal ties as part of this RFC?

Without causal-tie functionality, at-least-once separation can silently break when the separating system is disabled.

### Why don't we support more complex causal tie operations?

Compelling use cases (other than the one created by at-least-once separation) have not been demonstrated.
If and when those exist, we can add the functionality with APIs that map well to user intent.

### Why not cache the new set of systems whose run criteria changed?

Vector allocations are slow, but bit set comparisons and branchless mass assignments are fast.
It's faster to repeat work here than verify if it needs to be done.

At extremely high causal-tie graph depths and numbers of systems, the asymptotic performance gains may be worth it, but that seems unlikely in realistic scenarios.

## Prior art

The existing scheduler and basic strict ordering constraints are discussed at length in [Bevy #1144](https://github.com/bevyengine/bevy/pull/1144).

Additional helpful technical foundations and analysis can be found in the [linked blog post](https://ratysz.github.io/article/scheduling-1/) by @Ratysz.

## Unresolved questions

1. How *precisely* should schedules store the systems in them?

## Future possibilities

These additional constraints lay the groundwork for:

1. More elegant, efficient versions of higher-level APIs such as a [system graph specification].
2. Techniques for automatic schedule construction, where the scheduler is permitted to infer hard sync points and insert cleanup systems as needed to meet the constraints posed.
3. [Robust schedule-configurable plugins](https://github.com/bevyengine/rfcs/pull/33).

In the future, pre-computed schedules that are loaded from disk could help mitigate some of the startup performance costs of this analysis.
Such an approach would only be worthwhile for production apps.

The maintainability of system ordering constraints can be enhanced in the future with the addition of tools that serve to verify that they are still useful.
For example, if-needed ordering constraints that can never produce any effect on system ordering could be flagged at schedule initialization time and reported as a warning.

Additional graph-style APIs for specifying causal ties may be helpful, as would the addition of a `.always_run_together` method that can be added to labels to imply mutual causal ties between all systems that share that label.

# Feature Name: `global-task-pools`

## Summary
Globally initialize the newtyped TaskPools (ComputeTaskPool,
AsyncComputeTaskPool, IoTaskPool) as statics and provide static accessors to
them.

## Motivation
Bevy's TaskPools are currently passed around as ref-counted shared references,
and accessed primarily as ECS resources. This presents an ergonomics challenge
when integrating third-party crates without direct access to the ECS systems or
World. An example of this [Backroll's use of the TaskPools][backroll]
where the references must be passed and clone repeatedly.

This also presents an ergonomics issue for existing Bevy APIs, like
`par_for_each(_mut)`. All internally parallel systems must remember to add
a `Res<ComputeTaskPool>` parameter. This also presents a (rather small) footgun
for new users as to which task pool resource they should use, and choosing the
wrong one will give the wrong performance characteristics due to improper thread
allocation, or higher thread contention from the threads already in the wrong
pool.

Multitenant projects (processes with multiple Bevy apps running concurrently)
currently spin up new task pools for every top-level App in the process. The
number of threads per CPU core scales with the number of Apps in the process,
which may cause OS level thrashing from the context switches. This is most
commonly seen in server environments where processes may spin up and spin down
Apps in response to players starting and ending multiplayer matches.

## User-facing explanation
Instead of being publicly constructable, the newtyped TaskPools will become
static singletons, initialized on their first access.

Users can get a static reference to a task pool via the `get` associated
function (i.e. `ComputeTaskPool::get`), which will initialize the TaskPool with
the default configuration if it hasn't already been initialized.

For more explicit control over initialization, a corresponding `init` associated
function will take a task pool configuration and initialize the task pool, or
panic if the pool is already initialized. By convention, libraries, including
Bevy itself, should just use `get` and defer configuration to the final end
user's configuration.

## Implementation strategy
Prototype implementation: https://github.com/bevyengine/bevy/pull/2250

This implementation is rather simple and can be broken down into the following
smaller tasks:

 - The public construction of the newtyped TaskPools should be disabled/removed.
 - New static variables holding [`OnceCell`][once-cell] wrappers around each
   task pool will be added.
 - Move the default task pool initialization configuration from `bevy_core` into
   `bevy_tasks`.
 - Implement `get` and `init` functions as described above for each type.
 - Replace `Res<*TaskPool>` accesses with `*TaskPool::get` calls.
 - Optional: Remove the internal `Arc`s as the task pools to remove one level of
   indirection.
 - Optional: Remove the `TaskPool` parameters from `par_for_each(_mut)` on Query

## Drawbacks
Reconfiguring the task pools is now impossible under this initial design.
Calling `init` a second time will panic. This could be remedied by
moving reconfiguration into the TaskPool itself. The current way of
reconfiguring the task pools is to construct a new task pool and replace the
resource. However, this does not properly clean up the TaskPools if they
internally are self-referential: a long-lived, or indefinitely looping scheduled
task that contains a reference to the same task pool will keep the TaskPool
alive until all tasks terminate. Moving this reconfiguration as a first class
operation on TaskPool may be required to support these use cases.

Multitenant processes now share the same threads for all apps. Highly active or
long running apps may starve out others. Whether or not this is preferable over
OS-level thread thrashing is likely use case dependent.

## Rationale and alternatives
This is a significant improvement in terms of ergonomics over the status quo.
This API design removes the need to have a task pool parameter on systems,
`Query`, and `QueryState` types. It also removes integration friction for third
party crates that aren't directly integrating with the ECS state.

### Alternative: One Global Task Pool, Enum Tasks
This alternative proposes to remove the newtyped task pools entirely and just
have one global task pool instead. To get the same scheduling guarantee, tasks
would internally state which of the three task types it is. This would allow the
executors to independently schedule and prioritize tasks as they currently are,
but without forcing the user to make that decision, or at least allow users to
choose a common default.

This is still technically possible, but would require much more effort to
implement.

## Prior art
Other async executors also provide global versions of their executors:

 - [tokio][tokio]'s runtime is global by default
 - [async-global-executor][async-global-executor]

Other language's async runtimes also largely use globals as well:

 - Python's [asyncio event loop][asyncio-event-loop] is thread-local. Fetched
   via `asyncio.get_event_loop()`.
 - C#'s [managed-thread-pool][csharp-pool] is used for it's async runtime
 - Go spins up a 1:1 threadpool that all of it's async goroutines run on,
   scheduling is global and requires no reference.

Other game engines have a built in global threadpool used for their form of
multithreading:

 - Unity has dedicated threads for rendering, audio, and a dedicated worker
   thread pool for running multihreaded jobs.
 - Unreal: ???
 - Godot: ???

[backroll]: https://github.com/HouraiTeahouse/backroll-rs/blob/219f4b6fda27250a7c4f7928a381418faebd5544/backroll/src/backend/p2p.rs
[once-cell]: https://docs.rs/once_cell/latest/once_cell/sync/struct.OnceCell.html
[asyncio-event-loop]: https://docs.python.org/3/library/asyncio-eventloop.html
[csharp-pool]: https://docs.microsoft.com/en-us/dotnet/standard/threading/the-managed-thread-pool

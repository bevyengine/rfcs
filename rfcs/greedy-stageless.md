# Feature Name: Greedy Run Systems Loop for Stageless

## Summary

This is a possible design for a stageless schedule. It proposes adding an api for scheduling command queue application, breaking up RunCriteria into RunCriteria and LoopingCriteria, and presents an algorithm for how to schedule systems that require a hard sync point. Systems are grouped to run together depending on if they are parallel or exclusive systems.

## Motivation

**Parallel systems** in bevy interact with the ecs through `SystemParams`, which includes `Query`, `Res`, and `Command`. `Query` and `Res` are allowed to read and write data owned by the ecs, but actions like "spawn/despawn an entity", or "add/remove a component" require exclusive world access. Systems that require exclusive world access cannot be run in parallel with any other system. `Commands` are a way for parallel systems to queue up interactions with the world. When a parallel system calls a command like `command.spawn()` the entity is not actually spawned, but is queued to spawn at a special part of the schedule where systems are allowed exclusive world access. This special point is termed a **hard sync point**. Besides applying command queues to the world, systems that need direct world access are also allowed to run at this point. Bevy calls these **exclusive systems**.

The current model for introducing hard sync points are `Stages`. A Stage is a group of systems where systems are arranged into a graph. A stage runs the systems in the graph and then executes the systems that require a hard sync. So a hard sync occurs when a stage loops or when transitioning to a new stage. These stages are then run sequentially by the game loop.

Something a user often wants to do is put a hard sync point after a system that spawns entities to act on the entity data in the same tick. In the current model this is done by moving one of the systems to another stage to make a hard sync point between them. When the user does this they need to reason:

> I need to add a sync point between these two systems. So which stage should I move my system to and which system should I move?
> Maybe POSTUPDATE? Oh that doesn't work because it needs to be before a hard sync with some system in POSTUPDATE.
> Maybe move the spawning system to PREUPDATE? no that doesn't work either.
> So I'll add a stage, but not it only has one system in it.
> Should I move some other systems into the new stage? will moving systems to another stage mess something up?

Stages tend to be a very heavy abstraction, which require reasoning about the global behavior of your game and grouping systems in not necessarily the most logical way, but in a way that tries to maximize parallelism. This is because systems that are not in the same stage cannot be run in parallel. Trying to understand the possible parallelism of the program is hard for the user to reason about. Instead the user should only be thinking about the data flow through the program. "I need to spawn this entity and then act on the data, so I need to order these systems one after the other."" The scheduler should be in charge of all potential parallelism.

To solve this we should remove stages and create an api that allows scheduling of all systems (parallel, exclusive, and command queue application) in the same graph. The current API allows creating a schedule graph inside a stage with `label`, `before`, and `after`. For the user API we need to extend these API's to apply to exclusive systems and add an API for scheduling command queue application.

## User-facing explanation

### New Scheduling API

- `.label(Label)`
- **New** `.apply_label(Label)`
  - `.before(Label)`
  - `.after(Label)`
  - **New** `.apply_after(Label)`
  - **New** `.apply_before(Label)`
- `.with_run_criteria(RunCriteriaSystem)`
- **New** `.with_looping_criteria(LoopingCriteriaSystem)`
- `.startup()`
- _Removed_ ~~`.stage(Stage)`~~
- `.exclusive()`
  - _Removed_ ~~`.at_start()`~~
  - _Removed_ ~~`.at end()`~~

### Scheduling when to apply commands

Systems with commands have two parts, a parallel part that can be run in parallel with other systems that the user codes and a command queue that needs to be applied to the world during a hard sync point. Since these happen at different times they should be scheduled separately. This is done through three new system descriptor builder functions.

- `.commands_label(Label)` gives the point where the command queue is applied a `Label` which other systems can be scheduled around.
- `.commands_after(Label)` is used to schedule the command queue to be applied after `Label`
- `.commands_before(Label)` is used to schedule the command queue to be applied before `Label`

```rs
fn main() {
    App::new()
        // add a label to systemA's commands
        .add_system(systemA.label("System A").commands_label("System A Commands"))

        // places systemB after systemA's commands are applied
        .add_system(systemB.after("System A Commands"));

        // schedule systemC between systemA's commands are applied
        .add_system(systemC.after("System A").before("System A Commands"));

        // schedule systemD's commands after system A's commands
        .add_system(systemD.commands_after("System A Commands"))

        // schedule systemE's commands before system A's commands
        .add_system(systemE.commands_before("System A commands"));
}
```

### Exclusive Systems

Exclusive systems are systems that depend on direct world access. They are marked with the `.exclusive()` builder function and only have the `World` parameter. The `label`, `before`, and `after` apis can be used to order exclusive systems and can be mixed with labels for parallel systems.

### Run Criteria

Run criteria determine whether or not a system will run. They now are separated from LoopingCriteria. A RunCriteria system is a system that returns a bool with true for run the dependent system and false for not running the system. Systems without run criteria will be run once per tick.

```rs
fn should_run_once(hasRun: Local<bool>) -> bool {
    if *hasRun {
        false
    } else {
        *hasRun = true
        true
    }
}

fn spawn_player(mut commands: Commands) {
    commands.spawn().insert(Player);
}

fn main() {
    App::new()
        .add_system(
            spawn_player.with_run_criteria(should_run_once)
        );
}
```

### Looping Criteria

A looping criteria is a system that returns a bool. Returning true makes a system reevaluate its run criteria and false means the system is done running for this tick. Adding looping criteria introduces an implicit requirement for a hard sync. The hard sync is required to reevaluate looping criteria and run criteria. Systems scheduled after a looping system will run after the looping criteria returns false.

```rs
fn loop_twice(loop_count: Local<int32>) {
    if loop_count < 2 {
        loop_count += 1;
        true
    } else {
        false
    }
}

fn count_up_system(count: Local<int32>) {
    count += 1;
    println!("count: ${}", count);
}

fn main() {
    App::new()
        .add_system(
            count_up_system.with_looping_criteria(loop_twice)
        ).run_once();
}

// outputs:
// 1
// 2
```

### Removed APIs

- `.stage()` removed in favor just scheduling everything in the same graph
- `.exclusive().at_start()` and `.exclusive().at_end()` no longer make sense since they were for running at stage boundaries. The functionality can now be emulated with smart application of `.before()` and `.after()` since exclusive systems and parallel systems can be scheduled in relation to each other.

## Implementation strategy

### 2-step Run Systems Loop

The purpose of the run sytems loop is to run all systems that should run during a tick. We can do this with a two step loop.

1. The first step is a hard sync point. We run all systems that need **exclusive world access** that are allowed to be run by the system graph. These systems include exclusive systems and applying command queues to the world. Some of these systems will not be run because of **blocking dependencies** on parallel systems.
1. In the second step we run all **parallel systems** that the graph allows. Some of these systems will be blocked from running because of **blocking dependencies** on systems requiring exclusive world access. Blocking dependencies for parallel systems are dependcies on exclusive systems, command queue application labels, and unfinished looping systems.

By repeating these two steps, we will be be able to run all required systems.

### Counting Dependencies

A key part of the current run parallel systems loop is the counting and clearing of dependencies. When the dependency count for a system goes to zero it is run. This needs to be changed so that we don't queue systems that still have blocking dependencies. So we need to keep a separate count of non-blocked dependencies and blocking dependencies. When the blocking dependencies count reach zero we can queue the system to be run in the correct step. When non-blocking dependencies reach zero the system can be run in the step it is queued in.

#### Some Scenarios

- If _parallel system A_ depends on _parallel system B_ it can be queued to run in a step because parallel system B will be able to run in the current parallel step.
- If _parallel system A_ depends on _exclusive system B_ that is a direct blocking dependency and cannot run in the current parallel step. It has to wait until system B runs in some exclusive step.
- It gets a little trickier because if a system has a dependency on a system that has blocking dependency it will also be blocked. i.e. If _parallel system C_ depends on _parallel system A_ which has a dependency on _exclusive system B_, it also cannot be queued to run, because A is blocked by B which blocks C from running. In this scenario a system A will move from a blocking dependency to a non-blocking dependency once C is run.

### Splitting up Run Criteria

Run criteria currently can evaluate to one of four values, Yes, No, YesAndCheckAgain, and NoAndCheckAgain. This has made it difficult to combine run criteria. Simple generic boolean operations like `and` and `or` are not possible to make, because combining a simple yes/no value with YesAndCheckAgain or NoAndCheckAgain doesn't have a clear answer. Should `No && YesAndCheckAgain` evaluate to `No`, `NoAndCheckAgain`, or even stay as `YesAndCheckAgain`?

The simple solution to this problem is to separate whether a system should run from whether the run criteria should be checked again. Run Criteria becomes a boolean and a new type of system called Looping Criteria returns a boolean with whether the run criteria should be checked again.

### Implementation of Looping Criteria

Systems with a looping criteria are run in a loop until the looping criteria returns false. Systems with looping criteria are considered a blocking dependency to systems that depend on them until the looping criteria returns false.

A problem with looping criteria that will need to be solved is that they will need to work across hard sync points. i.e. If a looping criteria is applied to a system set. The looping criteria can apply to a sub graph that might require multiple hard syncs before reevaluating the run criteria. Looping criteria will need to keep track of whether the systems dependent on it have run for the current loop before evaluating.

It might be possible to reuse the logic for the system graph and consider looping criteria to have a `after` dependency on all systems that have a dependency on the looping criteria.

### When should run criteria be evaluated?

Currently bevy evaluates all run criteria once before any systems are run and again after all parallel systems run. The proposed loop does not have an equivalent to this as parallel systems are split up between hard sync points. I think there are a couple ways of doing this.

- Before step 1
- Before step 1 and before step 2

Not sure which would be preferred or if there are any options closer to the current model.

### Iteration count statistic

A statistic should be added that says what iteration of the run systems loop it was executed on. Knowning in which loop systems are run will tell the user which systems do not block each other and could help with debugging other scheduling problems.

### Bye-Bye Stages

Stages will be removed as part of this RFC. Not sure on the exact plan yet, but I think it should be straightforward.

Current stages will probably be replaced by system sets. But chould also be redone with more mind payed to the data flow.

## Drawbacks

- This makes a bevy tick a lot less linear and harder to reason about. There will probably need to be good visualization tools to help deal with this.
- Mapping the current stages onto the new method may get confusing. If a
  system is in PREUPDATE and another system is in POSTUPDATE what is the relationship between the two? Are they related at all? Are they implicitly depending on something happending in UPDATE? It will take some careful thought to correctly map the current stages and systems to this new api.
- It might be very easy to add a bunch of hard sync points into your tick making things slow. I think in practice the automatic grouping should prevent this in most cases, but something to watch out for. System visualization tools will help with this.

## Rationale

The base API makes a lot of sense in that a system with commands has 2 points that need to be scheduled. This API is as explicit as possible with that. There are some alternative scheduling apis mentioned below that are more implicit with how the commands are scheduled.

The run systems algorithm is fairly simple, but should do a decent job of keeping the number of hard sync points down and a decent job of parallelizing. There are possibly more optimal algorithms for grouping systems, but this is at least straightforward and will provide a baseline for future improvements.

## Alternative scheduling APIs

These APIs to be not as powerful as the one proposed and schedule commands more implicitly. However more implicitness might be desired if they are more intuitive. However I'm not sure they are more intuitive.

- `.hard_after(Label)` and `.hard_before(Label)` these would require there to be a hard sync point between the system and the `Label`
- `.sync()` this would require any system that depended on the system to wait until after a hard sync point before running

## Alternatives for introducting hard sync points

### name the sync points

An possible alternative for adding hard sync points would be to explicitly create named sync points, which systems can be then placed before or after. This would be much closer to how stages currently work, but much easier than stages to add sync points. However I don't think this is a good idea. While more flexible than stages, it still has the same cognitive load of which logical division of your program you should group unrelated systems into to maximize parallelism.

## Prior art

- The current scheduler is described here https://ratysz.github.io/article/scheduling-1/

## Unresolved questions

- need some basic run criteria and looping criteria to be implemented. What should these be? `fixed timestep`, `run once`, `on true`, `on false`. This could be a different RFC.
- What will states look like? States will need to be reworked too. States are a way of controlling whether a group of systems should be run or not. A typical flow of states would be something like start menu -> Load Level -> InGame -> InGame/Paused -> InGame -> Load Level -> etc. States might be able to be implemented through run criteria and looping criteria. But it's possible states may need to be lifted into it's own first class primitive. This should probably be in a separate RFC, but may affect this one.
- `.startup()` for scheduling systems at the startup of the application could possibly be replaced with run criteria and a special `STARTUP` label that doesn't allow `.before`. Though this might need to be considered with the things that address scheduling things around the application lifecycle.
- should RunCriteria and LoopingCriteria return a special type (Yes/No) rather than a bool? This could prevent them from being used where they're not supposed to be used.
- There should probably be a default place to insert systems into the schedule? The current add_system inserts systems into the UPDATE stage. What is the equivalent to this?

## Future possibilities

- Possibly a better scheduling algorithms to balance load between threads. This would probably need some sort of user hinting.
- Built in states/labels for lifecycle scheduling
- The multipiping PR #0000 is needed to allow run criteria to be easily composable. Another RFC should map out some basic run criteria operators to make run criteria easily composable. Things like `and`, `or`, or `not`.

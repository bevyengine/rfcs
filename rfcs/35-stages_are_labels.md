# Feature Name: `stages_are_labels`

##  Summary

Reincarnates stages as hierarchical labels and splits run criteria into conditions and transitions; solution for the problems collated in [#2801][1]. 

## Motivation

Too many scheduling rework discussions get caught up on sync points when those are really a question for the end user. Any potential scheduling API that would prescribe how often they should be used or demand highly granular constraints would likely create a poor user experience.

Instead, this RFC takes a "less is more" approach and describes how to revise the existing primitives to fix problems, capture missing functionality, and improve ergonomics.

## User-facing explanation

### Commands

Commands are structural changes to the world. These include spawning or despawning entities and inserting or removing their components. These actions are noteworthy because they indirectly affect components with table storage.

### Systems

Systems are functions that can read and write data stored in the world. They come in two types—**concurrent** and **exclusive**.

**Concurrent** systems are scheduled by the executor according to what data they access, the nature of that access, and any user-given ordering constraints. Concurrent systems with non-conflicting data access can be executed in parallel. For example, a concurrent system that only reads `Transform` components cannot execute at the same time as another concurrent system that can write them. Concurrent systems may produce commands, but the actual application of those commands must be deferred.

**Exclusive** systems mutably borrow the entire world, so by definition they cannot execute in parallel with other systems. Only exclusive systems have the ability to apply structural changes (e.g. deferred commands) to the world.

### (System) Sets

System sets are a tool for applying the same constraints to multiple systems. Their purpose is to cut down on boilerplate.

### Stages

Stages represent **atomic** blocks of systems. Assigning systems to a stage informs the executor that, as a group, those systems should be preceded and followed by an exclusive system that applies any pending commands to the world. By virtue of sitting between these "sync points," systems inside a stage are completely isolated from systems outside of it.

It's worth highlighting that there is no overlap between stages and system sets.

### Why are people so eager to remove stages?

Stages explicitly limit concurrency, so some people see them as a particularly "heavy" scheduling constraint. While this is true in a sense, loss of potential concurrency is necessary to apply pending structural changes. This is a conscious design decision.

Unfortunately, naming the overall design exploration "stageless" has caused many people to mistakenly believe that *having stages* is a problem. It isn't. Although many problems highlighted in [#2801][1] can indeed be traced back to their current *implementation*, stages themselves are not problematic.

So what are the root problems?

1. The `SystemStage` type is [a very large monolithic thing][2] that owns its systems and has its own executor.
2. They can't be nested.

These properties severely limit stages from reaching their full potential. For example, because `SystemStage` instances cannot be nested, if users want to implement a common flow control pattern, like making a frame-independent loop, they have to do something awkward like this.

```rust
fn main() {
    App::new()
        /* ... */
        .with_stage("frame-independent loop", 
            Schedule::default()
                .with_stage(/* ... */)
                /* ... */
        ).with_run_criteria(FixedTimestep::steps_per_second(60))
        /* ... */    
    .run();  
}
```

### RunCriteria

Run criteria are special systems that users can attach to systems and stages to control when they run. The executor evaluates system run criteria at the beginning of each stage and then orders the systems whose criteria have been met. 

There's more to them, though. Run criteria are also responsible for making stages loop and transitioning between states. This extra responsibility makes combining run criteria rather messy since, instead of booleans, run criteria return a [`ShouldRun`][3] enum.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ShouldRun {
    Yes,
    No,
    YesAndCheckAgain,
    NoAndCheckAgain,
}
```

An unfortunate side effect of the current implementation is that `SystemStage` loops will "lose" to `State` transitions whenever they conflict.

## Proposed Changes

### One Executor

A big, recurring idea in "stageless" design discussions is having one executor for the entire schedule. This lone executor would simplify internal code and give apps maximum flexibility to order systems concurrently (since it knows about all systems). Implementing this is basically renaming `SystemStage` to `Schedule`.

### Hierarchical Stage Labels

The lone executor in place, stages can be reincarnated as specialized system labels. Since the executor has knowledge of all systems, stages being labels would still provide enough information to the app builder to insert exclusive sync systems. However, we can go a step further and organize these stage labels in a tree.

These **stage trees** are new, purely topological building blocks designed to provide scaffolding for complex flow control patterns, replacing nested schedules. What's cool is that users can create this scaffolding independently of adding systems to the schedule.

![A visual example of a stage tree](../media/stage_tree.png "A visual example of a stage tree")
(The blocks in this picture are stages.)

The API for creating the pictured tree could look like this.

```rust
impl App {
    pub fn minimal() -> Self {
        Self::default()
            .add_stage(CoreStage::Startup)
            .add_stage(CoreStage::Frame.with_transition(loop_until_shutdown))
            .add_stage(CoreStage::Shutdown)
    }

    pub fn standard() -> Self {
        Self::minimal()
            .add_stage(CoreStage::First.append_to(CoreStage::Frame))
            .add_stage(CoreStage::PreUpdate.append_to(CoreStage::Frame))
            .add_stage(CoreStage::Update.append_to(CoreStage::Frame))
            .add_stage(CoreStage::PostUpdate.append_to(CoreStage::Frame))
            .add_stage(CoreStage::Last.append_to(CoreStage::Frame))
    }
}
```

### RunCriteria are split into transitions and conditions

This rework consolidates stage loops and state transitions into one primitive called **transitions**, putting them both on the same level. 

Transitions are now a first-class type users have to take control of the "main execution stack" (see next section). Transitions share many properties with commands. All systems can queue transitions, but the final decision cannot be made until the end of a stage. Unlike commands, users must write and add their own transition-handling systems to stages in order to use them.

```rust
enum AppState {
    Loading,
    InGame,
    Paused,
}

fn set_app_state_to_paused_if_condition(
    mut transitions: Transitions, // Not sure what fields this type should have.
    mut stack: ResMut<ExecutorStack>,
    current_state: Res<ExecutorState>,
) {
    for transition in transitions {
        if /* ... */ {
            // Change the global app state in the upcoming state to Paused.
            // Alt. you could queue an entire paused stage tree.
            // Lots of options, which is why the user is responsible.
            let mut next_state = stack.peek();
            stack.set(next_state.set_app_state(AppState::Paused));
        }
    }
}
```

With their transition responsibilities removed, run criteria become simple boolean **conditions** that can be combined in intuitive ways through `and`, `or`, `xor` and `not` modifiers. Conditions can be attached to systems, stages, and complete schedules.

System conditions are re-evaluated every time their stage is executed. Stage conditions are a little more complicated (explained in pseudocode below). Stage execution follows post-order (left to right, children before parents).

```
loop
    pop stage from stack

    if stage is running 
        if all children have completed
            break
        else
            panic

    if stage condition == true
        if stage is leaf 
            break
        else
            set stage status to running
            push stage back onto stack
            push children onto stack in reverse order (right to left)
            continue

execute stage
```

### States and the execution stack

Schedules can have more than one stage tree, because why not? Disjoint stage trees can be linked using transitions. As mentioned in the previous section, transitions allow users to control something called the "execution stack," which is a [stack-based state machine][9]. The executor itself basically doubles as one big [FSM driver][10], where the next stage to be executed is popped from the top of this stack. When handling transitions, the stack can be modified using familiar `push`, `set`, and `replace` operations.

```rust
pub struct ExecutorState {
    platform_state: BoxedPlatformStateLabel,
    app_state: BoxedAppStateLabel,
    stage_tree: BoxedStageTreeLabel,
    stage: BoxedStageLabel,
}

pub struct Executor {
    /* ... */
    current: ExecutorState,
    stack: Vec<ExecutorState>,
    /* ... */
}
```

What about states then?

Well in this rework, app states (and "lifecycle" states) simply increase the number of combinations that can be pushed onto the stack. States and stage trees have a decoupled* many-to-many relationship which, when combined with the utility of states in general pattern matching throughout an app, makes the whole setup very flexible. 

Too flexible, actually. Transitions and the execution stack are safe abstractions given the atomicity of stages, but they leave users wide open to writing [spaghetti][7]. For a better user experience, Bevy likely needs a higher-level abstraction with more guard rails (i.e. state charts) and visualization tools.

*While decoupled, transitions are still aligned to stage boundaries, so the state cannot be changed in the middle of a leaf stage.

### Overall Scheduling API

```rust
#[derive(SystemLabel)]
struct Label(&'static str);

#[derive(SetLabel)]
struct Set(&'static str);

#[derive(StageLabel)]
struct Stage(&'static str);

#[derive(StageTreeLabel)]
struct Tree(&'static str);

#[derive(ConditionLabel)]
struct Condition(&'static str);

#[derive(TransitionLabel)]
struct Transition(&'static str);

// Add to default stage tree
.add_stage(Stage)
.add_stage(Stage.prepend_to(Stage))
.add_stage(Stage.append_to(Stage))
.add_stage(Stage.before(Stage))
.add_stage(Stage.after(Stage))
.add_stage(Stage
    .with_condition(Condition)
    .with_transition(Transition))

// Add new stage tree
.add_tree(Tree)
    .add_stage(Stage
        .with_condition(Condition)
        .with_transition(Transition))

.add_set(Set
    .label(Label)
    .to_stage(Stage)
    .after(Label)
    .with_condition(Condition.and(Condition.not())))

.add_system(System
    .label(Label)
    .to_set(Set)
    .to_stage(Stage)
    .before(Label)
    .with_condition(Condition))

// conditions
.and(Condition)
.or(Condition)
.xor(Condition)
.not()
```

### So what does all this accomplish?

Achieves clearer separation of responsibilities, facilitates more code reuse, and greatly increases flexibility.

- Having one executor maximizes potential concurrency.
- Stages are just labels, so:
  - They can be optional. Schedules can have a default stage tree.
  - They can be used with system sets.
  - They can be used to block out a schedule separately from adding systems.
  - They can be exported by plugins for users to incorporate "pre-packaged" logic into their schedule.
    - Users can insert plugin stages into their stages and vice-versa.
- Stages can completely define an app. System modifiers like `.startup()` are no longer needed.
- The same stage can be added to multiple trees.
- The same tree can be paired with multiple states.
- Conditions can be combined through simple boolean operations.
- Transitions no longer have implicit conflicts.
- States no longer require a unique stage variant.

## Implementation Details

(TODO)

### Constructing stage trees
```rust
use indextree::{Arena as Tree, NodeId};

pub struct StageTree {
    // Map stage labels to nodes in the tree.
    map: HashMap<BoxedStageLabel, NodeId>,
    // The actual tree.
    // bool for "is running"
    tree: Tree<(BoxedStageLabel, bool)>,
}

impl Default for StageTree {
    fn default() -> Self {
        let mut map = HashMap::new();
        let mut tree = Tree::new();
        map.insert(Core::Main, tree.insert(Core::Main));
        Self {
            map,
            tree,
        }
    }
}

impl StageTree {
    pub fn insert_stage_before(&mut self, ref_label: BoxedStageLabel, label: BoxedStageLabel) {
        let ref_node_id = self.map.get_mut(ref_label).unwrap();
        let node_id = self.tree.insert(label);
        self.map.insert(label, node_id);
        ref_node_id.insert_before(node_id, &mut self.tree);
    }
    
    pub fn insert_stage_after(&mut self, ref_label: BoxedStageLabel, label: BoxedStageLabel) {
        let ref_node_id = self.map.get_mut(ref_label).unwrap();
        let node_id = self.tree.insert(label);
        self.map.insert(label, node_id);
        ref_node_id.insert_after(node_id, &mut self.tree);
    }
    
    pub fn append_substage(&mut self, ref_label: BoxedStageLabel, label: BoxedStageLabel) {
        let ref_node_id = self.map.get_mut(ref_label).unwrap();
        let node_id = self.tree.insert(label);
        self.map.insert(label, node_id);
        ref_node_id.append(node_id, &mut self.tree);
    }
    
    pub fn prepend_substage(&mut self, ref_label: BoxedStageLabel, label: BoxedStageLabel) {
        let ref_node_id = self.map.get_mut(ref_label).unwrap();
        let node_id = self.tree.insert(label);
        self.map.insert(label, node_id);
        ref_node_id.prepend(node_id, &mut self.tree);
    }

    pub fn prepend_stage(&mut self, label: BoxedStageLabel) {
        self.prepend_substage(Core::Main, label)
    }
    
    pub fn append_stage(&mut self, label: BoxedStageLabel) {
        self.append_substage(Core::Main, label)
    }
}
```

### Single schedule and executor

```rust
pub struct Schedule {
    world_id: Option<WorldId>,
    condition: BoxedCondition,
    stage_trees: HashMap<BoxedStageTreeLabel, StageTree>,
    stages: HashMap<BoxedStageLabel, StageContainer>,
    concurrent_systems: Vec<ConcurrentSystemContainer>,
    exclusive_systems: Vec<ExclusiveSystemContainer>,
    system_conditions: Vec<ConditionContainer>,
    uninit_concurrent_systems: Vec<usize>,
    uninit_exclusive_systems: Vec<usize>,
    uninit_system_conditions: Vec<(usize, DuplicateLabelStrategy)>,
    systems_were_modified: bool,
    executor_was_modified: bool,
}

pub struct StageContainer {
    condition: BoxedCondition,
    transition: BoxedTransition,
}

```
### How the executor quickly finds all the systems to be run in a stage

```rust
TODO
```

## Drawbacks

- TBD

## Rationale and alternatives

### Why can't stages/states run in parallel?

First, it really only makes sense to ask this question about stages imported from plugins. Users can always add their own systems to the same stage. 

Now, to answer the question:

1. Commands require exclusive mutable world borrows, so sync points can only exist in series.
2. Transitions cannot compose.
3. Bevy respects plugin privacy rules and does not allow users to freely dissect imported stages.

Hypothetically, if Bevy knew that some commands would never affect certain archetypes, it could scope the world access appropriately and run other systems in parallel. However, that would require **(a)** a much more verbose API for specifying exact system side effects and relationships and **(b)** hard-coding many more special cases into the schedule builder. 

Forcing this level of explicitness on the general user would be bad. 

Still, for the sake of argument, if we had such an API available to complement this one, maximizing concurrency would require dissolving plugin stages and adding new constraints to their systems to deal with the resulting ambiguities, which are both things that plugin privacy rules specifically prevent.

So ultimately, the only way to accommodate everyone is for plugins to export both stage labels and system (set) labels, and then either provide a means to [reconfigure labeled elements][4] or just export the system functions directly. (`@cart` [is against requiring plugins to make everything public][5].)

### Isn't it better to avoid sync points?

Sync points are neither good nor bad. They have to exist whenever you want to apply commands. Reducing the number of sync points in a schedule requires deliberate decisions to redesign logic to defer commands or accept stale reads, and Bevy can't make those decisions for you.

Optimal scheduling is an NP-complete problem. It's okay to not have a perfect schedule.

## Unresolved questions

- What should happen when a user appends/prepends a sub-stage to a stage that already has systems?
  - Group those systems into an anonymous sub-stage? 
- Constraints on transitions?
  - Should they be marked with their "terminal" stage and be discarded once that stage completes?
    - If so, how? `ExecutorState`?
  - Should any system be able to queue a transition for any stage or should there be constraints?
- Can we get rid of exclusive system modifiers in favor of ordering them like everything else?
  - `.at_start()`
  - `.before_commands()`
  - `.at_end()`

## Future Possibilities

- Having a single executor and reducing stages to labels should make it easier to [rebuild the schedule at runtime][8].

- The proposed changes should be compatible with the plugin configuration machinery described in [#33][4]. In fact, stages being labels should make it easier for plugins to transparently share "pre-packaged" functionality. Plugins could simply check which of their stage labels have been inserted, then add the appropriate systems and stage trees.

    ```rust
    impl Plugin for MyPlugin {
        fn build(state: &mut PluginState) {
            for label in my_plugin_crate_labels {
                if state.labels().contains(&label) {
                    // add (sub) stages, systems, and system sets
                    // insert and initialize resources
                    // build other dependencies
                    /* ... */
                }
            }
        }
    }
    ```

- If a user is willing to forgo commands and accept stale reads for some calculation, they can spawn a task, copy the required data into it (eliminating race conditions), run it asynchronously in the background, then poll or block on its completion later. Perhaps Bevy could highlight the task API for this (or come up with a higher-level abstraction if it would help). Unity's [Job System][6] is a good example. The new pipelined renderer operates on a similar principle by having a separate world.

[1]: https://github.com/bevyengine/bevy/discussions/2801
[2]: https://github.com/bevyengine/bevy/blob/997eae61854b362c10810884966ac605121e04d7/crates/bevy_ecs/src/schedule/stage.rs#L53
[3]: https://github.com/bevyengine/bevy/blob/5ba2b9adcf5e686273cf024acf1ad8ddfb4f8e18/crates/bevy_ecs/src/schedule/run_criteria.rs#L32
[4]: https://github.com/alice-i-cecile/rfcs/blob/apps-own-scheduling/rfcs/33-apps_own_scheduling.md#configuring-systems-by-their-label
[5]: https://github.com/bevyengine/rfcs/pull/33#discussion_r709665190
[6]: https://docs.unity3d.com/Manual/JobSystemSafetySystem.html
[7]: https://statecharts.dev/state-machine-state-explosion.html
[8]: https://github.com/bevyengine/bevy/pull/2507
[9]: https://bevyengine.org/news/bevy-0-5/#states-v2
[10]: https://github.com/bevyengine/bevy/discussions/2801#discussioncomment-1303893
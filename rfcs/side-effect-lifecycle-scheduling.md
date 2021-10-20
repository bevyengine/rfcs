
>Draft

# RFC: Encoding Side Effect Lifecycles as Subgraphs

This RFC presents an API for scheduling around the lifecycle of side effects. This assumes that exclusive systems can be labeled and ordered in the same way parallel systems can and does not include how the executor might need to be changed to support this API.

## Motivation

A system may produce a side effect that other systems must read, write, and potentially consume. Systems that do this have a very specific order since the side effect must be produced before it can be acted on and cannot be acted upon after they have been consumed. An example of this would be commands. Commands are created in one system and then consumed later when they're applied to the world. If you tried to read the command queue after the commands have been applied, you would only find an empty queue.

We can express this required ordering with labels, before, and after, but they lack semantic meaning. This is an api for creating subgraphs that express the required ordering between producers, consumers, readers, and writers.

## User Facing Description

### Scheduling Side Effects API
* `.produces(Label)` System produces a side effect of type `Label`
* `.consumes(Label)` No read/modifies systems can run after this system. An example that would use this is the system that applies commands to the world.
* `.reads(Label)` system has read access to side effect and must run after the producers and before the consumer. An example of this type of system would be systems that depend on a run criteria.
* `.writes(Label)` system writes to the side effect and must run after the producers and before the consumer

#### Relationships enforced between producers, readers, modifiers, and consumers

* Producers must be before readers, writers, and the consumer.
* Readers and writers must be after the producer and before the consumer. They can run in any order in relation to other readers and writers. If there needs to be an order between readers and writers, use the normal before and after labeling.
* Consumers are run after producers, readers, and writers. There should only be one consumer of a type of side effect.

### Some Simple Examples

#### Scheduling around command application
```rs
fn main() {
    App::new()
        .add_system(apply_commands_buffer.consumes("commands"))
        // if we don't care about when the commands are applied
        // we can ignore the new api
        .add_system(create_commands_A)
        
        // if we need to schedule a system after commands are applied
        .add_system(create_commands_A.label("system A").produces("commands"))
        .add_system(after_commands_A.after("system A").after("commands"))
}
```

#### Systems dependent on app state
```rs
// this is not an example of how a final api for app state
// or run criteria is expected to look. Just a way to highlight
// the relations between the systems that produce those side effects

fn main() {
    App::new()
        .add_system(system_that_queues_state_change.produces("state change"))
        .add_system(system_that_applies_state_queue
            .consumes("state change")
            .produces("state")
        )
        .add_system(run_criteria
            .reads("state")
            .produces("on update state=A")
        )
        .add_system(system_A.reads("on update state=A"))
        .add_system(system_B.reads("on update state=A"))
}
```

## Implementation

If we tried to create the relationships between producers, consumers, readers, and writers with system sets and labeling, it would look something like this.
```rs
// expressing the subgraph with system sets and labels

fn main() {
    let SideEffectLabel = Label::new("some side effect");
    App::new()
        .add_system_set(SystemSet::new()
            .label(SideEffectLabel)
            .add_system_set(SystemSet::new()
                .label(produces(SideEffectLabel))
                .before(reads(SideEffectLabel))
                .before(writes(SideEffectLabel))
                .before(consumes(SideEffectLabel))
                .add_system(side_effect_producer)
            )
        )
        // TODO: add consumers, readers, and writer system sets
}
```

In practice we can't create these system sets like this as it would be very unergonomic for users. But systems sets are just sugar anyways and don't exist at runtime. We can just apply the correct labels to individual systems.

```rs
// .produces
.add_system(side_effect_producer.produces(SideEffectLabel))
// is equivalent to
.add_system(side_effect_producer
    .label(SideEffectLabel)
    .label(produces(SideEffectLabel))
    .before(reads(SideEffectLabel))
    .before(writes(SideEffectLabel))
    .before(comsumes(SideEffectLabel))
)
```

> TODO: add desugaring of readers, writers, and consumers.

## Drawbacks

I expect that these relationships will mostly be added automatically when using run criteria, app state, and the like with a higher level api. Users would only be expected to interact with this API when things can't be done automatically or the default behavior doesn't match their needs.

## Open questions

* With looping criteria it might be possible to create a valid dependency cycle that cannot be resolved in the current API. If that is the case then we might need an `.after_can_break` type of edge that could be ignored in dependency cycle checking.

## Alternatives Considered

* The example above for expressing the relationships between side effects using just labeling and system sets has one major problem. There is no current api for adding systems to system sets after the system set has been created. If such an api did exist we could potentially just add systems to the correct system set. This didn't seem better than the proposed API and has some implementation complexity since system sets are mostly just api sugar for labeling multiple systems and does not exist at runtime.

## Future Work

* This is part of a bigger redesign of the ECS trying to solve problems with scheduling around hard sync points and data consistency between states. This RFC by itself doesn't solve those problems, but it does encode information about dependencies that other RFCs might need to actually solve the data consistency problems. 
* Side effects are effectively typed. Build an API to pass and send side effects and enforce the typing between producers and consumers. This could either be with data stored in the executor or a Resource that manages access through access tokens (what Boxy is working on).
* How to specify an order to how commands are applied?
* This API doesn't really directly address data consistency problems due to changes to run criteria. Ideally we'd figure out a way to enforce that the consumer needs to run if side effects have been produced.

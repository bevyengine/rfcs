
>Draft

# RFC: Encoding Side Effect Lifecycles as Subgraphs

The purpose of this RFC is to convince you that side effect lifetimes exist and to propose an API for annotating those lifetimes. It does not try to give any guarantees that the actual lifetimes of the side effects match the annotations. Solutions for this are pushed into future RFCs.

## Motivation

Side Effects in an ECS are a bit of an abstract concept. Systems almost always produce side effects since they are trying to do something and they don't have the clean functional ability to return a value. Some of these side effects are particularly troublesome. These side effects are born, spread their influence, and then die all within a single tick of the schedule graph. If we can express this lifecycle and influence as a subgraph of the graph of all systems, we'll be better able to handle the awkwardness of side effects.

To try and put this in more concrete terms, Bevy produces a command queues when it wants to spawn an entity. We could consider the lifespan of the side effect from this system to be from when the system produces the command queue until that command queue is applied to the world. Or on the other hand,the lifespan could be from when the system produces the command queue until the entity is despawned. The exact lifetimes of the side effects are not important to this RFC. What is important is finding a way to express the relationship between the systems that deal with a specific side effect.

We can think of this as being similar to rust lifetimes. The system that a side effect is produced in is where a variable is instantiated. Other systems borrow the value temporarily. The consuming system takes or moves the side effect, so no other systems in the original scope can see the side effect after the value is taken.

The ordering constraints are very simple in practice. Systems that want to react to a side effect must run within the lifetime of the side effect. Or in more detail: There can be multiple producers of a side effect. Anything that handles these side effects has to come after all the producers run. There can only be one consumer as there needs to be a singular node after which the side effects no longer exist. So systems that handle side effects must come before the consumer. These ordering constraints can be expressed with the existing before and after labeling methods, but are important enough that they deserve their own API.

## User Facing Description

> I expect that these relationships will mostly be added automatically when using run criteria, app state, and the like with a higher level API. Users would only be expected to interact with this API when things can't be done automatically or the default behavior doesn't match their needs.

### Scheduling Side Effects API
* `.produces(Label)` System produces a side effect of type `Label`
* `.handles(Label)` system writes or reads the side effect and must run after the producers and before the consumer
* `.consumes(Label)` No read/modifies systems can run after this system. An example that would use this is the system that applies commands to the world.

#### Relationships enforced between producers, handlers, and consumers

* Producers must be before readers, writers, and the consumer.
* Handlers must be after the producer and before the consumer. They can run in any order in relation to other handlers. If there needs to be an order between handlers, use the normal before and after labeling.
* Consumers are run after all producers, and handlers. There should only be one consumer of a type of side effect.

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

#### Ordering command buffers
```rs
// this is currently not possible
fn main() {
    App::new()
        .add_system(create_commands_A.produces("commands A"))
        .add_system(create_commands_B.produces("commands B"))
        .add_system(apply_commands_buffer
            .consumes("commands A")
            .label("apply commands A")
        )
        .add_system(apply_commands_buffer
            .consumes("commands B")
            .before("apply commands A")
        );
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

If we tried to create the relationships between producers, handlers, and consumers with system sets and labeling, it would look something like this.
```rs
// expressing the subgraph with system sets and labels

fn main() {
    let SideEffectLabel = Label::new("some side effect");
    App::new()
        .add_system_set(SystemSet::new()
            .label(SideEffectLabel)
            .add_system_set(SystemSet::new()
                .label(produces(SideEffectLabel))
                .before(handles(SideEffectLabel))
                .before(consumes(SideEffectLabel))
                .add_system(side_effect_producer)
            )
            .add_system_set(SystemSet::new()
                .label(handles(SideEffectLabel))
                .after(produces(SideEffectLabel))
                .before(consumes(SideEffectLabel))
                .add_system(side_effect_handler)
            )
            .add_system_set(SystemSet::new()
                .label(consumes(SideEffectLabel))
                .after(produces(SideEffectLabel))
                .after(handles(SideEffectLabel))
                .add_system(side_effect_consumer)
            )
        ))
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
    .before(handles(SideEffectLabel))
    .before(consumes(SideEffectLabel))
)

// .handles
.add_system(side_effect_reader.handles(SideEffectLabel))
// is equivalent to
.add_system(side_effect_reader
    .label(SideEffectLabel)
    .label(handles(SideEffectLabel))
    .after(produces(SideEffectLabel))
    .before(consumes(SideEffectLabel))
)

// .consumes
.add_system(side_effect_consumer.consumes(SideEffectLabel))
// is equivalent to
.add_system(side_effect_consumer
    .label(SideEffectLabel)
    .label(consumes(SideEffectLabel))
    .after(produces(SideEffectLabel))
    .after(handles(SideEffectLabel))
)
```
## Drawbacks

* The major drawback is that this API is adding a lot more interconnection in the scheduling graph which can potentially make things harder to reason about or accidentally create dependency cycles. I think this is unavoidable and the advantages for data consistency are clear.

## Open questions

* It is possible to create a *valid* dependency cycle that cannot be resolved. This can be caused when systems need to run repeatedly as with looping run criteria. If that is the case then we might need an `.after_can_break` type of edge that could be ignored in dependency cycle checking.
* `.uses(Label)` There is a case to be made for wanting to order things after the consumer has done something with the side effect. I'm not sure if this is a separate case though or just another side effect. Like in the case of commands the consumer consumes the command queue, but it also produces an entity. So in this case what is the difference between `.uses("entity")` vs `.consumes("entity")`. It could potentially be more ergonomic as `.uses` is a more generalized version of `.after_commands` from https://github.com/bevyengine/rfcs/pull/34

## Alternatives Considered

* The example above for expressing the relationships between side effects using just labeling and system sets has one major problem. There is no current api for adding systems to system sets after the system set has been created. If such an api did exist we could potentially just add systems to the correct system set. This didn't seem better than the proposed API and has some implementation complexity since system sets are mostly just api sugar for labeling multiple systems and does not exist at runtime.

## Future Work

* This is part of a bigger redesign of the ECS trying to solve problems with scheduling around hard sync points and data consistency between states. This RFC by itself doesn't solve those problems, but it does encode information about dependencies that other RFCs might need to actually solve the data consistency problems. 
* Side effects are effectively typed. Build an API to pass and send side effects and enforce the typing between producers and consumers. This could either be with data stored in the executor or a Resource that manages access through access tokens (what Boxy is working on).
* This API doesn't really directly address data consistency problems due to changes to run criteria. Ideally we'd figure out a way to enforce that the consumer needs to run if side effects have been produced. I don't think this can be done in the system graph construction stage and probably needs to be done at run time somehow.
* I initially had `.writes` and `.reads`, but merged them together into `.handles` because they have the same ordering constraints. It's possible that this API could be used for a data transport layer for shipping around data associated with side effects and `.handles` would then need to be split apart again, because reads can happen in parallel, but writes can't.
# Feature Name: Topics

:warning: This proposal depends on [one-shot systems](https://github.com/bevyengine/bevy/pull/4090), which is not yet a merged feature, and possibly [trait queries](https://github.com/bevyengine/rfcs/pull/39) which is not yet a merged RFC.

## Summary

This RFC introduces a feature to help with developing reactive applications.
Topics are data channels for moving asynchronous messages through many-to-many relationships between publishers and subscribers.
Each subscriber can listen to messages from many publishers, and many subscribers can listen to each publisher.

## Motivation

Topics are a common mechanism for middlewares to pass data between distributed systems.
Often called pub-sub (publish-subscribe), this pattern is supported by middlewares like ROS, MQTT, ZeroMQ, Kafka, RabbitMQ, and many others.
In these systems, anonymous publishers are able to push out messages to "topics" (data channels) which anonymous subscribers can listen to.
This allows the development of distributed reactive systems, organized by the abstraction of "topics" so that publishers can be easily discovered by relevant subscribers without needing to know details of how the overall distributed system is implemented.

This RFC proposes implementing publishers and subscribers, with topic-based discovery inside of Bevy as ergonomic Systems, Resources, and Components.

Topics can be used inside of Bevy to help Bevy users implement their application using a "distributed" architecture to decouple and organize event flows from various agents/entities.
This is similar to a previous suggestion about [Entity Events](https://github.com/bevyengine/bevy/issues/2070) and has similar motivations.
Bevy's current resource-based Event mechanism is not convenient for representing more complex flows of information that may be entity/agent-driven or entity/agent-sensitive.

### Example Use-Case
* A game has NPCs that can react to various events.
* The NPCs are split into two teams: `"Red"` and `"Blue"` team and there are team-wide events.
* There are team-agnostic event types, but not all NPCs should listen to all of these events.
* Not all NPCs should react to events in the same way.

To program the logic for this game with the existing `Events` system, the programmer would need to define one single system that accounts for every possible combination of these factors.
The many conditional combinations will cause the single system to have many complex branches.

With pub-sub and topics, the programmer could break down the problem as follows:
* NPC entities on the `"Red"` team subscribe to the `"Red"` topic.
* NPC entities on the `"Blue"` team subscribe to the `"Blue"` topic.
* Additional topics are only subscribed by each entity if they are relevant for the entity.
* Each subscription specifies the appropriate behavior for its own NPC.

The different conditional combinations that need to be accounted for are expressed more easily through topic subscription management.

### Networking
At the same time, defining this generic ergonomic pipeline for topics within Bevy should help facilitate the integration of various middlewares into Bevy.
For example, a `bevy_mqtt` plugin could simply subscribe to bevy topics and then effortlessly publish the received messages to MQTT topics, and vice-versa for two-way communication.

Bevy users who want their application to have networking or inter-process communication can implement their systems in terms of the bevy topics, then plug in whichever middleware suits their overall networked / multi-process system.
They could also change their choice of middleware later without changing how the systems of their application are implemented.
Or multiple middlewares can be used by one application without affecting how the systems are implemented.

## User-facing explanation

### Topics

These are the key structs for topics:
* `Publisher<T>`: a component struct that provides a `publish(data: T)` method
* `Subscriber<T>`: a component struct that provides a `subscribe<T, P: SystemParam, F: Fn(T, P)>(publisher: Entity, callback: F)` method
* `Domain`: a component that maintains a dictionary of topic names along with the publishers and subscribers that are interested in each topic
* `GlobalDomain<T>`: a Domain that can be used as a resource instead of as a component

To send out messages, spawn a `Publisher<T>` component for an entity, optionally giving some initial messages:

```rust
enum Mission {
    Attack,
    Hold,
    Retreat,
}

/// Colonels issue missions to their battalions, so they need to publish Mission messages
fn connect_colonels(
    commands: Commands,
    new_colonels: Query<(Entity, &Battalion), Added<Colonel>>,
    domain: ResMut<GlobalDomain<BattleSim>>,
) {
    for (e, battalion) in &new_colonels {
        let mut publisher = Publisher::<Mission>::new();
        publisher.publish(Mission::Hold);
        commands.entity(e).insert(publisher);
    }
}
```

To use a spawned publisher, query for a mutable reference and use its `publish(data: T)` method:

```rust
/// On each update cycle, the colonels decide what action to take
fn update_colonels(
    armies: Query<(&BattleConditions, &mut Publisher<Mission>), With<Colonel>>,
) {
    for (condition, mission) in &mut publishers {
        if condition.is_critical() {
            mission.publish(Mission::Retreat);
        } else if condition.have_advantage() {
            mission.publish(Mission::Attack);
        } else {
            mission.publish(Mission::Hold);
        }
    }
}
```

To help subscribers discover the publisher, use the `advertise(publisher: Entity, topic: String)` domain method to make the publisher visible in that domain:

```rust
/// Colonels issue missions to their battalions, so they need to publish Mission messages
fn connect_colonels(
    commands: Commands,
    new_colonels: Query<(Entity, &Battalion), Added<Colonel>>,
    domain: ResMut<GlobalDomain<BattleSim>>,
) {
    for (e, battalion) in &new_colonels {
        let mut publisher = Publisher::<Mission>::new();
        publisher.publish(Mission::Hold);
        commands.entity(e).insert(publisher);
        // Colonels advertise their mission publisher to the battalion-wide mission topic
        domain.advertise(e, battalion.mission_topic());
    }
}
```

To receive and respond to messages from a specific publisher, spawn a `Subscriber<T>` component for an entity and use its `subscribe(publisher: Entity, callback: F)` method:

```rust
/// Soldiers follow the missions issued to them by their colonels
fn connect_soldiers(
    commands: Commands,
    new_cavalries: Query<(Entity, &Battalion), Added<Cavalry>>,
    new_archers: Query<(Entity, &Battalion), Added<Archer>>,
) {
    // Cavalry units listen for missions from their colonel and then charge, defend, or flee
    for (e, battalion) in initial_soldiers.cavalry() {
        let mut subscriber = Subscriber::<Mission>::new();
        subscriber.subscribe(
            battalion.colonel(),
            move |mission: &Mission, orders: Query<&mut CavalryOrders>| {
                let orders = orders.get_mut(e).unwrap();
                match mission {
                    Mission::Attack => {
                        orders.charge_forward();
                    }
                    Mission::Hold => {
                        orders.defense_formation();
                    }
                    Mission::Retreat => {
                        orders.flee();
                    }
                }
            }
        );

        commands.entity(e).insert(subscriber);
    }

    // Archer units listen for missions from the colonel and then fire, aim, or flee
    for (e, battalion) in initial_soldiers.archers() {
        let mut subscriber = Subscriber::<Mission>::new();
        subscriber.subscribe(
            battalion.colonel(),
            move |mission: &Mission, orders: Query<&mut ArcherOrders>| {
                let orders = orders.get_mut(unit).unwrap();
                match mission {
                    Mission::Attack => {
                        orders.fire_salvo();
                    }
                    Mission::Hold => {
                        orders.take_aim();
                    }
                    Mission::Retreat => {
                        orders.flee();
                    }
                }
            }
        );

        commands.entity(e).insert(subscriber);
    }
}
```

Alternatively, to automatically subscribe to all publishers (current and future) for a topic, use the `discover(for_subscriber: Entity, on_topic: String, callback: F)` domain method:

```rust
/// Spies intercept or interfere with the mission information of a battalion
fn connect_spies(
    commands: Commands,
    new_informants: Query<(Entity, &Informant, &Battalion), Added<Informant>>,
    new_imposters: Query<(Entity, &Imposter), Added<Imposter>>,
    domain: ResMut<GlobalDomain<BattleSim>>,
) {
    for (e, informant, battalion) in &new_informants {
        // Informants listen to the battalion missions and then re-publish them
        // for an enemy army to hear.
        domain.advertise(e, informant.report_topic());
        let battalion_id = battalion.id();
        commands.entity(e).insert(Publisher::<(Entity, Mission)>::new());
        domain.discover(
            battalion.mission_topic(),
            move |mission: &Mission, report: Query<&mut Publisher<(Entity, Mission)>| {
                report.get_mut(e).publish((battalion_id, mission.clone()));
            }
        )
    }

    for (e, imposter) in &new_imposters {
        // Imposters also publish battalion missions to confuse informants.
        // Ordinary soldiers are subscribed directly to their colonels so they
        // are not confused by imposters.
        commands.entity(e).insert(Publisher::<Mission>::new());
        domain.advertise(e, imposter.mission_topic());
    }
}
```

When a subscriber is removed from an entity or an entity that has a subscriber is despawned, the callback that was subscribed will be dropped and not get triggered anymore.

## Implementation strategy

Many details of this implementation proposal will likely need significant iteration with help from someone more familiar than I am with the internals of Bevy.
I am currently not clear on how exactly dynamic system construction can/does work, nor the right way to leverage one-shot systems to make this work.

### Publishing
* Published messages are stored in the `Publisher<T>` struct as a `Smallvec`.
* The `Publisher<T>` will also store a `SmallVec<Entity>` to keep track of its subscribers.
* Once per cycle, the message queue in the publisher will be processed against all current subscribers.
  * For each subscriber of each publisher that has a non-empty queue, generate a one-shot system. The one-shot system would query for:
    * immutable reference to the publisher, to have access to its message queue
    * the system parameter needed by the subscriber
  * The one-shot system would iterate through the message queue of the publisher, passing an immutable reference to each message into the subscriber's callback, along with the system parameter needed by the callback.
  * The subscription callbacks will be able to run in parallel, as long as their additional system parameters don't have conflicts.

### Subscribing
* Subscribers maintain a dictionary of which callback should be used for each `(publisher entity, message type)` pair
* When a new subscription is created using `Subscriber<T>::subscribe`, the entity of the publisher is saved in a special queue for new subscriptions
  * On each cycle, before published messages are processed, a system runs through all subscribers to process any changes in subscriptions
    * (Consider ways to reduce overhead so this isn't an O(N) operation over all the subscribers on every cycle, maybe using a shared atomic bool to decide if changes even need to be checked)
  * When a new subscription is found, the publisher is informed about the entity for that subscriptions
  * When an old subscription is removed, the publisher is informed so it does not keep trying to send messages to the subscriber
  * Once the publisher is informed of the new subscriber, the publisher's entity is removed from the special new subscription queue
* The one-shot system produced by the publisher will use a private function to retrieve the correct callback from the subscriber based on the publisher's entity

### Domain management
The Domain needs to keep track of which topics the entities of publishers and subscribers are interested in.
Whenever a publisher is advertised or a subscriber requests discovery, the Domain must form a connection between the two, informing the publisher about the subscriber and vice-versa.

* New publishers and new subscribers are each kept in a special queue
* On each cycle:
  * The queue of new publishers is cross-examined against the queue of new subscribers as well as the dictionary of old subscribers to look for matches
  * New subscribers are cross-examined against old publishers to look for matches
  * Matching publishers + subscribers are informed of each other
  * Dropped publishers/subscribers are identified and any lingering connections to them are erased

The connections created by the `Domain` need to be semantically different from the connections formed by directly subscribing to a publisher.
For one thing, Domain-created connections need to be reference-counted: If a subscriber is subscribed to multiple topics and a publisher is publishing to more than one of those topics, then when the subscriber unsubscribes from one of the topics it should not unsubscribe from the publisher.
The subscriber's connection to the publisher should only be broken when both conditions are met:
1. Any topics subscriptions that connected the publisher and subscriber are dropped
2. Any direct connection that was made between the publisher and subscriber is dropped

### Additional thoughts

To facilitate cleanup of spontaneously dropped connections (e.g. a Publisher or Subscriber component is suddenly removed from an entity) it might be necessary to automatically insert a secondary component on each Publisher/Subscriber entity that keeps track of what its last set of connections were before the component was removed.
Otherwise each time a Publisher/Subscriber is removed from an entity an O(N) operation will be needed to cleanup its connections.
It may be possible for [relations](https://github.com/bevyengine/bevy/issues/3742) :rainbow: to help with this.

We may want to consider having a `Subscription` object that acts like a reference counter to keep direct publisher<->subscriber connections alive.
Otherwise there could be conflicts where one system tells a subscriber to disconnect from a publisher while another system still wants the connection to exist.

## Drawbacks

## Rationale and alternatives

### One message at a time
In the current design of Bevy's `Events`, systems are given access to the entire `Event` queue whereas this proposal only gives the callback system access to one `&Message` at a time.
This is done to reflect the following information:
1. The callback will clearly only be triggered when at least one new message has arrived, so there is no need to question whether the system will be triggered with an empty message queue.
2. Users do not have to be concerned about how the message queue is implemented or that a message queue even exists at all.

### Callback driven

Being driven by callbacks provides a few helpful benefits:
* The user does not need to think of where to place the responding system within their application stages
* The user can copy runtime data into their callback which wouldn't be available at compile time
* Callbacks offer a simple way to turn reactions on/off by toggling subscriptions

However, if the user has no control over when the callbacks are processed it may be harder for them to synchronize the overall behavior of their application.
Ideally users should implement their applications to not be sensitive to callback ordering, but that might not be a realistic requirement to impose.
Maybe subscriptions should also be able to specify what stage their callback should be processed in.
This concern will also likely be affected by the upcoming stageless systems design.

## Prior art

As mentioned in the motivation section, [pub-sub patterns](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) are common throughout middlewares to support inter-process and network communication.

The concept is also associated with a more general pattern of "reactive programming", which can be seen in frameworks like [ReactiveX](https://reactivex.io/).

## Unresolved questions

### How to automatically operate on publishers/subscribers with different message types
The `bevy_topics` plugin somehow needs to create systems that operate on `Publisher<T>` and `Subscriber<T>` components where `T` may vary.

One option would be to require users to register each type they want to support messages for when adding the `bevy_topics` plugin.
I would prefer to avoid this because requirement because I think users would be confused when they create a connection between entities with `Publisher<Foo>` and `Subscriber<Foo>` components but the connection quietly fails because they forgot to register `Foo`.

The only other way I can think of making this work would be to use something like [trait queries](https://github.com/bevyengine/rfcs/pull/39) where `Publisher<Foo>` and `Subscriber<Foo>` implement traits that allow them to be discovered by the `bevy_topics` system.

### Topic names
Should topic names use `String` or should they be more like `StageLabel` where they can use any data structure with Hash and Eq traits?

### Callback timing
In what stage should subscription callbacks be processed? Is that something that users should be able to set, and if so how? Should that be customizable per subscription?

## Future possibilities

### Publisher `Description`

In addition to the message type field, allow publishers to provide a "description" value that allows them to describe themselves.
This may help subscribers filter publishers whose outputs are not relevant for them.

Using the description, the `Domain` discovery feature should allow subscribers to provide a callback to filter out publishers for the topics that the subscriber is trying to subscribe to.

To allow descriptions to be generic without being coupled to the `Publisher<T>` type, we could use the following pattern:

```rust
impl<T> Publisher<T> {
    /// Add a description of any kind to this publisher
    pub fn add_description<Description>(&mut self, description: Description);

    /// View a description of any kind for this publisher, if the publisher
    /// provides it.
    pub fn get_description<Description>(&self) -> Option<&Description>;
}
```

### Connection events

Allow a subscriber to define an `on_connect` and `on_disconnect` callback that will be triggered when successfully connecting or disconnecting with a publisher. If we implement the `Description` generic parameter mentioned above, then that `Description` value can be passed into these callbacks.

### History

Allow publishers to be configured to maintain a history that subscribers can choose to review. E.g.:

```rust
impl<T> Publisher<T> {
    /// Keep the last n messages that were published so that new subscribers can
    /// look through them. By default this is 0.
    pub fn keep_last(self, n: usize) -> Self;

    /// Keep the all messages that are ever published so that new subscribers
    /// can look through them.
    pub fn keep_all(self) -> Self;
}

/// Returned by `Subscribe::subscribe`
impl Subscription {
    /// Receives the last n messages that were published before this subscriber
    /// subscribed, if and only if the publisher has been maintaining its history.
    pub fn receive_last(self, n: usize) -> Self;

    /// Receives all available messages in the publisher's history.
    pub fn receive_all(self) -> Self;
}
```

This may require adding a second message queue to the `Publisher<T>` struct.

### Parallel Publishing

In the current implementation proposal, a system needs to obtain a mutable reference to a publisher.
This means at most one system can be publishing for a certain type of publisher at a time (without needing complicated system-deconflicting strategies).
It would be good to have ways for systems to publish with the same publisher in parallel. Some possibilities:

* Custom command that flushes messages into publishers
  * Advantage: commands are a familiar way of marshalling parallel requests in Bevy
  * Disadvantage: it may feel clunky for users because they would need to explicitly communicate the publisher entity to the command
* Additional method `Publisher<T>::parallel_publish(&self, msg: T)` that uses a mutex to protect access to the message queue
  * Advantage: ergonomic and easy to understand
  * Disadvantage: introducing mutexes could mean a noticeable performance hit

### Custom message buffer / pre-allocated size

It would be good if it were possible for users to specify how they want the publisher to store messages.
Even if we lock users into `SmallVec` (which is probably a good choice for +90% of use cases), users may still want to specify the default pre-allocation size based on their knowledge of what publishing patterns they can expect.

Since the container or size information would need to be part of the Publisher's type information as a generic parameter, the publisher would become harder to query, and the queries would become sensitive to the choice of container.
To work nicely, this capability would require [trait queries](https://github.com/bevyengine/rfcs/pull/39) where `Publisher<T>` is a trait instead of a struct.

### Stream operations

It may be worth considering whether/how bevy can facilitate transformations and operations on data streams the way [ReactiveX](https://reactivex.io/documentation/operators.html) does.

As a simple example, consider an operator that zips two topics/publishers `Publisher<U>`, `Publisher<V>` together to effectively provide `Publisher<(U, V)>` that can be subscribed to. The zipped publisher would publish a new message each time a new message has arrived for both `U` and `V`.

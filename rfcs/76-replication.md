# Feature Name: Replication

## Summary

A simple way to replicate states between Bevy instances.

## Motivation

At the moment, Bevy doesn't have its own networking. There are external crates that solve this problem to some extent, but having a solution in Bevy would ensure that it will be maintained in lockstep with Bevy. As networking is a core feature of every newer game engine, having a direct solution in Bevy would allow plugin authors to also consider a replication-friendly design.

The goal is to be able to replicate resources, events, and components peer-to-peer, with options to configure how and when replication happens, as generally as possible.

## User-facing explanation

Replication allows for transferring state between multiple instances, either Server-Client, or P2P. And is split up into the actual replication part and transport.

### Replication

For replication to work, every resource, event or component has to be registered for sending or receiving. Which would also allow for some configuration, like relevancy or which serializer to use.
```rs
app.add_plugins(ReplicationPlugin);

app.recv_resource::<Resource>();
app.send_resource::<Resource>();
app.recv_event::<Event>();
app.send_event::<Event>();
app.recv_component::<Component>();
app.send_component::<Component>();
```
recv requires the DeserializeOwned trait, and send the Serialize trait.

### Transport

The transport part is responsible for the underlying protocol and how the data is transmitted and received.
There is a server and client plugin; they don't have any impact on replication itself.
```rs
app.add_plugins(ServerPlugin {
    address: todo!(),
});
app.add_plugins(ClientPlugin {
})
```

Except for the part that a server listens and a client connects, there are no other differences between client and server, which also allows for all sorts of topologies.

A connection is an entity with a connection component, so that the lifetime of the connection component is coupled with the lifetime of the actual connection. This has the additional effect of ownership, so that an entity is owned by the connection, including their children.

### Events

As there are no RPCs, events are the closest thing to achive similar results. Events can either be send normally, or using the `Directed<Event>` wrapper, which allows for event routing.
When events are received the event is wrapped inside an `Received<Event>` wrapper.

## Implementation strategy

A new crate gets added called bevy_net, or bevy_replication, which contains a replication plugin and could contain multiple transport plugins for every type of transport, and server and client are also two distinguished plugins.

The replication plugin contains the methods and interfaces for transport to implement and the systems to spawn new connections, send/queue, and handle updates.

To register resources, events, and components, a resource map is created containing hashes of the corresponding types and their serializers and deserializers. Where the deserializers also have the job of handling the state, like inserting resources, sending events, or inserting components,.

It's theoretically possible to also use Reflect, but it would be substantially slower.

The transport plugin interfaces with the replication plugin to handle the queues and send new packets into the queue.

## Drawbacks

- Additional burden of maintaining.
- "No one solution fits all"

## Rationale and alternatives

Keeping things as-is and recommending external crates.

As stated above, one major benefit of having an in-house solution is that crate authors writing plugins for Bevy may also consider how replication would interact with their plugin. There is less incentive to add support for external plugins.

No replication at all and just adding a way of sending and receiving packets would also be a solution, but then a user could also just write the boilerplate for creating a server and client, which are like 200 LoC, and it also might fit their needs better.

## Prior art

[bevy_net_valaphee](https://github.com/valaphee/bevy_net)
PoC, which adheres to this RFC and is sufficiently covered with tests, could also be upstreamed.

[bevy_replicon](https://github.com/projectharmonia/bevy_replicon)
Similar to my implementation also supporting different transports. But no resource replication and separate client and server logic.

[lightyear](https://github.com/cBournhonesque/lightyear)
Also supports different transports.

## Unresolved questions

- How to implement scoping and prioritization.
  - Unreal Engine, for example, has a way to specify if a property should be replicated (Initial, Owner, Owner+Initial) and what priority an entity has (distance-based).
- How to handle reliability and ordering.
  - For out-of-order packets, theoretically only the newest state for a resource or component+entity pair has to be applied, ignoring all datagrams that are older.<br>
    This also means that the state might not be the same as on the server, for example. (Maybe configurable?)
  - Events should never be accepted out-of-order, but they can be unreliable for playing animations.

## Future possibilities

- Builtin rollback and/or lock-step.

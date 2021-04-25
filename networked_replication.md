# Feature Name: `networked-replication`

## Summary

This RFC describes an implementation of engine features for developing networked games. Its main focus is replication and its key interest is providing these systems transparently (i.e. minimal, if any, networking boilerplate).

## Motivation

Networking is unequivocally the most lacking feature in all general-purpose game engines.

Bevy has an opportunity to be among the first open game engines to provide a truly plug-and-play networking API. This RFC focuses on *replication*, the part of networking that deals with simulation behavior and the only one that directly involves the ECS.

> The goal of replication is to ensure that all of the players in the game have a consistent model of the game state. Replication is the absolute minimum problem which all networked games have to solve in order to be functional, and all other problems in networked games ultimately follow from it. - [Mikola Lysenko](https://0fps.net/2014/02/10/replication-in-networked-games-overview-part-1/)

Most engines provide "low level" connectivity—virtual connections, optionally reliable UDP channels, rooms—and stop there. Those are not very useful to developers without "high level" replication features—prediction, reconciliation, lag compensation, interest management, etc.

Among Godot, Unity, and Unreal, only Unreal provides [some](https://youtu.be/JOJP0CvpB8w) [features](https://www.unrealengine.com/en-US/tech-blog/replication-graph-overview-and-proper-replication-methods) built-in.

IMO the broader absence of these systems leads many to conclude that every multiplayer game must need its own unique solution. This is not true. While the exact replication "strategy" depends of the game, all of them—lockstep, rollback, client-side prediction with server reconciliation, etc.—pull from the same bag of tricks. Their differences can be captured with simple configuration options. Really, only *massive* multiplayer games require custom solutions.

In general, I think that building *up* from the socket layer leads to the wrong intuition about what "networking" is. If you start from what the simulation needs and design *down*, defining the problem is easier and routing becomes an implementation detail.

What I hope to explore in this RFC is:
- How do game design and networking constrain each other?
- How do these constraints affect user decisions?
- What should developing a networked game look like in Bevy?

## Guide-level explanation

[Link to my explanation of important replication concepts.](../main/replication_concepts.md)

> Please treat all terms like determinism, state transfer, snapshots, and eventual consistency as placeholders. We could easily label them differently.

Bevy aims to make developing networked games as simple as possible. There isn't one replication strategy that works best for every game, but Bevy provides those it has under one API.

First think about your game and consider which form of replication might fit best. Players can *either* send their inputs to each other (or through a relay) and independently and deterministically simulate the game *or* they can send their inputs to a single machine (the server) who simulates the game and sends back updated game state.

> Honestly, Bevy could put something like a questionnaire in the docs. Genre and player count pretty much choose the replication strategy for you.

Next, determine which components and systems affect the global simulation state and tag them accordingly. Usually adding `#[derive(Replicate)]` to all replicable components is enough. You can additionally decorate gameplay logic and systems with `#[client]` or `#[server]` for conditional compilation.

Lastly, add these simulation systems to the `NetworkFixedUpdate` app state. Bevy will take care of all state rollback, serialization, and compression internally. Other than that, you're free to write your game as if it were local multiplayer. 

> This guide is pretty lazy lol, but that's the gist of it.

### Example: "Networked" Components

```rust
#[derive(Replicate)]
struct NetworkTransform {
    #[replicate(precision=0.001)]
    translation: Vec3,
    #[replicate(precision=0.01)]
    rotation: Quat,
    #[replicate(precision=0.1)]
    scale: Vec3,
}

#[derive(Replicate)]
struct Health {
    #[replicate(precision=0.1, range=(0.0, 100.0))]
    hp: f32,
}
```

### Example: "Networked" Systems
```rust
// No networking boilerplate. Just swap components.
// Same code runs on client and server.
fn check_zero_health(mut query: Query<(&Health, &mut NetworkTransform)>){
    for (health, mut transform) in query.iter_mut() {
        if health.hp <= 0.0 {
            transform.translation = Vec3::ZERO;
        }
    }
}
```

### Example: "Networked" App
```rust
#[derive(Debug, Hash, PartialEq, Eq, Clone, SystemLabel)]
pub enum NetworkLabel {
    Input,
    Gameplay,
    Physics,
    LagComp
}

fn main() {
    App::build()
        .add_plugins(DefaultPlugins)
        .add_plugins(NetworkPlugins)

        // Add the fixed update state.
        .add_state(AppState::NetworkFixedUpdate)
            .run_criteria(FixedTimestep::step(1.0 / 60.0))

        // Add our game systems:
        .add_system_set(
            SystemSet::new()
                .label(NetworkLabel::Input)
                .before(NetworkLabel::Gameplay)
                .with_system(sample_inputs.system())
        )
        .add_system_set(
            SystemSet::on_update(AppState::NetworkFixedUpdate)
                .label(NetworkLabel::Gameplay)
                .before(NetworkLabel::Physics)
                .with_system(check_zero_health.system())
                // ... Most user systems would go here.
        )
        #[server]
        .add_system_set(
            SystemSet::on_update(AppState::NetworkFixedUpdate)
                .label(NetworkLabel::LagComp)
                .after(NetworkLabel::Physics)
                // ...
        )
        // ...
        .run();
}
```

### Example Configuration Options
```
players: 32
max networked entities: 1024
replication strategy: snapshots
  mode: listen server
  simulation tick rate: 60Hz
  client send interval: 1 tick
  server send interval: 2 ticks
  client input delay: 0
  server input delay: 2 ticks
  prediction: true
    rollback window: 250ms
    min interpolation delay: 32ms
    lag compensation: true
      compensation window: 200ms
```

## Reference-level explanation

[Link to more in-depth implementation details.](../main/implementation_details.md)

### Macros
- `#[derive(Replicate)]` for identification and serialization; also adds `[repr(C)]`
- `#[replicate(precision=?)]` for quantization
- `#[replicate(range=(?, ?))]` for range compression
- `#[client]` and `#[server]` for conditional compilation

### Saving and Restoring Game State

Requirements

- Replicable components should only be mutated inside `NetworkFixedUpdate`.
- Networked entities will have a global `NetworkID` component at minimum.
- Clients shouldn't try to "network" local entities through the addition of replicable components since those entities do not exist on the server. 
- `World` should reserve an `Entity` ID range and track metadata for it separately.
  - [Sub-worlds](https://github.com/bevyengine/rfcs/pull/16) seem like a potential candidate for this.

Saving
- At the end of each fixed update, server iterates `Changed<T>` and `Removed<T>` for all replicable components and duplicates them to an isolated collection of `SparseSet<T>`.
  - You could pass this "read-only" copy to another thread to do the remaining work.
  - Tables would be rebuilt when restoring.

Preparing Packets
- Snapshots (full state updates) will use delta compression.
  - Server keeps a ring buffer of patches for the last `N` snapshots.
  - Server computes the latest patch by xor'ing the copy and the latest changes (before applying them) and pushes it into the ring buffer. The servers also updates the earlier patches by xor'ing them with the latest patch. (The xor'ing is basically a pre-compression step that produces long zero chains with high probability.)
  - Server compresses whichever patches clients need and hands them off to the protocol layer. (The same patch can be sent to multiple clients, so it scales pretty well.)

- Eventual consistency (partial state updates) will use interest management. 
  - Entities accrue send priority over time. Maybe we can use the magnitude of component changes as the base amount to accrue. 
  - Server runs users-defined rules for gameplay relevance.
  - Server runs collision detection to prioritize physical entities inside each client's area of interest.
  - Server writes the payload for each client and hands them off to the protocol layer.

Restoring
- At the beginning of each fixed update, the client decompresses the received update and writes its changes to the appropriate `SparseSet<T>` collection (several will be buffered).
- Client then uses this updated collection to write the prediction copy that has all the tables and non-replicable components.

### NetworkFixedUpdate
Clients
1. Iterate received server updates.
2. Update simulation and interpolation timescales.
3. Sample inputs and push them to send buffer.
4. Rollback and re-sim *if* a new update was received.
5. Simulate predicted tick.

Server
1. Iterate received client inputs.
2. Sample buffered inputs.
3. Simulate authoritative tick.
4. Duplicate state changes to copy.
5. Push client updates to send buffer.

Everything aside from the simulation steps could be auto-generated.

### Network Modes
| Mode | Playable? | Authoritative? | Open to connections? |
| :--- | :---: | :---: | :---: |
| Client | ✓ | ✗ | ✗ | 
| Standalone | ✓ | ✓ | ✗ | 
| Listen Server | ✓ | ✓ | ✓ | 
| Dedicated Server | ✗ | ✓ | ✓ |
| Relay | ✗ | ✗ | ✓ |

- Listen servers have client and server instances on the same machine. Standalone is a listen server configured with a dummy socket and closed to connections.
- Relays are for managing deterministic and client-authoritative games. They can do "clock" synchronization, input validation, interest management, etc. Just no simulation.

## Drawbacks
- Lots of potentially cursed macro magic.
- Direct writes to `World`.
- Seemingly limited to components that implement `Clone` and `Serialize`.

## Rationale and alternatives
> Why is this design the best in the space of possible designs?

Networking is a widely misunderstood problem domain. The proposed implementation should suffice for most games while minimizing design friction—users need only annotate gameplay-related components and systems, put those systems in `NetworkFixedUpdate`, and configure some settings.

> What other designs have been considered and what is the rationale for not choosing them?

Replication always boils down to sending inputs or state, so the space of alternative designs includes different choices for the end-user interface and different implementations of save/restore functions.

Frankly, given the abundance of confusion surrounding networking, polluting the API with "networked" variants of structs and systems (aside from `Transform`, `Rigidbody`, etc.) would just make life harder for everybody, both game developers and Bevy maintainers.

People who want to make multiplayer games want to focus on designing their game and not worry about how to implement prediction, how to serialize their game, how to keep packets under MTU, etc. All of that should just work. I think the ease of macro annotations is worth any increase in compile times when networking features are enabled.

> What is the impact of not doing this?

It'll only grow more difficult to add these features as time goes on. Take Unity for example. Its built-in features are too non-deterministic and its only working solutions for state transfer are paid third-party assets. Thus far, said assets cannot integrate deeply enough to be transparent (at least not without substituting parts of the engine).

> Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

I strongly doubt that fast, efficient, and transparent replication features can be implemented without directly manipulating a `World` and its component storages.

## Unresolved questions
- What components and resources can't be serialized?
- Is there a better way to isolate replicable and non-replicable entities?
- Can we provide lints for undefined behavior like mutating networked state outside of `NetworkFixedUpdate`? 
- How should UI widgets interact with networked state? Exclusively poll verified data?
- How should we deal with correcting mispredicted events and FX? 
- Does rolling back break existing change detection or events?
- Can we replicate animations exactly without explicitly sending animation parameters?
- When sending partial state updates, how should we deal with weird stuff like there being references to entities that haven't been spawned or have been destroyed?

## Future possibilities
- With some tool to visualize game state diffs, these replication systems could help detect non-determinism in other parts of the engine. 
- Much like how Unreal has Fortnite, Bevy could have an official (or curated) collection of multiplayer samples to dogfood these features.
- Bevy's future editor could automate most of the configuration and annotation.
- Beyond replication, Bevy need only provide one good default for protocol and I/O for the sake of completeness. I recommend dividing crates at least to the extent shown below to make it easy for developers to swap the low-level stuff with [whatever](https://partner.steamgames.com/doc/features/multiplayer) [alternatives](https://developer.microsoft.com/en-us/games/solutions/multiplayer/) [they](https://dev.epicgames.com/docs/services/en-US/Overview/index.html) [want](https://docs.aws.amazon.com/gamelift/latest/developerguide/gamelift-intro.html). Replication addresses all the underlying ECS interop, so it should be settled first.

    **replication** ← this RFC
    - save and restore
    - prediction
    - serialization and compression
    - interest management, prioritization, level-of-detail
    - smooth rendering
    - lag compensation
    - statistics

    **protocol**
    - (N)ACKs and reliability
    - channels
    - connection authentication and management
    - encryption
    - statistics

    **I/O**
    - send, recv, poll, etc.

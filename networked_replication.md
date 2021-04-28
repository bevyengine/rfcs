# Feature Name: `networked-replication`

## Summary

This RFC proposes an implementation of engine features for developing networked games. It abstracts away the (mostly irrelevant) low-level transport details to focus on high-level *replication* features, with key interest in providing them transparently (i.e. minimal, if any, networking boilerplate).

## Motivation

Networking is unequivocally the most lacking feature in all general-purpose game engines. 

While most engines provide low-level connectivity—virtual connections, optionally reliable UDP channels, rooms—almost none of them ([except][1] [Unreal][2]) provide high-level *replication* features like prediction, interest management, or lag compensation, which are necessary for most networked multiplayer games.

This broad absence of first-class replication features stifles creative ambition and feeds into an idea that every multiplayer game needs its own unique implementation. Certainly, there are idiomatic "strategies" for different genres, but all of them—lockstep, rollback, client-side prediction with server reconciliation—pull from the same bag of tricks. Their differences can be captured in a short list of configuration options. Really, only *massive* multiplayer games require custom solutions.

Bevy's ECS opens up the possibility of providing a near-seamless, generalized networking API.

What I hope to explore in this RFC is:
- What game design choices and constraints does networking add?
- How does ECS make networking easier to implement?
- What should developing a networked multiplayer game in Bevy look like?

## User-facing Explanation

[Link to my explanation of important replication concepts.](../main/replication_concepts.md)

Bevy's aim here is to make writing local and networked multiplayer games indistinguishable, with minimal added boilerplate. Having an exact simulation timeline simplifies this problem, thus the core of this unified approach is a fixed timestep—`NetworkFixedUpdate`.

As a user, you only have to annotate your gameplay-related components and systems, add those systems to `NetworkFixedUpdate` (currently would be an `AppState`), and configure a few simulation settings to get up and running. That's it! Bevy will transparently handle separating, reconciling, serializing, and compressing the networked state for you. (Those systems can be exposed for advanced users, but non-interested users need not concern themselves.)

> Game design should (mostly) drive networking choices. Future documentation could feature a questionnaire to guide users to the correct configuration options for their game. Genre and player count are generally enough to decide.

The core primitive here is the `Replicate` trait. All instances of components and resources that implement this trait will be automatically detected and synchronized over the network. Simply adding a `#[derive(Replicate)]` should be enough in most cases. 

```rust
#[derive(Replicate)]
struct Transform {
    #[replicate(precision=0.001)]
    translation: Vec3,
    #[replicate(precision=0.01)]
    rotation: Quat,
    #[replicate(precision=0.1)]
    scale: Vec3,
}

#[derive(Replicate)]
struct Health {
    #[replicate(range=(0, 1000))]
    hp: u32,
}
```
By default, both client and server will run every system you add to `NetworkFixedUpdate`. If you want systems or code snippets to run exclusively on one or the other, you can annotate them with `#[client]` or `#[server]` for the compiler.

```rust
#[server]
fn ball_movement_system(
    mut ball_query: Query<(&Ball, &mut Transform)>)
{
    for (ball, mut transform) in ball_query.iter_mut() {
        transform.translation += ball.velocity * FIXED_TIMESTEP;
    }
}
```

For more nuanced runtime cases—say, an expensive movement system that should only process the local player entity on clients—you can use the `Predicted<T>` query filter. If you need an explicit request or notification, you can use `Message` variants.
```rust
fn update_player_velocity(
    mut q: Query<(&Player, &mut Rigidbody)>)
{
    for (player, mut rigidbody) in q.iter_mut() {
        // DerefMut flags these rigidbodies as predicted on the client.
        *rigidbody.velocity = player.aim_direction * player.movement_speed * FIXED_TIMESTEP;
    }
}

fn expensive_physics_calculation(
    mut q: Query<(&mut Rigidbody), Predicted<Rigidbody>>)
{
    for rigidbody in q.iter_mut() {
        // Do stuff with only the predicted rigidbodies...
    }
}
```

```
TODO: Message Example
```

Bevy can configure an `App` to operate in several different network modes.

| Mode | Playable? | Authoritative? | Open to connections? |
| :--- | :---: | :---: | :---: |
| Client | ✓ | ✗ | ✗ | 
| Standalone | ✓ | ✓ | ✗ | 
| Listen Server | ✓ | ✓ | ✓ | 
| Dedicated Server | ✗ | ✓ | ✓ |
| Relay | ✗ | ✗ | ✓ |

<br>

```rust
// TODO: Example App configuration.
```

## Implementation Strategy
[Link to more in-depth implementation details (more of an idea dump atm).](../main/implementation_details.md)
  
### What is required?
- In order for servers to send state to clients, `ComponentId` should be stable.
- `World` should be able to reserve an `Entity` ID range, with separate metadata.
  - If merged, [#16](https://github.com/bevyengine/rfcs/pull/16) can probably be used to handle this cleanly.
- The ECS scheduler should support nested loops.
  - (I'm pretty sure this isn't an actual blocker, but the workaround feels a little hacky.)
- Replicable components should only be mutated inside `NetworkFixedUpdate`.
- Networked entities should have a `NetworkID` component at minimum. Could be auto-added.
- Adding replicable components to non-networked entities should be avoided unless client-authoritative.

### The Replicate Trait
```rust
// TODO
impl Replicate for T {
    ...
}
```

### Specialized Change Detection
```rust
// TODO
// Predicted<T>
// Confirmed<T>
// Cancelled<T>
// Added<T> and Removed<T> variants for each also
```

### Rollback via Run Criteria
```rust 
// TODO
```

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
### Saving Game State
- At the end of each fixed update, server iterates `Changed<T>` and `Removed<T>` for all replicable components and duplicates them to an isolated collection of `SparseSet<T>`.
  - You could pass this "read-only" copy to another thread to do the remaining work.
  - Tables would be rebuilt when restoring.

### Preparing Server Packets
- Snapshots (full state updates) will use delta compression.
  - Server keeps a ring buffer of patches for the last `N` snapshots.
  - Server computes the latest patch by xor'ing the copy and the latest changes (before applying them) and pushes it into the ring buffer. The servers also updates the earlier patches by xor'ing them with the latest patch. (The xor'ing is basically a pre-compression step that produces long zero chains with high probability.)
  - Server compresses whichever patches clients need and hands them off to the protocol layer. (The same patch can be sent to multiple clients, so it scales pretty well.)

- Eventual consistency (partial state updates) will use interest management. 
  - Entities accrue send priority over time. Maybe we can use the magnitude of component changes as the base amount to accrue. 
  - Server runs users-defined rules for gameplay relevance.
  - Server runs collision detection to prioritize physical entities inside each client's area of interest.
  - Server writes the payload for each client and hands them off to the protocol layer.

### Restoring Game State
- At the beginning of each fixed update, the client decompresses the received update and writes its changes to the appropriate `SparseSet<T>` collection (several will be buffered).
- Client then uses this updated collection to write the prediction copy that has all the tables and non-replicable components.

## Drawbacks
- Lots of potentially cursed macro magic.
- Direct writes to `World`.
- Seemingly limited to components that implement `Clone` and `Serialize`.

## Rationale and Alternatives

### Why *this* design?
Networking is a widely misunderstood problem domain. The proposed implementation should suffice for most games while minimizing design friction—users need only annotate gameplay-related components and systems, put those systems in `NetworkFixedUpdate`, and configure some settings.

Polluting the API with "networked" variants of structs and systems (aside from `Transform`, `Rigidbody`, etc.) would just make life harder for everybody, both game developers and Bevy maintainers. IMO the ease of macro annotations is worth any increase in compile times when networking features are enabled.

### Why should Bevy provide this?
People who want to make multiplayer games want to focus on designing their game and not worry about how to implement prediction, how to serialize their game, how to keep packets under MTU, etc. Having these come built-in would be a huge selling point. 

### Why not wait until Bevy is more mature?
It'll only grow more difficult to add these features as time goes on. Take Unity for example. Its built-in features are too non-deterministic and its only working solutions for state transfer are paid third-party assets. Thus far, said assets cannot integrate deeply enough to be transparent (at least not without substituting parts of the engine).

### Why does this need to involve `bevy_ecs`?
I strongly doubt that fast, efficient, and transparent replication features can be implemented without directly manipulating a `World` and its component storages.

## Unresolved Questions
- What components and resources can't be serialized?
- Is there a better way to isolate replicable and non-replicable entities?
- Can we provide lints for undefined behavior like mutating networked state outside of `NetworkFixedUpdate`? 
- Does rolling back break existing change detection or events?
- When sending partial state updates, how should we deal with weird stuff like there being references to entities that haven't been spawned or have been destroyed?
- How should UI widgets interact with networked state? Exclusively poll verified data?
- How should we deal with correcting mispredicted events and FX? 
- Can we replicate animations exactly without explicitly sending animation parameters?

## Future Possibilities

- With some tools to visualize game state diffs, these replication systems could help detect non-determinism in other parts of the engine.
- Much like how Unreal has Fortnite, Bevy could have an official (or curated) collection of multiplayer samples to dogfood these features.
- Bevy's future editor could automate most of the configuration and annotation.
- Replication addresses all the underlying ECS interop, so it should be settled first. But beyond replication, Bevy need only provide one good default for protocol and I/O for the sake of completeness. I recommend dividing crates at least to the extent shown below to make it easy for developers to swap the low-level transport with [whatever][3] [alternatives][4] [they][5] [want][7].

| `bevy_net_replication` | `bevy_net_protocol` | `bevy_net_io` | 
| -- | -- | -- |
| <ul><li>save and restore</li><li>prediction</li><li>serialization</li><li>interest management</li><li>visual error correction</li><li>lag compensation</li><li>statistics (high-level)</li></ul> | <ul><li>(N)ACKs</li><li>reliability</li><li>virtual connections</li><li>channels</li><li>encryption</li><li>statistics (low-level)</li></ul> | <ul><li>send</li><li>recv</li><li>poll</li></ul> |


[1]: https://youtu.be/JOJP0CvpB8w "Unreal Networking Features"
[2]: https://www.unrealengine.com/en-US/tech-blog/replication-graph-overview-and-proper-replication-methods "Unreal Replication Graph Plugin"
[3]: https://github.com/quinn-rs/quinn
[4]: https://partner.steamgames.com/doc/features/multiplayer
[5]: https://developer.microsoft.com/en-us/games/solutions/multiplayer/
[6]: https://dev.epicgames.com/docs/services/en-US/Overview/index.html
[7]: https://docs.aws.amazon.com/gamelift/latest/developerguide/gamelift-intro.html
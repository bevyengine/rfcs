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

[Recommended reading on replication concepts.](../main/replication_concepts.md)

Bevy's aim here is to make writing local and networked multiplayer games indistinguishable, with minimal added boilerplate. Having an exact simulation timeline simplifies this problem, thus the core of this unified approach is a fixed timestep—`NetworkFixedUpdate`.

As a user, you only have to annotate your gameplay-related components and systems, add those systems to `NetworkFixedUpdate`, and configure a few simulation settings to get up and running. That's it! Bevy will transparently handle separating, reconciling, serializing, and compressing the networked state for you. (Those systems can be exposed for advanced users, but non-interested users need not concern themselves.)

> Game design should (mostly) drive networking choices. Future documentation could feature a questionnaire to guide users to the correct configuration options for their game. Genre and player count are generally enough to decide.

The core primitive here is the `Replicate` trait. All instances of components and resources that implement this trait will be automatically registered and synchronized over the network. Simply adding a `#[derive(Replicate)]` should be enough in most cases.

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

```plaintext
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

We'll also need a mode similar to listen server for deterministic peers.

```plaintext
TODO: Example App configuration.
```

## Implementation Strategy
  
[See here for a big idea dump.](../main/implementation_details.md) (Hopefully, I can clean this up later.)

## Drawbacks

- Serialization strategy is `unsafe` (might be possible to do it entirely with safe Rust, idk).
- Macros might be gnarly.
- At first, only POD components and resources will be supported. DST support will come later.

## Rationale and Alternatives

### Why *this* design?

Networking is a widely misunderstood problem domain. The proposed implementation should suffice for most games while minimizing design friction—users need only annotate gameplay-related components and systems, put those systems in `NetworkFixedUpdate`, and configure some settings.

Polluting the API with "networked" variants of structs and systems (aside from `Transform`, `Rigidbody`, etc.) would just make life harder for everybody, both game developers and Bevy maintainers. IMO the ease of macro annotations is worth any increase in compile times when networking features are enabled.

### Why should Bevy provide this?

People who want to make multiplayer games want to focus on designing their game and not worry about how to implement prediction, how to serialize their game, how to keep packets under MTU, etc. Having these come built-in would be a huge selling point.

### Why not wait until Bevy is more mature?

It'll only grow more difficult to add these features as time goes on. Take Unity for example. Its built-in features are too non-deterministic and its only working solutions for state transfer are paid third-party assets. Thus far, said assets cannot integrate deeply enough to be transparent (at least not without substituting parts of the engine).

### Why does this need to involve `bevy_ecs`?

For better encapsulation, I'd prefer if multiple world functionality and nested loops were standard ECS features. Nesting an outer fixed timestep loop and an inner rollback loop doesn't seem possible without a custom stage or scheduler right now.

## Unresolved Questions

- Can we provide lints for undefined behavior like mutating networked state outside of `NetworkFixedUpdate`?
- ~~Will rollbacks break change detection?~~ As long as we're careful to update the appropriate change ticks, it should be okay.
- Will rollbacks break events?
- ~~When sending interest-managed updates, how should we deal with weird stuff like there being references to entities that haven't been spawned or have been destroyed?~~ I believe this is solved by using generational indexes for the network IDs.
- How should UI widgets interact with networked state? React to events? Exclusively poll verified data?
- How should we handle correcting mispredicted events and FX?
- Can we replicate animations exactly without explicitly sending animation data?

## Future Possibilities

- With some tools to visualize game state diffs, these replication systems could help detect non-determinism in other parts of the engine.
- Much like how Unreal has Fortnite, Bevy could have an official (or curated) collection of multiplayer samples to dogfood these features.
- Bevy's future editor could automate most of the configuration and annotation.
- Replication addresses all the underlying ECS interop, so it should be settled first. But beyond replication, Bevy need only provide one good default for protocol and I/O for the sake of completeness. I recommend dividing crates at least to the extent shown below to make it easy for developers to swap the low-level transport with [whatever][3] [alternatives][4] [they][5] [want][7].

| `bevy::net::replication` | `bevy::net::protocol` | `bevy::net::io` |
| -- | -- | -- |
| <ul><li>save and restore</li><li>prediction</li><li>serialization</li><li>delta compression</li><li>interest management</li><li>visual error correction</li><li>lag compensation</li><li>statistics (high-level)</li></ul> | <ul><li>(N)ACKs</li><li>reliability</li><li>virtual connections</li><li>channels</li><li>encryption</li><li>statistics (low-level)</li></ul> | <ul><li>send</li><li>recv</li><li>poll</li></ul> |

[1]: https://youtu.be/JOJP0CvpB8w "Unreal Networking Features"
[2]: https://www.unrealengine.com/en-US/tech-blog/replication-graph-overview-and-proper-replication-methods "Unreal Replication Graph Plugin"
[3]: https://github.com/quinn-rs/quinn
[4]: https://partner.steamgames.com/doc/features/multiplayer
[5]: https://developer.microsoft.com/en-us/games/solutions/multiplayer/
[6]: https://dev.epicgames.com/docs/services/en-US/Overview/index.html
[7]: https://docs.aws.amazon.com/gamelift/latest/developerguide/gamelift-intro.html

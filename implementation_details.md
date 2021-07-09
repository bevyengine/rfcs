
# Implementation Details

## Requirements

Type identifiers (in some form) need to match between all connected machines. Options so far:

- `TypeId`
  - [debatable stability][14]
  - no additional mapping needed (just reuse the world's)
- ["`StableTypeId`"][13]
  - currently unavailable
  - no additional mapping needed (just reuse the world's)
- `ComponentId`
  - fragile, requires networked components and resources registered first and in a fixed order (for all relevant worlds)
  - no mapping needed
- `unique_type_id`
  - uses ordering described in a `types.toml` file
  - needs mapping between these and `ComponentId`
- `type_uuid`
  - not an index
  - needs mapping between these and `ComponentId`

## Wants
  
- Ability to split-off a range of entities into a [sub-world][12] with its own, separate component storages. This would remove a layer of indirection, bringing back the convenience of the `World` interface.
- Scheduler support for arbitrary cycles in the stage graph (or "stageless" equivalent). I believe this boils down to arranging stages (or labels) in a hierarchical FSM (state chart) or behavior tree.

## Practices users should follow or they'll have UB

- Entities must be spawned (non-)networked. They cannot become (non-)networked.
- Networked entities must be spawned with a "network ID" component at minimum.
- (Non-)networked components and resources should only hold or reference (non-)networked data.
- Networked components should only be mutated inside the fixed update.

## `Connection` != `Player`

I know I've been using the terms "client" and "player" somewhat interchangeably, but `Connection` and `Player` should be separate tokens. There's no reason the engine should limit things to one player per connection. Having `Player` be its own thing makes it easier to do stuff like online splitscreen, temporarily substituting vacancies with bots, etc. Likewise, a `Connection` should be a platform-agnostic handle.

## Storage

IMO a fast, data-agnostic networking solution is impossible without the ability to handle things on the bit-level. Memcpy and integer compression are orders of magnitude faster than deep serialization and DEFLATE. To that end, each snapshot should be a pre-allocated memory arena. The core storage resource would then basically amount to a ring buffer of these arenas.

On the server, this would translate to a ring buffer of deltas (for the last `N` snapshots) along with a full copy of the latest networked state. On the client this would hold a bunch of snapshots.

```plaintext
                     delta ringbuf                      copy of latest
                           v                                  v
[(0^8), (1^8), (2^8), (3^8), (4^8), (5^8), (6^8), (7^8)]     [8]
                                                    ^         
                                               newest delta 
```

This architecture has a lot of advantages. It can be pre-allocated when the resource is initialized and it's the same for all replication modes. I.e. no storage differences between input determinism, full state transfer, or interest-managed state transfer.

These storages can be lazily updated using Bevy's built-in change detection. At the end of every tick, the server zeroes the space for the newest delta, then iterates `Changed<T>` and `Removed<T>`:

- Generating the newest delta by xor'ing the changes with the stored copy.
- Updating the rest of the ring buffer by xor'ing the older deltas with the newest.
- Writing the changes to the stored copy.

Even if change detection became optional, I don't think much speed would be lost if we had to scan two snapshots for bitwise differences.

TODO

- Store/serialize networked data without struct padding so we're not wasting bandwidth.
- Components and resources that allocate on the heap (backed by the arena) may have some issues with the interest management send strategy. First, finding all of an entity's heap allocations is its own problem. Then, writing partial heap information could invalidate existing data on the client.

### Input Determinism

In deterministic games, the server bundles received inputs and re-distributes them back to the clients. Clients generate their own snapshots locally whenever they have a full set inputs for a tick. Only one snapshot is needed. Clients also send checksum values to the server, that the server can use to detect desyncs.

### Full State Transfer

(aka. delta-compressed snapshots)

For delta-compression, the server just compresses whichever deltas clients need using some variant of run-length encoding (currently looking at [Simple8b + RLE][2]). If the compressed payload is too large, the server will split it into fragments. Overall, this is a very lightweight replication method because the server only needs to compress deltas that are going to be sent and the same compressed payload can be sent to any number of clients.

### Interest-Managed State Transfer

(aka. eventual consistency)

Eventual consistency isn't inherently reliant on prioritization and filtering, but they're essential for an optimal player experience.

If we can't send everything, we should prioritize what players want to know. They want live updates on objects that are close or occupy a big chunk of their FOV. They want to know about their teammates or projectiles they've fired, even if those are far away. The server has to make the most of each packet.

Similarly, game designers often want to hide certain information from certain players. Limiting the amount of hidden information that gets leaked and exploited by cheaters is often crucial to a game's long-term health. Battle-royale players, for example, don't need and probably shouldn't even have their opponents' inventory data. In practice, these barriers are never perfect (e.g. *Valorant's* Fog of War not preventing wallhacks), but something is better than nothing.

Anyway, to do all this interest management, the server needs to track some extra metadata.

```rust
struct InterestMetadata {
    changed: Vec<u32>, 
    relevant: Vec<BitSet>,
    lost: Vec<BitSet>,
    priority: Vec<Option<u32>>, 
}
```

Essentially, the server wants to send clients all the data that:

- belongs entities they're interested in AND
- has changed since they were last received AND
- is currently relevant to them (i.e. they're allowed to know)

I'm gonna gloss over how to check if an entity is inside someone's area of interest (AOI). It's just an application of collision detection. You'll create some interest regions for each client and write from entities that fall within them.

I don't see a need for ray, distance, and shape queries where BVH or spatial partitioning structures excel, so I'm looking into doing something like a [sweep-and-prune][9] (SAP) with Morton-encoding. Since SAP is essentially  sorting an array, I imagine it might be faster. Alternatives like grids and potentially visible sets (PVS) can be explored and added later.

Those entities get written in the order of their oldest relevant changes. That way that the most pertinent information gets through if not everything can fit. Entities that the server always wants clients to know about or those that clients themselves always want to know about can be written without going through these checks or bandwidth constraints.

So how is that metadata tracked?

The **changed** field is simply an array of change ticks. We could reuse Bevy's built-in change tracking, but ideally each *word* in the arena would be tracked separately (would enable much better compression). These change ticks serve as the basis for send priority.

The **relevant** field tracks whether or not a component should be sent to a certain client. This is how information is filtered at the component-level. By default, new changes would mark components as relevant for everybody, and then some form of filter rules (maybe [entity relations][10]) could selectively modify those. If a component value is sent, its relevance is reset to `false`.

The **lost** field has bit per change tick, per client. Whenever the server sends a packet, it jots down somewhere which entities were in it and their priorities. Later, if the server is notified that a packet was probably lost, it can pull this info and set the lost bits. If the delta matching the stored priority still exists, the server can use that as a reference to only set a minimal amount of lost bits for an entity. Otherwise, all its lost bits would be set. Similarly, on send the lost bit would be cleared.

Honestly, I believe this is pretty good, but I'm still looking for something that's more accurate while using fewer bits (if possible).

The **priority** field just stores the end result of combining the other metadata. This array gets sorted and the `Some(entity)` are written in that order, until the client's packet is full or they're all written. I think having the server only send undelivered changes and prioritizing the oldest ones is better than assigning entities arbitrary update frequencies.

### Interest Management Edge Cases

Unfortunately, the most generalized strategy comes with its own headaches.

- What should a client do when it misses the first update for a entity? Is it OK to spawn a entity with incomplete information? If not, how does the client know when it's safe?

AFAIK this is only a problem for "kinded" entities that have archetype invariants (aka spawn info). I'm thinking two potential solutions:

1. Have the client spawn new remote entities with any missing components in their invariants set to a default value.
2. Have the server redundantly send the full invariants for a new interesting entity until that information has been delivered once.

I think #2 is the better solution.

TBD, I'm sure there are more of these.

## Replicate Trait

TBD

```rust
pub unsafe trait Replicate {
    fn quantize(&mut self);
}

unsafe impl Replicate for T {}
```

## How to rollback?

There are two loops over the same chain of logic.

```plaintext
The first loop is for re-simulating older ticks.
The second loop is for executing the newly-accumulated ticks.
```

I think a nice solution would be to queue stages/labels using a hierarchical FSM (state chart) or behavior tree. Looping would stop being a special edge case and `ShouldRun` could reduce to a `bool`. The main thing is that transitions cannot be completely determined in the middle of a stage / label. The final decision has to be deferred to the end or there will be conflicts.

## Unconditional Rollbacks

Every article on "rollback netcode" and "client-side prediction and server reconciliation" encourages having clients compare their predicted state to the authoritative state and reconciling *if* they mispredicted, but well... How do you actually detect a mispredict?

AFAIK, there are two ways to do it:

- Iterate both copies and look for the first difference.
- Have both server and client compute a checksum for their copy and have the client compare them.

The first option has an unpredictable speed. The second option requires an ordered scan of the entire networked game state. Checksums *are* worth having for deterministic desync detection, but that can be deferred. The point I'm trying to make is that detecting state differences isn't cheap (especially once DSTs are involved).

Let's consider a simpler default:  

- Always rollback and re-simulate when you receive a new update.

This might seem wasteful, but think about it. If-then is really an anti-pattern that just hides performance problems from you. Mispredictions will exist regardless of this choice, and they're *especially* likely during heavier computations like physics. Having clients always rollback and re-sim makes it easier to profile and optimize your worst-case. It's also more memory-efficient, since clients never need to store old predicted states.

## Time Synchronization

Networked applications have to deal with relativity. Clocks will drift. Some router between you and the game server will randomly go offline. Someone in your house will start streaming Netflix. Et cetera. The slightest change in latency (i.e. distance) between two clocks will cause them to shift out of phase.

So how do two computers even agree on *when* something happened?

It'd be really easy to answer that question if there was an *absolute* time reference. Luckily, we can make one. See, there are [two kinds of time][15]—plain ol' **wall-clock time** and **game time**—and we have complete control over the latter. The basic idea is pretty simple: Use a fixed timestep simulation and number the ticks in order. Doing that gives us a timeline of discrete moments that everyone can share (i.e. Tick 742 is the same in-game moment for everyone).

With this shared timeline strategy, clients have two, mutually exclusive options:

- Try to simulate ticks at the same wall-clock time.
- Try to have their inputs reach the server at same wall-clock time.

When using state transfer, I'd recommend against having clients try to simulate ticks at the same time. To accomodate inputs arriving at different times, the server itself would have to rollback and resimulate or you'd have to change the strategy. For example, [Source engine][3] games (AFAIK) simulate the movement of each player at their individual send rates and *then* simulate the world at the regular tick rate. However, doing things their way makes having lower ping a technical advantage (search "lag compensation" in this [this article][4]), which I assume is the reason why ~~melee is bad~~ trading kills is so rare in Source engine games.

### A Relatively Fixed Timestep

Fixed timesteps are typically implemented as a kind of currency exchange. The time that elapsed since the previous frame is deposited in an accumulator and converted into simulation steps according to the exchange rate (tick rate).

```rust
pub struct Accumulator {
    accum: f64,
    ticks: usize,
}

impl Accumulator {
    pub fn add_time(&mut self, time: f64, timestep: f64) {
        self.accum += time;
        while self.accum >= timestep {
            self.accum -= timestep;
            self.ticks += 1;
        }
    }

    pub fn ticks(&self) -> usize {
        self.ticks
    }

    pub fn overtick_percentage(&self, timestep: f64) -> f64 {
        self.accum / timestep
    }

    pub fn consume_tick(&mut self) -> Option<usize> {
        let remaining = self.ticks.checked_sub(1);
        remaining
    }

    pub fn consume_ticks(&mut self) -> Option<usize> {
        let ticks = if self.ticks > 0 { Some(self.ticks) } else { None };
        self.ticks = 0;
        ticks
    }
}
```

Here's how it's typically used. Notice the time dilation. It's changing the time->tick exchange rate to produce more or fewer simulation steps per unit time. Just so you know, this time dilation should only affect the tick rate. Inside the systems running in the fixed update, you should always use the normal fixed timestep for the value of dt.

```rust
// Determine the exchange rate.
let x = (1.0 * time_dilation);

// Accrue all the time that has elapsed since last frame.
accumulator.add_time(time.delta_seconds(), x * timestep);

for step in 0..accumulator.consume_ticks() {
  /* ... */
}

// Calculate the blend alpha for rendering simulated objects.
let alpha = accumulator.overtick_percentage(x * timestep);
```

Ideally, clients simulate any given tick ahead by just enough so their inputs reach the server right before it does.

One way I've seen people try to do this is to have clients estimate the wall-clock time on the server (using an SNTP handshake or similar) and from that schedule their next tick. That does work, but IMO it's too inaccurate. What we really care about is how much time passes between the server receiving an input and consuming it. That's what we want to control. The server can measure these wait times exactly and include them in the corresponding snapshot headers. Then clients can use those measurements to modify their tick rate and adjust their lead. E.g. if its inputs are arriving too late (too early), a client can briefly simulate more (less) frequently to converge on the correct lead.

```rust
if received_newer_server_update { 
  /* ... updates packet statistics ... */
  // measurements of the input wait time and input arrival delta come from the server
  target_input_wait_time = max(timestep, avg_input_arrival_delta + safety_factor * input_arrival_dispersion)
  
  // I'm negating here because I'm scaling the timestep and not the tick rate.
  // i.e. 110% tick rate => 90% timestep
  error = -(target_input_wait_time - avg_input_wait_time);
}

// This logic executes every tick.

// Anything we hear back from the server is always a round-trip old.
// We want to drive this feedback loop with a more up-to-date estimate
// to avoid overshoot / oscillation. 
// Obviously, it's impossible for the client to know the current wait time.
// But it can fancy a guess by assuming every adjustment it made since
// the latest received update succeeded.
predicted_error = error;
for tick in (recv_tick..curr_tick) {
    predicted_error += ringbuf[tick % ringbuf.len()];
}

// This is basically just a proportional controller.
time_dilation = (predicted_error - min_error) * (max_dilation - min_dilation) / (max_error - min_error);
time_dilation = time_dilation.clamp(min_dilation, max_dilation);

// Store the new adjustment in the ring buffer.
*ringbuf[curr_tick % ringbuf.len()] = time_dilation * timestep;
```

### Snapshot Interpolation

Interpolating received snapshots is very similar. What we're interested in is the remaining time left in the snapshot buffer. You want to always have at least one snapshot ahead of the current "playback" time (so the client always has something to interpolate to).

```rust
if received_newer_server_update {
  /* ... updates packet statistics ... */
  target_interpolation_delay = max(server_send_interval, avg_update_arrival_delta + safety_factor * update_arrival_dispersion);
  time_since_last_update = 0.0;
}

// This logic executes every frame.

// Calculate the current interpolation.
// Network conditions are assumed to be constant between updates.
current_interpolation_delay = last_update_received_time + time_since_last_update - playback_time;

// I'm negating here because I'm scaling time and not frequency.
// i.e. 110% freq => 90% time
error = -(target_interpolation_delay - current_interpolation_delay);
time_dilation = (error - min_error) * (max_dilation - min_dilation) / (max_error - min_error);
time_dilation = time_dilation.clamp(min_dilation, max_dilation);

playback_time += time.delta_seconds() * (1.0 + time_dilation);
time_since_last_update += time.delta_seconds();

// Determine the two snapshots and blend alpha. 
let i = buf.partition_point(|&snapshot| (snapshot.tick as f32 * timestep) < playback_time);
let (from, to, blend) = if i == 0 {
    // Current playback time is behind all buffered snapshots.
    (buf[0].tick, buf[0].tick, 0.0)
} else if i == buffer.len() {
    // Current playback time is ahead of all buffered snapshots.
    // Here, I'm just clamping to the latest, but you could extrapolate instead.
    (buf[-1].tick, buf[-1].tick, 0.0)
} else {
    let a = buf[i-1].tick;
    let b = buf[i].tick;
    let blend = (playback_time - (a as f32 * timestep)) / ((b - a) as f32 * timestep);
    (a, b, blend)
}

// Go forth and (s)lerp.
```

## Predict or Delay?

The higher a client's ping, the more ticks they'll need to resim. Depending on the game, that might be too expensive to support all the clients in your target audience. In those cases, we can trade more input delay for fewer resim ticks. Essentially, there are three meaningful moments in the round-trip of an input:

1. When the inputs are sent.
2. When the true simulation tick happens (conceptually).
3. When resulting update is received.

```plaintext
0 <-+-+-+-+-+-+-+-+-+-+-+-> t
             sim
             / \
            /   \
           /     \
          /       \
         /         \
       send       recv
      |<--   RTT   -->|
```

This gives us a few options for time sync:

1. **No rollback and adaptive input delay**, preferred for games with prediction disabled
2. **Bounded rollback and adaptive input delay**, preferred for games using input determinism with prediction enabled
3. **Unbounded rollback and no/fixed input delay**, preferred for games using state transfer with prediction enabled

"Adaptive input delay" here means "fixed input delay with more as needed."

Method #1 basically tries to ensure packets are always received before they're needed by the simulation. The client will add as much input delay as needed to avoid stalling.

Method #2 is best explained as a sequence of fallbacks. Clients first add a fixed amount of input delay. If that's more than their current RTT, they won't need to rollback. If that isn't enough, the client will rollback, but only up to a limit. If even the combination of fixed input delay and maximum rollback doesn't cover RTT, more input delay will be added to fill the remainder.

Method #3 is preferred for games that use state transfer. Adding input delay would negatively impact the accuracy of server-side lag compensation, so it should almost always be set to zero in those cases. Games that use input determinism might prefer a constant input delay even if it means their game might stutter, unlike method #2.

## Predicted <-> Interpolated

When the server has full authority, clients cannot directly write persistent changes to the authoritative state. However, it's perfectly okay for them to do whatever they want locally. That's all client-side prediction really is—local changes. Clients can just copy the latest authoritative state as a starting point.

We can also shift components between being predicted (extrapolated) and being interpolated. Either could be default. If interpolation is default, entities would reset to interpolated when modified by a server update. Users could then use specialized `Predicted<T>` and `Confirmed<T>` query filters to address the two separately. These can piggyback off of Bevy's built-in reliable change detection.

This means systems predict by default, but users can opt-out with the `Predicted<T>` filter to only process components that have already been mutated by an earlier system. Clients will naturally predict any entities driven by their input and any spawned by their input (until confirmed by the server).

## Predicted FX Events

Sounds and particles need special consideration, since they can be predicted but are also typically handled outside of the fixed update.

We'll need events that can be confirmed or cancelled. The main requirement is tagging them with a unique identifier. Maybe hashing together the tick number, system ID, and entity ID would suffice.

TBD

## Predicted Spawns

This too requires special consideration.

The naive solution is to have clients spawn dummy entities so that when an update that confirms the result arrives, they'll simply destroy the dummy and spawn the true entity. IMO this is a poor solution because it prevents clients from smoothly blending errors in the predicted spawn's rendered transform. Snapping its visuals wouldn't look right.

A better solution is for the server to assign each networked entity a global ID that the spawning client can predict and map to its local instance. There are 3 variants that I know of:

1. Use an incrementing generational index (reuse `Entity`) and fix its upper bits to match the ID of the spawning player.

2. Use PRNGs to generate shared keys (I've seen these dubbed "prediction keys") for pairing local and global IDs. Rather than predict the global ID directly, clients predict the shared keys. Server updates that confirm a predicted entity would include both its global ID and the shared key. Once acknowledged, later updates can include just the global ID. This method is more complicated but does not share the previous method's implicit entity limit.

3. Bake it into the memory layout. If the layout and order of the snapshot storage is identical on all machines, array indexes and relative pointers can double as global IDs. They wouldn't need to be explicitly written into packets, potentially reducing packet size by 4-8 bytes per entity (before compression). However, we'd probably end up wanting generations anyway to not confuse destroyed entities with new ones.

I recommend 1 as it's the simplest method. Bandwidth and CPU resources would run out long before the reduced entity ranges do. My current strategy is a mix of 1 and 3.

## Smooth Rendering

Rendering should happen later in the frame, sometime after the fixed update.

Whenever clients receive an update with new remote entities, those entities shouldn't be rendered until that update is interpolated. We can do this through a marker component or with a field in the render transform.

Cameras need some special treatment. Look inputs need to be accumulated at the render rate and re-applied to the predicted camera rotation just before rendering.

We'll also need some way for developers to declare their intent that a motion should be instant instead of smoothly interpolated. Since it needs to work for remote entities as well, maybe this just has to be a bool on the networked transform.

While most visual interpolation is linear, we'll want another blend for quickly but smoothly correcting visual misprediction errors, which can occur for entities that are or just stopped being predicted. [Projective velocity blending][11] seems like the de facto standard method for these, but I've also seen simple exponential decays used. There may be better error correction methods.

## Lag Compensation

Lag compensation deals with colliders and needs to run after all motion and physics systems. All positions have to be settled or you'll get unexpected results.

Similar to inputs, I've seen people try to have the server estimate which snapshots each client was interpolating based on their ping, but we can easily do better than that. Clients can just tell the server directly by sending their interpolation parameters along with their inputs. With this information, the server knows to do with *perfect* accuracy. No guesswork necessary.

```plaintext
<input packet header>
tick number (predicted)
tick number (interpolated from)
tick number (interpolated to)
interpolation blend value
<rest of payload>
```

So there are two ways to do the actual compensation:

- Compensate upfront by bringing new projectiles into the present (similar to a rollback).
- Compensate over time ("amortized"), constantly testing projectiles against a history buffer.

There's a lot to learn from *Overwatch* here.

*Overwatch* shows that [we can treat time as another spatial dimension][5], so we can put the entire collider history in something like a BVH and test it all at once (the amortized method). Essentially, you'd generate a bounding box for each collider that surrounds all of its historical poses and then test projectiles for hits against those first (broad-phase), then test those hits against bounding boxes blended between two snapshots (optional mid-phase), then the precise geometry blended between two snapshots (narrow-phase).  

For clients with too-high ping, their interpolation will lag far behind their prediction. If you only compensate up to a limit (e.g. 200ms), [those clients will have to extrapolate the difference][6]. Doing nothing is also valid, but lagging clients would abruptly have to start leading their targets.

You'd constrain the playback time like below and then run some extrapolation logic pre-update.

```rust
playback_time = playback_time.max((curr_tick * timestep) - max_lag_compensation);
```

*Overwatch* [allows defensive abilities to mitigate compensated projectiles][7]. AFAIK this is simple to do. If a player activates any defensive bonus, just apply it to all their buffered colliders.

When a player is the child of another, uncontrolled entity (e.g. the player is a passenger in a vehicle), the non-predicted movement of that parent entity must be rewound during lag compensation, so that any projectiles fired by the player spawn in the correct location. [See here.][8]

## Messages (RPCs and events you can send!)

Sometimes raw inputs aren't expressive enough. Examples include choosing a loadout and buying items from an in-game menu. Mispredicts aren't acceptable in these cases, however servers don't typically simulate UI.

So there's need for a dedicated type of optionally reliable message for text/UI-based and "send once" gameplay interactions. Similarly, global alerts from the server shouldn't clutter the game state.

These messages can optionally be postmarked to be processed on a certain tick like inputs, but that can only be best effort (i.e. tick N or earliest).

And while I gave examples of "requests" using these messages, those don't have to receive explicit replies. If the server confirms your purchased items, those would just appear in your inventory in a later snapshot.

IDK what these should look like yet. A macro might be the most ergonomic choice, if it means a message can be defined in its relevant system.

TBD

[1]: https://github.com/bevyengine/rfcs/pull/16
[2]: https://github.com/lemire/FastPFor/blob/master/headers/simple8b_rle.h
[3]: https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking
[4]: https://www.ea.com/games/apex-legends/news/servers-netcode-developer-deep-dive
[5]: https://youtu.be/W3aieHjyNvw?t=2226 "Tim Ford explains Overwatch's hit registration"
[6]: https://youtu.be/W3aieHjyNvw?t=2347 "Tim Ford explains Overwatch's lag comp. limits"
[7]: https://youtu.be/W3aieHjyNvw?t=2492 "Tim Ford explains Overwatch's lag comp. mitigation"
[8]: https://alontavor.github.io/AdvancedLatencyCompensation/
[9]: https://github.com/mattleibow/jitterphysics/wiki/Sweep-and-Prune
[10]: https://github.com/bevyengine/rfcs/pull/18
[11]: https://www.researchgate.net/publication/293809946_Believable_Dead_Reckoning_for_Networked_Games
[12]: https://github.com/bevyengine/rfcs/pull/16#issuecomment-849878777
[13]: https://github.com/bevyengine/bevy/issues/32
[14]: https://github.com/bevyengine/bevy/issues/32#issuecomment-821510244
[15]: https://johnaustin.io/articles/2019/fix-your-unity-timestep
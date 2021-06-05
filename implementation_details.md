
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
  
- If it were possible to split off a range of entities into a [sub-world][12] (with its own, separate component storages), that would remove a layer of indirection and allow reusing the `World` API more directly.
- If the ECS scheduler supported arbitrary cycles in the stage graph (or the "stageless" equivalent), I imagine it'd be possible to make the nested loop criteria completely transparent to the end user (no custom stages needed, perhaps).

## Practices users should follow or they'll have UB

- Entities must be spawned (non-)networked. They cannot become (non-)networked.
- Networked entities must be spawned with a "network ID" component at minimum.
- (Non-)networked components and resources should only hold or reference (non-)networked data.
- Networked components should only be mutated inside `NetworkFixedUpdate`.

## `Connection` != `Player`

I know I've been using the terms "client" and "player" somewhat interchangeably, but `Connection` and `Player` should be separate tokens. There's no reason the engine should limit things to one player per connection. Having `Player` be its own thing makes it easier to do stuff like online splitscreen, temporarily substituting vacancies with bots, etc. Likewise, a `Connection` should be a platform-agnostic handle.

## Storage

The server maintains a storage resource containing a full copy of the latest networked state as well as a ring buffer of deltas (for the last `N` snapshots). Both are updated lazily using Bevy's built-in change detection.

```plaintext
                     delta ringbuf                      copy of latest
                           v                                  v
[(0^8), (1^8), (2^8), (3^8), (4^8), (5^8), (6^8), (7^8)]     [8]
                                                    ^         
                                               newest delta 
```

This structure is pre-allocated when the resource is initialized and is the same for both full and interest-managed updates.

At the end of every tick, the server zeroes the space for the newest delta, then iterates `Changed<T>` and `Removed<T>`:

- Generating the newest delta by xor'ing the changes with the stored copy.
- Updating the rest of the ring buffer by xor'ing the older deltas with the newest.
- Writing the changes to the stored copy.

TODO

- Store/serialize networked data without struct padding so we're not wasting bandwidth.
- Support components and resources that allocate on the heap (DSTs). Won't be possible at first but the solution most likely will be backing this resource with its own memory region (something like `bumpalo` but smarter). That will be important for deterministic desync detection as well.

### Full Updates

(a.k.a. delta-compressed snapshots)

For delta-compression, the server just compresses whichever deltas clients need using some variant of run-length encoding (currently looking at [Simple8b + RLE][2]). If the compressed payload is too large, the server will split it into fragments. There's no unnecessary work either. The server only compresses deltas that are going to be sent and the same compressed payload can be sent to any number of clients.

### Interest-Managed Updates

(a.k.a. eventual consistency)

Eventual consistency isn't inherently reliant on prioritization and filtering, but they're essential for an optimal player experience.

If we can't send everything, we should prioritize what players want to know. They want live updates on objects that are close or occupy a big chunk of their FOV. They want to know about their teammates or projectiles they've fired, even if those are far away. The server has to make the most of each packet.

Similarly, game designers often want to hide certain information from certain players. Limiting the amount of hidden information that gets leaked and exploited by cheaters is often crucial to a game's long-term health. Battle-royale players, for example, don't need and probably shouldn't even have their opponents' inventory data. In practice, these barriers are never perfect (e.g. *Valorant's* Fog of War not preventing wallhacks), but something is better than nothing.

Anyway, to do all this interest management, the server needs to track some extra metadata.

```rust
struct InterestMetadata<const P: usize, const E: usize, const C: usize> {
    // P: players, E: entities, C: components
    position: [Option<(usize, SpatialIndex)>; 2 * E],
    within_aoi: [[Option<(usize, f32)>; E]; P],
    relevance: [[BitSet; C]; P]
    age: [[[usize; C]; E]; P] 
    priority: [[usize; E]; P] 
}
```

This metadata contains a few things:

- position: the min and max AABB coordinates of every networked entity (if available)
- within_aoi: the entities that are inside the area of interest of this player

Checking if an entity is inside someone's area of interest (AOI) is just an application of collision detection. AOI results are used to filter information at the entity-level. "Intangible" entities that lack a transform will auto-pass this check. For players on the same client, their results are merged with an OR.

I don't see a need for ray, distance, and shape queries where BVH or spatial partitioning structures excel, so I'm looking into doing something like a [sweep-and-prune][9] (SAP) with Morton-encoding. Since SAP is essentially just sorting the array, I imagine it might have better performance. Alternatives like grids and potentially visible sets (PVS) can be explored and added later.

- relevance: whether or not a component value should be sent to this player

Relevance is used to filter information at the component-level. By default, change detection will mark components as relevant for everybody, and then some form of rule-based filtering (maybe [entity relations][10]) can be used to selectively omit or force-include them. When sent, a component's relevance is reset to false. For players on the same client, their relevances are merged with an OR.

- age: time (in ticks) each component has had a pending change since it was last sent to this player

Age serves as the basis for send priority. The send priority of an entity is simply the age of its oldest relevant component. When an entity is sent, the ages of its relevant components are reset to zero. Age is tracked per component to be more robust against the receiving client having packet loss. For players on the same client, this field will be identical.

I think having the server only send undelivered changes and prioritizing the oldest ones is better than assigning entities arbitrary update frequencies. That said, I'll be testing a "zero-cost" alternative where I just count the number of due relevant components per entity. Unlike age, I believe this wouldn't require storing any sent packet metadata (see below).

- priority: how important it is to send the current state of each entity to this player

Priority is the end result of combining the above metadata. After filtering entities to those within a player's area of interest and filtering their components to only what that player needs to know, the server sorts the interesting entities in priority order and writes each one until the client's packet is full or they're all written.

*So what do we do if packets are lost?*

Whenever the server sends a packet, it remembers how old the included entities (actually their row indexes) were, then resets the age and relevance of their components. Later, if the server is notified that some previously sent packets were probably lost, it can pull this info and restore the components to a more indicative age (plus however many ticks have passed).

For restoring the relevance of an entity's components, there are two cases. If the delta matching the stored age still exists, the server can use that as a reference and only mark its changed components. Otherwise, they all get marked.

*So how 'bout those edge cases?*

Unfortunately, the most generalized strategy comes with its own headaches.

- What should a client do when it misses the first update for a entity? Is it OK to spawn a entity with incomplete information? If not, how does the client know when it's safe?

AFAIK this is only a problem for "kinded" entities that have archetype invariants. I'm thinking two potential solutions:

1. Have the client spawn new remote entities with any missing components in their invariants set to a default value.
2. Have the server redundantly send the full invariants for a new interesting entity until that information has been delivered once.

TBD

## Replicate Trait

```rust
pub unsafe trait Replicate {
    fn quantize(&mut self);
}

unsafe impl Replicate for T {}
```

## How to rollback?

TODO

probably a custom schedule/stage

```plaintext
The "outer" loop is the number of fixed update steps as determined by the fixed timestep accumulator.
The "inner" loop is the number of steps to re-simulate.
```

## Unconditional Rollbacks

Every article on "rollback netcode" and "client-side prediction and server reconciliation" encourages having clients compare their predicted state to the authoritative state and reconciling *if* they mispredicted, but well... How do you actually detect a mispredict?

AFAIK, there are two ways to do it:

- Iterate both copies and look for the first difference.
- Have both server and client compute a checksum for their copy and have the client compare them.

The first option has an unpredictable speed. The second option requires an ordered scan of the entire networked game state. Checksums *are* worth having for deterministic desync detection, but that can be deferred. The point I'm trying to make is that detecting state differences isn't cheap (especially once DSTs are involved).

Let's consider a simpler default:  

- Always rollback and re-simulate when you receive a new update.

This might seem wasteful, but think about it. If-then is really an anti-pattern that just hides performance problems from you. Mispredictions will exist regardless of this choice, and they're *especially* likely during heavier computations like physics. Having clients always rollback and re-sim makes it easier to profile and optimize your worst-case. It's also more memory-efficient, since clients never need to store old predicted states.

## "Clock" Synchronization

Using a fixed rate, tick-based simulation simplifies how we need to think about time. It's like scrubbing a timeline, from one "frame" to the next. The key point is that everyone follows the same sequence. Clients may be simulating different points on the timeline, but tick 480 is the same simulation step for everyone.

Ideally, clients predict ahead by just enough to have their inputs for each tick reach the server right before it simulates that tick. A commonly discussed strategy is to have clients estimate the clock time on the server (through some SNTP handshake) and use that to schedule their next simulation step, but IMO that's too indirect.

What we really care about is: How much time passes between when the server receives my input and when that input is consumed? If the server just tells me—in its update for tick N—how long my input for tick N sat in its buffer, I can use that information to converge on the correct lead.

```rust
if received_newer_server_update:
    // an exponential moving average is a simple smoothing filter
    avg_age = (31 / 32) * avg_age + (1 / 32) * age

    // too late  -> positive error -> speed up
    // too early -> negative error -> slow down
    error = target_age - avg_age

    // reset accumulator
    accumulated_correction = 0.0


time_dilation = remap(error + accumulated_correction, -max_error, max_error, -0.1, 0.1)
accumulated_correction += time_dilation * fixed_delta_time

cost_of_one_tick = (1.0 + time_dilation) * fixed_delta_time
```

If its inputs are arriving too early, a client can temporarily run fewer ticks each second to relax its lead. For example, a client simulating 10% slower would shrink their lead by 1 tick for every 10.

Interpolation is the same. You want the interpolation delay to be as small as possible. All that matters is the interval between received packets and how it varies (or maybe the number of buffered snapshots ahead of your current interpolation time).

```rust
if received_newer_server_update:
     // an exponential moving average is simple smoothing filter
    avg_delay = (31 / 32) * avg_delay + (1 / 32) * delay
    avg_jitter = (31 / 32) * avg_jitter + (1 / 32) * abs(avg_delay - delay)

    target_interp_delay = avg_delay + (2.0 * avg_jitter);
    avg_interp_delay = (31 / 32) * avg_interp_delay + (1 / 32) * (latest_snapshot_recv_time - interp_time);
    
    // too early -> positive error -> slow down
    // too late  -> negative error -> speed up
    error = -(target_interp_delay - avg_interp_delay)

    // reset accumulator
    accumulated_correction = 0.0


time_dilation = remap(error + accumulated_correction, -max_error, max_error, -0.1, 0.1)
accumulated_correction += time_dilation * delta_time

interp_time += (1.0 + time_dilation) * delta_time
interp_time = max(interp_time, predicted_time - max_lag_comp)
```

The key idea here is that simplifying the client-server relationship is more efficient and has less problems. If you followed the Source engine model described [here][3], the server would have to apply inputs whenever they arrive, meaning the server also has to rollback and it also must deal with weird ping-related issues (see the lag compensation section in [this article][4]). If the server never accepts late inputs and never changes its pace, no one needs to coordinate.

## Predicted <-> Interpolated

Clients can't directly make persistent changes the authoritative state, but they're allowed to do whatever they want locally. Current plan is to just copy the latest authoritative state. If this ends up being too expensive (or when DSTs are supported), a copy-on-write layer is another option.

We want to shift components between being predicted (extrapolated) and being interpolated. Either could be default. If interpolation is default, entities would reset to interpolated when modified by a server update. Users could then use specialized `Predicted<T>` and `Confirmed<T>` (equivalent to `Not(Predicted<T>)`) query filters to address the two groups separately. I think these can piggyback off of Bevy's built-in reliable change detection.

Systems would generate predictions by default, but users can opt-out with the `Predicted<T>` filter to only process entities that have already been mutated by an earlier system. Users would typically use these filters for heavier logic like physics. Clients will naturally predict any entities driven by their input and any spawned by their input (until confirmed by the server).

Since sounds and particles require special consideration, they're probably best realized through dispatching events to be handled *outside* `NetworkFixedUpdate`. We can use these query filters to generate events that only trigger on authoritative changes and events that trigger on predicted changes to be confirmed or cancelled later.

How to uniquely identify these events is another question, though.

## Predicting Entity Creation

This requires some special consideration.

The naive solution is to have clients spawn dummy entities so that when an update that confirms the result arrives, they'll simply destroy the dummy and spawn the true entity. IMO this is a poor solution because it prevents clients from smoothly blending errors in the predicted spawn's rendered transform. Snapping its visuals wouldn't look right.

A better solution is for the server to assign each networked entity a global ID that the spawning client can predict and map to its local instance. There are 3 variants that I know of:

1. Use an incrementing generational index (reuse `Entity`) and fix its upper bits to match the ID of the spawning player.

2. Use PRNGs to generate shared keys (I've seen these dubbed "prediction keys") for pairing local and global IDs. Rather than predict the global ID directly, clients predict the shared keys. Server updates that confirm a predicted entity would include both its global ID and the shared key. Once acknowledged, later updates can include just the global ID. This method is more complicated but does not share the previous method's implicit entity limit.

3. Bake it into the memory layout. If the layout and order of the snapshot storage is identical on all machines, array indexes and relative pointers can double as global IDs. They wouldn't need to be explicitly written into packets, potentially reducing packet size by 4-8 bytes per entity (before compression). However, we'd probably end up wanting generations anyway to not confuse destroyed entities with new ones.

I recommend 1 as it's the simplest method. Bandwidth and CPU resources would run out long before the reduced entity ranges do. My current strategy is a mix of 1 and 3.

## Smooth Rendering

Rendering should happen after `NetworkFixedUpdate`.

Whenever clients receive an update with new remote entities, those entities shouldn't be rendered until that update is interpolated. We can do this through a marker component or with a field in the render transform.

Cameras need some special treatment. Look inputs need to be accumulated at the render rate and re-applied to the predicted camera rotation just before rendering.

We'll also need some way for developers to declare their intent that a motion should be instant instead of smoothly interpolated. Since it needs to work for remote entities as well, maybe this just has to be a bool on the networked transform.

While most visual interpolation is linear, we'll want another blend for quickly but smoothly correcting visual misprediction errors, which can occur for entities that are or just stopped being predicted. [Projective velocity blending][11] seems like the de facto standard method for these, but I've also seen simple exponential decays used. There may be better smoothing filters.

## Lag Compensation

Lag compensation deals with colliders and needs to run after all motion and physics systems. All positions have to be settled or you'll get unexpected results.

It seems like a common strategy is to have the server estimate what interpolated state the client was looking at based on their RTT, but we can resolve this without any guesswork. Clients can just tell the server what they were looking at by bundling their interpolation parameters along with their inputs. With this information, the server can reconstruct what each client saw with *perfect* accuracy.

```plaintext
<packet header>
tick number (predicted)
tick number (interpolated from)
tick number (interpolated to)
interpolation blend value
<rest of payload>
```

So there are two ways to do the actual compensation:

- Compensate upfront by bringing new projectiles into the present (similar to a rollback).
- Compensate over time ("amortized"), constantly testing projectiles against the history buffer.

There's a lot to learn from *Overwatch* here.

*Overwatch* shows that [we can treat time as another spatial dimension][5], so we can put the entire collider history in something like a BVH and test it all at once (with the amortized method).

For clients with too-high ping, their interpolation will lag far behind their prediction. If you only compensate up to a limit (e.g. 200ms), [those clients will have to extrapolate the difference][6]. Doing nothing is also valid, but lagging clients would abruptly have to start leading their targets.

*Overwatch* [allows defensive abilities to mitigate compensated projectiles][7]. AFAIK this is simple to do. If a player activates any defensive bonus, just apply it to all their buffered hitboxes.

When a player is the child of another, uncontrolled entity (e.g. the player is a passenger in a vehicle), the non-predicted movement of that parent entity must be rewound during lag compensation, so that any projectiles fired by the player spawn in the correct location. See [here.][8]

## Messages (RPCs)

Messages are good for sending global alerts and any gameplay mechanics where raw inputs aren't expressive enough. For example, buying items bulk from an in-game menu. The server won't simulate UI, so it'd probably be simplest if the client sent a reliable message describing what they want. The "reply" in this example would be implicit in the received state.

Messages can be reliable. They can also be postmarked to be processed on a certain tick like inputs. That can only be best effort (i.e. tick N or earliest), though.

I don't really know what these should look like yet. A macro might be the most ergonomic choice, if it means a message can be completely defined in its relevant system.

TODO

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
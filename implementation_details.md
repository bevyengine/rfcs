
# Implementation Details

## `Connection` != `Player`

I know I've been using the terms "client" and "player" somewhat interchangeably, but `Connection` and `Player` should be separate tokens. There's no benefit in forcing one player per connection. Having `Player` be its own thing makes it easier to do stuff like online splitscreen, temporarily fill team slots with bots, etc.

## "Clock" Synchronization

Using a fixed rate, tick-based simulation simplifies how we need to think about time. It's like scrubbing a timeline, from one "frame" to the next. The key point is that everyone follows the same sequence. Clients may be simulating different points on the timeline, but tick 480 is the same simulation step for everyone.

Ideally, clients predict ahead by just enough to have their inputs reach the server right before they're needed. People often try to have clients estimate the clock time on the server (with some SNTP handshake) and use that to schedule the next simulation step, but that's too indirect IMO.

What we really care about is: How much time passes between when the server receives my input and when that input is consumed? If the server just tells me—in its update for tick N—how long my input for tick N sat in its buffer, I can use that information to converge on the correct lead.

```rust
if received_newer_server_update:
    // an exponential moving average is a simple smoothing filter
    smoothed_age = (31 / 32) * smoothed_age + (1 / 32) * age

    // too late  -> positive error -> speed up
    // too early -> negative error -> slow down
    error = target_age - smoothed_age

    // reset accumulator
    accumulated_correction = 0.0


time_dilation = remap(error + accumulated_correction, -max_error, max_error, -0.1, 0.1)
accumulated_correction += time_dilation * simulation_timestep

tick_cost = (1.0 + time_dilation) * fixed_delta_time
```

If its inputs are arriving too early, a client can temporarily run fewer ticks each second to relax its lead. For example, a client simulating 10% slower would shrink their lead by 1 tick for every 10.

Interpolation is the same. All that matters is the interval between received packets and how it varies. You want the interpolation delay to be as small as possible.

```rust
if received_newer_server_update:
     // an exponential moving average is simple smoothing filter
    smoothed_delay = (31 / 32) * smoothed_delay + (1 / 32) * delay
    smoothed_jitter = (31 / 32) * smoothed_jitter + (1 / 32) * abs(smoothed_delay - delay)

    target_interp_delay = smoothed_delay + (2.0 * smoothed_jitter);
    smoothed_interp_delay = (31 / 32) * smoothed_interp_delay + (1 / 32) * (latest_snapshot_time - interp_time);
    
    // too early -> positive error -> slow down
    // too late  -> negative error -> speed up
    error = -(target_interp_delay - smoothed_interp_delay)

    // reset accumulator
    accumulated_correction = 0.0


time_dilation = remap(error + accumulated_correction, -max_error, max_error, -0.1, 0.1)
accumulated_correction += time_dilation * delta_time

interp_time += (1.0 + time_dilation) * delta_time
interp_time = max(interp_time, predicted_time - max_lag_comp)
```

The key idea here is that simplifying the client-server relationship makes the problem easier. You *could* have the server apply inputs whenever they arrive, rolling back if necessary, but that would only complicate things. If the server never accepts late inputs and never changes its pace, no one needs to coordinate.

## Prediction <-> Interpolation

Clients can't directly modify the authoritative state, but they should be able to predict whatever they want locally. One obvious implementation is to literally fork the latest authoritative state. If copying the full state ends up being too expensive, we can probably use a copy-on-write layer.

My current idea to shift components between prediction and interpolation is to default to interpolated (reset upon receiving a server update) and then use specialized change detection `DerefMut` magic to flag as predicted.

```rust
Predicted<T>
PredictAdded<T>
PredictRemoved<T>
Confirmed<T>
ConfirmAdded<T>
ConfirmRemoved<T>
Cancelled<T>
CancelAdded<T>
CancelRemoved<T>
```

Everything is predicted by default, but users can opt-out by filtering on `Predicted<T>`. In the more conservative cases, clients would predict the entities driven by their input, the entities they spawn (until confirmed), and any entities mutated as a result of the first two. Systems with filtered queries (i.e. physics, path-planning) should typically run last.

We can also use these filters to generate events that only trigger on authoritative changes and events that trigger on predicted changes to be confirmed or cancelled later. The latter are necessary for handling sounds and particle effects. Those shouldn't be duplicated during rollbacks and should be faded out if mispredicted.

Should UI be allowed to reference predicted state or only verified state?

## Predicting Entity Creation

This requires some special consideration.

The naive solution is to have clients spawn dummy entities. When an update that confirms the result arrives, clients can simply destroy the dummy and spawn the true entity. IMO this is a poor solution because it prevents clients from smoothly blending these entities from predicted time into interpolated time. It won't look right.

A better solution is for the server to assign each networked entity a global ID (`NetworkID`) that the spawning client can predict and map to its local instance.

- The simplest form of this would be an incrementing generational index whose upper bits are fixed to match the spawning player's ID. This is my recommendation. Basically, reuse `Entity` and reserve some of the upper bits in its ID.

- Alternatively, PRNGs could be used to generate shared keys (called "prediction keys" in some places) for pairing global and local IDs. Rather than predict the global ID, the client would predict the shared key. Server updates that confirm the predicted entity would include both its global ID and the shared key, which the client can then use to pair the IDs. This method adds complexity but bypasses the previous method's implicit entity limit.

- A more extreme solution would be to somehow bake global IDs directly into the memory allocation. If memory layouts are mirrored, relative pointers become global IDs, which don't need to be explicitly written into packets. This would save 4-8 bytes per entity before compression.

## Smooth Rendering

Rendering should come after `NetworkFixedUpdate`.

Whenever clients receive an update with new remote entities, those entities shouldn't be rendered until that update is interpolated.

Cameras need a little special treatment. Inputs to the view rotation need to be accumulated at the render rate and re-applied just before rendering.

We'll also need to distinguish instant motion from integrated motion when interpolating. Moving an entity by modifying `transform.translation` and `rigidbody.velocity` should look different.

We'll need a special blending for predicted entities and entities transitioning between prediction and interpolation. "Projective velocity blending" seems to be a common way to smooth extrapolation errors, but I've also seen a simple exponential decay recommended. Are there better smoothing algorithms? (Any econ grads good with time-series?)

## Lag Compensation

Lag compensation deals with colliders. To avoid weird outcomes, lag compensation needs to run after all motion and physics systems.

Again, people often imagine having the server estimate what interpolated state the client was looking at based on their RTT, but we can resolve this without any guesswork.

Clients can just tell the server what they were looking at by bundling the interpolated tick numbers and the blend value inside the input payloads. With this information, the server can reconstruct *exactly* what each client saw.

```plaintext
<packet header>
tick number (predicted)
tick number (interpolated from)
tick number (interpolated to)
interpolation blend value
<rest of payload>
```

So there are two ways to go about the actual compensation:

- Compensate upfront by bringing new projectiles into the present (similar to a rollback).
- Compensate over time ("amortized"), constantly testing projectiles against the history buffer.

There's a lot to learn from *Overwatch* here.

*Overwatch* shows that [we can treat time as another spatial dimension](https://youtu.be/W3aieHjyNvw?t=2226), so we can put the entire collider history in something like a BVH and test it all at once (with the amortized method).

*Overwatch* [allows defensive abilities to mitigate compensated projectiles](https://youtu.be/W3aieHjyNvw?t=2492). AFAIK this is simple to do. If a player activates any defensive bonus, just apply it to all their buffered hitboxes.

For clients with too-high ping, their interpolation will lag far behind their prediction. If you only compensate up to a limit (e.g. 200ms), [those clients will have to extrapolate the difference](https://youtu.be/W3aieHjyNvw?t=2347). Doing nothing is also valid, but lagging clients would abruptly have to start leading their targets.

When a player is parented to another entity, which they have no control over (e.g. the player is a passenger in a vehicle), the non-predicted movement of that parent must be rewound during compensation to spawn any projectiles fired by the player in the correct location.

## Unconditional Rollbacks

Every article on "rollback netcode" and "client-side prediction and server reconciliation" encourages having clients compare their predicted state to the authoritative state and reconciling *if* they mispredicted. But how do you actually detect a mispredict?

I thought of two methods while I was writing this:

- Unordered scan looking for first difference.
- Ordered scan to compute checksum and compare.

The first option has an unpredictable speed. The second option requires a fixed walk of the game state (checksums *are* probably worth having even if only for debugging non-determinism). There may be options I didn't consider, but the point I'm trying to make is that detecting changes among large numbers of entities isn't cheap.

Let's consider a simpler default:  

- Always rollback and re-simulate.

Now, you may think that's wasteful, but I would say "if mispredicted" gives you a false sense of security. Mispredictions can occur at any time, *especially* during long-lasting complex physics interactions. Those would show up as CPU spikes. Instead, it's much easier to profile and optimize for your worst-case if clients *always* rollback and re-sim. It's also more memory-efficient, since clients never need to store old predicted states.

## Delta-Compressed Snapshots

- The server keeps an incrementally updated copy of the networked state.
  - Components are stored with their global ID instead of the local ID.
- The server keeps a ring buffer of "patches" for the last `N` snapshots.
- At the end of every `NetworkFixedUpdate`, the server iterates `Changed<T>` and `Removed<T>`, then:
  - Generates the latest patch as the copy `xor` changes.
  - Applies the changes to the copy and pushes the latest patch into the ring buffer.
  - `Xors` older patches with the latest patch to update them.
- The server reads the needed patches as `&[u8]` (or `&[u64]`) and compresses them using run-length encoding (RLE) or similar.
  - No "serialization" needed. If networked DSTs (dynamically-sized types) are stored in their own heap allocation, we can literally send the bits. `rkyv` is a good reference (relative pointers).
- Pass compressed payloads to protocol layer.
- Protocol and I/O layers do whatever they do and send the packet.

## Interest-Managed Updates

- Uses the same latest copy + delta buffer data structure, but with additional metadata and filter "components" to track priority and relevance per-*player*.
- Changing components (note: `Transform` is special) adds to the send priority of their entities. Existing send priority doubles every tick. Importantly, entities that haven't changed won't accumulate priority.
- Server writes the entities relevant to each client in priority order, until the packet is full or all entities are written. The send priority of written entities are reset for that client.
- If the server is notified of packet loss, it checks the patch of the update that was lost and re-prioritizes its changed entities for that client. If the patch no longer exists, everything is prioritized.
- (Again, handling networked DSTs needs more consideration.)

## Messages

TODO

Messages are best for sending global alerts and any gameplay mechanics you explicitly want modeled as request-reply (or one-way) interactions. They can be unreliable or reliable. You can also postmark messages to be executed on a certain tick like inputs. That can only be best effort, though.

The example I'm thinking of is buying items from an in-game vendor. The server doesn't simulate UI, but ideally we can write the message transaction in the same system. A macro might end up being the most ergonomic choice.

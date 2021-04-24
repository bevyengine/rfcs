
# Implementation Details
## Delta Compression
TBD

## Area of Interest
TBD

## "Clock" Synchronization
Ideally, clients predict ahead by just enough to have their inputs reach the server right before they're needed. For some reason, people frequently arrive at the idea that clients should estimate the clock time on the server (with some SNTP handshake) and use that to schedule the next simulation step.

That's overcomplicating it. What we really care about is: How much time passes between when the server receives my input and when that input is consumed? If the server simply tells clients how long their inputs are waiting in its buffer, the clients can use that information to converge on the correct lead.

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

## Lag Compensation
Lag compensation mainly deals with colliders. To avoid weird outcomes, lag compensation needs to run after all motion and physics systems.

Again, people get weird ideas about having the server estimate what interpolated state the client was looking at based on their RTT. Again,  that kind of guesswork is unnecessary.

Clients can just tell the server what they were looking at by bundling the interpolated tick numbers and the blend value inside the input payloads.

```
<packet header>
tick number (predicted)
tick number (interpolated from)
tick number (interpolated to)
interpolation blend value
<rest of payload>
```
With this information, the server can reconstruct *exactly* what each client saw. 

Lag compensation goes like this:
1. Queue projectile spawns, tagged with their shooter's interpolation data.
2. Restore all colliders to the earliest interpolated moment.
3. Replay forward to the current tick, spawning the projectiles at the appropriate times and registering hits.

After that's done, any surviving projectiles will exist in the correct time. The process is the same for raycast weapons.

There's a lot to learn from *Overwatch* here.

*Overwatch* [allows defensive abilities to mitigate lag-compensated shots](https://youtu.be/W3aieHjyNvw?t=2492). AFAIK this is simple to do. If a player activates any defensive bonus, just apply it to all their buffered hitboxes.

*Overwatch* also [finds the movement envelope of each entity](https://youtu.be/W3aieHjyNvw?t=2226), the "sum" of its bounding volumes over the full lag compensation window, to reduce the number of intersection tests, only rewinding characters whose movement envelopes intersect projectiles.

For clients with very high ping, their interpolated time will lag too far behind their predicted time. You generally don't want to favor the shooter past a certain limit (e.g. 250ms), so [those clients have to extrapolate the difference](https://youtu.be/W3aieHjyNvw?t=2347). Not extrapolating is also valid, but then lagging clients would abruptly have to start leading their targets.

This limit is the only relation between the predicted time and the interpolated time. They're otherwise decoupled.

## Smooth Rendering
Whenever clients receive an update with new remote entities, those entities shouldn't be rendered until that update is interpolated.

Cameras need a little special treatment. Inputs to the view rotation need to be accumulated at the render rate and re-applied just before rendering.

Is an exponential decay enough for smooth error correction or are there better algorithms?

## Prediction <-> Interpolation
Clients can't directly modify the authoritative state, but they should be able to predict whatever they want locally. One obvious implementation is to literally fork the latest authoritative state. If copying the full state ends up being too expensive, we can probably use a copy-on-write layer.

Clients should predict the entities driven by their input, the entities they spawn (until confirmed), and any entities mutated as a result of the first two. I think that should cover it. Predicting *everything* would be a compile-time choice.

I said entities, but we can predict with component granularity. The million-dollar question is how to shift things between prediction and interpolation. My current idea is for everything to default to interpolation (reset upon receiving a server update) and then use specialized change detection `DerefMut` magic.

```
Predicted<T>
PredictAdded<T>
PredictRemoved<T>
Confirmed<T>
ConfirmAdded<T>
ConfirmRemoved<T>
Canceled<T>
CancelAdded<T>
CancelRemoved<T>
```

With these, we can generate events that only trigger on authoritative changes and events that trigger on predicted changes to be confirmed or cancelled later. The latter are necessary for handling sounds and particle effects. Those shouldn't be duplicated during rollbacks and should be faded out if mispredicted.

All systems that handle "predictable" interactions (pushing a button, putting an item in your inventory) should run *before* physics. Everything in `NetworkFixedUpdate` should run before rendering.

Should UI be allowed to reference predicted state or only verified state?

## Predicting Entity Creation
This requires some special consideration.

The naive solution is to have clients spawn dummy entities. When an update that confirms the result arrives, clients can simply destroy the dummy and spawn the true entity. IMO this is a poor solution because it prevents clients from smoothly blending these entities from predicted time into interpolated time. It won't look right.

A better solution is for the server to assign each networked entity a global ID that the spawning client can predict and map to its local instance.

- The simplest form of this would be an incrementing index whose upper bits are fixed to match the spawning player's ID. This is my recommendation.

- Alternatively, PRNGs could be used to generate shared keys for pairing global and local IDs. Rather than predict the global ID, the client would predict the shared key. Server updates that confirm the predicted entity would include both its global ID and the shared key, which the client can then use to pair the IDs. This method adds complexity but bypasses the previous method's implicit entity limit.

- A more extreme solution would be to somehow bake global IDs directly into the memory allocation. If memory layouts are mirrored, relative pointers become global IDs, which don't need to be explicitly written into packets. This would save 4-8 bytes per entity before compression.

## Unconditional Rollbacks
Every article on "rollback netcode" and "client-side prediction and server reconciliation" encourages having clients compare their predicted state to the authoritative state and reconciling *if* they mispredicted. But how do you actually detect a mispredict?

I thought of two methods while I was writing this:

1. Unordered scan looking for first difference.
2. Ordered scan to compute checksum and compare.

The first option has an unpredictable speed. The second option requires a fixed walk of the game state (checksums *are* probably worth having even if only for debugging non-determinism). There may be options I didn't consider, but the point I'm trying to make is that detecting changes among large numbers of entities isn't cheap. 

Let's consider a simpler default:  

3. Always rollback and re-simulate.

Now, you might be thinking, "Isn't that wasteful?"

*If* gives a false sense of security. If I make a game and claim it can rollback 250ms, that basically should mean *any* 250ms, with no stuttering. If clients *always* rollback and re-sim, it'll be easier to profile and optimize for that. As a bonus, clients can immediately toss old predicted states.

Constant rollbacks may sound expensive, but there were games with rollback running on the original Playstation 20+ years ago.
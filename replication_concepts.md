# Replication
> The goal of replication is to ensure that all of the players in the game have a consistent model of the game state. Replication is the absolute minimum problem which all networked games have to solve in order to be functional, and all other problems in networked games ultimately follow from it. - [Mikola Lysenko][1]

---

Abstractly, you can think of a game as a pure function that accepts an initial state and player inputs and generates a new state.
```rust
let new_state = simulate(&state, &inputs);
```
Fundamentally, if several players want to perform a synchronized simulation over a network, they have basically two options:

- Send their inputs to each other and independently and deterministically simulate the game. 
  - <details><summary>also known as</summary>active replication,  lockstep, state-machine synchronization, determinism</details>
- Send their inputs to a single machine (the server) who simulates the game and broadcasts updates back.
  - <details><summary>also known as</summary>passive replication, client-server, primary-backup, state transfer</details>

In other words, players can either run the "real" game or follow it.

For the rest of this RFC, I'll refer to them as determinism and state transfer, respectively. I just think they're the most literal terminology.

## Why determinism?
Deterministic multiplayer is basically local multiplayer but with *really* long controller cables. The netcode simply supplies the gameplay code with inputs. They're basically decoupled.

Determinism has low infrastructure costs, both in terms of bandwith and server hardware. All steady-state network traffic is input, which is not only small but also compresses well. (Note that as player count increases, there *is* a crossover point where state transfer becomes more efficient). Likewise, as the game runs completely on the clients, there's no need to rent powerful servers. Relays are still handy for efficiently managing rooms and scaling to higher player counts, but those could be cheap VPS instances.

Determinism is also tamperproof. It's impossible to do anything like speedhack or teleport as running these exploits would simply cause cheaters to desync. On the other hand, determinism inherently suffers from total information leakage.  

That every client must run the *entire* world is also determinism's biggest limit. While this works well for games with thousands of micro-managed entities like *Starcraft 2*, you won't be seeing games with expansive worlds like *Genshin Impact* networked this way any time soon.

## Why state transfer?
Determinism is awesome when it fits but it's generally unavailable. Neither Godot nor Unity nor Unreal can make this guarantee for large parts of their engines, particularly physics.

Whenever you can't have or don't want determinism, you should use state transfer.

Its main underlying idea is **authority**, which is just like ownership in Rust. Those who own state are responsible for broadcasting up-to-date information about it. I sometimes see authority divided into *input* authority (control permission) and *state* authority (write permission), but usually authority means state authority.

The server usually owns everything, but authority is very flexible. In games like *Destiny* and *Fall Guys*, clients own their movement state. Other games even trust clients to confirm hits. Distributing authority like this adds complexity and obviously leaves the door wide open for cheaters, but sometimes it's necessary. In VR, it makes sense to let clients claim and relinquish authority over interactable objects.

## Why not messaging patterns?
The only other strategy you really see used for replication is messaging. RPCs. I actually see these most often in the free asset space. (I guess it's the go-to pattern outside of games?)

Take chess for example. Instead of sending polled player inputs or the state of the chessboard, you could just send the moves like "white, e2 to e4," etc.

Here's the issue. Messages are tightly coupled to their game's logic. They can't be generalized. Chess is simple—one turn, one event—but what about an FPS? What messages would it need? How many? When and where would those messages need be sent and received?

If those messages have cascading effects, they can only be sent reliable, ordered.
```rust
let mut s = state[n];
for message in queue.iter() {
    s.apply(&message);
}

// The key thing to note is that state[n+1]
// cannot be correct unless all messages were
// applied and applied in the right order.
*state[n+1] = s;
```
Messages are great for when you want explicit request-reply interactions and global alerts like players joining or leaving. They just don't cut it as a replication mechanism for real-time games. Even if you avoided send and receive calls everywhere (i.e., collect and send in batches), messages don't compress as well as inputs or state.

# Latency
Networking is hard because we want to let players who live in different countries play together *at the same time*, something that special relativity tells us is [strictly impossible][2]... unless we cheat.

### Lockstep
The simplest solution is to concede to the universe with grace and have players stall until they've received whatever data they need to execute the next simulation step. Blocking is fine for most turn-based games but it just doesn't cut it for real-time games.
                
### Adding Local Input Delay
The first trick we can pull is have each player delay their own input for a bit, trading responsiveness for more time to receive the incoming data.

Our brains are pretty lenient about this, so we can actually *reduce* the latency between players. Two players in a 1v1 match actually could experience simultaneity if each delayed their input by half the round-trip time.

This trick has powered the RTS genre for decades. With a large enough input delay and a stable connection, the game will run smoothly. However, there's still a problem because the game stutters whenever the window is missed. This leads to the next trick.

> determinism + lockstep + local input delay = delay-based netcode 

### Predict-Reconcile
Instead of blocking, what if players just guess the missing data and keep going? Doing that would let us avoid stuttering, but then we'd have to deal with guessing incorrectly.

Well, when the player finally has that missing remote data, what they can do is restore their simulation to the previous verified state, update it with the received data, and then re-predict the remaining steps.

This retroactive correction is called **rollback** or **reconciliation**, and it ensures that players never desync *too much*. Honestly, it's practically invisible with a high tick rate and good visual smoothing. (Apparently it's been around since [1996][3].)

With prediction, input delay is no longer needed, but it's still useful. Reducing latency reduces how many steps players need to re-simulate.

> determinism + predict-rollback + local input delay (optional) = rollback netcode 

### Selective Prediction
Once again, determinism is an all or nothing deal. If you predict, you predict everything. 

State transfer has the flexibility to predict only *some* things, letting you offload expensive systems onto the server. Games like *Rocket League* still predict everything, including other clients (the server re-distributes their inputs along with game state so that this is more accurate). However, most games choose not to do this. It's more common for clients to predict only what they control and interact with. 

# Visual Consistency
**tl;dr**: Hard snap the simulation state and subtly blend the view. Time travel if needed.
## Smooth Rendering and Lag Compensation

Predicting only *some* things adds implementation complexity. 

When clients predict everything, they produce renderable state at a fixed pace. Now, anything that isn't predicted must be rendered using data received from the server. The problem is that server updates are sent over a lossy, unreliable internet that disrupts any consistent spacing between packets. This means clients need to buffer incoming server updates long enough to have two authoritative updates to interpolate most of the time.

Gameplay-wise, not predicting everything also divides entities between two points in time: a predicted time and an interpolated time. Clients see themselves in the future and everything else in the past. Because players demand a WYSIWYG experience, the server must compensate for this "remote lag" by allowing certain things, mainly projectiles, to interact with the past.

Visually, we'll often have to blend between extrapolated and authoritative data. Simply interpolating between two authoritative updates is incorrect. The visual state can and will accrue errors, but that's what we want. Those can be tracked and smoothly reduced (to some near-zero threshold, then cleared).

# Bandwidth
## How much can we fit into each packet?
Not a lot.

You can't send arbitrarily large packets over the internet. The information superhighway has load limits. The conservative, almost universally supported "maximum transmissible unit" or MTU is 1280 bytes. Accounting for IP and UDP headers and some connection metadata, you realistically can send ~1200 bytes of game data per packet.

If you significantly exceed this, some random stop along the way will delay the packet and break it up into fragments. 

[Fragmentation](https://packetpushers.net/ip-fragmentation-in-detail/) [sucks](https://blog.cloudflare.com/ip-fragmentation-is-broken) because it multiplies the likelihood of the overall packet being lost (all fragments have to arrive to read the full packet). Getting fragmented along the way is even worse because of the added delay. It's okay if the sender manually fragments their packet (like 2 or 3) *upfront*, although the higher loss does limit simulation rate, just don't rely on the internet to do it.

## Okay, but that doesn't seem like much?
Well, there are two more reasons not to yeet giant 100kB packets across the network:
- Bandwidth costs are the lion's share of hosting expenses.
- Many players still have limited bandwidth.

So unless we limit everyone to <20Hz tick rates, our only options are:
- Send smaller things.
- Send fewer things.

### Snapshots
Alright then, state transfer. The most obvious strategy is to send full **snapshots**. All we can do with these is make them smaller (i.e. quantize floats, then compress everything).

Fortunately, snapshots are very compressible. An extremely popular idea called **delta compression** is to send each client a diff (often with further compression on top) of the current snapshot and the latest one they acknowledged receiving. Clients can then use these to patch their existing snapshots into the current one.

The server can fragment payloads as a last resort. 

### Eventual Consistency
When snapshots fail or hidden information is needed, the best alternative is to prioritize sending each client the state most relevant to them. This technique is commonly called **eventual consistency**.

Determining relevance is often called **interest management** or **area of interest**. Each granular piece of state is given a "send priority" that accumulates over time and resets when sent. How quickly priority accumulates for different things is up to the developer, though physical proximity and visual salience usually have the most influence.

Eventual consistency can be combined with delta compression, but I wouldn't recommend it. Many AAA games have done it, but IMO it's just too much bookkeeping. Unlike snapshots, the server would have to track the latest received state for each *item* on each client separately and create diffs for each client separately.

[1]: https://0fps.net/2014/02/10/replication-in-networked-games-overview-part-1/
[2]: https://en.wikipedia.org/wiki/Relativity_of_simultaneity
[3]: https://en.wikipedia.org/wiki/Client-side_prediction
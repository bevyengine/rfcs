# Feature Name: `Deterministic RNG`

## Summary

Include a source of entropy to (optionally) enable deterministic random number generation.

## Motivation

Bevy games / applications often need to use randomness[<sup>1</sup>](#1) but doing so makes execution non-deterministic. This is problematic for automated testing as well as deterministic execution. Deterministic execution is important for those wishing to create games with [deterministic lockstep](https://gafferongames.com/post/deterministic_lockstep/) as well as high-fidelity simulations.

Currently there is no official way to introduce randomness in bevy, a plugin, or an app. Other engines such as [Godot](https://docs.godotengine.org/en/stable/tutorials/math/random_number_generation.html), [Unity](https://docs.unity3d.com/ScriptReference/Random.html), and [Unreal](https://docs.unrealengine.com/4.27/en-US/BlueprintAPI/Math/Random/) include engine support for randomness.

<small>
<sup>1</sup><a name="1"></a> For example, 16 of bevy's examples currently use `rand`
<details>
<pre>
examples/games/contributors.rs
examples/games/alien_cake_addict.rs
examples/async_tasks/external_source_external_thread.rs
examples/async_tasks/async_compute.rs
examples/ecs/iter_combinations.rs
examples/stress_tests/transform_hierarchy.rs
examples/stress_tests/many_lights.rs
examples/animation/custom_skinned_mesh.rs
examples/stress_tests/bevymark.rs
examples/stress_tests/many_sprites.rs
examples/ecs/parallel_query.rs
examples/ecs/component_change_detection.rs
examples/ecs/ecs_guide.rs
examples/app/random.rs
crates/bevy_ecs/examples/resources.rs
crates/bevy_ecs/examples/change_detection.rs
</pre>
</details>
</small>

## User-facing explanation

### Overview

Games often use randomness as a core mechanic.
For example, card games generate a random deck for each game and killing monsters in an RPG often rewards players with a random item.
While randomness makes games more interesting and increases replayability, it also makes games harder to test and prevents advanced techniques such as [deterministic lockstep](https://gafferongames.com/post/deterministic_lockstep/).

Let's pretend you are creating a poker game where a human player can play against the computer.
The computer's poker logic is very simple--when the computer has a good hand, it bets all of its money.
To make sure the behavior works, you write a test to first check the computer's hand and if it is good confirm that all its money is bet.
If the test passes does it ensure the computer behaves as intended? Sadly, no.

Because the deck is randomly shuffled for each game--without doing so the player would already know the card order from the previous game--it is not guaranteed that the computer player gets a good hand and thus the betting logic goes unchecked.
While there are ways around this--a fake deck that is not shuffled, running the test many times to increase confidence, breaking the logic into units and testing those--it would be very helpful to have randomness as well as a way to make it _less_ random.

Luckily, when a computer needs a random number it doesn't use real randomness and instead uses a [pseudorandom number generator](https://en.wikipedia.org/wiki/Pseudorandom_number_generator).
Popular Rust libraries containing pseudorandom number generators are [`rand`](https://crates.io/crates/rand) and [`fastrand`](https://crates.io/crates/fastrand).

Pseudorandom number generators require a source of [entropy](https://en.wikipedia.org/wiki/Entropy) called a [random seed](https://en.wikipedia.org/wiki/Random_seed).
The random seed is used as input to generate numbers that _appear_ random but are instead in a specific and deterministic order.
For the same random seed, a pseudorandom number generator always returns the same numbers in the same order.

For example, let's say you seed a pseudorandom number generator with `1234`.
You then ask for a random number between `10` and `99` and the pseudorandom number generator returns `12`.
If you run the program again with the same seed (`1234`) and ask for another random number between `1` and `99`, you will again get `12`.
If you then change the seed to `4567` and run the program, more than likely the result will not be `12` and will instead be a different number.
If you run the program again with the `4567` seed, you should see the same number from the previous `4567`-seeded run.

There are many types of pseudorandom number generators each with their own strengths and weaknesses.
Because of this, Bevy does not include a pseudorandom number generator.
Instead, the `bevy_entropy` plugin includes a source of entropy to use as a random seed for your chosen pseudorandom number generator.

Note that Bevy currently has [other sources of non-determinism](https://github.com/bevyengine/bevy/discussions/2480) unrelated to pseudorandom number generators.

### Usage

The `bevy_entropy` plugin ships with Bevy and is enabled by default.
If you do not need randomness, you may [disable the plugin](https://docs.rs/bevy/latest/bevy/app/struct.PluginGroupBuilder.html#method.disable) or use Bevy's [minimal set of plugins](https://docs.rs/bevy/latest/bevy/struct.MinimalPlugins.html).

When enabled, `bevy_entropy` provides one world resource: `Entropy`.
`Entropy` is then used as the seed for the pseudorandom number generator of your choosing.
A complete example leveraging the [`StdRng`](https://docs.rs/rand/latest/rand/rngs/struct.StdRng.html) pseudorandom number generator from the [`rand`](https://crates.io/crates/rand) crate can be found in the examples.

The default source of world entropy [`Entropy::default()`] is non-deterministic and seeded from the operating system.
It is guarenteed to be suitable for [cryptographic applications](https://en.wikipedia.org/wiki/Pseudorandom_number_generator#Cryptographic_PRNGs) but the actual algorithm is an implementation detail that will change over time.

You may choose to determinstically seed your own world entropy via [`Entropy::from`].
The seed you choose may have security implications or influence the distribution of the resulting random numbers.
See [this resource](https://rust-random.github.io/book/guide-seeding.html) for more details about how to pick a "good" random seed for your needs.

Depending on your game and the type of randomness you require, when specifying a seed you will normally do one of the following:

1. Get a good random seed out-of-band and hardcode it in the source.
2. Dynamically call to the OS and print the seed so the user can rerun deterministically. In games like [Factorio](https://www.factorio.com/) sharing random seeds is encouraged and supported.
3. Dynamically call to the OS and share the seed with a server so the client and server deterministically execute together.
4. Load the seed from a server so the client and server deterministically execute together.

## Implementation strategy

https://github.com/bevyengine/bevy/pull/2504

## Drawbacks

- This may not be general enough to include in Bevy.
- This may not be general enough to be on by default.
- Includes `rand_core` and `rand_chacha` as a dependency.
  - But not in public API.

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
  - It gives flexibility for PRNG crates while also enforcing standards on the ecosystem.
  - It defaults to what someone would resonably expect if they wrote it themselves.
  - It defaults to something safe (suitable for cryptographic functions), removing a possible footgun.
- What other designs have been considered and what is the rationale for not choosing them?
  - We could go higher and expose an API closer to `rand`. This is what [Godot](https://docs.godotengine.org/en/stable/tutorials/math/random_number_generation.html), [Unity](https://docs.unity3d.com/ScriptReference/Random.html), and [Unreal](https://docs.unrealengine.com/4.27/en-US/BlueprintAPI/Math/Random/) do. We are not experts in PRNG API design, and while `rand` is clearly the most popular random crate in the Rust ecosystem we don't currently want to tie bevy to any particular API in case something better emerges.
  - We could go lower and merely expose a static `WorldSeed`. I'm worried about what the default would be and forcing a seed at world creation feels heavyweight.
  - We could default to a faster PRNG rather than a safer one. I wanted folks to fall into the pit of success.
- What objections immediately spring to mind? How have you addressed them?
  - `rand_core` and `rand_chacha`are too heavy of a dependency.
  - This should not be in core.
- What is the impact of not doing this?
  - There is no chance of the Bevy ecosystem supporting deterministic execution.
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?
  - There needs to be one true way to keep the ecosystem coherant.

## Prior art

- https://github.com/bevyengine/bevy/discussions/2480
- https://github.com/bevyengine/bevy/discussions/1678

Other engines such as [Godot](https://docs.godotengine.org/en/stable/tutorials/math/random_number_generation.html), [Unity](https://docs.unity3d.com/ScriptReference/Random.html), and [Unreal](https://docs.unrealengine.com/4.27/en-US/BlueprintAPI/Math/Random/) include engine support for randomness (with a higher-level API).

- https://towardsdatascience.com/how-to-use-random-seeds-effectively-54a4cd855a79
- https://www.whatgamesare.com/determinism.html
- https://gafferongames.com/post/deterministic_lockstep/
- https://ruoyusun.com/2019/03/29/game-networking-2.html
- https://arxiv.org/abs/2104.06262

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
  - Do we want this?
  - ~~What about `rand`?~~
    - `rand` was [switched to](https://github.com/bevyengine/bevy/pull/2504/commits/d255fc40b65ea358c40c71a17852cd865387b869) `rand_core` and `rand_chacha`, thanks @bjorn3!
  - Are `rand_core` and `rand_chacha` small enough?
  - Is `Entropy` naming too obscure?
  - How do we deal with security vs speed tradeoff here?
  - Is this the right level of abstraction? Lower? Higher?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - Built-in higher-level randomness APIs
  - Entropy per-system rather than per-world

## Future possibilities

- Higher-level randomness APIs
- Entropy per-system rather than per-world
- Seed from arbitrary-length [`&[u8]`](https://github.com/bevyengine/rfcs/pull/55#issuecomment-1138159824)

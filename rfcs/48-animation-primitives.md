# Feature Name: `animation-primitives`

## Summary

Animation is a particularly complex, with many stateful and intersecting
elements. This RFC aims to detail a set of lowest-level APIs for authoring and
playing animations within Bevy.

## Motivation

Animation is at the heart of modern game development. A game engine without an
animation system is generally considered not yet production ready.

This RFC aims to detail the absolute lowest level APIs to unblock ecosystem
level experimentation with building more complex animation systems (i.e. inverse
kinematics, animation state machine, humanoid retargetting, etc.)

## Scope

Animation is a huge area that spans multiple problem domains:

 1. **Storage**: this generally covers on-disk storage in the form of assets as
    well as the in-memory representation of static animation data.
 2. **Sampling**: this is how we sample the animation assets over time and
    transform it into what is applied to the animated entities.
 3. **Application**: this is how we applied the sampled values to animated
    entities.
 4. **Composition**: this is how we compose simple clips to make more complex
    animated behaviors.
 4. **Authoring**: this is how we create the animation assets used by the engine.

This RFC specifically aims to resolve only problems within the domains of storage
and sampling. Application can be distinctly decoupled from these earlier two
stages, treating the sampled values as a black box output, and composition and
authoring can be built separately upon the primitives provided by this RFC and
thus are explicit non-goals here.

## User-facing explanation

The core of this system is a trait called `Sample<T>` which allows sampling
values of `T` from an underlying type at a provided time. `T` here can be
anything considered animatable. A few examples of high priority types to be
supported here are:

 - `f32`/`f64`
 - `Vec2`/`Vec3`/`Vec3A`/`Vec4` (and their `f64` variants)
 - `Color`
 - `Option<T>` where `T` can also be sampled
 - `bool` for toggling on and off features.
 - `Range<T>` for a range for randomly sampling from (think particle systems)
 - `Handle<T>` for sprite animation, though can be generically used for any asset
   swapping.
 - etc.

Built on top of this trait is the concept of a **animation graph**, a runtime
mutable directed acyclic graph of nodes for altering the sampling behavior for
a given property. There is always one root level node that is directly sampled
from the app world. It can either be a terminal node, or be a composition node
that samples it's children for values and combines the outputs before passing it
upstream. Some examples include:

 - `MixerNode<T>` - a stateful multi-input composition node that outputs a
   weighted sum of it's inputs. Can be used to make arbitrary blending of it's
   inputs.
 - `SelectorNode<T>` - a multi-input composition node that outputs only the
   currently selected input as it's output. All other inputs are not evaluated.
   Like a MixerNode with a one-hot weight blend, but more computationally
   efficient.
 - `ConstantNode<T>` - only outputs a constant value all times.
 - `RepeatNode<T>` - a single input node that loops it's initial input over time.
 - `PingPongNode<T>` - a single input node that loops it's initial input over
   time, will
 - `Arc<dyn Sample<T>>`/`Box<dyn Sample<T>>` - Anything that can be sampled can
   be used as an input to the graph.

The final lowest level and the concrete implementors of Sample are implemetors
of the trait `Curve<T>`. Curves are meant to store the raw data for time-sampled
values. There may be multiple implementations of this trait, and they're
typically what is serialized and stored in assets.

Finally the last major introduction is the `AnimationClip` asset, which bundles a
set of curves to the asoociated `Reflect` path they're bound to. This is the main
metadata source for actually binding sampled outputs to the fields they're
animating.

## Implementation strategy

Protoytpe implementation: https://github.com/HouraiTeahouse/bevy_prototype_animation

TODO: Complete this section.

## Drawbacks

The main drawback to this approach is code complexity. There are a lot of `dyn
Trait` or `impl Trait` in these APIs and it might get a bit difficult to follow
without external 1000ft views of the entire system. However, this is meant to be
a flexible low-level API so this might be easier to gloss over once a more higher
level solution built on top of it is made.

## Rationale and alternatives

Bevy absolutely needs some form of a first-party animation system, no modern game
engine can be called production ready without one. Having this exist solely as a
third-party ecosystem crate is definitely unacceptable as it would promote a
facturing of the ecosystem with multiple incompatible baseline animation system
implementations.

The design chosen here was explicitly to allow for maximum flexibility for both
engine and game developers alike. The main alternative is to completely remove
`Sample<T>`, the animation graph, and it's nodes, and let developers directly
hook up animation curves from clips to entities. However, this lacks flexibility
and does not allow for users of the API to inject their own alterations to the
animation stream.

The main potential issue with the current implementation is the very heavy use of
`Arc<dyn Sample<T>>` which has CPU cache, lifetime, and performance implications
on the stored data. Atomic operations disrupt the CPU cache; however, they're
only used when cloning or dropping an `Arc`. The structure of the animation
graphs are, for the most part, static. Likewise, the trait object use is likely
unavoidable so long as we rely on traits as a point of abstraction within the
graph. An alternative mgiht be to we want to transfer ownership of the curve, and
just make multiple copies of a potentially large and immutable animation data
buffer, but that comes with a signfigant memory and CPU cache performance cost.

## Prior art

This proposal is largely inspired by Unity's [Playable][playable] API, which has
a similar goal of building composable time-sequenced graphs for animation, audio,
and game logic. Several other game engines have very similar APIs and features:

 - Unreal has [AnimGraph][animgraph] for creating dynamic animations in
   Blueprints.
 - Godot has [animation trees][animation-trees] for creating dynamic animations in
   Blueprints.

The proposed API here doesn't purport or aim to directly replicate the features
seen in these other engines, but provide the absolute bare minimum API so that
engine developers or game developers can build them if they need to.

Currently nothing like this exists in the entire Rust ecosystem.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

[playable]: https://docs.unity3d.com/Manual/Playables.html
[animgraph]: https://docs.unrealengine.com/4.27/en-US/AnimatingObjects/SkeletalMeshAnimation/AnimBlueprints/AnimGraph/
[animation-trees]: https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html

## Unresolved questions

TODO: Complete

## Future possibilities

TODO: Complete

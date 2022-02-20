# Feature Name: `animation-primitives`

## Summary

Animation is particularly complex, with many stateful and intersecting
elements. This RFC aims to detail a set of lowest-level APIs for authoring and
playing animations within Bevy.

## Motivation

Animation is at the heart of modern game development. A game engine without an
animation system is generally not considered production-ready.

This RFC aims to detail the absolute lowest-level APIs to unblock ecosystem
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

This RFC primarily aims to resolve only problems within the domains of storage
and sampling. Composition will only be touched briefly upon for the purposes of
animated type selection. Application can be distinctly decoupled from these
earlier two stages, treating the sampled values as a black box output, and
authoring can be built separately upon the primitives provided by this RFC and
thus are explicit non-goals here.

## User-facing explanation

The end goal of this, and subsequent RFCs, is to deliver on a end-to-end
property-based animation system. Animation here specifically aims to provide
asset based, not-code based, workflows for altering *any* visibly mutable
property of a entity. This may include skeletal animation, where the bones deform
the verticies of a mesh, to make a 3D character run, but also may include
3D material animation, swapping sprites in a cycle for a 2D game,
enabling/disabling gameplay elements based on time and character movements
(i.e. hitboxes), etc.

To oversimplify the entire system, this involves a pipeline that samples values,
based on time, optionally alters or blends multiple samples together, then
applies the final value to a component in the ECS World.

The core of this system's storage is a trait called `Curve<T>` which defines a
a serializable set of time-sequenced values that a user can opaquely sample
values of type `T` at a provided time. `T` here can be anything considered
animatable. A few examples of high priority types to be supported here are:

 - `f32`/`f64`
 - `Vec2`/`Vec3`/`Vec3A`/`Vec4` (and their `f64` variants)
 - Any integer type, usually as a quantization of `f32` or `f64`.
 - `Transform` (see "Special Case: Mesh Deformation Animation" below)
 - `Color`
 - `bool` for toggling on and off features.
 - `Range<T>` for a range for randomly sampling from (think particle systems)
 - `Handle<T>` for sprite animation, though can be generically used for any asset
   swapping.
 - etc.

These curves can be used by themselves in dedicated contexts (i.e. particle
system sampling), or used as a part of a `AnimationClip` asset.
AnimationClips bundle multiple curves together in a cohesive manner to allow
sampling all curves in the clip together. Curves are keyed by the associated
`Reflect` path that is animated by the curve. AnimationClips are meant to sample
most, if not all, of the curves at the same time.

These base primitives can then be combined and composed together to build the
rest of the system, which will be detailed in one or more followup RFCs.

## Implementation strategy

Prototype implementation: https://github.com/HouraiTeahouse/bevy_prototype_animation

### `Curve<T>` Trait

TODO: Detail why this exists.

```rust
trait Curve<T> {
  fn duration(&self) -> f32;
  fn sample(&self, time: f32) -> T;
}
```

#### `Animatable` Trait

The sampled values of `Curve<T>` need to implement `Animatable`, a trait that
allows interpolating and blending the stored values in curves. The general trait
may look like the following:

```rust
struct BlendInput<T> {
  weight: f32,
  value: T,
}

trait Animatable {
  fn interpolate(a: &Self, b: &Self, time: f32) -> Self;
  fn blend(inputs: impl Iterator<Item=BlendInput<Self>>) -> Self;
}
```

`interpolate` implements interpolation between two values of a given type given a
time. This typically will be a [linear interpolation][lerp], and have the `time`
parameter clamped to the domain of [0, 1]. However, this may not necessarily be
strictly be a continuous interpolation for discrete types like the integral
types, `bool`, or `Handle<T>`. This may also be implemented as [spherical linear
interpolation][slerp] for quaternions.  This will typically be required to
provide smooth sampling from the variety of curve implementations. If it is
desirable to "override" the default lerp behavior, newtype'ing an underlying
`Animatable` type and implementing `Animatable` on the newtype instead.

`blend` expands upon this and provides a way to blend a collection of weighted
inputs into one output. This can be used as the base primitive implementation for
building more complex compositional systems. For typical numerical types, this
will often come out to just be a weighted sum. For non-continuous discrete types
like `Handle<T>`, it may select the highest weighted input.

A blanket implementation could be done on types that implement `Add +
Mul<Output=Self>`, though this might conflict with a need for specialized
implementations for the following types:
 - `Vec3` - needed to take advantage of SIMD instructions via `Vec3A`.
 - `Handle<T>` - need to properly use `clone_weak`.

[lerp]: https://en.wikipedia.org/wiki/Linear_interpolation
[slerp]: https://en.wikipedia.org/wiki/Slerp

#### Concrete Implementation: `CurveFixed`

`CurveFixed` is the typical initial attempt at implementing an animation curve.
It assumes a fixed frame rate and stores each frame as individual elements in a
`Vec<T>` or `Box<[T]>`. Sampling simply requires finding the corresponding
indexes in the buffer and calling `interpolate` on with the between the two
frames. This is very fast, typically incuring only the cost of one cache miss per
curve sampled. The main downside to this is the memory usage: a keyframe must be
stored every for `1 / framerate` seconds, even if the value does not change,
which may lead to storing a large amount of redundant data.

#### Concrete Implementation: `CurveVariable`

`CurveVariable`, like `CurveFixed`, stores its keyframe values in a contiguous
buffer like `Vec<T>`. However, it stores the exact keyframe times in a parallel
`Vec<T>`, allowing for the time between keyframes to be variable. This comes at a
cost: sampling the curve requires performing a binary search to find the
surroudning keyframe before sampling a value from the curve itself. This can
result in multiple cache misses just to sample from one curve.

This cost can be mitigated by using cursor based sampling, where sampling returns
a keyframe index alongside the sampled value, which can be reused when sampling
again to minimize the time spent searching for the correct value. This notably
complicates sampling from these curves

#### Concrete Implementation: `CurveVariableLinear`

TODO: Complete this section.

### Special Case: Mesh Deformation Animation (Skeletal/Morph)

The heaviest use case for this animation system is for 3D mesh deformation
animation. This includes [skeletal animation][skeletal_animation] and [morph
target animation][morph_target_animation]. This section will not get into the
requisite rendering techniques to display the results of said deformations and
only focus on how to store curves for animation.

Both `AnimationClip` and surrounding types that utilize it may need a specialized
shortcut for accessing and sampling `Transform` based poses from the clip.

[skeletal_animation]: https://en.wikipedia.org/wiki/Skeletal_animation
[morph_target_animation]: https://en.wikipedia.org/wiki/Morph_target_animation

### Curve Compression and Quantization

Curves can be quite big. Each buffer can be quite large depending on the number
of keyframes and the representation of said keyframes, and a typical game with
multiple character animations may several hundreds or thousands of curves to
sample from. To both help cut down on memory usage and minimize cache misses,
it's useful to compress the most usecases seen by a

`f32` is by far the most commonly animated type, and is foundational for building
more complex types (i.e. Vec2, Vec3, Vec4, Quat, Transform), which makes it a
prime target for compression. An effective method of compression is use of
quantization, where the upper and lower bounds of a curve are computed, and all
intermediate values are stored as discrete `u16`. This can cut memory usage of
`f32` curves by up to 50%. As `u16` can be converted back to `f32` losslessly,
the only loss in accuracy is when a source curve is quantized. An example of how
an implementation of how a struct might look can be seen below.

```rust
struct QuantizedFloatCurve {
  frame_rate: f32,
  start_time: f32,
  min_value: f32,
  increment: f32,
  frames: Vec<u16>,
}
```

One optional optimization for `Curve<bool>` is to use a bitvector to store values
instead a `Vec`, which would allow storing up to 8 fixed-framerate keyframes in
one byte.

Another common low hanging fruit for compression is static curves: curves with a
singular value throughout the full duration of the curve itself. This drops the
need to store a full `Vec` to store only one value. This can be easily
represented as a enum. Extending the above example implementation, a resultant
implemntation might look like the following:

```rust
enum CompressedFloatCurve {
  Static {
    start_time: f32,
    duration: f32,
  },
  Quantized {
    start_time: f32,
    frame_rate: f32,
    min_value: f32,
    increment: f32,
    frames: Vec<u16>,
  },
}
```

TODO: Add note about [quaternion compression][quat-compression]

We can then use this compose these compressed structs together to construct
more complex curves:

```rust
struct CompressedVec3Curve {
  x: CompressedVec3Curve,
  y: CompressedVec3Curve,
  z: CompressedVec3Curve,
}

struct CompressedTransformCurve {
  translation: CompressedVec3Curve,
  scale: CompressedVec3Curve,
  rotation: CompressedQuatCurve,
}
```

[quat_compression]: https://technology.riotgames.com/news/compressing-skeletal-animation-data

### `AnimationClip`

TODO: Complete this section.

## Drawbacks

The main drawback to this approach is the complexity. There are many different
implementors of `Curve<T>`, required largely out of necessity for performance. It
may be difficult to know which one to use for when and will require signifigant
documentation effort. The core also centering around `Curve<T>` as a trait also
encourages use of trait objects over concrete types, which may have runtime costs
associated with its use.

## Rationale and alternatives

Bevy absolutely needs some form of a first-party animation system, no modern game
engine can be called production ready without one. Having this exist solely as a
third-party ecosystem crate is unacceptable as it would promote a
facturing of the ecosystem with multiple incompatible baseline animation system
implementations.

## Prior art

On the subject of animation data compression, [ACL][acl] aims to provide a set of
animation compression algorithms for general use in game engines. It is
unfortunately written in C++, which may make it difficult to directly integrate
with Bevy, but it's open sourced under MIT, which would make a reimplementation
of its algorithms in Rust compatible with Bevy.

[acl]: https://technology.riotgames.com/news/compressing-skeletal-animation-data

## Unresolved questions

 - Can we safely interleave multiple curves together so that we do not incur
   mulitple cache misses when sampling from animation clips?

## Future possibilities

TODO: Complete

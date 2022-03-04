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
and sampling. The following will only be touched upon briefly:

 - Composition will only be touched briefly upon for the purposes of animated
   type selection.
 - Application is only touched upon in relation to how to select which property
   is to be animated by which asset, as well as sampled value post-processing.
   The rest can be distinctly decoupled from these earlier two stages, treating
   the sampled values as a black box output.

Authoring can be built separately upon the primitives provided by this RFC and
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

### `Animatable` Trait
To define values that can be properly smoothly sampled and composed together, a
trait is needed to determine the behavior when interpolating and blending values
of the type together. The general trait may look like the following:

```rust
struct BlendInput<T> {
  weight: f32,
  value: T,
}

trait Animatable {
  fn interpolate(a: &Self, b: &Self, time: f32) -> Self;
  fn blend(inputs: impl Iterator<Item=BlendInput<Self>>) -> Option<Self>;
  unsafe fn post_process(&mut self, world: &World) {}
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
like `Handle<T>`, it may select the highest weighted input. Even though a
iterator is inherently ordered in some way, the result provided by `blend` must
be order invariant for all types. If the provided iterator is empty, `None`
should be returned to signal that there were no values to blend.

A blanket implementation could be done on types that implement `Add +
Mul<Output=Self>`, though this might conflict with a need for specialized
implementations for the following types:
 - `Vec3` - needed to take advantage of SIMD instructions via `Vec3A`.
 - `Handle<T>` - need to properly use `clone_weak`.

An unsafe `post_process` trait function is going to be required to build values
that are dependent on the state of the World. An example of this is `Handle<T>`,
which requires strong handles to be used properly: a `Curve<HandleId>` can
implement `Curve<Handle<T>>` by postprocessing the `HandleId` by reading the
associated `Assets<T>` resource to make a strong handle. This is applied only
after blending is applied so post processing is only applied once per sampled
value. This function is unsafe by default as it may be unsafe to read any
non-Resource or NonSend resource from the World if application is run over
multiple threads, which may cause aliasing errors if read. Other unsafe
operations that mutate the World from a read-only reference is also unsound. The
default implementation here is a no-op, as most implementations do not need this
functionality, and will be optimized out via monomorphization.

[lerp]: https://en.wikipedia.org/wiki/Linear_interpolation
[slerp]: https://en.wikipedia.org/wiki/Slerp

### `Curve<T>` Trait
To store the raw values that can be sampled from, we introduce the concept of an
animation curve. An animation curve defines the function over which a value
evolves over a set period of time. These curves typically define a set of timed
keyframes that paramterize how the value changes over the established time
period.

We may have multiple ways an animation curve may store it's underlying keyframes,
so a generic trait `Curve<T>` can be defined as follows:

```rust
trait Curve<T: Animatable> : Serialize, DeserializeOwned {
  fn duration(&self) -> f32;
  fn sample(&self, time: f32) -> T;
}
```

A curve always covers the time range of `[0, duration]`, but must return valid
values when sampled at any non-negative time. What a curve does beyond this range
is implementation specifc. This typically will just return the value that would
be sampled at time 0 or `duration`, whichever is closer, but it doesn't have to
be.

As shown in the above trait definition, curves must be serializable in some form,
either directly implementing `serde` traits, or via `Reflect`. This serialization
must be accessible even when turned into a trait object. This is required for
`AnimationClip` asset serialization/deserialization to work.

#### `Animatable` Newtype Pattern
Since `Curve<T>::sample` returns `T` and `T` is not a associated type but rather
a generic parameter, a type can implement multiple variants of `Curve`. One
possible common pattern is to define a newtype around `T` that defines new
interpolation or blending behavior when implementing `Animatable`. A blanket impl
can then be made for curves of the newtype as follows:

```rust
impl<C: Curve<NewType>> Curve<T> for C {
    fn duration(&self) -> f32 {
        <Self as Curve<NewType>>::duration()
    }
    fn sample(&self, time: f32) -> T {
        <Self as Curve<NewType>>::sample(time)
          .map(|value| value.0)
    }
}
```

One very common example of this might be quaternions, where either normal linear
interpolation can be used, or spherical linear interpolation can be used instead.
The underlying storage would not change, but the interpolation behavior would.

#### Concrete Implementation: `CurveFixed`
`CurveFixed` is the typical initial attempt at implementing an animation curve.
It assumes a fixed frame rate and stores each frame as individual elements in a
`Vec<T>` or `Box<[T]>`. Sampling simply requires finding the corresponding indexes
in the buffer and calling `interpolate` on with the between the two frames. This
is very fast, typically incuring only the cost of one cache miss per curve
sampled. The main downside to this is the memory usage: a keyframe must be stored
for every `1 / framerate` seconds, even if the value does not change, which may
lead to storing a large amount of redundant data.

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

#### Curve Compression and Quantization
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

### PropertyPath
To reduce reliance on expensive runtime parsing and raw stringly-typed
operations, the path to which a property is animated can be parsed into a
`PropertyPath` struct, which contains the metadata for fetching the correct
property in an component to animate during application. Property paths roughly
take the form of `path/to/entity@crate::module::Component.field.within.component`.

The first part of the path is the named path in the Entity hierarchy. This relies
on the `Name` component to be populated at every Entity along a Transform
hierarchy to be present. If there are multiple children at any level with the
same name, the first child in `Children` with the matching name will be used.

The second part of path is the component within the entity that is being
animated. This is the fullly qualified type name of the component being animated.
Dynamic components are not supported by this initial design.

The third and final part of the path is a field path within the given component.
This should be a vailid path usable with the `GetPath` trait functions. This is
likely to be the most costly operation when implementing application, so a
pre-parsed subpath struct `FieldPath` should be used here if possible.

A rough outline of how such a `PropertyPath` struct might look like:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, ParitialOrd)]
pub struct EntityPath {
  parts: Box<[Name]>,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, ParitialOrd)]
pub struct PropertyPath {
  entity: EntityPath,
  component_type_id: TypeId,
  component_name: String,
  field: FieldPath,
}
```

A component TypeId will be stored alongside the component name. The TypeId is
required to remove string based component name lookups during application.
Using the TypeId also accelerates `Eq`, `Ord`, and `Hash` implementations as
these paths are commonly used as HashMap and BTreeMap keys. The raw component
name is only used when converting back into a raw string.

As a result of needing runtime type information, serialization and deserialzation
cannot be implemented with serde derive macros, but will require a `TypeRegistry`
to handle the TypeId lookup. For simpler file representations, upon serialization,
PropertyPaths are written out as raw strings, and parsed on deserialization.

### `AnimationClip`
A `AnimationClip` is a serializable asset type that encompasses a mapping of
property paths to curves. It effectively acts as a `HashMap<PropertyPath,
Curve<T>>`, but will require a bit of type erasure to ensure that the generics of
each curve is not visible in the concrete type definition.

As PropertyPath cannot be serialized without a TypeRegistry, AnimationClips will
also require one to be present during both serialization and deserialization. The
type erasure of each curve will also likely require `typetag`-like serialization
using the `TypeRegistry`.

#### Special Case: Mesh Deformation Animation (Skeletal/Morph)
The heaviest use case for this animation system is for 3D mesh deformation
animation. This includes [skeletal animation][skeletal_animation] and [morph
target animation][morph_target_animation]. This section will not get into the
requisite rendering techniques to display the results of said deformations and
only focus on how to store curves for animation.

Both `AnimationClip` and surrounding types that utilize it may need a specialized
shortcut for accessing and sampling `Transform` based poses from the clip. This
will likely take the form of a separate internal map from EntityPath to a
concrete curve type for Transform to avoid any dynamic dispatch overhead, and
dedicated lookup and sampling functions explicitly for this special case.

[skeletal_animation]: https://en.wikipedia.org/wiki/Skeletal_animation
[morph_target_animation]: https://en.wikipedia.org/wiki/Morph_target_animation

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

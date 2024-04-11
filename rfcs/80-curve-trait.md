Feature Name: `curve-trait`

## Summary

This RFC introduces a general trait API, `Curve<T>`, for shared functionality of curves within Bevy's
ecosystem. This encompasses both *abstract* curves, such as those produced by splines (and hence defined
by equations), as well as *concrete* curves that are defined by sampling and interpolation (e.g. 
animation keyframes), among others. 

The purpose is to provide a common baseline of useful functionality that unites these disparate
implementations, making curves of many kinds pleasant to work with for authors of plugins and games.

## Motivation

Curves tend to have quite disparate underlying implementations at the level of data. As a result,
they are prone to causing ecosystem fragmentation, as authors of various plugins (including internal ones)
are liable to create the curve abstractions that suit their own particular needs.

By providing a common baseline in `Curve<T>`, we can ensure a significant degree of portability between these
disparate areas and, ideally, allow for authors to reuse the work of others rather than reinventing the
wheel every time they want to write libraries that involve generating or working with curves. 

Furthermore, something like this is also a prerequisite to bringing more complex geometric operations
(some of them with broad implications) into `bevy_math` itself. For instance, this lays the natural bedwork
for more specialized libraries in curve geometry, such as those needed by curve extrusion (e.g. for mesh
generation of roads and other surfaces). Without an internal curve abstraction in Bevy itself, we would
be forced to repeat our work for each class of curves that we want to work with.

## User-facing explanation

### Introduction

The trait `Curve<T>` provides a generalized API surface for curves. Its requirements are quite simple:
```rust
pub trait Curve<T>
where
    T: Interpolable,
{
    fn duration(&self) -> f32;
    fn sample(&self, t: f32) -> T;
}
```
At a basic level, it encompasses values of type `T` parametrized over an interval that starts at 0 and ends
at the curve's given `duration`, allowing that data to be sampled by providing the associated time parameter. 

### Interpolation

The only requirement of `T` to be used with `Curve<T>` is that it implements the `Interpolable` trait, so we
will briefly examine that next:

```rust
pub trait Interpolable: Clone {
    fn interpolate(&self, other: &Self, t: f32) -> Self;
}
```

As you can see, in addition to the `Clone` requirement, `Interpolable` requires a single method, `interpolate`,
which takes two items by reference and produces a third by interpolating between them. The idea is that
when `t` is 0, `self` is recovered, whereas when `t` is 1, you receive `other`. Intermediate values are to be
obtained by an interpolation process that is provided by the implementation. 

(Intuitively, `Clone` is required by `Interpolable` because at the endpoints 0 and 1, clones of the starting and
ending points should be returned. Frequently, `Interpolable` data will actually be `Copy`.)

For example, Bevy's vector types (including `f32`, `Vec2`, `Vec3`, etc.) implement `Interpolable` using linear 
interpolation (the `lerp` function here):
```rust
impl<T> Interpolable for T
where
    T: VectorSpace,
{
    fn interpolate(&self, other: &Self, t: f32) -> Self {
        self.lerp(*other, t)
    }
}
```
Other forms of interpolation are possible; for example, given two points and tangent vectors at each of
them, one might perform Hermite interpolation to create a cubic curve between them, whose points and tangent
values can then be sampled at any point on the curve. Another common example would be "spherical linear
interpolation" (`slerp`) of quaternions, used in animation and other rigid motions. 

To aid you in using this trait, `Interpolable` has blanket implementations for tuples whose members are
`Interpolable` (which simultaneously interpolate each constituent); in the same way, the `Interpolable` trait 
can be derived for structs whose members are `Interpolable` with `#[derive(Interpolable)]`:
```rust
impl<S, T> Interpolable for (S, T)
where
    S: Interpolable,
    T: Interpolable,
{ //... }
//... And so on for (S, T, U), etc.
```
```rust
#[derive(Interpolable)]
struct MyCurveData {
    position: Vec3,
    velocity: Vec3,
}
```

### The Curve API

Now, let us turn our attention back to `Curve<T>`, which exposes a functional API similar to that of `Iterator`.
We will explore its main components one-by-one. The first of those is `map`:
```rust
/// Create a new curve by mapping the values of this curve via a function `f`; i.e., if the
/// sample at time `t` for this curve is `x`, the value at time `t` on the new curve will be
/// `f(x)`.
fn map<S>(self, f: impl Fn(T) -> S) -> impl Curve<S>
where
    Self: Sized,
    S: Interpolable,
{ //... }
```
As you can see, `map` takes our curve and, consuming it, produces a new curve whose sample values are the images
under `f` of those from the function that we started with. For example, if we started with a curve in 
three-dimensional space and wanted to project it onto the XY-plane, we could do that with `map`:
```rust
// A 3d curve, implementing `Curve<Vec3>`
let my_3d_curve = function_curve(2.0, |t| Vec3::new(t * t, 2.0 * t, t - 1.0));
// Its 2d projection, implementing `Curve<Vec2>`
let my_2d_curve = my_3d_curve.map(|v| Vec2::new(vec.x, vec.y));
```
As you might expect, `map` is lazy like its `Iterator` counterpart, so the function it takes as input is only
evaluated when the resulting curve is actually sampled.

---

Next up is `reparametrize`, one of the most important API methods:
```rust
/// Create a new [`Curve`] whose parameter space is related to the parameter space of this curve
/// by `f`. For each time `t`, the sample from the new curve at time `t` is the sample from
/// this curve at time `f(t)`. The given `duration` will be the duration of the new curve. The
/// function `f` is expected to take `[0, duration]` into `[0, self.duration]`.
fn reparametrize(self, duration: f32, f: impl Fn(f32) -> f32) -> impl Curve<T>
where
    Self: Sized,
{ //... }
```
As you can see, `reparametrize` is like `map`, but the function is applied in parameter space instead of in
output space. This is somewhat counterintuitive, because it means that many of the functions that we might want
to use it with actually need to be inverted. For example, here is how you would use `reparametrize` to change
the `duration` of a curve from `1.0` to `2.0` by linearly stretching it:
```rust
let my_curve = function_curve(1.0, |x| x + 1.0);
let dur = my_curve.duration();
let scaled_curve = my_curve.reparametrize(dur * 2.0, |t| t / 2.0);
```
(A convenience method `reparametrize_linear` exists for this specific thing.)

However, `reparametrize` is vastly more powerful than just this. For example, here we use `reparametrize` to
create a new curve that is a segment of the original one:
```rust
let my_curve = function_curve(1.0, |x| x * 2.0);
// The segment of `my_curve` from `0.5` to `1.0`:
let curve_segment = my_curve.reparametrize(0.5, |t| 0.5 + t);
```

And here, we use it to reverse our curve:
```rust
let my_curve = function_curve(2.0, |x| x * x);
let dur = my_curve.duration();
let reversed_curve = my_curve.reparametrize(dur, |t| dur - t);
```

And here, we reparametrize by an easing curve:
```rust
let my_curve = function_curve(1.0, |x| x + 5.0);
let easing_curve = function_curve(1.0, |x| x * x * x);
let eased_curve = my_curve.reparametrize(1.0, |t| easing_curve.sample(t));
```
(The latter also has a convenience method, `reparametrize_by_curve`, which handles the duration automatically.)

---

Next, we have `graph`, the last of the general functional methods of the API:
```rust
/// Create a new [`Curve`] which is the graph of this one; that is, its output includes the
/// parameter itself in the samples. For example, if this curve outputs `x` at time `t`, then
/// the produced curve will produce `(t, x)` at time `t`.
fn graph(self) -> impl Curve<(f32, T)>
where
    Self: Sized,
{ //... }
```
This is a subtle method whose main applications involve allowing more complex things to be expressed. For example,
here we modify a curve by making its output value attenuate with time:
```rust
// A curve with a value of `3.0` over a duration of `5.0`:
let my_curve = const_curve(5.0, 3.0);
// The same curve but with exponential falloff in time:
let new_curve = my_curve.graph().map(|(t, x)| x * (-t).exp2());
```

Most general operations that one could think up for curves can be achieved by clever combinations of `map` and
`reparametrize`, perhaps with a little `graph` sprinkled in.

---

The next two API methods concern themselves with more imperative (i.e. data-focused) matters. Often one needs to 
take a curve and actually render it concrete by sampling it at some resolution — for the purposes of storage, 
serialization, application of numerical methods, and so on. That's where `resample` comes in:
```rust
/// Resample this [`Curve`] to produce a new one that is defined by interpolation over equally
/// spaced values. A total of `samples` samples are used.
fn resample(&self, samples: usize) -> SampleCurve<T> { //... }
```
The main thing to notice about this is that, unlike the previous functional methods which merely spit out some kind
of `impl Curve<T>`, `resample` has a concrete return type of `SampleCurve<T>`, which is an actual struct that holds
actual data that you can access. 

Notably, `SampleCurve<T>` still implements `Curve<T>`, but its curve implementation is set in stone: it uses
equally spaced sample points together with the interpolation provided by `T` to provide something which is loosely
equivalent to your original curve, perhaps with a loss of fine detail. The sampling resolution is controlled by
the `samples` parameter, so higher values will ensure something that closely matches your curve.

For example, with a `Curve<T>` whose `T` performs linear interpolation, a `samples` value of 2 will yield a 
`SampleCurve<T>` that represents a line passing between the two endpoints, and as `samples` grows larger,
the original curve is approximated by greater and greater quantities of line segments equally spaced in its parameter
domain. 

Commonly, `resample` is used to obtain something more concrete after applying the `map` and `reparametrize` methods:
```rust
let my_curve = function_curve(3.0, |x| x * 2.0 + 1.0);
let modified_curve = my_curve.reparametrize(1.0, |t| 3.0 * t);
for (t, v) in modified_curve.graph().resample(100).samples {
    println!("Value of {v} at time {t}");
}
```

A variant of `resample` called `resample_uneven` allows for choosing the sample points directly instead of having them
evenly spaced:
```rust
/// Resample this [`Curve`] to produce a new one that is defined by interpolation over samples
/// taken at the given set of times. The given `sample_times` are expected to be strictly
/// increasing and nonempty.
fn resample_uneven(&self, sample_times: impl IntoIterator<Item = f32>) -> UnevenSampleCurve<T> { //... }
```
This is useful for strategically saving space when performing serialization and storage.

### Making curves

The curve-creation functions `constant_curve` and `function_curve` that we have been using in examples are, in fact,
real functions that are part of the library. They look like this:
```rust
/// Create a [`Curve`] that constantly takes the given `value` over the given `duration`.
pub fn constant_curve<T: Interpolable>(duration: f32, value: T) -> impl Curve<T> { //... }
```
```rust
/// Convert the given function `f` into a [`Curve`] with the given `duration`, sampled by
/// evaluating the function.
pub fn function_curve<T, F>(duration: f32, f: F) -> impl Curve<T>
where 
    T: Interpolable,
    F: Fn(f32) -> T,
{ //... }
```
Note that, while the examples used only functions `f32 -> f32`, `function_curve` can convert any function of the 
parameter domain (valued in something `Interpolable`) into a `Curve<T>`. For example, here is a rotation over time,
expressed as a `Curve<Quat>` using this API:
```rust
let rotation_curve = function_curve(f32::consts::TAU, |t| Quat::from_rotation_z(t));
```
Furthermore, all of `bevy_math`'s curves (e.g. those created by splines) implement `Curve<T>` for suitable values of
`T`, including information like derivatives in addition to positional data. Additionally, authors of other Bevy
libraries and internal modules may provide additional `Curve<T>` implementors, either to provide functionality
for specific problem domains or to expand the variety of curve constructions available in the Bevy ecosystem.

It is worth remembering that implementing `Curve<T>` yourself, too, is extraordinarily straightforward, since the
only required methods are `duration` and `sample`, so you can hook into this API functionality yourself with ease.

## Implementation strategy

The API is really segregated into two parts, the functional and the concrete. The functional part, whose outputs are
only guaranteed to be `impl Curve<T>` of some kind, uses wrapper structs for its outputs, which take ownership of 
the original curve data, along with any closures needed to perform combined sampling. For example, `map` is powered
by `MapCurve<S, T, C, F>`, which looks like this:
```rust
/// A [`Curve`] whose samples are defined by mapping samples from another curve through a
/// given function.
pub struct MapCurve<S, T, C, F>
where
    S: Interpolable,
    T: Interpolable,
    C: Curve<S>,
    F: Fn(S) -> T,
{
    preimage: C,
    f: F,
    _phantom: PhantomData<(S, T)>,
}

impl<S, T, C, F> Curve<T> for MapCurve<S, T, C, F>
where
    S: Interpolable,
    T: Interpolable,
    C: Curve<S>,
    F: Fn(S) -> T,
{
    fn duration(&self) -> f32 {
        self.preimage.duration()
    }
    fn sample(&self, t: f32) -> T {
        (self.f)(self.preimage.sample(t))
    }
}
```
There are two things to be noted here:
- Since it takes ownership of a closure along with the source curve, which has unspecified serialization properties,
a `MapCurve` cannot in general be serialized or used in storage.
- This is just the default implementation, which can be overridden by other constructions where it makes sense.

The implementation of `reparametrize` is similar, relying on a `ReparamCurve<T, C, F>` which owns the source curve
in addition to the reparametrization function. The function `graph` is powered by `GraphCurve<T>` which, like the others,
must also own its source data; however, it doesn't require any function information, so it is essentially a plain
wrapper struct on the level of data (providing only a different implementation of `Curve::sample`).

On the other hand, the "concrete" part of the API consists of `SampleCurve<T>`, `UnevenSampleCurve<T>`, and the 
methods that yield them. These are implemented essentially as one would imagine, holding vectors of data and interpolating
it in `Curve::sample`. E.g.:

```rust
/// A [`Curve`] that is defined by neighbor interpolation over a set of samples.
pub struct SampleCurve<T>
where
    T: Interpolable,
{
    duration: f32,

    /// The list of samples that define this curve by interpolation.
    pub samples: Vec<T>,
}
// ...

    #[inline]
    fn sample(&self, t: f32) -> T {
        let num_samples = self.samples.len();
        // If there is only one sample, then we return the single sample point. We also clamp `t`
        // to `[0, self.duration]` here.
        if num_samples == 1 || t <= 0.0 {
            return self.samples[0].clone();
        }
        if t >= self.duration {
            return self.samples[self.samples.len() - 1].clone();
        }

        // Inside the curve itself, interpolate between the two nearest sample values.
        let subdivs = num_samples - 1;
        let step = self.duration / subdivs as f32;
        let lower_index = (t / step).floor() as usize;
        let upper_index = (t / step).ceil() as usize;
        let f = (t / step).fract();
        self.samples[lower_index].interpolate(&self.samples[upper_index], f)
    }
```

The main thing here is that `SampleCurve<T>` is actually returned *by type* from the `resample` API,
and it is clearly suitable for serialization and storage in addition to numerical applications. This
allows consumers to work flexibly with the functional API and then cast down to concrete values when
they need them. (And of course, `UnevenSampleCurve<T>` is implemented similarly, instead using a binary
search in its sequence of time-values to find its interpolation interval.)

Because these types actually store concrete sample data, they have special implementations of `map`
which are not lazy, instead returning values of type `SampleCurve<S>`/`UnevenSampleCurve<S>`. These
can be accessed from a type-level API as `map_concrete`, so that the user can avoid erasing the
type if it is convenient to do so. The same goes for `graph`; however, because of its contravariance,
the same cannot be said of `reparametrize`, which maintains its default functional implementation.

When `resample` and `resample_uneven` are told to sample 0 or 1 points, they should panic or return an error.
Empty curves are not considered conceptually valid.

Everything else should be fairly clear based on the user-facing API descriptions. 

## Drawbacks

The main risk in implementing this is that we cement a poor or incomplete curve API within the Bevy
ecosystem, so that authors of internal and external modules need to sidestep it or do reimplementation
themselves. To avoid this, we should ensure that this system is adequately all-encompassing if we
choose to adopt it.

## Rationale and alternatives

An API like `Curve<T>` is natural for a problem domain in which underlying data is extremely varied;
the only real alternatives involve cementing ourselves around limited data-implementations that are
necessarily inadequate or inefficient for some problem domains. By contrast, the `Curve<T>` API is
extremely flexible, allowing concrete data to interoperate with function closures before being 
resampled again, serialized, and so on. These benefits apply immediately to any curve implementation 
that can satisfy the measly requirements of `Curve<T>`. 

One of the main apparent drawbacks of this interface is that the functional API methods `map`, 
`reparametrize`, and `graph` implicitly perform type erasure, and this has implications for
serialization. For instance, even if an implementor `MyCurveType: Curve<T>` also holds serialization
and deserialization traits (or other traits of relevance), and *even if these traits are implemented
by the output type of its mapping methods*, they cannot be accessed in the output value. I believe
that, if the need arises, this can be addressed by developing a small extension (say, `ConcreteCurve<T>`)
to `Curve<T>`, with methods like `map_concrete` whose output data are known to have stronger
guarantees. With just the present API alone, this would have to be addressed by resampling.

This cannot merely be an ecosystem crate because the purpose is for a part of the ecosystem to 
centralize around it.

Finally, if we do not implement something like this, we lose out on the opportunity to centralize
a fragemented area. We also lose out on the opportunity to provide exceptionally useful functionality 
to a great many users and inadvertently stifling future innovation in related areas, since implementation
burden becomes so much greater.

## Unresolved questions

Do we need more than this at the level of `bevy_math` to meet the needs of `bevy_animation` and perhaps `bevy_audio`?

Are there other major stakeholders that I haven't thought of?

I would also like additional thoughts related to the matter of serialization/deserialization that I raised
in the preceding section.

## Future possibilities

This paves the groundwork for future work on curve geometry. Among the lowest-hanging fruit after the
fact would be an algorithmic implementation of the Implicit Function Theorem, which also swiftly leads
to arclength reparametrization of any curve with enough data (e.g. positions and derivatives), a feature
that has been requested in the context of cubic curves already. 

In particular, I imagine a library built on top of this which takes specifically geometric curve data
and operates algorithmically on it to produce things like:
- derivative estimation from positions (easy but non-obvious)
- arclength estimation and reparametrization (easy with IFT)
- rotation-minimizing frames for arbitrary regular curves (easy)

Further along that path are things like mesh extrusion. In particular, I really view this as the first step
in the program of making roads in Bevy. 

It's possible also that the API itself might like to be expanded — by introducing further convenience methods,
more sophistocated forms of resampling, and so on. Some of these will, no doubt, rely on additional 
specialization of the interface to more particular values of `T` that are more closely associated to 
particular problem domains.


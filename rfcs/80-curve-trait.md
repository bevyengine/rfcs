Feature Name: `curve-trait`

## Summary

This RFC introduces a general trait API, `Curve<T>`, for shared functionality of curves within Bevy's
ecosystem. This encompasses both *abstract* curves, such as those produced by splines (and hence defined
by equations), as well as *concrete* curves that are defined by sampling and interpolation (e.g. 
animation keyframes), among others. 

The purpose is to provide a common baseline of useful functionality that unites these disparate
implementations, making curves of many kinds pleasant to work with for authors of plugins and games.
Common uses of curves in games include:
* Describing animations as transformations over time
* Describing movement of game entities
* Describing camera moves
* Describing easing and attenuation with time and/or distance
* Describing geometry (e.g. of roads, surfaces of extrusion, etc.)
* Describing paths of particles and effects
* Defining experience and drop rate curves

## Motivation

Curves tend to have quite disparate underlying implementations at the level of data. As a result,
they are prone to causing ecosystem fragmentation, as authors of various plugins (including internal ones)
are liable to create the curve abstractions that suit their own particular needs.

By providing a common baseline in `Curve<T>`, we can ensure a significant degree of portability between these
disparate areas and, ideally, allow for authors to reuse the work of others rather than reinventing the
wheel every time they want to write libraries that involve generating or working with curves. 

Ideally, many APIs that consume curves will be able to use something like `impl Curve<T>` (or even `dyn Curve<T>`
where it makes sense) instead of relying on specific representations.

Furthermore, something like this is also a prerequisite to bringing more complex geometric operations
(some of them with broad implications) into `bevy_math` itself. For instance, this lays the natural foundation
for more specialized libraries in curve geometry, such as those needed by curve extrusion (e.g. for mesh
generation of roads and other surfaces). Without an internal curve abstraction in Bevy itself, we would
be forced to repeat our work for each class of curves that we want to work with.

## User-facing explanation

### Introduction

The trait `Curve<T>` provides a generalized API surface for curves. In principle, a `Curve<T>` is a family
of values of type `T` parametrized over some interval which is typically thought of as something like time
or distance.

For example, we might use the `Curve` APIs to construct a `Curve<Transform>` in order to describe the motion 
of an Entity with time:
```rust
// This is a `Curve<Quat>` that describes a rotation with time:
let rotation_over_time = function_curve(interval(0.0, std::f32::consts::TAU).unwrap(), |t| Quat::from_rotation_z(t));
// Here is a `Curve<Vec3>` that describes a translation with time:
let translation_over_time = function_curve(interval(0.0, 1.0).unwrap(), |t| Vec3::splat(t));
// Let's reparametrize the rotation so that it goes from 0 to 1 instead:
let new_rotation = rotation_over_time.reparametrize_linear(interval(0.0, 1.0).unwrap()).unwrap();
// Combine the two into a `Curve<(Vec3, Quat)>` by zipping them together:
let translation_and_rotation = translation_over_time.zip(new_rotation).unwrap();
// Join the two families of data to get a `Curve<Transform>`:
let transform_curve = translation_and_rotation.map(|(t, r)| Transform::from_translation(t).with_rotation(r));
```

This could be used to actually set an entity's `Transform` in a system or otherwise:
```rust
*my_entity_transform = transform_curve.sample(0.6);
```

However, `Curve` is quite general, and it can be used for much more, including describing and manipulating
easings, animations, geometry, camera moves, and more!

The trait itself looks like this:

```rust
pub trait Curve<T>
{
    fn domain(&self) -> Interval;
    fn sample(&self, t: f32) -> T;
}
```
At a basic level, it encompasses values of type `T` parametrized over some particular `Interval` (its `domain`,
allowing that data to be sampled by providing the associated time parameter. 

### Intervals

The `Interval` type plays a part of the definition of each `Curve`. This is a type which
represents a nonempty closed interval over the `f32` numbers whose endpoints may be at infinity. Its provided methods are
mostly self-explanatory:
```rust
/// Create a new [`Interval`] with the specified `start` and `end`. The interval can be infinite
/// but cannot be empty; invalid parameters will result in an error.
pub fn new(start: f32, end: f32) -> Result<Self, InvalidIntervalError> { //... }

/// Get the start of this interval.
pub fn start(self) -> f32 { //... }

/// Get the end of this interval.
pub fn end(self) -> f32 { //... }

/// Get the length of this interval. Note that the result may be infinite (`f32::INFINITY`).
pub fn length(self) -> f32 { //... }

/// Returns `true` if this interval is finite.
pub fn is_finite(self) -> bool { //... }

/// Returns `true` if `item` is contained in this interval.
pub fn contains(self, item: f32) -> bool { //... }

/// Create an [`Interval`] by intersecting this interval with another. Returns an error if the
/// intersection would be empty (hence an invalid interval).
pub fn intersect(self, other: Interval) -> Result<Interval, InvalidIntervalError> { //... }

/// Clamp the given `value` to lie within this interval.
pub fn clamp(self, value: f32) -> f32 { //... }

/// Get the linear map which maps this curve onto the `other` one. Returns an error if either
/// interval is infinite.
pub fn linear_map_to(self, other: Self) -> Result<impl Fn(f32) -> f32, InfiniteIntervalError> { //... }

/// Get an iterator over equally-spaced points from this interval in increasing order.
/// Returns an error if `points` is less than 2 or if the interval is unbounded.
pub fn spaced_points(
    self,
    points: usize,
) -> Result<impl Iterator<Item = f32>, SpacedPointsError> {
```

The `Interval` type also implements `TryFrom<RangeInclusive>`, which may be desirable if you want to use
the `start..=end` syntax. One of the primary benefits of `Interval` (in addition to these methods) is
that it is `Copy`, so it is easy to take intervals and throw them around. 

### Sampling

The `Curve::sample` method is not intrinsically constrained by the curve's `domain` interval. Instead,
implementors of `Curve<T>` are free to determine how samples drawn from outside the `domain` will behave.
However, variants of `sample` (as well as other important methods) use the `domain` explicitly:
```rust
/// Sample a point on this curve at the parameter value `t`, returning `None` if the point is
/// outside of the curve's domain.
fn sample_checked(&self, t: f32) -> Option<T> { //... }

/// Sample a point on this curve at the parameter value `t`, clamping `t` to lie inside the
/// domain of the curve.
fn sample_clamped(&self, t: f32) -> T { //... }
```

### The Main Curve API

Now, let us turn our attention back to `Curve<T>`, which exposes a functional API similar to that of `Iterator`.
We will explore its main components one-by-one. The first of those is `map`:
```rust
/// Create a new curve by mapping the values of this curve via a function `f`; i.e., if the
/// sample at time `t` for this curve is `x`, the value at time `t` on the new curve will be
/// `f(x)`.
fn map<S>(self, f: impl Fn(T) -> S) -> impl Curve<S>
where
    Self: Sized,
{ //... }
```
As you can see, `map` takes our curve and, consuming it, produces a new curve whose sample values are the sample
values of the starting curve mapped through `f`. For example, if we started with a curve in three-dimensional space
and wanted to project it onto the XY-plane, we could do that with `map`:
```rust
// A 3d curve, implementing `Curve<Vec3>`
let my_3d_curve = function_curve(interval(0.0, 2.0).unwrap(), |t| Vec3::new(t * t, 2.0 * t, t - 1.0));
// Its 2d projection, implementing `Curve<Vec2>`
let my_2d_curve = my_3d_curve.map(|v| Vec2::new(vec.x, vec.y));
```
As you might expect, `map` is lazy like its `Iterator` counterpart, so the function it takes as input is only
evaluated when the resulting curve is actually sampled.

---

Next up is `reparametrize`, another of the most important API methods:
```rust
/// Create a new [`Curve`] whose parameter space is related to the parameter space of this curve
/// by `f`. For each time `t`, the sample from the new curve at time `t` is the sample from
/// this curve at time `f(t)`. The given `domain` will be the domain of the new curve. The
/// function `f` is expected to take `domain` into `self.domain()`.
fn reparametrize(self, domain: Interval, f: impl Fn(f32) -> f32) -> impl Curve<T>
where
    Self: Sized,
{ //... }
```
As you can see, `reparametrize` is like `map`, but the function is applied in parameter space instead of in
output space. This is somewhat counterintuitive, because it means that many of the functions that we might want
to use it with actually need to be inverted. For example, here is how you would use `reparametrize` to change
the `domain` of a curve from `[0.0, 1.0]` to `[0.0, 2.0]` by linearly stretching it:
```rust
let my_curve = function_curve(interval(0.0, 1.0).unwrap(), |x| x + 1.0);
let domain = my_curve.domain();
let scaled_curve = my_curve.reparametrize(interval(0.0, 2.0).unwrap(), |t| t / 2.0);
```
(A convenience method `reparametrize_linear` exists for this specific kind of thing.)

However, `reparametrize` can do much more than this alone. For example, here we use `reparametrize` to
create a new curve that is a segment of the original one:
```rust
let my_curve = function_curve(interval(0.0, 1.0).unwrap(), |x| x * 2.0);
// The segment of `my_curve` from `0.5` to `1.0`, shifted back to `0.0:
let curve_segment = my_curve.reparametrize(interval(0.0, 0.5).unwrap(), |t| 0.5 + t);
```

And here, we use it to reverse our curve:
```rust
let my_curve = function_curve(interval(0.0, 2.0).unwrap(), |x| x * x);
let domain = my_curve.domain();
let reversed_curve = my_curve.reparametrize(domain, |t| domain.end() - t);
```

And here, we reparametrize by an easing curve:
```rust
let my_curve = function_curve(interval(0.0, 1.0).unwrap(), |x| x + 5.0);
let easing_curve = function_curve(interval(0.0, 1.0).unwrap(), |x| x * x * x);
let eased_curve = my_curve.reparametrize(easing_curve.domain(), |t| easing_curve.sample(t));
```
(The latter also has a convenience method, `reparametrize_by_curve`, which handles the domain automatically.)

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
// A curve with a value of `3.0` over the `Interval` from `0.0` to `5.0`:
let my_curve = const_curve(interval(0.0, 5.0).unwrap(), 3.0);
// The same curve but with exponential falloff in time:
let new_curve = my_curve.graph().map(|(t, x)| x * (-t).exp2());
```

Most general operations that one could think up for curves can be achieved by clever combinations of `map` and
`reparametrize`, perhaps with a little `graph` sprinkled in.

### Sampling, resampling, and interpolation

The next part of the API concerns itself with more imperative (i.e. data-focused) matters. 

Often, one will want to define a curve using some kind of interpolation over discrete data or, conversely,
extract lists of samples that approximate a curve, or convert a curve to one which has been discretized.

For the first of these, there is the type `SampleCurve<T, I>`, whose constructor looks like this:
```rust
/// Create a new `SampleCurve`, using the specified `interpolation` to interpolate between
/// the given `samples`. An error is returned if there are not at least 2 samples or if the
/// given `domain` is unbounded.
///
/// The interpolation takes two values by reference together with a scalar parameter and
/// produces an owned value. The expectation is that `interpolation(&x, &y, 0.0)` and
/// `interpolation(&x, &y, 1.0)` are equivalent to `x` and `y` respectively.
pub fn new(
    domain: Interval,
    samples: impl Into<Vec<T>>,
    interpolation: I,
) -> Result<Self, SampleCurveError>
where
    I: Fn(&T, &T, f32) -> T,
{ //... }
```
Here, the `samples` become a `Vec<T>` whose elements represent equally-spaced sample values over the given
`domain`. Adjacent samples are interpolated using the given `interpolation`. The result is a type that
implements `Curve<T>`, on which the preceding operations can be performed.

Conversely, if one wants discrete samples from a curve, there is the function `samples`:
```rust
/// Extract an iterator over evenly-spaced samples from this curve. If `samples` is less than 2
/// or if this curve has unbounded domain, then an error is returned instead.
fn samples(&self, samples: usize) -> Result<impl Iterator<Item = T>, ResamplingError> { //... }
```

Timed samples can be achieved by an application of `graph`:
```rust
for (t, v) in my_curve.by_ref().graph().samples(100).unwrap() {
    println!("Value {v} at time {t}.");
}
```

Finally, the preceding two operations can be combined in one fell swoop with the method `resample`, resulting
in a curve that approximates the original curve using discrete samples:

```rust
/// Resample this [`Curve`] to produce a new one that is defined by interpolation over equally
/// spaced values, using the provided `interpolation` to interpolate between adjacent samples.
/// A total of `samples` samples are used, although at least two samples are required to produce
/// well-formed output. If fewer than two samples are provided, or if this curve has an unbounded
/// domain, then a [`ResamplingError`] is returned.
fn resample<I>(
    &self,
    samples: usize,
    interpolation: I,
) -> Result<SampleCurve<T, I>, ResamplingError>
where
    Self: Sized,
    I: Fn(&T, &T, f32) -> T,
{ //... }
```

This story has a parallel in `UnevenSampleCurve`, which behaves more like keyframes in that the samples need not be evenly spaced:
```rust
/// Create a new [`UnevenSampleCurve`] using the provided `interpolation` to interpolate
/// between adjacent `timed_samples`. The given samples are filtered to finite times and
/// sorted internally; if there are not at least 2 valid timed samples, an error will be
/// returned.
///
/// The interpolation takes two values by reference together with a scalar parameter and
/// produces an owned value. The expectation is that `interpolation(&x, &y, 0.0)` and
/// `interpolation(&x, &y, 1.0)` are equivalent to `x` and `y` respectively.
pub fn new(
    timed_samples: impl Into<Vec<(f32, T)>>,
    interpolation: I,
) -> Result<Self, UnevenSampleCurveError> { //... }
```

This kind of construction can be useful for strategically saving space when performing serialization and storage (although, keep in mind
that the interpolating function may need to be separately handled in user-space). The `UnevenSampleCurve` type also supports 
*forward* mapping of its sample times via the method `map_sample_times`, which accepts a function like the inverse of the one that 
would be used in `Curve::reparametrize`.

The associated resampling method is `resample_uneven`:
```rust
/// Resample this [`Curve`] to produce a new one that is defined by interpolation over samples
/// taken at the given set of times. The given `sample_times` are expected to contain at least
/// two valid times within the curve's domain range.
///
/// Irredundant sample times, non-finite sample times, and sample times outside of the domain
/// are simply filtered out. With an insufficient quantity of data, a [`ResamplingError`] is
/// returned.
///
/// The domain of the produced [`UnevenSampleCurve`] stretches between the first and last
/// sample times of the iterator.
fn resample_uneven(
    &self,
    sample_times: impl IntoIterator<Item = f32>,
) -> Result<UnevenSampleCurve<T>, ResamplingError> { //... }
```

Note that there is no equivalent to `samples` for uneven spacing; however, these can be obtained easily using `Iterator::map`:
```rust
for (t, v) in (0..100).map(|x| x * x).map(|t| (t, my_curve.sample(t))) {
    println!("Value {v} at time {t}.");
}
```
The reason for this shortfall is that user expectations surrounding filtering, sorting, clamping, and so on are hard to predict. For example, 
`sample` here could be replaced with `sample_clamped`, or the second `map` with `filter_map` and `sample` with `sample_checked`, among many other 
variations. With time, perhaps a good default will become apparent and be added to the API.

Finally, for some common math types whose interpolation is especially obvious and well-behaved (and enshrined in a trait), there are
convenience methods `resample_auto` and `resample_uneven_auto` that use this type-inferred interpolation automatically. The expectation is not
that interpolation in user-space will commonly use this trait and the associated methods, since the bar for having a valid trait implementation
is rather high; rather, in domains where weaker interpolation notions are prevalent (e.g. animation), the expectation is that consumers will
primarily go through `SampleCurve` or `UnevenSampleCurve` or define their own curve constructions entirely.

### Combining curves

There are a couple of common ways of combining curves that are supported by the Curve API. The first of these is `compose`, which appends two curves together end-to-end:
```rust
/// Create a new [`Curve`] by composing this curve end-to-end with another, producing another curve
/// with outputs of the same type. The domain of the other curve is translated so that its start
/// coincides with where this curve ends. A [`CompositionError`] is returned if this curve's domain
/// doesn't have a finite right endpoint or if `other`'s domain doesn't have a finite left endpoint.
fn compose<C>(self, other: C) -> Result<impl Curve<T>, CompositionError> { //... }
```

This is useful for doing things like joining paths; note, however, that it cannot generally provide any guarantees that the resulting curve doesn't abruptly transition from the first curve to the second.

The second useful API method in this category is `zip`, which behaves much like its `Iterator` counterpart:
```rust
/// Create a new [`Curve`] by joining this curve together with another. The sample at time `t`
/// in the new curve is `(x, y)`, where `x` is the sample of `self` at time `t` and `y` is the
/// sample of `other` at time `t`. The domain of the new curve is the intersection of the
/// domains of its constituents. If the domain intersection would be empty, an
/// [`InvalidIntervalError`] is returned.
fn zip<S, C>(self, other: C) -> Result<impl Curve<(T, S)>, InvalidIntervalError> { //... }
```

### Other ways of making curves

The curve-creation functions `constant_curve` and `function_curve` that we have been using in examples are in fact
real functions that are part of the library. They look like this:
```rust
/// Create a [`Curve`] that constantly takes the given `value` over the given `domain`.
pub fn constant_curve<T>(domain: Interval, value: T) -> impl Curve<T> { //... }
```
```rust
/// Convert the given function `f` into a [`Curve`] with the given `domain`, sampled by
/// evaluating the function.
pub fn function_curve<T, F>(domain: Interval, f: F) -> impl Curve<T>
where 
    F: Fn(f32) -> T,
{ //... }
```
Note that, while the examples used mostly functions `f32 -> f32`, `function_curve` can convert any function of the 
parameter domain into a `Curve<T>`. For example, here is a rotation over time, expressed as a `Curve<Quat>` using this
API:
```rust
let rotation_curve = function_curve(interval(0.0, std::f32::consts::TAU).unwrap(), |t| Quat::from_rotation_z(t));
```
Furthermore, all of `bevy_math`'s curves (e.g. those created by splines) implement `Curve<T>` for suitable values of
`T`, including information like derivatives in addition to positional data. Additionally, authors of other Bevy
libraries and internal modules may provide additional `Curve<T>` implementors, either to provide functionality
for specific problem domains or to expand the variety of curve constructions available in the Bevy ecosystem.

It is worth remembering that implementing `Curve<T>` yourself, too, is extraordinarily straightforward, since the
only required methods are `domain` and `sample`, so you can hook into this API functionality yourself with ease.

### Borrowing

One other minor point is that you may not always want functions like `map` and `reparametrize` to take ownership of
the input curve. For example, the following code takes ownership of `my_curve`, so it cannot be reused, even though
`resample` requires only a reference:
```rust
let mapped_sample_curve = my_curve.map(|x| x * 2.0).resample_auto(100).unwrap();
```
The `by_ref` method exists to circumvent this problem, allowing curve data to be used through borrowing; this is 
supported by a blanket implementation which allows any type which dereferences to a `Curve` to be a `Curve` itself. 
For example, the following are equivalent and do not take ownership of `my_curve`:
```rust
let mapped_sample_curve = my_curve.by_ref().map(|x| x * 2.0).resample_auto(100).unwrap();
let mapped_sample_curve = (&my_curve).map(|x| x * 2.0).resample_auto(100).unwrap();
```

## Implementation strategy

### API Implementation

The API is really segregated into two parts, the functional and the concrete. The functional part, whose outputs are
only guaranteed to be `impl Curve<T>` of some kind, uses wrapper structs for its outputs, which take ownership of 
the original curve data, along with any closures needed to perform combined sampling. For example, `map` is powered
by `MapCurve<S, T, C, F>`, which looks like this:
```rust
/// A [`Curve`] whose samples are defined by mapping samples from another curve through a
/// given function.
pub struct MapCurve<S, T, C, F>
where
    C: Curve<S>,
    F: Fn(S) -> T,
{
    preimage: C,
    f: F,
    _phantom: PhantomData<(S, T)>,
}

impl<S, T, C, F> Curve<T> for MapCurve<S, T, C, F>
where
    C: Curve<S>,
    F: Fn(S) -> T,
{
    fn domain(&self) -> Interval {
        self.preimage.domain()
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

---

On the other hand, the "concrete" part of the API consists of `SampleCurve<T, I>`, `UnevenSampleCurve<T, I>`, and the 
methods that yield them. These are implemented essentially as one would imagine, holding vectors of data and interpolating
it in `Curve::sample`. E.g.:

```rust
/// A [`Curve`] that is defined by neighbor interpolation over a set of samples.
pub struct SampleCurve<T, I>
{
    domain: Interval,
    samples: Vec<T>,
    interpolation: I,
}
// ...

    fn sample(&self, t: f32) -> T {
        // We clamp `t` to the domain.
        let t = self.domain.clamp(t);

        // Inside the curve itself, interpolate between the two nearest sample values.
        let subdivs = self.samples.len() - 1;
        let step = self.domain.length() / subdivs as f32;
        let t_shifted = t - self.domain.start();
        let lower_index = (t_shifted / step).floor() as usize;
        let upper_index = (t_shifted / step).ceil() as usize;
        let f = (t_shifted / step).fract();
        (self.interpolation)(&self.samples[lower_index], &self.samples[upper_index], f) 
    }
```

The main thing here is that `SampleCurve<T, I>` is actually returned *by type* from the `resample` API,
and it is clearly suitable for serialization and storage in addition to numerical applications. This
allows consumers to work flexibly with the functional API and then cast down to concrete values when
they need them. (And of course, `UnevenSampleCurve<T, I>` is implemented similarly, instead using a binary
search in its sequence of time-values to find its interpolation interval.)

Because these types actually store concrete sample data, they have special `map`-like methods
which are not lazy, instead returning values of type `SampleCurve<S, I>`/`UnevenSampleCurve<S, I>`. These
can be accessed from a type-level API as `map_concrete`, so that the user can avoid erasing the
type if it is convenient to do so. The same goes for `graph`; however, because of its contravariance
(it is function precomposition), the same cannot be said of `reparametrize`, which maintains its default
functional implementation. The latter gap is filled partially by `UnevenSampleCurve::map_sample_times`.

Everything else should be fairly clear based on the user-facing API descriptions.

### Mathematical and especially nice interpolation

It is natural to ask what kind of interpolation is especially well-behaved from the perspective of resampling
operations. One answer is that we should expect that the interpolation have *subdivision-stability*: if a curve
between `p` and `q` is defined by interpolation, then resampling the curve at intermediate points and using their
interpolation should result in the same curve as the one we started with. This leads, for example, to the following
law (modulo some omitted `unwrap`s):

* `curve.resample_auto(n).resample_auto(k * n)` is equivalent to `curve.resample_auto(n)` (where `k > 0`).

Something similar should hold for uneven sample points:

* If `iter_few` and `iter_many` are `f32`-valued iterators and all of `iter_few`'s values are contained in those of `iter_many`, then:
  `curve.resample_uneven_auto(iter_few).resample_uneven_auto(iter_many)` is equivalent to `curve.resample_uneven_auto(iter_few)`.

That is to say: subsampling has no effect on sample-interpolated curves with values in such types. This motivates a trait; on the 
level of syntax, the trait for such things is completely innocuous, carrying only the shape of an interpolation function:
```rust
pub trait StrongInterpolate {
    fn interpolate_strong(&self, &Self, t: f32) -> Self;
}
```

However, there are strong expectations that come with implementing this:
1. `interpolate_strong(&x, &y, 0.0)` and `interpolate_strong(&x, &y, 1.0)` are equivalent to `x` and `y` respectively.
2. This mode of interpolation should be "obvious" based on the semantics of the type.
3. This interpolation is subdivision-stable: if `original_curve` is the curve formed by interpolation between two
   arbitrary points, and `(t0, p)`, `(t1, q)` are two timed samples from this curve, then the curve formed by
   interpolation between `p` and `q` must be the unique linear reparametrization of the curve-segment between `t0` and `t1`
   in `original_curve`.

Equivalently, the second condition may be stated as follows: if `iter` is an iterator yielding `f32` values between `0.0`
and `1.0` (and including `0.0` and `1.0`), then `original_curve.resample_uneven(iter)` is equivalent to `original_curve`.

Note that here, "equivalent" does not mean "taking the same values in the same order" — it means that they actually
produce equivalent samples at any given parameter value.

This is the trait that is used for `resample_auto` and `resample_uneven_auto`, and it is to be implemented for `NormedVectorSpace`
types using `lerp`, as well as for rotation and direction types using `slerp`.

### Shared interpolation interfaces

Under the hood, the aforementioned `SampleCurve`/`SampleAutoCurve`/`UnevenSampleCurve`/`UnevenSampleAutoCurve` constructions
use shared API backends that abstract away the meat of the data access patterns. For example, evenly-spaced samples use a data
structure that looks like this:

```rust
pub struct EvenCore<T> {
    /// The domain over which the samples are taken, which corresponds to the domain of the curve
    /// formed by interpolating them.
    ///
    /// # Invariants
    /// This must always be a bounded interval; i.e. its endpoints must be finite.
    pub domain: Interval,

    /// The samples that are interpolated to extract values.
    ///
    /// # Invariants
    /// This must always have a length of at least 2.
    pub samples: Vec<T>,
}
```

Unevenly-spaced samples have something similar:

```rust
pub struct UnevenCore<T> {
    /// The times for the samples of this curve.
    ///
    /// # Invariants
    /// This must always have a length of at least 2, be sorted, and have no
    /// duplicated or non-finite times.
    pub times: Vec<f32>,

    /// The samples corresponding to the times for this curve.
    ///
    /// # Invariants
    /// This must always have the same length as `times`.
    pub samples: Vec<T>,
}
```

Notably, these have public fields (despite requiring invariants) because they are intended to be extensible by 
authors of libraries and so on; on the other hand, each has a `new` function which enforces the invariants,
returning an error if they are not met.

The benefit of using these is that the interpolation access patterns are done for you; the methods
`sample_interp` and `sample_interp_timed` do the meat of the interpolation, allowing custom interpolation to
be easily defined. Here is what those look like:

```rust
/// Given a time `t`, obtain a [`InterpolationDatum`] which governs how interpolation might recover
/// a sample at time `t`. For example, when a [`Between`] value is returned, its contents can
/// be used to interpolate between the two contained values with the given parameter. The other
/// variants give additional context about where the value is relative to the family of samples.
///
/// [`Between`]: `InterpolationDatum::Between`
pub fn sample_interp(&self, t: f32) -> InterpolationDatum<&T> { //... }
```

```rust
/// Like [`sample_interp`], but the returned values include the sample times. This can be
/// useful when sampling is not scale-invariant.
///
/// [`sample_interp`]: EvenCore::sample_interp
pub fn sample_interp_timed(&self, t: f32) -> InterpolationDatum<(f32, &T)> { //... }
```

Most of the time, `sample_interp` is sufficient, but `sample_interp_timed` may be required when interpolation
is not scale-invariant.

`InterpolationDatum` is just an enum that looks like this:
```rust
pub enum InterpolationDatum<T> {
    /// This value lies exactly on a value in the family.
    Exact(T),

    /// This value is off the left tail of the family; the inner value is the family's leftmost.
    LeftTail(T),

    /// This value is off the right tail of the family; the inner value is the family's rightmost.
    RightTail(T),

    /// This value lies on the interior, in between two points, with a third parameter expressing
    /// the interpolation factor between the two.
    Between(T, T, f32),
}
```

For simple cases (e.g. creating something which behaves like `SampleCurve`), there is a helper function `sample_with` 
which takes an explicit interpolation and does the obvious thing, cloning the value of any case except `Between`, where
the interpolation is used.

Here is an example that demonstrates how to use `EvenCore` to create a `Curve` that contains its interpolation mode
in an enum:
```rust
use bevy_math::curve::*;
use bevy_math::curve::builders::*;

enum InterpolationMode {
    Linear,
    Step,
}

trait LinearInterpolate {
    fn lerp(&self, other: &Self, t: f32) -> Self;
}

fn step<T: Clone>(first: &T, second: &T, t: f32) -> T {
    if t >= 1.0 {
        second.clone()
    } else {
        first.clone()
    }
}

struct MyCurve<T> {
    core: SampleCore<T>,
    interpolation_mode: InterpolationMode,
}

impl<T> Curve<T> for MyCurve<T>
where
    T: LinearInterpolate + Clone,
{
    fn domain(&self) -> Interval {
        self.core.domain()
    }
    
    fn sample(&self, t: f32) -> T {
        match self.interpolation_mode {
            InterpolationMode::Linear => self.core.sample_with(t, <T as LinearInterpolate>::lerp),
            InterpolationMode::Step => self.core.sample_with(t, step),
        }
    }
}
```

I think this does a pretty good job of demonstrating how the "core" concept can be useful to implementors.

### Object safety

The `Curve<T>` trait should be object-safe. While the functional `Curve` API methods are automatically
excluded from dynamic dispatch because they move `self` into another struct (and hence require `Self: Sized`),
others should be explicitly given the `Self: Sized` constraint in order to ensure object safety.

This prevents methods like `map` from being used by `dyn Curve<T>` (which was hopeless regardless); however,
by providing a blanket implementation over `Deref`, pointers to trait objects can still be used as `Curve<T>`.
For instance, `Box<dyn Curve<T>>`, `Arc<dyn Curve<T>>`, etc. all implement `Curve<T>` through this blanket
implementation, which means that they can use the default implementations of `map`, `reparametrize`, etc. built
on top of the dispatchable core containing `domain` and `sample`.

This is the reason for including a `?Sized` constraint on the underlying curve in the blanket implementation,
which has a signature that looks like this:
```rust
impl<T, C, D> Curve<T> for D
where
    C: Curve<T> + ?Sized,
    D: Deref<Target = C>,
{ //... }
```

## Drawbacks

The main risk in implementing this is that we cement a poor or incomplete curve API within the Bevy
ecosystem, so that authors of internal and external modules need to sidestep it or do reimplementation
themselves.

Furthermore, a poor implementation would lead to additional maintenance burden and compile times with
little benefit.

Another limitation of the API presented here is that it uses `f32` exclusively as its parameter type,
which presents another area where the API could be deemed incomplete in the future. The reason that `f32`
was deemed adequate are as follows:
- Existing constructions in `bevy_math`, `bevy_animation`, and `bevy_color` all use `f32`.
- Basically nothing in Bevy on the CPU side uses `f64` internally.
- Curve applications tend to be artistic in nature rather than demanding high precision, so that `f32` is
  generally good enough.
- If strictly necessary, making the change to use generics is straightforward, but unifying around `f32`
  is simpler and lowers the barrier to interoperation. 

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

I believe that all major questions for this particular project scope have been resolved satisfactorily.

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

Another important future direction to keep in mind is the serialization and deserialization of curve data,
coupled with the use of items like `dyn Curve<T>` and similar. This will likely require extensions of the
`Curve` API (or specializations thereof) to work well (e.g. a specialized subtrait). 

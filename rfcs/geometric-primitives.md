# Feature Name: `geometric-primitives`

## Summary

Lightweight geometric primitive types for use across bevy engine crates, and as interoperability types for external libraries.

## Motivation

This would provide a stardard way to model primitives across the bevy ecosystem to prevent ecosystem fragmentation amongst plugins.

## Guide-level explanation

Geometric primitives are lightweight representations of geometry that describe the type of geometry as well as fully defined dimensions. These primitives are *not* discrete meshes, but the underlying precise geometric definition. For example, a circle is:

```rust
pub struct Circle {
  origin: Vec2
  radius: f32,
}
```

Geometric primitives have fully defined shape, size, position, and orientation.  Position and orientation are **not** handled by `Transform` systems.

TODO:
Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Reference-level explanation

### `bevy_geom::2d`

- `Line`
- `Arc`
- `Circle`
- `Triangle`
- `AABB`
- `OBB`

### `bevy_geom::3d`

- `Line`
- `Sphere`
- `Cylinder`
- `Capsule`
- `Torus`
- `Cone`
- `Pyramid`
- `Frustum`
- `AABB`
- `OBB`

## Drawbacks

An argument could be made to use an external crate for this, however these types are so fundamental, I think it's important that they are optimized for the engine's uses, and are not a generalized solution.

This is also a technically simple addition that shouldn't present maintenance burden. The real challenge is upfront in ensuring the API is designed well, and the primitives are performant for their most common use cases. If anything, this furthers the argument for not using an external crate.

## Rationale and alternatives

- Why is this design the best in the space of possible designs?

### Not Using Transforms

Primitives are fully defined in space, and do not use `Transform` or `GlobalTransform`. This is an intentional decision.

It's unsurprisingly much simpler to use these types when the primitives are fully defined internally, but maybe somewhat surprisingly, more efficient.

#### Cache Efficiency

- Some primitives such as AABB and Sphere don't need a rotation to be fully defined. 
- By using a `GlobalTransform`, not only is this an unused Quat that fills the cache line, it would also cause redundant change detection on rotations.
- This is especially important for AABBs and Spheres, because they are fundamental to collision detection and BV(H), and as such need to be as efficient as possible.
- I still haven't found a case where you would use a `Primitive3d` without needing this positional information that fully defines the primitive in space. If this holds true, it means that storing the positional data inside the primitive is _not_ a waste of cache, which is normally why you would want to separate the transform into a separate component.

#### CPU Efficiency

- Storing the primitive's positional information internally serves as a form of memoization.
- Because you need the primitive to be fully defined in world space to run useful operations, this means that with a `GlobalTransform` you would need to apply the transform to the primitive every time you need to use it.
- By applying this transformation only once (e.g. during transform propagation), we only need to do this computation a single time.

#### Ergonomics

- As I've already mentioned a few times, primitives need to be fully defined in world space to do anything useful with them.
- By making the primitive components fully defined and standalone, computing operations is as simple as: `primitive1.some_function(primitive_2)`, instead of also having query and pass in 2 `GlobalTransform`s in the correct order.

#### Use with Transforms

- For use cases such as oriented bounding boxes, a primitive should be defined relative to its parent.
- In this case, the primitive would still be fully defined internally, but we would need to include primitive updates analogous to the transform propagation system.
- For example, a bounding sphere entity would be a child of a mesh, with a `Sphere` primitive and a `Transform`. On updates to the parent's `GlobalTransform`, the bounding sphere's `Transform` component would be used to update the `Sphere`'s position and radius by applying the scale and translation to a unit sphere. This could be applied to all primitives, with the update system being optimized for each primitive.

- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

TODO

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Future possibilities

- Bounding boxes
- Collisions
- Frustum Culling
- Debug Rendering

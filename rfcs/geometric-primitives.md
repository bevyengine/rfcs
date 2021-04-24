# Feature Name: `geometric-primitives`

## Summary

Lightweight geometric primitive types for use across bevy engine crates, and as interoperability types for external libraries.

## Motivation

This would provide a standard way to model primitives across the bevy ecosystem to prevent ecosystem fragmentation amongst plugins.

## Guide-level explanation

Geometric primitives are lightweight representations of geometry that describe the type of geometry as well as fully defined dimensions. These primitives are *not* discrete meshes, but the underlying precise geometric definition. For example, a circle is:

```rust
pub struct Circle {
  origin: Point,
  radius: f32,
}
```

Geometric primitives have a defined shape, size, position, and orientation. Position and orientation are **not** handled by bevy's `Transform` systems. These are fundamental geometric primitives that must be usable and comparable as-is. 

TODO:
Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Reference-level explanation

### `bevy_geom::Shape2d`

These types only exist in 2d space: their dimensions and location are only defined in `x` and `y` unlike their 3d counterparts.

- `Point`: type alias of Vec2
- `Direction`: Vec2 that is guaranteed to be normalized through its getter/setter

#### Axis

Line with infinite length on the x/y plane.


```rust
struct Axis { point: Point, normal: Direction }
``` 

#### Line

- `Line`: (start: Point, end: Point) line bounded by two points
- `Arc`: (start: Point, end: Point, radius_origin: Point) segment of a circle

```rust
struct Circle` { origin: Point, radius: f32 }
```
- `Triangle`
- `AABBCentered` origin and half-extents
- `AabbExtents` minimum and maximum extents
- `OBB` origin, orthonormal basis, and half-extents

### `bevy_geom::Shape3d`

These types are fully defined in 3d space.

- `Point`: type alias of Vec3
- `Direction`: Vec3 that is guaranteed to be normalized through its getter/setter
- `Axis`
- `Line`
- `Plane`
- `Quad`
- `Sphere`
- `Cylinder`
- `Capsule`
- `Torus`
- `Cone`
- `Frustum`: defined with 6 planes
- `AabbCentered`
- `AabbExtents`
- `OBB`

### Bounding Boxes

This section is based off of prototyping work done in [bevy_mod_bounding](https://github.com/aevyrie/bevy_mod_bounding).

Because bounding boxes are fully defined in world space, this leads to the natural question of how they are kept in sync with their parent.

### Frustum Culling

This section is based off of prototyping work done in [bevy_frustum_culling](https://github.com/aevyrie/bevy_frustum_culling).

### Ray Casting

This section is based off of prototyping work done in [bevy_mod_picking](https://github.com/aevyrie/bevy_mod_picking).

## Drawbacks

An argument could be made to use an external crate for this, however these types are so fundamental I think it's important that they are optimized for the engine's uses, and are not from a generalized solution.

This is also a technically simple addition that shouldn't present maintenance burden. The real challenge is upfront in ensuring the API is designed well, and the primitives are performant for their most common use cases. If anything, this furthers the argument for not using an external crate.

## Rationale and alternatives

### Lack of `Transform`s

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

Unity `PrimitiveObjects`: https://docs.unity3d.com/Manual/PrimitiveObjects.html
Godot `PrimitiveMesh`: https://docs.godotengine.org/en/stable/classes/class_primitivemesh.html#class-primitivemesh

These examples intermingle primitive geometry with the meshes themselves. This RFC makes these distinct.


## Unresolved questions
TODO: discuss unresolved questions.
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?

### Out of Scope

- Value types, e.g. float vs. fixed is out of scope. This RFC is focused on the core geometry types and is intended to use Bevy's common rendering value types such as `f32`.

## Future possibilities

- Bounding boxes
- Collisions
- Frustum Culling
- Debug Rendering

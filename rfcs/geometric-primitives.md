# Feature Name: `geometric-primitives`

## Summary

Lightweight geometric primitive types for use across bevy engine crates, and as interoperability types for external libraries.

## Motivation

This would provide a standard way to model primitives across the bevy ecosystem to prevent ecosystem fragmentation amongst plugins.

## User-Facing Explanation

Geometric primitives are lightweight representations of geometry that describe the type of geometry as well as fully defined dimensions. These primitives are *not* meshes, but the underlying precise mathematical definition. For example, a circle is:

```rust
pub struct Circle2d {
  origin: Point2d,
  radius: f32,
}
```

Geometric primitives have a defined shape, size, position, and orientation. Position and orientation are **not** defined using bevy's `Transform` components. This is because these are fundamental geometric primitives that must be usable and comparable as-is.

`bevy_geom` provides two main modules, `geom_2d` and `geom_3d`. Recall that the purpose of this crate is to provide lightweight types, so there are what appear to be duplicates in 2d and 3d, such as `geom_2d::Line2d` and `geom_3d::Line`. Note that the 2d version of a line is lighter weight, and is only defined in 2d. 3d geometry (or 2d with depth which is 3d) is assumed to be the default for most cases. The names of the types were chosen with this in mind, to guide you toward using Line instead of Line2d for example, unless you know why you are making this choice.

## Implementation strategy

### Helper Types

```rust
/// Stores an angle in radians, and supplies builder functions to prevent errors (from_radians, from_degrees)
struct Angle(f32);
```

### 3D Geometry Types

These types are fully defined in 3d space.

```rust
/// Point in 3D space
struct Point(Vec3)

/// Vector direction in 3D space that is guaranteed to be normalized through its getter/setter.
struct Direction(Vec3)

// Plane in 3d space defined by a point on the plane, and the normal direction of the plane
struct Plane {
  point: Point,
  normal: Direction,
}

/// Unbounded line in 3D space with direction
struct Ray { 
  point: Point, 
  normal: Direction,
}

/// Line in 3D space bounded by two points
struct Line { 
  start: Point, 
  end: Point,
}

struct Triangle([Point; 3]);

struct Quad([Point; 4]);

struct Sphere{
  origin: Point,
  radius: f32,
}

struct Cylinder {
  /// The bottom center of the cylinder.
  origin: Point,
  radius: f32,
  /// The extent of the cylinder is the vector that extends from the origin to the opposite face of the cylinder.
  extent: Vec3,
}

struct Cone{
  /// Origin of the cone located at the center of its base.
  origin: Point,
  /// The extent of the cone is the vector that extends from the origin to the tip of the cone.
  extent: Vec3,
  base_radius: f32,
}

struct Torus{
  origin: Point,
  normal: Direction,
  major_radius: f32,
  tube_radius: f32,
}

struct Capsule {
  origin: Point,
  height: Vec3,

}

// A 3d frustum used to represent the volume rendered by a camera, defined by the 6 planes that set the frustum limits.
struct Frustum{
  near: Plane,
  far: Plane,
  top: Plane,
  bottom: Plane,
  left: Plane,
  right: Plane,
}

// 3D Axis Aligned Bounding Box defined by its extents, useful for fast intersection checks and frustum culling.
struct AabbExtents {
  min: Vec3
  max: Vec3
}
impl Aabb for AabbExtents {} //...

// 3D Axis Aligned Bounding Box defined by its center and half extents, easily converted into an OBB
struct AabbCentered {
  origin: Point,
  half_extents: Vec3,
}
impl Aabb for AabbCentered {} //...

// 3D Axis Aligned Bounding Box defined by its eight vertices, useful for culling or drawing
struct AabbVertices([Point; 8]);
impl Aabb for AabbVertices {} //...

// 3D Oriented Bounding Box
struct ObbCentered {
  origin: Point,
  orthonormal_basis: Mat3,
  half_extents: Vec3,
}
impl Obb for ObbCentered {} //...

struct ObbVertices([Point; 4]);
impl Obb for ObbVertices {} //...
```

### 2D Geometry Types

These types only exist in 2d space: their dimensions and location are only defined in `x` and `y` unlike their 3d counterparts. These types are suffixed with "2d" to disambiguate from the 3d types in user code, guide users to using 3d types by default, and remove the need for name-spacing the 2d and 3d types when used in the same scope.

```rust
/// Point in 2D space
struct Point2d(Vec2)

/// Vector direction in 2D space that is guaranteed to be normalized through its getter/setter.
struct Direction2d(Vec2)

/// Unbounded line in 2D space with direction
struct Ray2d { 
  point: Point2d, 
  normal: Direction2d,
}

/// Line in 2D space bounded by two points
struct Line2d { 
  start: Point2d, 
  end: Point2d,
}

/// A line list represented by an ordered list of vertices in 2d space
struct LineList<const N: usize>{
  points: [Point; N],
  /// True if the LineList is a closed loop
  closed: bool,
}

struct Triangle2d([Point2d; 3]);

struct Quad2d([Point2d; 4]);

/// A regular polygon, such as a square or hexagon.
struct RegularPolygon2d{
  circumcircle: Circle2d,
  /// Number of faces.
  faces: u8,
  /// Clockwise rotation of the polygon about the origin. At zero rotation, a point will always be located at the 12 o'clock position.
  orientation: Angle,
}

struct Circle2d {
  origin: Point2d, 
  radius: f32,
}

/// Segment of a circle
struct Arc2d {
  circle: Circle2d,
  /// Start of the arc, measured clockwise from the 12 o'clock position
  start: Angle,
  /// Angle in radians to sweep clockwise from the [start_angle]
  sweep: Angle,
}

// 2D Axis Aligned Bounding Box defined by its extents, useful for fast intersection checks and culling with an axis-aligned viewport
struct AabbExtents2d {
  min: Vec2
  max: Vec2
}
impl Aabb2d for AabbExtents2d {} //...

// 2D Axis Aligned Bounding Box defined by its center and half extents, easily converted into an OBB.
struct AabbCentered2d {
  origin: Point2d,
  half_extents: Vec2,
}
impl Aabb2d for AabbCentered2d {} //...

// 2D Axis Aligned Bounding Box defined by its four vertices, useful for culling or drawing
struct AabbVertices2d([Point2d; 4]);
impl Aabb2d for AabbVertices2d {} //...

// 2D Oriented Bounding Box
struct ObbCentered2d {
  origin: Point2d,
  orthonormal_basis: Mat2,
  half_extents: Vec2,
}
impl Obb2d for ObbCentered2d {} //...

struct ObbVertices2d([Point2d; 4]);
impl Obb2d for ObbVertices2d {} //...
```

### Meshing

While these primitives do not provide a meshing strategy, future work could build on these types so users can use something like `Sphere.mesh()` to generate meshes.

```rust
let unit_sphere = Sphere::UNIT;
// Some ways we could build meshes from primitives
let sphere_mesh = Mesh::from(unit_sphere);
let sphere_mesh = unit_sphere.mesh();
let sphere_mesh: Mesh = unit_sphere.into();
```

### Bounding Boxes/Volumes

This section is based off of prototyping work done in [bevy_mod_bounding](https://github.com/aevyrie/bevy_mod_bounding).

A number of bounding box types are provided for 2d and 3d use. Some representations of a bounding box are more efficient depending on the use case. Instead of storing all possible values in a component (wasted space) or computing them with a helper function every time (wasted cpu), each representation is provided as a distinct component that can be updated independently. This gives users of Bevy the flexibility to optimize for their use case without needing to write their own incompatible types from scratch. Consider the functionality built on top of bounding such as physics or collisions - because they are all built on the same fundamental types, they can interoperate.

Note that the bounding types presented in this RFC implement their respective `Obb` or `Aabb` trait. Bounding boxes can be represented by a number of different underlying types; by making these traits instead of structs, systems can easily be made generic over these types depending on the situation.

Because bounding boxes are fully defined in world space, this leads to the natural question of how they are kept in sync with their parent. The plan would be to provide a system similar to transform propagation, that would apply an OBB's precomputed `Transform` to its parent's `GlobalTransform`. Further details are more appropriate for a subsequent bounding RFC/PR. The important point to consider is how this proposal provides common types that can be used for this purpose in the future.

### Frustum Culling

This section is based off of prototyping work done in [bevy_frustum_culling](https://github.com/aevyrie/bevy_frustum_culling).

The provided `Frustum` type is useful for frustum culling, which is generally done on the CPU by comparing each frustum plane with each entity's bounding volume.

```rust
// What frustum culling might look like:
for bounding_volume in bound_vol_query.iter() {
    for plane in camera_frustum.planes().iter() {
        if bounding_volume.outside(plane) {
            // Cull me!
            return;
        }
    }
}
```

This data structure alone does not ensure the representation is valid; planes could be placed in nonsensical positions. To prevent this, the struct's fields should be made private, and constructors and setters should be provided to ensure `Frustum`s can only be initialized or mutated into valid arrangements.

In addition, by defining the frustum as a set of planes, it is also trivial to support oblique frustums.

### Ray Casting

This section is based off of prototyping work done in [bevy_mod_picking](https://github.com/aevyrie/bevy_mod_picking).

The bounding volumes section covers how these types would be used for the bounding volumes which are used for accelerating ray casting. In addition, the `Ray` component can be used to represent rays. Applicable 3d types could implement a `RayIntersection` trait to extend their functionality.

```rust
let ray = Ray::X;
let sphere = Sphere::new(Point::x(5.0), 1.0);
let intersection = ray.cast(sphere);
```

## Drawbacks

An argument could be made to use an external crate for this, however these types are so fundamental I think it's important that they are optimized for the engine's uses, and are not from a generalized solution.

This is also a technically simple addition that shouldn't present maintenance burden. The real challenge is upfront in ensuring the API is designed well, and the primitives are performant for their most common use cases. If anything, this furthers the argument for not using an external crate.

## Rationale and alternatives

### Lack of `Transform`s

Primitives are fully defined in space, and do not use `Transform` or `GlobalTransform`. This is an intentional decision.

It's unsurprisingly much simpler to use these types when the primitives are fully defined internally, but maybe somewhat surprisingly, more efficient.

### Cache Efficiency

- Some primitives such as AABB and Sphere don't need a rotation to be fully defined. 
- By using a `GlobalTransform`, not only is this an unused Quat that fills the cache line, it would also cause redundant change detection on rotations.
- This is especially important for AABBs and Spheres, because they are fundamental to collision detection and BV(H), and as such need to be as efficient as possible.
- I still haven't found a case where you would use a `Primitive3d` without needing this positional information that fully defines the primitive in space. If this holds true, it means that storing the positional data inside the primitive is _not_ a waste of cache, which is normally why you would want to separate the transform into a separate component.

### CPU Efficiency

- Storing the primitive's positional information internally serves as a form of memoization.
- Because you need the primitive to be fully defined in world space to run useful operations, this means that with a `GlobalTransform` you would need to apply the transform to the primitive every time you need to use it.
- By applying this transformation only once (e.g. during transform propagation), we only need to do this computation a single time.

### Ergonomics

- As I've already mentioned a few times, primitives need to be fully defined in world space to do anything useful with them.
- By making the primitive components fully defined and standalone, computing operations is as simple as: `primitive1.some_function(primitive_2)`, instead of also having query and pass in 2 `GlobalTransform`s in the correct order.

### Use with Transforms

- For use cases such as oriented bounding boxes, a primitive should be defined relative to its parent.
- In this case, the primitive would still be fully defined internally, but we would need to include primitive updates analogous to the transform propagation system.
- For example, a bounding sphere entity would be a child of a mesh, with a `Sphere` primitive and a `Transform`. On updates to the parent's `GlobalTransform`, the bounding sphere's `Transform` component would be used to update the `Sphere`'s position and radius by applying the scale and translation to a unit sphere. This could be applied to all primitives, with the update system being optimized for each primitive.

## Prior art

- Unity `PrimitiveObjects`: https://docs.unity3d.com/Manual/PrimitiveObjects.html
- Godot `PrimitiveMesh`: https://docs.godotengine.org/en/stable/classes/class_primitivemesh.html#class-primitivemesh

These examples intermingle primitive geometry with the meshes themselves. This RFC makes these distinct.

## Unresolved questions

What is the best naming scheme, e.g., `2d::Line`/`3d::Line` vs. `Line2d`/`Line3d` vs. `Line2d`/`Line`.

How will fully defined primitives interact with `Transforms`? Will this confuse users? How can the API be shaped to prevent this?

### Out of Scope

- Value types, e.g. float vs. fixed is out of scope. This RFC is focused on the core geometry types and is intended to use Bevy's common rendering value types such as `f32`.

## Future possibilities

- Bounding boxes
- Collisions
- Frustum Culling
- Ray casting
- Physics
- SDF rendering
- Debug Rendering

# Feature Name: `primitive-shapes`

## Summary

These geometric primitives, or "primitive shapes", are lightweight types for use across Bevy engine crates and as interoperability types for plugins and libraries. The goal is to provide an API that considers ergonomics for the common use cases, such as meshing, bounding, and collision, without sacrificing performance.

## Motivation

This provides a type-level foundation that subsequent engine (and plugin!) features can be built upon incrementally, preventing ecosystem fragmentation. A major goal is to make it possible for engine and game developers to prototype physics, bounding, collision, and raycasting functionality using the same base types. This will make engine integration and cross-plugin interoperability much more feasible, as the engine will have opinionated types that will be natural to implement `into` or `from`. I hope this opens the door to experimentation with multiple physics and bounding acceleration backends. 

There is significant complexity in the way seemingly equivalent shapes are defined depending on how they are used. By considering these use cases *first*, the goal is that geometric data structures can be composed depending on the needed functionality, with traits used to ensure the right data structure is used for the right task.

## User-Facing Explanation

Geometric primitives are lightweight representations of geometry that describe the type of geometry as well as its dimensions. These primitives are *not* meshes, but the underlying precise mathematical definition. For example, a circle is:

```rust
pub struct Circle2d {
  radius: f32,
}
```

Note that a `Circle2d` does not contain any information about the position of the circle. This is due to how shapes are composed to add more complex functionality. Consider the following common use cases of primitives shapes, as well as the the information (Translation and Rotation) that are needed to fully define the shapes for these cases:

| Shape     | Mesh | Bounding | Collision |
|---        |---|---|---|
| Sphere    | ✔ | ✔  + Trans | ✔ + Trans |
| Box    | ✔ | ✔ + Trans (AABB) | ✔ + Trans + Rot (OBB) |
| Capsule   | ✔ | ❌ | ✔ + Trans + Rot |
| Cylinder  | ✔ | ❌ | ✔ + Trans + Rot |
| Cone      | ✔ | ❌ | ✔ + Trans + Rot |
| Wedge     | ✔ | ❌ | ✔ + Trans + Rot |
| Plane     | ✔ | ❌ | ✔ |
| Torus     | ✔ | ❌ | ❌ |

### Bounding vs. Collision

The difference between the two is somewhat semantic. Both bounding and collision check for intersections between bounding volumes or areas. However, bounding volumes are generally used for broad phase checks due to their speed and small size, and are consequently often used in tree structures for spacial queries. Note that both bounding types are not oriented - this reduces the number of comparisons needed to check for intersections during broad phase tests. 

Although the traits for these may look similar, by making them distinct it will better conform to industry terminology and help guide users away from creating performance problems. For example, because a torus is a volume, it is conceivably possible to implement the `Bounding` trait, however it would be a terrible idea to be encouraging users to build BVTs/BVHs out of torii. This is also why the Oriented Bounding Box (OBB) is a collider, and not `Bounding`. Bounding spheres or AABBs are generally always preferred for broad phase bounding checks, despite the OBB's name. For advanced users who want to implement `Bounding` for OBBs, they always have the option of wrapping the OBB in a newtype and implementing the trait.

### Where are the `Transform`s?

 Translation and rotation are **not** defined using bevy's `Transform` components. This is because types that implement `Bounding` and `Collider` must be fully self-contained. This:
 
 * makes the API simpler when using these components in functions and systems
 * ensures bounding and collision types use an absolute minimum of memory
 * prevents errors caused by nonuniform scale invalidating the shape of the primitive.

Instead, changes to the parent's `GlobalTransform` should be used to derive a new primitive on scale changes - as well as a new translation and rotation if required.

### Meshing

Both 2d and 3d primitives can implement the `Meshable` trait to provide the ability to generate a tri mesh:

```rust
let circle_mesh: Mesh = Circle2d{ radius: 2.0 }.mesh();
```
The base primitive types only define the shape (`Circle2d`) and size (`radius`) of the geometry about the origin. Once generated, the mesh can have a transform applied to it like any other mesh, and it is no longer tied to the primitive that generated it. The `Default::default()` implementation of primitives should be a "unit" variant, such as a circle with diameter 1.0, or a box with all edges of length 1.0.

Meshing could be naturally extended with other libraries or parameters. For example, a sphere by default might use `Icosphere`, but could be extended to generate UV spheres or quad spheres. For 2d types, we can use a crate such as `lyon` to generate 2d meshes from parameterized primitive shapes within the `Meshable` interface.

### Bounding

For use in spatial queries, broad phase collision detection, and raycasting, `Bounding` and `Bounding2d` traits are provided. These traits are implemented for types that define position in addition to the shape and size defined in the base primitive. The basic functionality of this trait is to check whether one bounding shape is contained within another bounding shape.

### Colliders

Colliders can provide intersection checks against other colliders, as well as check if they are within a bounding volume.

### 3D and 2D

This RFC provides independent 2d and 3d primitives. Recall that the purpose of this is to provide lightweight types, so there are what appear to be duplicates in 2d and 3d, such as `Line` and `Line2d`. Note that the 2d version of a line is smaller than its 3d counterpart because it is only defined in 2d. 3d geometry (or 2d with depth) is assumed to be the default for most cases. The names of the types were chosen with this in mind.

## Implementation strategy

### Helper Types

```rust
/// Stores an angle in radians, and supplies builder functions to prevent errors (from_radians, from_degrees)
struct Angle(f32);
```
### Traits

These traits are provided as reference to illustrate how these might be used. The details of implementation and interface should be determined in a separate RFC, PR, or independent prototypes.

```rust
trait Meshable{
  fn mesh(&self) -> Mesh;
};

trait Bounding {
  fn within(&self, other: &impl Bounding) -> bool;
  fn contains(&self, collider: &impl Collider) -> bool;
}

trait Bounding2d {
  fn within(&self, other: &impl Bounding2d) -> bool;
  fn contains(&self, collider: &impl Collider2d) -> bool;
}

trait Collider {
  fn collide(&self, other: &impl Collider) -> Option(Collision);
  fn within(&self, bounds: &impl Bounding) -> bool;
}

trait Collider2d {
  fn collide(&self, other: &impl Collider2d) -> Option(Collision2d);
  fn within(&self, bounds: &impl Bounding2d) -> bool;
}
```

### 3D Geometry Types

```rust
struct Point(Vec3)

/// Vector direction in 3D space that is guaranteed to be normalized through its getter/setter.
struct Direction(Vec3)
impl Meshable for Direction {}

struct Plane {
  point: Point,
  normal: Direction,
}
impl Meshable for Plane {}

/// Differentiates a line from a ray, where a line is infinite and a ray is directional half-line, although their underlying representation is the same.
struct Ray(
  point: Point, 
  direction: Direction,
);
impl Meshable for Ray {}

// Line types

/// Unbounded line in 3D space with directionality
struct Line { 
  point: Point, 
  direction: Direction,
}
impl Meshable for Line {}

/// A line segment bounded by two points
struct LineSegment { 
  start: Point, 
  end: Point,
}
impl Meshable for LineSegment {}

/// A line drawn along a path of points
struct PolyLine {
  points: Vec<Point>,
}
impl Meshable for PolyLine {}

struct Triangle([Point; 3]);
impl Meshable for Triangle {}
impl Collider for Triangle {}

struct Quad([Point; 4]);
impl Meshable for Quad {}
impl Collider for Quad {}

/// Sphere types

struct Sphere {
  radius: f32,
}
impl Meshable for Sphere {}

struct BoundingSphere {
  sphere: Sphere,
  translation: Vec3,
}
impl Meshable for BoundingSphere {}
impl Bounding for BoundingSphere {}

struct SphereCollider {
  sphere: Sphere,
  translation: Vec3,
}
impl Meshable for SphereCollider {}
impl Collider for SphereCollider {}

// Box Types

struct Box {
  half_extents: Vec3,
}
impl Meshable for Box

struct BoundingBox {
  box: Box,
  translation: Vec3,
}
impl Meshable for BoundingBox {}
impl Bounding for BoundingBox {}
type Aabb = BoundingBox;

struct BoxCollider {
  box: Box,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for BoxCollider {}
impl Collider for BoxCollider {}
type Obb = BoxCollider;

// Cylinder Types

/// A cylinder with its origin at the center of the volume
struct Cylinder {
  radius: f32,
  height: f32,
}
impl Meshable for Cylinder {}

struct CylinderCollider {
  cylinder: Cylinder,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for CylinderCollider {}
impl Collider for CylinderCollider {}

// Capsule Types

struct Capsule {
  // Height of the cylindrical section
  height: f32,
  radius: f32,
}
impl Meshable for Capsule {}

struct CapsuleCollider {
  capsule: Capsule,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for CapsuleCollider {}
impl Collider for CapsuleCollider {}

// Cone Types

/// A cone with the origin located at the center of the circular base
struct Cone {
  height: f32,
  radius: f32,
}
impl Meshable for Cone {}

struct ConeCollider {
  cone: Cone,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for ConeCollider {}
impl Collider for ConeCollider {}

// Wedge Types

/// A ramp with the origin centered on the width, and coincident with the rear vertical wall.
struct Wedge {
  height: f32,
  width: f32,
  depth: f32,
}
impl Meshable for Wedge {}

struct WedgeCollider {
  wedge: Wedge,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for WedgeCollider {}
impl Collider for WedgeCollider {}

// Other Types

struct Torus {
  major_radius: f32,
  tube_radius: f32,
}
impl Meshable for Torus {}

// A 3d frustum used to represent the volume rendered by a camera, defined by the 6 planes that set the frustum limits.
struct Frustum {
  near: Plane,
  far: Plane,
  top: Plane,
  bottom: Plane,
  left: Plane,
  right: Plane,
}
impl Meshable for Frustum {}

```

### 2D Geometry Types

These types only exist in 2d space: their dimensions and location are only defined in `x` and `y` unlike their 3d counterparts. These types are suffixed with "2d" to disambiguate from the 3d types in user code, guide users to using 3d types by default, and remove the need for name-spacing the 2d and 3d types when used in the same scope.

```rust
struct Point2d(Vec2)

struct Direction2d(Vec2)
impl Meshable for Direction2d {}

struct Ray2d {
  point: Point2d, 
  direction: Direction2d
}
impl Meshable for Ray2d {}

struct Line2d {
  point: Point2d, 
  direction: Direction2d,
}
impl Meshable for Line2d {}
impl Collider2d for Line2d {}

struct LineSegment2d { 
  start: Point2d, 
  end: Point2d,
}
impl Meshable for LineSegment2d {}
impl Collider2d for LineSegment2d {}

struct PolyLine2d<const N: usize>{
  points: [Point2d; N],
}
impl Meshable for PolyLine2d {}
impl Collider2d for PolyLine2d {}

struct Triangle2d([Point2d; 3]);
impl Meshable for Triangle2d {}
impl Collider2d for Triangle2d {}

struct Quad2d([Point2d; 4]);
impl Meshable for Quad2d {}
impl Collider2d for Quad2d {}

/// A regular polygon, such as a square or hexagon.
struct RegularPolygon2d {
  /// The circumcircle that all points of the regular polygon lie on.
  circumcircle: Circle2d,
  /// Number of faces.
  faces: u8,
  /// Clockwise rotation of the polygon about the origin. At zero rotation, a point will always be located at the 12 o'clock position.
  orientation: Angle,
}
impl Meshable for RegularPolygon2d {}
impl Collider2d for RegularPolygon2d {}
 
struct Polygon2d <const N: usize>{
  points: [Point; N],
}
impl Meshable for Polygon2d {}
impl Collider2d for Polygon2d {}

/// Circle types

struct Circle2d {
  radius: f32,
}
impl Meshable for Circle2d {}

struct BoundingCircle2d {
  circle: Circle2d,
  translation: Vec2,
}
impl Meshable for BoundingCircle2d {}
impl Bounding2d for BoundingCircle2d {}

struct CircleCollider2d {
  sphere: Circle2d,
  translation: Vec2,
}
impl Meshable for CircleCollider2d {}
impl Collider2d for CircleCollider2d {}

// Box Types

struct Box2d {
  half_extents: Vec2,
}
impl Meshable for Box2d

struct BoundingBox2d {
  box: Box2d,
  translation: Vec2,
}
impl Meshable for BoundingBox2d {}
impl Bounding2d for BoundingBox2d {}
type Aabb2d = BoundingBox2d;

struct BoxCollider2d {
  box: Box2d,
  translation: Vec2,
  rotation: Mat2,
}
impl Meshable for BoxCollider2d {}
impl Collider2d for BoxCollider2d {}
type Obb2d = BoxCollider2d;

// Capsule Types

struct Capsule2d {
  // Height of the cylindrical section
  height: f32,
  radius: f32,
}
impl Meshable for Capsule2d {}

struct CapsuleCollider2d {
  capsule: Capsule2d,
  translation: Vec2,
  rotation: Mat2,
}
impl Meshable for CapsuleCollider2d {}
impl Collider2d for CapsuleCollider2d {}

```
### Lack of `Transform`s

Primitives colliders and bounding volumes are fully defined in space, and do not use `Transform` or `GlobalTransform`. This is an intentional decision. Because transforms can include nonuniform scale, they are fundamentally incompatible with shape primitives. We could use transforms in the future if the `translation`, `rotation`, and `scale` fields were distinct components, and shape primitives could be bundled with a `translation` or `rotation` if applicable.

#### Cache Efficiency

- Some primitives such as AABB and Sphere don't need a rotation (or scale) to be fully defined. 
- Using a `GlobalTransform` adds an unused Quat and Vec3 to the cache line.
- This is especially important for AABBs and Spheres, because they are fundamental to broad phase collision detection and BV(H), and as such need to be as efficient as possible.
- This also applies to all other primitives, which don't use the `scale` field of the `GlobalTransform`.

#### Ergonomics

- Primitives need to be fully defined in world space to compute collision or bounding.
- By making the primitive components fully defined and standalone, computing operations is as simple as: `primitive1.some_function(primitive_2)`, instead of also having query and pass in 2 `GlobalTransform`s in the (hopefully) correct order.

#### Use with Transforms

- The meshes generated from these primitives can be transformed freely. The primitive types, however, do not interact with transforms for the reasons stated above.

### Bounding Boxes/Volumes

Because bounding volumes and colliders are fully defined in world space, this leads to the natural question of how they are kept in sync with their parent. An implementation should provide a system similar to transform propagation, that would update the primitive as well as its translation and rotation if applicable. This can be accomplished in the transform propogation system itself, or in system that runs directly after. Further details are more appropriate for a subsequent bounding RFC/PR. The important point to consider is how this proposal provides common types that can be used for this purpose, whether for internal or external crates. The purpose of considering bounding and collision is that they represent common use cases of these primitives, and their potential implementation strategy should be considered.

### Frustum Culling

The provided `Frustum` type is useful for frustum culling, which is generally done on the CPU by comparing each frustum plane with each entity's bounding volume.

```rust
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

In addition, by defining the frustum as a set of planes, it is also trivial to support oblique frustums. Oblique frustums are useful for 2d projections used in CAD, as well as in games to take advantage of projection distortion to do things like emphasizing the feeling of speed in a driving game. See the Unity docs: https://docs.unity3d.com/Manual/ObliqueFrustum.html

### Ray Casting

The bounding volumes sections of this RFC cover how these types would be used for the bounding volumes which are used for accelerating ray casting. In addition, the `Ray` primitive component can be used to represent rays. Applicable 3d types could implement a `Raycast` trait to extend their functionality.

```rust
let ray = Ray::X;
let sphere = SphereCollider::new{Sphere{1.0}, Point::x(5.0));
let intersection = sphere.raycast(ray);
```

### Direction Normalization

The notes for `Direction` mention it is gauranteed to be normalized through its getter and setter. There are a few ways to do this, but I'd like to propose a zero-cost implementation using the typestate pattern. To absolutely minimize the use of `normalize` on the contained `Vec3`, we can memoize the result _only when accessed_. We don't want to make `Direction` an enum, as that will add a discriminant, and we don't want to have normalized and unnormalized types for of all our primitives. So instead, we could use the typestate pattern to do something like:

```rust
struct Plane {
  point: Point,
  normal: Direction,
}

struct Direction {
  direction: impl Normalizable,
}

struct UncheckedDir(Vec3);
impl Normalizable for UncheckedDir {}

struct NormalizedDir(Vec3);
impl Normalizable for NormalizedDir {}

```

When a `Direction` is mutated or built, the direction will be an `UncheckedDir`. Once it is accessed, the `Directions`s getter method will normalize the direction, and swap it out with a `NormalizedDir`. Now the normalized direction is memoized in the `Direction` without increasing the size of the type. This complexity can be completely hidden to users.

## Drawbacks

Adding primitives invariably adds to the maintenance burden. However, this cost seems worth the benefits of the needed engine functionality that can be built on top of it.

## Rationale and alternatives

An argument could be made to use an external crate for shape primitives, however these types are so fundamental It's important that they are optimized for the engine's most common use cases, and are not from a generalized solution. In addition, the scope of this RFC does not include the implementation of features like bounding, collision, raycasting, or physics. These are acknowledged as areas that (for now) should be carried out in external crates.

## Prior art

- Unity `PrimitiveObjects`: https://docs.unity3d.com/Manual/PrimitiveObjects.html
- Godot `PrimitiveMesh`: https://docs.godotengine.org/en/stable/classes/class_primitivemesh.html#class-primitivemesh
- The popular *Shapes* plugin for Unity https://acegikmo.com/shapes/docs/#line

Prior art was used to select the most common types of shape primitive, naming conventions, as well as sensible data structures for bounding, collision, and culling.

Many game engine docs appear to have oddly-named and disconnected shape primitive types that are completely unrelated. This RFC aims to ensure Bevy doesn't go down this path, and instead derives functionality from common types to take advantage of the composability of components in the ECS.

## Unresolved questions

What is the best naming scheme, e.g., `2d::Line`/`3d::Line` vs. `Line2d`/`Line3d` vs. `Line2d`/`Line`?

### Out of Scope

- Value types, e.g. float vs. fixed is out of scope. This RFC is focused on the core geometry types and is intended to use Bevy's common rendering value types such as `f32`.
- Implementation of bounding, collisions, physics, raycasting, meshing, etc.

## Future possibilities

- Bounding boxes
- Collisions
- Frustum Culling
- Froxels for forward clustered rendering
- Ray casting
- Physics
- SDF rendering
- Immediate mode debug Rendering (just call `.mesh()`!)
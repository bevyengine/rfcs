# Feature Name: `primitive-shapes`

## Summary

These geometric primitives, or "primitive shapes", are lightweight types for use across Bevy engine crates and as interoperability types for plugins and libraries. The goal is to provide an API that considers ergonomics for the common use cases, such as meshing, bounding, and collision, without sacrificing performance.

## Motivation

This provides a type-level foundation that subsequent engine (and plugin!) features can be built upon incrementally, preventing ecosystem fragmentation. A major goal is to make it possible for engine and game developers to prototype physics, bounding, collision, and raycasting functionality using the same base types. This will make engine integration and cross-plugin interoperability much more feasible, as the engine will have opinionated types that will be natural to implement `into` or `from`. I hope this opens the door to experimentation with multiple physics and bounding acceleration backends.

There is significant complexity in the way seemingly equivalent shapes are defined depending on how they are used. By considering these use cases *first*, the goal is that geometric data structures can be composed depending on the needed functionality, with traits used to ensure the right data structure is used for the right task.

## User-Facing Explanation

Geometric primitives are lightweight representations of geometry that describe the type of geometry as well as its dimensions. These primitives are *not* meshes, but the underlying precise mathematical definition. For example, a circle is:

```rust
pub struct Circle {
  radius: f32,
}
```

Note that a `Circle` does not contain any information about the position of the circle. This is due to how shapes are composed to add more complex functionality. Consider the following common use cases of primitives shapes, as well as the the information (Translation and Rotation) that are needed to fully define the shapes for these cases:

| Shape     | Mesh | Bounding | Collision |
|-----------|----|------------------|---|
| Sphere    | ✔ | ✔  + Trans       | ✔ + Trans |
| Box       | ✔ | ✔ + Trans (AABB) | ✔ + Trans + Rot (OBB) |
| Capsule   | ✔ | ❌               | ✔ + Trans + Rot |
| Cylinder  | ✔ | ❌               | ✔ + Trans + Rot |
| Cone      | ✔ | ❌               | ✔ + Trans + Rot |
| Wedge     | ✔ | ❌               | ✔ + Trans + Rot |
| Plane     | ✔ | ❌               | ✔ |
| Torus     | ✔ | ❌               | ❌ |

### Bounding vs. Collision

The difference between the two is somewhat semantic. Both bounding and collision check for intersections between bounding volumes or areas. However, bounding volumes are generally used for broad phase checks due to their speed and small size, and are consequently often used in tree structures for spatial queries. Note that both bounding types are not oriented - this reduces the number of comparisons needed to check for intersections during broad phase tests. Bounding spheres or AABBs are generally always preferred for broad phase bounding checks, despite the OBB's name.

**Note that the Bounding/Collision interface is not part of this RFC**, the purpose of this discussion is to present common use cases of these primitive shape types to uncover deficiencies in their design.

### Where are the `Transform`s?

In this potential bounding and collision interface, translation and rotation are **not** defined using bevy's `Transform` components, nor are they stored in the base primitive shape types. This is because types that implement `Bounding` and `Collider` must be fully self-contained. This:

* makes the API simpler when using these components in functions and systems
* ensures bounding and collision types use an absolute minimum of memory
* prevents errors caused by nonuniform scale invalidating the shape of the primitive.

Instead, changes to the parent's `GlobalTransform` should be used to derive a new primitive on scale changes - as well as a new translation and rotation if required.

Although bounding and collision are out of scope of this RFC, this helps to explain why primitive shape types contain no positional information. This hypothetical bounding/collision interface shows how the base primitive shapes can be composed to build common game engine functinality without adding memory overhead.

### Meshing

Both 2d and 3d primitives could implement the `Meshable` trait to provide the ability to generate a tri mesh:

```rust
let circle_mesh: Mesh = Circle{ radius: 2.0 }.mesh();
```
The base primitive types only define the shape (`Circle`) and size (`radius`) of the geometry about the origin. Once generated, the mesh can have a transform applied to it like any other mesh, and it is no longer tied to the primitive that generated it. The `Default::default()` implementation of primitives should be a "unit" variant, such as a circle with diameter 1.0, or a box with all edges of length 1.0.

Meshing could be naturally extended with other libraries or parameters. For example, a sphere by default might use `Icosphere`, but could be extended to generate UV spheres or quad spheres. For 2d types, we could use a crate such as `lyon` to generate 2d meshes from parameterized primitive shapes within the `Meshable` interface.

### Bounding

For use in spatial queries, broad phase collision detection, and raycasting, hypothetical `Bounding` and `Bounding2d` traits are provided. These traits are implemented for types that define position in addition to the shape and size defined in the base primitive. The basic functionality of this trait is to check whether one bounding shape is contained within another bounding shape.

### Colliders

Colliders can provide intersection checks against other colliders, as well as check if they are within a bounding volume.

### 2D and 3D

This RFC provides independent 2d and 3d primitives. Types that are present in both 2d and 3d space are suffixed with "2d" or "3d" to disambiguate them such as `Line2d` and `Line3d`. Types that only exist in either space have no suffix. 

* 2D shapes have their dimensions defined in `x` and `y`. 
* 3D shapes have their dimensions defined in `x`, `y` and `z`

The complete overview of shapes, their dimensions and their names can be seen in this table:

| Shape           | 2D              | 3D            | Description                                                                              |
|-----------------|-----------------|---------------|------------------------------------------------------------------------------------------|
| Rectangle       | Rectangle       | -             | A rectangle defined by its width and height                                              |
| Circle          | Circle          | -             | A circle defined by its radius                                                           |
| Polygon         | Polygon         | -             | A closed shape in a plane defined by a finite number of line-segments between vertices   |
| RegularPolygon  | RegularPolygon  | -             | A polygon where all vertices lies on the circumscribed circle, equally far apart         |
| Plane           | Plane2d         | Plane3d       | An unbounded plane defined by a position and normal                                      |
| Direction       | Direction2d     | Direction3d   | A normalized vector pointing in a direction                                              |
| Ray             | Ray2d           | Ray3d         | An infinite half-line pointing in a direction                                            |
| Line            | Line2d          | Line3d        | An infinite line                                                                         |
| LineSegment     | LineSegment2d   | LineSegment3d | A finite line between two vertices                                                       |
| Polyline        | Polyline2d      | Polyline3d    | A line drawn along a path of vertices                                                    |
| Triangle        | Triangle2d      | Triangle3d    | A polygon with 3 vertices                                                                |
| Quad            | Quad2d          | Quad3d        | A polygon with 4 vertices                                                                |
| Sphere          | -               | Sphere        | A sphere defined by its radius                                                           |
| Box             | -               | Box           | A cuboid with six quadrilateral faces, defined by height, width, depth                   |
| Cylinder        | -               | Cylinder      | A cylinder with its origin at the center of the volume                                   |
| Capsule         | -               | Capsule       | A capsule with its origin at the center of the volume                                    |
| Cone            | -               | Cone          | A cone with the origin located at the center of the circular base                        |
| Wedge           | -               | Wedge         | A ramp with the origin centered on the width, and coincident with the rear vertical wall |
| Torus           | -               | Torus         | A torus, shaped like a donut                                                             |
| Frustum         | -               | Frustum       | The portion of a pyramid that lies between two parallel planes                           |

## Implementation strategy

### Helper Types

```rust
/// Stores an angle in radians, and supplies builder functions to prevent errors (from_radians, from_degrees)
struct Angle(f32);
```

### Traits (REFERENCE ONLY!)

**These traits are provided as reference to illustrate how these primitive shape types might be used. The details of implementation and interface should be determined in a separate RFC, PR, or independent prototypes.**

```rust
trait Meshable{
  fn mesh(&self) -> Mesh;
};

trait Bounding2d {
  fn within(&self, other: &impl Bounding2d) -> bool;
  fn contains(&self, collider: &impl Collider2d) -> bool;
}

trait Bounding3d {
  fn within(&self, other: &impl Bounding3d) -> bool;
  fn contains(&self, collider: &impl Collider3d) -> bool;
}

trait Collider2d {
  fn collide(&self, other: &impl Collider2d) -> Option(Collision2d);
  fn within(&self, bounds: &impl Bounding2d) -> bool;
}

trait Collider3d {
  fn collide(&self, other: &impl Collider3d) -> Option(Collision3d);
  fn within(&self, bounds: &impl Bounding3d) -> bool;
}
```
### 2D Geometry Types

```rust
struct Direction2d(Vec2)
impl Meshable for Direction2d {}

struct Ray2d {
  point: Vec2, 
  direction: Direction2d
}
impl Meshable for Ray2d {}

struct Line2d {
  point: Vec2, 
  direction: Direction2d,
}
impl Meshable for Line2d {}

/// A finite line between two vertices
struct LineSegment2d { 
  start: Vec2, 
  end: Vec2,
}
impl Meshable for LineSegment2d {}

/// A line drawn along a path of vertices 
struct PolyLine2d<const N: usize>{
  points: [Vec2; N],
}
impl Meshable for PolyLine2d {}

struct Triangle2d([Vec2; 3]);
impl Meshable for Triangle2d {}

struct Quad2d([Vec2; 4]);
impl Meshable for Quad2d {}

/// A closed shape in a plane defined by a finite number of line-segments between vertices
struct Polygon2d <const N: usize>{
  points: [Vec2; N],
}
impl Meshable for Polygon2d {}

/// A polygon where all vertices lies on the circumscribed circle, equally far apart
/// Example: Square, Triangle, Hexagon
struct RegularPolygon2d {
  /// The circumcircle that all points of the regular polygon lie on.
  circumcircle: Circle2d,
  /// Number of faces.
  faces: u8,
  /// Clockwise rotation of the polygon about the origin. At zero rotation, a point will always be located at the 12 o'clock position.
  orientation: Angle,
}
impl Meshable for RegularPolygon2d {}


// Circle types

/// A circle defined by its radius
struct Circle {
  radius: f32,
}
impl Meshable for Circle {}

/* REFERENCE ONLY
struct BoundingCircle2d {
  circle: Circle,
  translation: Vec2,
}
impl Meshable for BoundingCircle2d {}
impl Bounding2d for BoundingCircle2d {}

struct CircleCollider {
  sphere: Circle,
  translation: Vec2,
}
impl Meshable for CircleCollider {}
impl Collider2d for CircleCollider {}
*/

// Box Types

/// A rectangle defined by its width and height
struct Rectangle {
  half_extents: Vec2,
}
impl Meshable for Rectangle

/* REFERENCE ONLY
struct BoundingBox2d {
  box: Rectangle,
  translation: Vec2,
}
impl Meshable for BoundingBox2d {}
impl Bounding2d for BoundingBox2d {}
type Aabb2d = BoundingBox2d;

struct RectangleCollider {
  box: Rectangle,
  translation: Vec2,
  rotation: Mat2,
}
impl Meshable for RectangleCollider {}
impl Collider2d for RectangleCollider {}
type Obb2d = RectangleCollider;
*/

// Capsule Types

struct Capsule2d {
  height: f32, // Height of the rectangular section
  radius: f32, // End cap radius
}
impl Meshable for Capsule2d {}

/* REFERENCE ONLY
struct CapsuleCollider2d {
  capsule: Capsule2d,
  translation: Vec2,
  rotation: Mat2,
}
impl Meshable for CapsuleCollider2d {}
impl Collider2d for CapsuleCollider2d {}
*/

```

### 3D Geometry Types

```rust
/// Vector direction in 3D space that is guaranteed to be normalized through its getter/setter.
struct Direction3d(Vec3)
impl Meshable for Direction3d {}

struct Plane3d {
  point: Vec3,
  normal: Direction3d,
}
impl Meshable for Plane3d {}

/// Differentiates a line from a ray, where a line is infinite and a ray is directional half-line, although their underlying representation is the same.
struct Ray3d(
  point: Vec3, 
  direction: Direction3d,
);
impl Meshable for Ray3d {}

// Line types

/// Unbounded line in 3D space with directionality
struct Line3d { 
  point: Vec3, 
  direction: Direction3d,
}
impl Meshable for Line3d {}

/// A line segment bounded by two points
struct LineSegment3d { 
  start: Vec3, 
  end: Vec3,
}
impl Meshable for LineSegment3d {}

/// A line drawn along a path of points
struct PolyLine3d {
  points: Vec<Vec3>,
}
impl Meshable for PolyLine3d {}

struct Triangle3d([Vec3; 3]);
impl Meshable for Triangle3d {}
impl Collider for Triangle3d {}

struct Quad3d([Vec3; 4]);
impl Meshable for Quad3d {}
impl Collider for Quad3d {}

// Sphere types

struct Sphere {
  radius: f32,
}
impl Meshable for Sphere {}

/* REFERENCE ONLY
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
*/

// Box Types

struct Box {
  half_extents: Vec3,
}
impl Meshable for Box

/* REFERENCE ONLY
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
*/

// Cylinder Types

/// A cylinder with its origin at the center of the volume
struct Cylinder {
  radius: f32,
  height: f32,
}
impl Meshable for Cylinder {}

/* REFERENCE ONLY
struct CylinderCollider {
  cylinder: Cylinder,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for CylinderCollider {}
impl Collider for CylinderCollider {}
*/

// Capsule Types

struct Capsule {
  // Height of the cylindrical section
  height: f32,
  radius: f32,
}
impl Meshable for Capsule {}

/* REFERENCE ONLY
struct CapsuleCollider {
  capsule: Capsule,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for CapsuleCollider {}
impl Collider for CapsuleCollider {}
*/

// Cone Types

/// A cone with the origin located at the center of the circular base
struct Cone {
  height: f32,
  radius: f32,
}
impl Meshable for Cone {}

/* REFERENCE ONLY
struct ConeCollider {
  cone: Cone,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for ConeCollider {}
impl Collider for ConeCollider {}
*/

// Wedge Types

/// A ramp with the origin centered on the width, and coincident with the rear vertical wall.
struct Wedge {
  height: f32,
  width: f32,
  depth: f32,
}
impl Meshable for Wedge {}

/* REFERENCE ONLY
struct WedgeCollider {
  wedge: Wedge,
  translation: Vec3,
  rotation: Quat,
}
impl Meshable for WedgeCollider {}
impl Collider for WedgeCollider {}
*/

// Other Types

struct Torus {
  major_radius: f32,
  tube_radius: f32,
}
impl Meshable for Torus {}

/// A 3d frustum used to represent the volume rendered by a camera, defined by the 6 planes that set the frustum limits.
struct Frustum {
  near: Plane3d,
  far: Plane3d,
  top: Plane3d,
  bottom: Plane3d,
  left: Plane3d,
  right: Plane3d,
}
impl Meshable for Frustum {}

```

### Bounding and Collision

Primitives colliders and bounding volumes are fully defined in space (in their hypothetical implementation here), and do not use `Transform` or `GlobalTransform`. This is an intentional decision. Because transforms can include nonuniform scale, they are fundamentally incompatible with shape primitives. We could use transforms in the future if the `translation`, `rotation`, and `scale` fields were distinct components, and shape primitives could be bundled with a `translation` or `rotation` if applicable.

The provided reference implementaiton of bounding and collision types demonstrate how primitive shape types can be composed with other types, triats, or components to build added functionality from shared parts. While the collder and bounding primitives provided as reference are out of scope, they also highlight the incompatibility of shape primitives with the `Transform` component.

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

In addition, by defining the frustum as a set of planes, it is also trivial to support oblique frustums. Oblique frustums are useful for 2d projections used in CAD, as well as in games to take advantage of projection distortion to do things like emphasizing the feeling of speed in a driving game. See the Unity docs: <https://docs.unity3d.com/Manual/ObliqueFrustum.html>

### Ray Casting

The bounding volumes sections of this RFC cover how these types could be used for the bounding volumes which are used for accelerating ray casting. In addition, the `Ray` primitive component can be used to naturally represent raycasting rays. Applicable 3d types could implement a `Raycast` trait to extend their functionality.

```rust
let ray = Ray3d::X;
let sphere = SphereCollider::new{Sphere{1.0}, Vec3::x(5.0));
let intersection = sphere.raycast(ray);
```

### Direction Normalization

The notes for `Direction` mention it is gauranteed to be normalized through its getter and setter. There are a few ways to do this, but I'd like to propose a zero-cost implementation using the typestate pattern. To absolutely minimize the use of `normalize` on the contained `Vec3`, we can memoize the result _only when accessed_. We don't want to make `Direction` an enum, as that will add a discriminant, and we don't want to have normalized and unnormalized types for of all our primitives. So instead, we could use the typestate pattern to do something like:

```rust
struct Plane3d {
  point: Vec3,
  normal: Direction3d,
}

struct Direction3d {
  direction: impl Normalizable,
}

struct UncheckedDir(Vec3);
impl Normalizable for UncheckedDir {}

struct NormalizedDir(Vec3);
impl Normalizable for NormalizedDir {}

```

When a `Direction` is mutated or built, the direction will be an `UncheckedDir`. Once it is accessed, the `Directions`s getter method will normalize the direction, and swap it out with a `NormalizedDir`. Now the normalized direction is memoized in the `Direction3d` without increasing the size of the type. This complexity can be completely hidden to users.

## Drawbacks

Adding primitives invariably adds to the maintenance burden. However, this cost seems worth the benefits of the needed engine functionality that can be built on top of it.

## Rationale and alternatives

An argument could be made to use an external crate for shape primitives, however these types are so fundamental It's important that they are optimized for the engine's most common use cases, and are not from a generalized solution. In addition, the scope of this RFC does not include the implementation of features like bounding, collision, raycasting, or physics. These are acknowledged as areas that (for now) should be carried out in external crates.

### Using Parry/Rapier Types

"Parry is the defacto standard for physics, why not use those types? If we make our own types, doesn't this add overhead if everyone is using Parry?"

The choice of what physics engine to integrate into Bevy, if at all, will require much more discussion than can be covered in this RFC. However, it is true that Parry appears to be the most common choice for physics in Bevy, and its types should be considered for this RFC.

Parry uses nalgebra for linear algebra; we would need to convert to/from glam types to interoperate with the rest of the engine whether or not we use Parry's geometric primitives. In that sense, making our own types doesn't add overhead to conversion, but we _would_ own the process. Parry plugins already exist for bevy. Adding our own primitive types to the engine wouldn't break those plugins, but it does add the ability to interoperate.

Using Parry types would also mean every bevy crate that wants to implement some geometry focused feature, now needs to use the nalgebra and glam math types. This runs counter to Bevy's goal of simplicity.

Perhaps most critically, Parry types are opinionated for physics/raycasting use. The goals of those types simply don't align with the goals outlined in this RFC. Bevy's shape primitives should be able to be used, for example, for meshing in 2d (UI/CAD) and 3d, frustum culling, clustered rendering, or anything else that that lives in the engine <-> application stack. If we want to add a 2d arc type for UI, we would need to to add primitives to Parry that are completely orthogonal to its goals.

## Prior art

* Unity `PrimitiveObjects`: <https://docs.unity3d.com/Manual/PrimitiveObjects.html>
* Godot `PrimitiveMesh`: <https://docs.godotengine.org/en/stable/classes/class_primitivemesh.html#class-primitivemesh>
* The popular *Shapes* plugin for Unity <https://acegikmo.com/shapes/docs/#line>
* Rapier/Parry

Prior art was used to select the most common types of shape primitive, naming conventions, as well as sensible data structures for bounding, collision, and culling.

Many game engine docs appear to have oddly-named and disconnected shape primitive types that are completely unrelated. This RFC aims to ensure Bevy doesn't go down this path, and instead derives functionality from common types to take advantage of the composability of components in the ECS.

### Out of Scope

* Value types, e.g. float vs. fixed is out of scope. This RFC is focused on the core geometry types and is intended to use Bevy's common rendering value types such as `f32`.
* Implementation of bounding, collisions, physics, raycasting, meshing, etc.

## Future possibilities

* Bounding boxes
* Collisions
* Frustum Culling
* Froxels for forward clustered rendering
* Ray casting
* Physics
* SDF rendering
* Immediate mode debug Rendering (just call `.mesh()`!)

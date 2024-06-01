# Dedicated 2D Transform

## Summary

The ergonomics of using Bevy's existing Transform for 2D games is poor. To counter this a 2D Transform component should be created that has improved ergonomics.

## Motivation

While it is readily possible to create 2D games in Bevy today, interacting with the existing Transform system is awkward, high in "ceremony", and prone to unintentional human error.  There are two main issues:

1. Translation is represented as a `Vec3` with the draw order set implicity based on the Z value
2. Rotation being represented as a Quat even though all rotations are around the Z axis

### The Problem with Translation

Translation in Bevy is stored as a `Vec3` in the Transform component. This is fine for 3D games, and is usable for 2D games, but has several issues.

#### Overloading the Concept

In 2D games this `Vec3` represents two seperate concepts - the 2D position (X,Y) of the entity, and the draw order (Z) of the entity. Vector maths, thus, can end up changing the draw order value when done on the full `Vec3`, leading to graphical issues being caused by systems that seem unrelated to graphics. This can be easy to forget, and difficult to debug. Avoiding this requires writing a lot boilerplate, which leads to...

#### Poor Ergonomics

In order to avoid issues caused by the above overloading any system that interacts with an entities translation will often store the entities Z, copy a `Vec2` of the translation using either xy() or truncate(), perform any necessary operations, then re-extend the `Vec2` back to a `Vec3` with the stored Z before reassigning the value to the Transforms translation. The effect of this is that a 2D games code is littered with truncate() and extend(). This is then further exasperated when doing operations like Raycasting where maintaining the existing Z value will give incorrect results (including Z distance in ray calculations). All in all, this is a poor user experience.

### The Problem with Rotation

Rotation in bevy is stored as a `Quat` in the Transform component. Again, this is fine for 3D games, and is usable for 2D games, but has poor ergonomics. Rotation in a 2D game in Bevy is always counterclockwise around the Z axis. As with using `Vec3`'s for Translation this causes a lot of boilerplate, extracting the value of a quaternions rotation around Z in radians, performing calculations on that value, then creating a new `Quat` rotating around Z before reassigning. Much as with Translation, this is a poor user experience.

## User-facing explanation

Transform2D represents a 2D entities position, rotation, and scale in world space. It is defined as follows:

```rust
pub struct Transform2d {
  position: Vec2,
  rotation: f32 //RFC Note: We could use Rotation2d here
  scale: Vec2
}
```

If you're making a 2D game and you want your entity to appear in the world at a given location it must have a `Transform2D` component. `Position` represents your entities location in the world (actually it's position on the 'xy-plane'). `Rotation` represents your entities anti-clockwise rotation in radians (around the Z-axis). `Scale` represents the entities scale in the world (like `Position` this is actually the entities scale on the 'xy-plane').

Many entities in a 2D game will have a sprite, a mesh, or some other graphics attached to them. By default, all these entities are drawn on the same 'layer'. When entities are drawn the last entity drawn appears on top of thosed drawn before it in the same location. Developers can control which layer an entity is on by changing that entities `DrawLayer`:

```rust
pub struct DrawLayer(pub f32);
```

As you can see `DrawLayer` is just a wrapper around an `f32`. Entities with a higher value for their `DrawLayer` will be drawn on top of those with lower values.

## Implementation strategy

### Components

#### Transform

A `Transform2d` component should be created as follows:

```rust
pub struct Transform2d {
  position: Vec2,
  rotation: f32 //Note: Optionally a Rotation2d
  scale: Vec2
}
```

The existing `Transform` component should be be kept as is to maintain backwards compatibility. 

(Note: If we don't care about said backwards compatibility I would choose to rename `Transform` to `Transform3d`)

#### DrawOrder

A `DrawOrder` component should be created as follows:

```rust
pub struct DrawLayer(pub f32);
```
For entities with parent/child relationships, child entities should have their `DrawOrder` propogated to an offset from their parents `DrawOrder` as with the existing `Transform` component. We should not do anything more than this (ie. we should not attempt to automatically set child entities to some lower value of `DrawOrder` than their parent).

### Bundles

#### Spatial2dBundle

A new spatial bundle should be created as per the existing `SpatialBundle`, but containing a `Transform2d` instead of a `Transform`.

#### SpriteBundle/SpriteSheetBundle

Two new bundles should be created `Sprite2dBundle` and `SpriteSheet2dBundle` matching the existing `SpriteBundle` and `SpriteSheetBundle` except replacing `Transform` with `Transform2d`.

The existing `SpriteBundle` and `SpriteSheetBundle` should be kept as is to maintain backwards compatibility.

(Note: If we don't care about said backwards compatibility I would choose to rename the existing bundles to `Sprite3dBundle` and `SpriteSheet3dBundle`)

### Systems

New versions of `sync_simple_transforms`, `propogate_transforms`, and `propogate_transforms_recursive` should be written that propogate the new `Transform2d` into the existing `GlobalTransform`. 

## Drawbacks

### Maintenance Burden

Beyond having three new systems and some new components to maintain Bevy as a whole would need to choose to either:

* Offer `Vec2` versions of many of it's `Vec3` functions, potentially labelling as eg. `RayCast2d` vs `RayCast`
* Require developers to call `extend()` and use the `Vec3` versions of these functions. Which may defy the point of doing this in the first place.

This burden is unlikely to go away once this decision is made, for the lifetime of the engine.

### Ecosystem Pressure

As above, many ecosystem crates (`bevy_xpbd`, `bevy_rapier`, for example) would need to answer the same question. Either offering `Vec2` versions or requiring use of `extend()`. In some cases these crates already do this, but this would 'force the question'.

### One Way Door?

Once this door is passed through, it may be difficult to walk back, its likely that UI will want similar versions of `Transform` for their needs and ecosystem crates would likely start to support `Transform2d`. Once this change is made, it is likely made permanently.

## Rationale and alternatives

There are really only two options:

* Have a `Tranform2d` component
* Don't have a `Transform2d` component

The existing system has poor ergonomics, not making this change will continue to favor the development of 3D games, make 2D games more difficult to develop and more prone to error. I believe this may, in the long term, harm uptake of the engine given the large proportion of indie games are 2D and Indie devs are more likely to risk using a new 'less proven' tech stack like Bevy and Rust.

For `DrawOrder` we could have a specific layer system, where a user registers draw layers, and selects from a set of pre-registered draw layers, but this doesn't handle relative positioning. We could handle relative positioning in parent/child relationships by associating some float range and sharing the range between children but many developers would need to manually work around this in order to have children appear on the layers they want (see the existing demand for better control of transform propogation).

In regards to support burden, this support burden must 'be paid' either way, either in the engine, or in the game. Right now we require that burden to be paid by the game developer. I believe it is more correct for that support burden to be paid by the engine and reduce the burden on the game developer.

In regards to the ecosystem effect, right now functionality like this already exists in several crates. There is no standardized solution, and developers would need to know of these possible solutions before starting - something they're unlikely to know given they are unlikely to be embedded in the ecosystem. A standardized in engine solution would improve user experience, improve indie uptake, and - I believe - improve the crate ecosystem by allowing for better targetting of 2D specific crates.

## Unresolved questions

- Do we want to take this burden ourselves?
- Are we happy about using a fairly simple/naive `DrawOrder` implementation?
- What are the knock-on effects on UI and its desire for a potential `UiTransform`/`TransformUi`

## Future possibilities

I have my own crate `rantz_spatial2d` which I'm using in my own game. It's far from perfect and poorly documented, but in it, I have split `Transform2d` into its component parts of `Position2D`, `Rotation2D`, and `Scale2D`. It's out of the scope of this RFC, but often, in 2D games, you actually don't want the whole Transform, only a segment of it, and it may be beneficial to consider splitting `Transform2d` and `Transform` in the same way at some point. Again, outside the scope of this RFC, but an interesting thought.

# Feature Name: `reflexive-world-queries`

## Summary

Types created using `#[derive(WorldQuery)]` are now reflexive,
meaning we no longer generate 'Item' structs for each custom WorldQuery.
This makes the API for these types much simpler.

## Motivation

You, a humble user, wish to define your own custom `WorldQuery` type.
It should display the name of an entity if it has one, and fall back
to showing the entity's ID if it doesn't.

```rust
#[derive(WorldQuery)]
struct DebugName {
    name: Option<&'static Name>,
    id: Entity,
}

impl Debug for DebugName {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt:Result {
        if let Some(name) = self.name {
            write!(f, "{name}")
        } else {
            write!(f, "Entity {:?}", self.id)
        }
    }
}
```

This type is easy define, and should be very useful.
However, you notice a strange error when trying to use it:

```rust
fn my_system(q: Query<DebugName>) {
    for dbg in &q {
        println!("{dbg:?}");
    }
}
```

```
error[E0277]: `DebugNameItem<'_>` doesn't implement `Debug`
   --> crates\bevy_core\src\name.rs:194:20
    |
194 |         println!("{dbg:?}");
    |                    ^^^ `DebugNameItem<'_>` cannot be formatted using `{:?}`
    |
    = help: the trait `Debug` is not implemented for `DebugNameItem<'_>`
    = note: add `#[derive(Debug)]` to `DebugNameItem<'_>` or manually `impl Debug for DebugNameItem<'_>`
    = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
```

"Rustc, what are you talking about?" you ask. You just implemented `Debug`!

The problem here is that the type returned from the query is *not* the type we just defined
-- it's an hidden macro-generated type with near-identical fields.
Here's what it looks like in the docs:

```rust
pub struct DebugNameItem<'__w> {
    pub name: <Option<&'static Name> as WorldQuery>::Item<'__w>,
    pub entity: <Entity as WorldQuery>::Item<'__w>,
}
```

In order to fix the error, we need to implement `Debug` for *this* type:

```rust
impl Debug for DebugNameItem<'_> { ... }
```

This successfully avoid the compile error, but it results in an awkward situtation where our documentation,
trait impls, and methods are spread accross two very similar types with a non-obvious distinction between them.
In addition, the `DebugNameItem` struct has necessarily vague documentation due to being generated in a macro.

## User-facing explanation

### **Reflexive**

(Adjective) *In reference to oneself.*

Any `WorldQuery` type used within `#[derive(WorldQuery)]` must be reflexive,
meaning it returns itself when used in a query.
Most `WorldQuery` types are reflexive, and may be used without consideration of this property.
A notable exception is `&mut T`, which is incompatible with the derive macro.
`Mut<T>` must be used instead.

Examples:

```rust
#[derive(WorldQuery)]
struct DebugName<'w> {
    name: Option<&'w Name>,
    id: Entity,
}

impl Debug for DebugName<'_> { ... }

#[derive(WorldQuery)]
#[world_query(mutable)]
struct PhysicsComponents<'w> {
    mass: &'w Mass,
    transform: Mut<'w, Transform>,
    velocity: Mut<'w, Velocity>,
}
```

## Implementation strategy

Changing the derive macro to be reflexive should be a trivial change,
and will not be discussed at length in this RFC.

To support this change, we will need to rework some of our built-in
`WorldQuery` types to be reflexive.

```rust
// Before: 
fn my_system(q: Query<Changed<T>>) {
    for flag in &q {
        ...
    }
}

// After:
fn my_system(q: Query<Changed<T>>) {
    for Changed(flag) in &q {
        ...
    }
}
```

Since `&mut T` is not reflexive, we will have to implement `WorldQuery` for `Mut<T>`.

## Drawbacks

This will add slightly more friction in some cases, non-reflexive `WorldQuery`
types such as `&mut T` are forbidden from being used in the derive macro.
However, these types can still be used in anonymous `WorldQuery`s or types
such as `type MyQuery = (&'static mut A, &'static mut U)`.

Since the `WorldQuery` derive is only used in cases where the user wishes
to expose a public API, we should prioritize the cleanliness of that API
over the ease of implementing it.

## Rationale and alternatives

Do nothing. The current situation is workable, but it's not ideal.
Generated 'Item' structs add signifcant noise to a plugin's API,
and make it more difficult to understand due to having several very similar types.

## Unresolved questions

None.

## Future possibilities

We should explore ways to make the `#[world_query(mutable)]` attribute unnecessary.

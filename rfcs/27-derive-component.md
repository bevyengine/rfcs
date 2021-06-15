# Feature Name: `derive_component`

## Summary

Remove blanket impl from `Component` trait, and require it to be manually implemented or derived using `#[derive(Component)]`.
Extend the `Component` trait to declare more information statically, instead of requiring runtime registration.

## Motivation

`Component` trait is being used inside Bundles and Queries. It's usage in Queries is enforced by the type system.
Unfortunately, because it's automatically derived, almost every type can be treated as a component, including Bundles.
That means `Component` APIs don't really have any way to prevent misuse, like `commands.insert(bundle)`. Right now
it's very easy to end up with that code because of a typo, as the `Bundle` api methods are very similarly named.

There are also other pitfalls, like primitive types being automatically `Component`s, but without clear meaning what they represent.
This is an especially sever issue for function pointer types. It's easy to use an enum variant or newtype constructor as
an component, like `.insert(MyEnum::Variant)`,  without realising it. This leads to to very hard to spot bugs without any hint
of either a compiler or runtime that something went wrong. This is easily preventable by not implementing `Component` trait for those types.

We also already have `#[derive(Bundle)]` and others. Adding that to `Component` keeps the syntax similar across the board.

Apart from requiring explicit opt-in, `derive` gives us a natural place to define more metadata about specific component type.
One way to use that is to declare component's Storage type statically, instead of requiring manual registration of that metadata
into the world.

```rust
#[derive(Component)]
#[component(storage = "SparseSet")]
struct MyComponent(..);
```

This enables compiler to perform much more optimizations in the query iteration code, as a lot of the code paths can be statically eliminated.

## User-facing explanation

In order to define your own component type, you have to implement a `Component` trait. The easiest way to do that is using a `derive` macro.

```rust
#[derive(Component)]
struct HitPoints(u32);
```

If you forget to properly annotate your component type, the compiler will remind you
of that as soon as you try to insert that component into the world or query for it. Apart from type safety, this also provides a visual indication that given type
is intended to be directly associated with entities in the world.

By default, your component will be stored using Dense storage strategy.
In order to declare your component's storage strategy manually, you have to provide an additional attribute
to specify the component settings.

```rust
#[derive(Component)]
#[component(storage = "SparseSet")]
struct Selected;
```

### Migration strategy

All your component types must be annotated with a derive.

```diff
+ #[derive(Component)]
  struct MyComponent { .. }
```

You can no longer use primitive types (e.g. `u32`, `bool`) as components.
Wrap them in a custom struct, and mark that as a component.

```diff
+ #[derive(Component)]
+ struct HitPoints(u32);
  
- commands.entity(id).insert(100);
+ commands.entity(id).insert(HitPoints(100));
```

Remove all registration of components. Make sure to correctly annotate components that were previously registered with `SparseSet` storage type.

```diff
- world.register_component(
-     ComponentDescriptor::new::<MyComponent>(StorageType::SparseSet)
- ).unwrap();

+ #[derive(Component)]
+ #[component(storage = "SparseSet")]
  struct MyComponent { .. }
```

## Implementation strategy

- remove the existing blanket impl
- implement basic derive macro that just implements a marker trait
- add associated `Storage` type to the `Component` trait
- provide a way to specify the `Storage` type using an macro attribute.
- use the associated `Storage` type inside query iterators to statically determine
  if the query is sparse or dense.

## Drawbacks

Requiring extra annotation on every component type adds some boilerplate.

## Rationale and alternatives

- instead of `derive`, use an attribute macro.
- Do nothing and keep registering metadata at runtime. Live with the branching inside query iterators.
- Require manual implementation of `Component` trait without providing derive macro.

## Unresolved questions

- How to support dynamic components for future scripting layer?

## Future possibilities

The Component derive could later lead to automatic derives for reflection, or in general serve
as a way to reduce boilerplate from new features of the engine added in the future.

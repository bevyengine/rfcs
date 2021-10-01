# Feature Name: `derive_component`

## Summary

Remove blanket impl from `Component` trait, and require it to be manually implemented or derived using `#[derive(Component)]`.
Extend the `Component` trait to declare more information statically, instead of requiring runtime registration.

## Motivation

The `Component` trait is being used inside Bundles and Queries. It's usage in Queries is enforced by the type system.
Unfortunately, because it's automatically derived, almost every type can be treated as a component, including Bundles.
That means `Component` APIs don't really have any way to prevent misuse, like `commands.insert(bundle)`. Right now
it's very easy to end up with that code because of a typo, as the `Bundle` api methods are very similarly named.

There are also other pitfalls, like primitive types being automatically `Component`s, but without clear meaning what they represent.
This is an especially severe issue for function pointer types. It's easy to use an enum variant or newtype constructor as
a component, like `.insert(MyEnum::Variant)`,  without realising it. This leads to very hard to spot bugs without any hint
by the compiler or at runtime that something went wrong. This is easily preventable by not implementing the `Component` trait for those types.

We also already have `#[derive(Bundle)]` and others. Adding that to `Component` keeps the syntax similar across the board.

Apart from requiring explicit opt-in, `derive` gives us a natural place to define more metadata about specific component type.
One way to use that is to declare component's Storage type statically, instead of requiring manual registration of that metadata
into the world.

```rust
#[derive(Component)]
#[component(storage = "SparseSet")]
struct MyComponent(..);
```

This enables the compiler to perform much more optimizations in the query iteration code, as a lot of the code paths can be statically eliminated.

## User-facing explanation

In order to define your own component type, you have to implement the `Component` trait. The easiest way to do that is using a `derive` macro.

```rust
#[derive(Component)]
struct HitPoints(u32);
```

If you forget to properly annotate your component type, the compiler will remind you
of that as soon as you try to insert that component into the world or query for it. Apart from type safety, this also provides a visual indication that the given type
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
### Why not use a manual implementation of `Component` to set the storage type?

As shown in [this playground link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ca85c5d3c561ef9d1feec143bbcd9611), we could force users to write out a manual implementation of the `Component` trait when they wanted to change the storage type.

This has a couple advantages:

1. More direct and less magical, at least for advanced Rust users.
2. No configuration DSL to maintain.
3. Better type checking.

However, it has several more serious disadvantages, which outweigh those:

1. Manual implementations of `Component` will break (often in dozens of places scattered around the code base) whenever we make breaking changes to the `Component` trait. Derive + attribute macros will continue working flawlessly, except where they directly touch the breaking changes.
2. Longer boilerplate, especially as the `Component` trait grows. This is particularly frustrating as you need to add / remove this repeatedly when benchmarking perf as the actual net impact is very fact-specific.
3. Exposes internal-ish implementation details to users who really don't care.

### Comparison of methods of defining storage type in `Component` trait

There are several possible ways to define the storage type, as shown in [this playground link](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4a2d47173f68137a34c31987d9d46bfa).

While any approach would work, an associated type is safer and offers better discoverability, due to the use of the type checker.
In addition, you can't assert specific associated const value in the trait bounds, which may be limiting in the future.
The slight increase in verbosity is worth it for these benefits.
## Unresolved questions

- How to support dynamic components for future scripting layer?

## Future possibilities

The Component derive could later lead to automatic derives for reflection, or in general serve
as a way to reduce boilerplate from new features of the engine added in the future.

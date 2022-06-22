# bevy_reflect: `Reflect` as a unique `Reflect`

## Summary

- create a new trait: `PartialReflect`
- `Reflect: PartialReflect`
- All methods on `Reflect` that **does not** depend on the uniqueness of the
  underlying type are moved to `PartialReflect`
- Most things should now accept a `dyn PartialReflect` rather than
  `dyn Reflect`.
- It is possible to convert a `PartialReflect` into a `Reflect` using a
  `into_full` method.
- `FromReflect` becomes `FromPartialReflect`.

## Jargon

For clarity, we will always call "new Reflect" the new version of Reflect,
with guaranteed uniqueness, and "old Reflect" the current implementation of
Reflect.

#### pass for

We say that old `T: Reflect` **passes for** old `U: Reflect` when `T`'s
`type_name` is equal to `U`'s, and it is possible to convert from a `T` to a `U`
using `apply`, `set` or `FromReflect::from_reflect`.

#### underlying

The **underlying** value of a dynamic object (eg: `dyn Reflect`) is the
concrete type backing it. In the following code, the **underlying type** of
`reflect_a` is `MysteryType`, while the **underlying value** is the value of
`a`.

```rust
let a = MysteryType::new();
let reflect_a: Box<dyn Reflect> = Box::new(a.clone()) as Box<dyn Reflect>;
```

When using `clone_value`, the underlying type changes: if `MysteryType` is a
struct, the underlying type of `reflect_clone` will be `DynamicStruct`:

```rust
let reflect_clone: Box<dyn Reflect> = reflect_a.clone_value();
```

#### canonical type

A `Box<dyn PartialReflect>` has a _canonical underlying type_ when the
underlying type is the actual struct it is supposed to reflect. All types
derived with `#[derive(Reflect)]` are canonical types. All new `Reflect`s are
canonical types.

#### proxies

A `PartialReflect` (or old `Reflect`) implementor is said to be "proxy" when it
doesn't have a canonical representation. The `Dynamic***` types are all proxies.
Something that implements `PartialReflect` but not new `Reflect` is a proxy by
definition.

#### Language from previous versions of this RFC

This RFC went through a few name changes, in order to understand previous
discussion, here is a translation table:
* `CanonReflect`: What is now the new `Reflect`
* `ReflectProxy`: A version of what is now `PartialReflect`, but as a concrete
  enum rather than a trait

## Motivation

The old `Reflect` trait is a trap, it doesn't work as expected due to proxies
mixing up with the "regular" types under the `dyn Reflect` trait object.

When the underlying type of an old `Box<dyn Reflect>` is a proxy, a lot of
the old `Reflect` methods become invalid, such as `any`, `reflect_hash`,
`reflect_partial_eq`, `serializable`, `downcast`, `is` or `take`. Yet, the user
can still call those methods, despite the fact that they are invalid in this
condition.

### What's under the old `dyn Reflect`?

Currently, `Reflect` has very a confusing behavior. Most of the methods on
old `Reflect` are highly dependent on the underlying type, regardless of whether
they are supposed to stand for the requested type or not.

For example

```rust
let a = MysteryType::new(); // suppose MysteryType implements Reflect
let reflect_a: Box<dyn Reflect> = Box::new(a.clone()) as Box<dyn Reflect>;
let dynamic_a: Box<dyn Reflect> = a.clone_value();
// This more or less always panic
assert!(dynamic_a.is::<MyteryType>())
```

This is because multiple different things can pretend to be the same thing. In
itself this shouldn't be an issue, but the old `Reflect` API is leaky, and
you need to be aware of both this limitation and the underlying type to be able
to use the old `Reflect` API correctly.


### `reflect_hash` and co. with proxies

The problem is not limited to `is`, but also extends to the `reflect_*`
methods.

```rust
let a = MysteryType::new(); // suppose MysteryType implements Reflect
let reflect_a: Box<dyn Reflect> = Box::new(a.clone()) as Box<dyn Reflect>;
let dynamic_a: Box<dyn Reflect> = a.clone_value();
let reflect_hash = reflect_a.reflect_hash();
let dynamic_hash = dynamic_a.reflect_hash();
// This more or less always panic if `MysteryType` implements `Hash`
assert_eq!(reflect_hash, dynamic_hash)
```

### Problem

Reflect has multiple distinct roles that require different assumptions on the
underlying type:
1. A trait for structural exploration of data structures independent from the
   data structure itself. Exposed by methods `reflect_ref`, `reflect_mut` and
   `apply`
2. An extension to `Any` that allows generic construction from structural
   descriptions (automatic deserialization) through registry.
3. "trait reflection" through registry.

Role (1) and (2, 3) require different assumptions. As noted in the earlier
sections, `Any` requires assumptions on the underlying types, it is the case of
trait reflection as well.

The problem stems from some methods of old `dyn Reflect` assuming that the
underlying type is always the same. 

The following methods are the one that assumes uniqueness:
- `any`
- `any_mut`
- `downcast`
- `take`
- `is`
- `downcast_ref`
- `downcast_mut`
- `set`

The `Any` trait bound on old `Reflect` is similarly problematic, as it introduces
the same issues as those methods.

Note that the following methods are also problematic, and require discussion,
but to keep to scope of this RFC short, we will keep them in `PartialReflect`.
- `reflect_hash`
- `reflect_partial_eq`
- `serializable`

## Proposition

This RFC separates the bits of `Reflect` needed for (1) and the bits needed for
(2) and (3).

We introduce `PartialReflect` to isolate the methods of `Reflect` for (1). This
also means that the old `Reflect` subtraits now are bound to `PartialReflect`
rather than `Reflect`. Proxy types such as `DynamicStruct` will also now only
implement `PartialReflect` instead of full `Reflect`, because they do not make
sense in the context of (2) and (3).

This makes `Reflect` "canonical", in that the underlying type is the one being
described all the time. And now the `Any` bound and any-related methods are all
"safe" to use, because the assumptions requires to be able to use them are
backed into the type system.

We still probably want a way to convert from a `PartialReflect` to a `Reflect`,
so we add a `into_full` method to `PartialReflect` to convert them into
`Reflect` in cases where the underlying type is canonical.

The `FromReflect` trait will be renamed `FromPartialReflect`, a future
implementation might also add the ability to go from any _proxy_ partial
reflect into a full reflect using a combination of `FromPartialReflect` and the
type registry.

In short:
- create a new trait: `PartialReflect`
- `Reflect: PartialReflect`
- All methods on `Reflect` that **does not** depend on the uniqueness of the
  underlying type are moved to `PartialReflect`
- Most things should now accept a `dyn PartialReflect` rather than
  `dyn Reflect`.
- It is possible to convert a `PartialReflect` into a `Reflect` using a
  `into_full` method.
- `FromReflect` becomes `FromPartialReflect`.


### `PartialReflect` trait

All methods that do not assume underlying uniqueness should be moved to a new
trait: `PartialReflect`. `PartialReflect` also has a `as_full` and `into_full`
to convert them into "full" `Reflect`. We also keep the "trait reflection"
methods, because changing the trait reflection design is complex and we want to
keep the scope of this RFC minimal.

```rust
pub trait PartialReflect: Send + Sync {
    // new
    fn as_full(&self) -> Option<&dyn Reflect>;
    fn into_full(self: Box<Self>) -> Option<Box<dyn Reflect>>;

    fn type_name(&self) -> &str;

    fn as_partial_reflect(&self) -> &dyn PartialReflect;
    fn as_partial_reflect_mut(&mut self) -> &mut dyn PartialReflect;
    fn reflect_ref(&self) -> ReflectRef;
    fn reflect_mut(&mut self) -> ReflectMut;

    fn apply(&mut self, value: &dyn PartialReflect);
    // Change name of `clone_value`
    // Ideally add documentation explaining that the underlying type changes.
    fn clone_partial(&self) -> Box<dyn PartialReflect>;

    fn debug(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {}

    // Questionable, but let's leave those for later.
    fn reflect_hash(&self) -> Option<u64> {}
    fn reflect_partial_eq(&self, _value: &dyn PartialReflect) -> Option<bool> {}
    fn serializable(&self) -> Option<Serializable> {}
}
```

### New `Reflect` trait

`Reflect` now, instead of implementing those methods, has `PartialReflect` as
trait bound:

```rust
pub trait Reflect: PartialReflect + Any {
    fn as_reflect(&self) -> &dyn Reflect;
    fn as_reflect_mut(&mut self) -> &mut dyn Reflect;

    fn any_mut(&mut self) -> &mut dyn Any {
      // implementation
    }
    // etc.
    fn any
    fn any_mut
    fn downcast
    fn take
    fn is
    fn downcast_ref
    fn downcast_mut
    fn set
}
```

This trait is automatically implemented by the `#[derive(Reflect)]` macro.

Note that when [trait upcasting] is implemented, the `any_mut` and other
`Any`-related methods should be moved into a `impl dyn Reflect` block,
`as_reflect` and `as_partial_reflect` can also be dropped with trait
upcasting.

### `PartialReflect` to `Reflect` conversion

We still want to be able to use the `Any` methods on our `PartialReflect`, so we
want a way to get a new `Reflect` from a `PartialReflect`. This is only possible
by checking if the underlying type is the one we expect OR converting into the
underlying type described by the `PartialReflect::type_name()` method.

To do so, we have two options:
1. Provide an expected type or access a constructor stored in type registry
   and convert with `FromReflect` from any `Reflect` to the canonical
   representation of a `Reflect`.
2. Add a `into_full(self: Box<Self>) -> Option<Box<dyn Reflect>>` method to
   `PartialReflect` that returns `Some` only for canonical types.

(1) Is complex and requires a lot more design, so we'll stick with (2) for this
RFC.

`into_full` will return `None` by default, but in the `#[derive(Reflect)]`
macro will return `Some(self)`. Converting from a proxy `PartialReflect`
to a `Reflect` is currently impossible.

## User-facing explanation

`PartialReflect` let you navigate any type independently of their shape
with the `reflect_ref` and `reflect_mut` methods. `Reflect` is a special
case of `PartialReflect` that also let you convert into a concrete type using the
`Any` methods. To get a `Reflect` from a `PartialReflect`, you should use
`PartialReflect::into_full`.

Any type derived with `#[derive(Reflect)]` implements both `Reflect` and
`PartialReflect`. The only types that implements `PartialReflect` but not
`Reflect` are "proxy" types such as `DynamicStruct`, or any third party
equivalent.

## Drawbacks

- It is impossible to make a proxy `PartialReflect` into a `Reflect` without
  knowing the underlying type.
- This splits the reflect API, that used to be neatly uniform. It will be
  more difficult to combine various use cases of `bevy_reflect`, requiring
  explicit conversions. (Note that however, the previous state of things would
  result in things getting silently broken)

## Unresolved questions

- Should `Reflect` still be named `Reflect`, does it need to be a subtrait of
  `PartialReflect`?
- Is the ability to convert from a proxy `PartialReflect` to `Reflect` a
  hard-requirement, ie: blocker? (I think only a first attempt at implementation
  can answer this)

## Future possibilities

- Could we modify the `ReflectRef` and `ReflectMut` enums to enable a sort of
  "partial" evaluation of structures by adding a `&dyn Reflect` variant?
- `impl<T: Struct> PartialReflect for T {}` and other subtraits rather than
  `Struct: PartialReflect`.
- Add a `reflect_owned` that returns a `Dynamic` equivalent of `ReflectRef`
  (As an earlier version of this RFC called `ReflectProxy`)
- Make use of [trait upcasting].
- An earlier version of this RFC combined the `FromReflect` trait with the
  `TypeRegistry` to enable conversion from any proxy `PartialReflect` to
  concrete `Reflect`, this method is probably still needed.
- (2) and (3) are still worth discriminating, this relates to the `reflect_hash`
  and `reflect_partial_eq` methods, which are explicitly left "broken as before"
  in this RFC.

[trait upcasting]: https://github.com/rust-lang/rust/issues/65991

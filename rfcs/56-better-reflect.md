# bevy_reflect: `CanonReflect` as a unique `Reflect`

## Summary

- Add a `CanonReflect: Reflect` trait that replaces `Reflect`
- Remove from `Reflect` all methods that assume uniqueness of underlying type.
- Add a `from_reflect` method to `Typed` that takes a `Box<dyn Reflect>` and
  returns a `Self` to convert a `Reflect` into a `CanonReflect`.

## Lexicon/Terminology/Nomenclature/Jargon

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

## Motivation

The `Reflect` trait is a trap, it doesn't work as expected due to `Dynamic***`
mixing up with the "regular" types under the `dyn Reflect` trait object.

When the underlying type of a `Box<dyn Reflect>` is a `Dynamic***`, a lot of
the `Reflect` methods become invalid, such as `any`, `reflect_hash`,
`reflect_partial_eq`, `serializable`, `downcast`, `is` or `take`. Yet, the user
can still call those methods, despite the fact that they are invalid in this
condition.

### What's under the `dyn Reflect`?

Currently, `Reflect` has very a confusing behavior. Most of the methods on
`Reflect` are highly dependent on the underlying type, regardless of whether
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
itself this shouldn't be an issue, but the current `Reflect` API is leaky, and
you need to be aware of both this limitation and the underlying type to be able
to use the `Reflect` API correctly.


### `reflect_hash` and co. with `Dynamic***`

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

The problem stems from some methods of `dyn Reflect` assuming that the
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

The `Any` trait bound on `Reflect` is similarly problematic, as it introduces
the same issues as those methods.

Note that the following methods are also problematic, and require discussion,
but to keep to scope of this RFC short, we will keep them in `Reflect`.
- `reflect_hash`
- `reflect_partial_eq`
- `serializable`

## Proposition

- Create a `CanonReflect` trait
- Remove all methods assuming uniqueness and `Any` trait bound from `Reflect`,
  add them to `CanonReflect`.
- Remove `set` method from `Reflect`
- Implement a way to convert from `Reflect` to `CanonReflect`:
  - New method to `Typed` trait: `from_reflect`
  - Remove `FromReflect` trait
  - New `constructor` field to `TypeRegistration` to convert a `Box<dyn Reflect>`
    into a `Box<dyn CanonReflect>` dynamically.
  - New `make_canon` method to `TypeRegistry`

### `CanonReflect` trait

We remove all methods assuming uniqueness and `Any` trait bound from
`Reflect` and only implement them on "canonical" types: ie: the ones implemented
by the `#[derive(Reflect)]` macros.

We introduce a new trait `CanonReflect`, that must be explicitly implemented
to mark a type as being the canonical type.
```rust
pub trait CanonReflect: Reflect + Any {
  fn any_mut(&mut self) -> &mut dyn Any {
    // implementation
  }
  // etc.
}
```

This trait is automatically implemented by the `#[derive(Reflect)]` macro.

This prevents any surprises caused by the underlying type of a reflect not being
the one expected.

Note that when [trait upcasting] is implemented, the `any_mut` and other
`Any`-related methods should be moved into a `impl dyn CanonReflect` block.

### `Reflect` to `CanonReflect` conversion

We still want to be able to use the `Any` methods on our `Reflect`, so we want
a way to get a `CanonReflect` from a `Reflect`. This is only possible by
checking if the underlying type is the one we expect OR converting into the
underlying type described by the `Reflect::type_name()` method.

To do so, it's necessary to either provide an expected type or access the type
registry to invoke the constructor associated with the `Reflect::type_name()`.

To define a constructor We move the `from_reflect` method from `FromReflect` to
`Typed`. For no particular reasons but to reduce how many traits we export from
`bevy_reflect`.

**Danger**: Using `type_name` from `Reflect` is error-prone. For the conversion
to pick up the right type, we need `type_name()` to match exactly the one
returned by `std::any::type_name`. Ideally we have an alternative that defines
the underlying type without ambiguity, such as `target_type_id` or
`full_type_name`.

We introduce a new field to `TypeRegistration`: `constructor`. It allows
converting from a `Reflect` to a `CanonReflect`, so that it's possible to
use its underlying-sensible methods.

```rust
trait Typed {
    fn from_reflect(reflect: Box<dyn Reflect>) -> Result<Self, Box<dyn Reflect>>;
}
pub struct TypeRegistration {
    short_name: String,
    data: HashMap<TypeId, Box<dyn TypeData>>,
    type_info: TypeInfo,
    // new: constructor
    constructor: fn(Box<dyn Reflect>)
        -> Result<Box<dyn CanonReflect>, Box<dyn Reflect>>,
}
```

We add a `make_canon` method to `TypeRegistry` and `TypeRegistration` to use
that new field.

```rust
impl TypeRegistration {
    pub fn new<T: Typed>() -> Self {
        Self {
            //...
            constructor: T::from_reflect(from).map(|t| Box::new(t)),
        }
    }
    pub fn make_canon(&self, reflect: Box<dyn Reflect>)
        -> Result<Box<dyn CanonReflect>, Box<dyn Reflect>>
    {
        (self.constructor)(reflect)
    }
    // ...
}
impl TypeRegistry {
    fn make_canon(&self, reflect: Box<dyn Reflect>)
        -> Result<Box<dyn CanonReflect>, Box<dyn Reflect>>
    {
        if let Some(registration) = self.get_with_name(reflect.type_name()) {
            registration.make_canon(reflect)
        } else {
            Err(reflect)
        }
    }
}
```

The `Reflect` trait now only has the following methods:
```rust
pub trait Reflect: Send + Sync {
    fn type_name(&self) -> &str;

    fn as_reflect(&self) -> &dyn Reflect;
    fn as_reflect_mut(&mut self) -> &mut dyn Reflect;
    fn reflect_ref(&self) -> ReflectRef;
    fn reflect_mut(&mut self) -> ReflectMut;

    fn apply(&mut self, value: &dyn Reflect);
    // Change name of `clone_value`
    // Ideally add documentation explaining that the underlying type changes.
    fn clone_dynamically(&self) -> Box<dyn Reflect>;

    fn debug(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {}

    // Questionable, but let's leave those for later.
    fn reflect_hash(&self) -> Option<u64> {}
    fn reflect_partial_eq(&self, _value: &dyn Reflect) -> Option<bool> {}
    fn serializable(&self) -> Option<Serializable> {}
}
impl dyn Reflect {
    pub fn make_canon<T: Typed>(self: Box<Self>)
        -> Result<Box<dyn CanonReflect>, Box<dyn Reflect>>
    {
        T::from_reflect(self).map(|t| Box::new(t))
    }
}
```

Note that [trait upcasting] would allow us to remove the `as_reflect` and
`as_reflect_mut` methods. In fact, I don't think they are necessary at all.

Note that the various `Dynamic***` types shouldn't implement `CanonReflect`,
only `Reflect`.

## User-facing explanation

`Reflect` let you navigate any type regardless independently of their shape
with the `reflect_ref` and `reflect_mut` methods. `CanonReflect` is a special
case of `Reflect` that also let you convert into a concrete type using the
`Any` methods. To get a `CanonReflect` from a `Reflect`, you should use any
of the `make_canon` methods on `dyn Reflect`, `TypeRegistry` or
`TypeRegistration`.

Any type derived with `#[derive(Reflect)]` implements `CanonReflect`. The only
types that implements `Reflect` but not `CanonReflect` are "proxy" types such
as `DynamicStruct`, or any third party equivalent.

## Drawbacks

- Users must be aware of the difference between `CanonReflect` and `Reflect`,
  and it must be explained.
- You cannot directly convert a `Reflect` into a `Any` or `T`, you must first
  make sure it is conforms a canonical representation by using either
  `TypeRegistry::make_canon` or `<dyn Reflect>::make_canon<T: Typed>(self:
  Box<self>)`.
- The addition of `constructor` to `Typed` will increase the size of generated
  code.
- `constructor` makes `Typed` not object safe. But it should be fine
  since `Typed` is not intended to be used as a dynamic trait objects.

## Unresolved questions

- `CanonReflect` name? My first instinct was `ReflectInstance`, but I changed it
  to `ReflectType`, then `CanonicalReflect` and finally `CanonReflect` thinking
  it was too long.
  \
  Technically, `Canon` doesn't mean "normal representation", as "canonical"
  does, but it's short and close enough to be understood. I thought about
  using `canonize` instead of `make_canon`, but it felt too religious.
- An earlier version of this RFC had the exact opposite approach to fix the
  uniqueness issue: Instead of removing all underlying-dependent methods from
  `Reflect` and adding them to a new trait, we kept the underlying-dependent
  methods but moved all the non-dependent methods to a new trait. Which one to
  go with?

## Future possibilities

- Add a `target_type(&self) -> TypeId` or `full_type_name` method to `Reflect`
  such that it's easier to check if we are converting into the proper canonical
  type. We might also benefit from a method that is capable of taking two
  type paths and comparing them accounting for shortcuts.
- Performance: currently we force-convert from `CanonReflect` to `Reflect`,
  then deeply traverse the data structure for all our reflect methods. It could
  be greatly optimized if we add a way to tell if the underlying type is
  already canon. (For example `make_canon` could be a no-op if the underlying
  type is already the canonical form)
  \
  This also interacts interestingly with `ReflectRef` and `ReflectMut`. Should
  the `Value` variant hold a `dyn Reflect` or a `dyn CanonReflect`?
- `impl<T: Struct> Reflect for T {}` and other subtraits rather than
  `Struct: Reflect`.
- Redesign `reflect_hash` and `reflect_partial_eq`, since this RFC doesn't fix
  the issue with them.
- Add a `reflect_owned` that returns a `Dynamic` equivalent of `ReflectRef`
  (As an earlier version of this RFC called `ReflectProxy`)
- Make use of [trait upcasting].

[trait upcasting]: https://github.com/rust-lang/rust/issues/65991

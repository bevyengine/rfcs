# bevy_reflect: Unique `Reflect`

This builds on https://github.com/bevyengine/bevy/pull/4042 and the `TypeInfo`
API.

## Summary

Add a `ReflectProxy` enum that generalizes the `Dynamic***` structs from
`bevy_reflect` to all types, and remove `Reflect` implementation for
`Dynamic***` structs.

I also suggest renaming all `Dynamic***` into `***Proxy` and the reflect
traits `clone_dynamic` methods to `proxy_clone`.

## Lexicon/Terminology/Nomenclature/Jargon

#### pass for

We say that `T: Reflect` **passes for** `U: Reflect` when `T`'s `type_name` is
equal to `U`'s, and it is possible to convert from a `T` to a `U` using
`apply`, `set` or `FromReflect::from_reflect`.

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

#### Underlying-sensible method

Those are methods of `Reflect` that act differently between `reflect_clone` and
`reflect_a` from the previous examples.


#### `Dynamic***`

The structures holding generic structural information about some proxied type. 
* `DynamicMap`
* `DynamicList`
* `DynamicTuple`
* `DynamicStruct`
* `DynamicTupleStruct`

#### Reflect subtraits

The top level traits in `bevy_reflect`: `Struct`, `TupleStruct`, `Tuple`,
`List` and `Map`. They are all subtraits of `Reflect`.

## Motivation

The `Reflect` trait is a trap, it doesn't work as expected due to `Dynamic***`
mixing up with the "regular" types under the `dyn Reflect` trait object.

To solve this, we do not allow more than a single underlying type to "pass for"
a concrete type in `Reflect`. This implies that the various `Dynamic***` types
shouldn't implement `Reflect`.

When the underlying type of a `Box<dyn Reflect>` is a `Dynamic***`, a lot of
the `Reflect` methods become invalid, such as `any`, `reflect_hash`,
`reflect_partial_eq`, `serializable`, `downcast`, `is` or `take`. However,
those methods are provided regardless.

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
assert_eq!((&*reflect_a).type_id(), (&*dynamic_a).type_id())
```

This is because multiple different things can pretend to be the same thing. In
itself this shouldn't be an issue, but the current `Reflect` API is leaky, and
you need to be aware of both this limitation and the underlying type to be able
to use the `Reflect` API correctly.


### `reflect_hash` and co. with `Dynamic***`

The problem is not limited to `type_id`, but also extends to the `reflect_*`
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


## Proposition

The problem stems from the duplicity of `dyn Reflect`. We want to rely on the
underlying methods of the same thing to be unique.

To solve this, we do not allow more than a single underlying type to "pass for"
a concrete type in `Reflect`. This implies that the various `Dynamic***` types
shouldn't implement `Reflect`.

## User-facing explanation

* `Reflect` maps 1-to-1 with an underlying type meaning that if a
  `Box<dyn Reflect>` cannot be downcasted to `T`, you cannot build a `T` from
  it.
* There is a `ReflectProxy` enum you can use to construct any `Reflect` type
  based on some dynamic structure. This can be used, for example, for
  deserialization of types that implement `Reflect` but not `Deserialize`.
  * You can "clone reflectively" a `Box<dyn Reflect>` by using
    `Reflect::proxy_clone`. It returns a `ReflectProxy`.
  * You can use `ReflectProxy::sidecast` to create a `Box<dyn Reflect>` from
    a `ReflectProxy`. The created trait object's underlying type is a concrete
    type you can downcast to.
* `FromReflect` doesn't exist anymore. Instead, `Reflect::downcast` provides
  now the same guarentees that `FromReflect::from_reflect` had. If you need to
  get a `T` from a `ReflectProxy`, use a combination of
  `ReflectProxy::sidecast_type<T>` and `downcast`.


## Implementation strategy

The `Reflect` API stays identical to the one we have today, at the exception of
`clone_value`.

We remove the `Dynamic***: Reflect` implementations and we introduce a
`ReflectProxy` enum.

```rust
pub enum ReflectProxy {
    Struct(DynamicStruct),
    TupleStruct(DynamicTupleStruct),
    Tuple(DynamicTuple),
    List(DynamicList),
    Map(DynamicMap),
    AlreadyReflect(Box<dyn Reflect>),
}
```

`ReflectProxy` mirrors the `ReflectRef` and `TypeInfo` enums.

`ReflectProxy` has a method to convert the underlying value into the
value it is proxying. This allows you to get a `Reflect` from a `ReflectProxy`
while upholding the `Reflect` uniqueness rule.

```rust
impl ReflectProxy {
  pub fn type_name(&self) -> &str {
    match self { /* etc. */ }
  }
  pub fn can_sidecast(&self, registry: &TypeRegistry) -> Result<(), SidecastError> {}
  pub fn sidecast(self, registry: &TypeRegistry)
    -> Result<Box<dyn Reflect>, SidecastError> {
    let type_name = self.type_name();
    let registration = registry.get_with_name(type_name).ok_or(SidecastError::Unregistered)?;
    Ok(registration.constructor(self)?)
  }
  pub fn sidecast_type<T: Reflect + Typed>(self) -> Result<Box<dyn Reflect>, SidecastError> {
    let registration = TypeRegistration::of::<T>();
    Ok(registration.constructor(self)?)
  }
}
```
 
`sidecast` requires introducing a new field to `TypeRegistration`:
```rust
pub struct TypeRegistration {
    short_name: String,
    // new: constructor
    constructor: fn(ReflectProxy) -> Result<Box<dyn Reflect>, ConstructorError>,
    data: HashMap<TypeId, Box<dyn TypeData>>,
    type_info: TypeInfo,
}
impl TypeRegistration {
  pub fn constructor(&self, proxy: ReflectProxy) ->  Result<Box<dyn Reflect>, ConstructorError> {
    (self.constructor)(proxy)
  }
  // ...
}
```

`constructor` would simply construct the `T` and wrap it into a `Box` before
returning it.

`constructor` can be derived similarly to `TypeInfo`. It will however
require additional consuming methods on `Dynamic***` structs in order to be
able to move their `Box<dyn Reflect>` fields into the constructed data
structure.

## Drawbacks

- Users cannot define their own proxy type, they must use the `Dynamic***`
  types.
- `Dynamic***` do not implement the `Reflect` subtraits anymore.
- There is some added complexity (although it fixes what I believe to be a
  major limitation of the `bevy_reflect` API)
- `Reflect::apply` now only accepts a `ReflectProxy` (see next section)

## Rationale and alternatives

- Divide `Reflect` in two distinct traits, one with the underlying-sensible
  method, and one with the other methods. (see next points)
- An earlier version of this RFC proposed `ReflectProxy` as a **trait**,
  however, I reformulated it as an enum when considering how to implement it.
  (How would you define the `TypeRegistration::constructor` field if passed a
  `Box<dyn ReflectProxy>`? You'd need something like `ReflectProxyRef`?)

I believe this change is needed, since `Reflect` duplicity is a large footgun.

I don't believe that having `Dynamic***` implement `Reflect` makes sense, I
think it introduces much more issues than it solves. The only use-case for
`Dynamic***` being `Reflect` are the `reflect_mut`, `reflect_ref` and `apply`
methods. However, those could as well be implemented outside of the `Reflect`
trait, as mentioned in the "Future possibilities" section.

## Unresolved questions

- How does this interact with `#[reflect(ignore)]`?
- Is a `trait ReflectProxy` more desirable than an `enum` and how to implement
  it?
- Should `Dynamic***` inner values be `Box<ReflectProxy>` or `Box<dyn Reflect>`?
  With `Box<dyn Reflect>`:
  * You must have a `TypeRegistry` and you must call `sidecast` both when
    _building_ the `Dynamic***` and when converting from `ReflectProxy` to
    `Reflect` (note that you most often already need the `TypeRegistry` when
    building the `Dynamic***` if you want to access `TypeInfo`)
  * The heavy lifting is mostly done when constructing the `ReflectProxy`,
    `sidecast` is only a shallow conversion, calling `downcast` on the top
    level data structure fields.
  * There is less change to do if we keep it as it is today.
  * With `Box<ReflectProxy>`, the opposite is true.
- Should we try to implement the reflect subtraits for the `Dynamic***`?
  (this would require removing the `Subtrait: Reflect` bounds)

## Future possibilities

We might be able to move `reflect_ref` and `reflect_mut` out of `Reflect` into
its own trait. (`Overlay`), make `Reflect: Overlay` and implement `Overlay` for
the `Dynamic***` types and `ReflectProxy`. `apply` could then accept a 
`T: Overlay` instead of a `ReflectProxy`.

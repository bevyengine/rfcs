# Feature Name: `from_reflect_ergonomics`

## Summary

Derive `FromReflect` by default when deriving `Reflect`, add `ReflectFromReflect` type data, and
make `ReflectDeserializer` easier to use.

## Terminology

### *Dynamic Types* vs *Real Types*

A **Dynamic** type refers to any of the `Dynamic***` structs used by `bevy_reflect` to represent a basic Rust data
structure.
For example, a `DynamicTupleStruct` is a Dynamic that represents a tuple struct in Rust.

A **Real** type, on the other hand, refers to any type that is *not* a Dynamic.
For example, `i32` is a Real type, and so is `Vec<T>`.

Since both may be represented as a `dyn Reflect`, examples in this RFC will try to use the following conventions:

* Variables named `dyn_***` contain a Dynamic type. The full name should match the name of the corresponding Dynamic
  type. For example, a `dyn_tuple` represents a `DynamicTuple`.
* Variables named `real_***` contain a Real type. The full name should match the corresponding data structure. For
  example, a `real_tuple` represents any Rust tuple, such as `(i32, f32, usize)`.

## Background

### What is `Reflect`?

Before diving into this RFC and the world of Bevy reflection, let's take a quick step back and recall what `Reflect` is
and why we need it.

Reflection in programming gives developers the power to analyze the code itself, and use that information to perform
dynamic operations.
This can be as simple as getting the name of a struct's field.
Or it can be more complex, such as iterating over all elements of a tuple and comparing them with values in a list.

Rust does not come with this functionality by default.
Bevy's `Reflect` instead powers this *meta-programming* mechanism.
`Reflect` can be derived on any struct using `#[derive(Reflect)]` and allows that struct to be reflected.
This derive will also implement a number of other necessary traits.
For structs, one such trait is `Struct` (or `TupleStruct` for tuple structs).

If a type implements `Reflect`, you'll often see it used in trait object form: `dyn Reflect`.

This whole system is very useful, especially for dynamically operating on a value based on the data we get from
reflecting it.

### Why do we need Dynamic types?

Reflection works great when you already have a value to reflect.
However, there are many times when you might not have a value (or the full value).

For example, take the following struct:

```rust
struct Foo {
    x: i32,
    y: i32
}
```

If we need to deserialize the following JSON:

```json
{
  "type": "my_crate::Foo",
  "struct": {
    "x": {
      "type": "i32",
      "value": 123
    }
  }
}
```

How can we do this? Notice that we're missing the `y` field!

This is where Dynamic types come in— specifically a `DynamicStruct`.
Dynamic types allow us to mimic *any* type.
We can use this Dynamic to apply its data atop an existing value where applicable.
Or, in the case above, we can use it to store partial information of a type.

The problem is that when we deserialize this and get that `DynamicStruct` back, how do we get to the Real type, `Foo`?
Enter `FromReflect`.

### What is `FromReflect`?

The `FromReflect` trait is meant to allow types to be constructed dynamically, often using Dynamic types.
Specifically, it enables a Real type to be generated from a Dynamic one.

As an example, we can convert a `DynamicTuple` to a Real tuple like so:

```rust
let mut dyn_tuple = DynamicTuple::default ();
dyn_tuple.insert(123usize);
dyn_tuple.insert(321usize);

let real_tuple = < (usize, usize) >::from_reflect( & dyn_tuple).unwrap();
```

A Dynamic value to a Real one. Neat!

### How does `FromReflect` work?

Basically, `FromReflect` methods are recursively called for each field.

Given a struct like:

```rust
struct Foo {
    value: (f32, f32)
}
```

Calling `Foo::from_reflect` will kick off a chain of `FromReflect`:

`Foo::from_reflect` calls `<(f32, f32)>::from_reflect` which itself calls `f32::from_reflect`.

This is why, when deriving `FromReflect`, all fields *must* also implement `FromReflect`[^1].
Without this, the trait implementation won't compile.

### Why is `FromReflect` useful?

So why does this matter? If we need the Real type we can just avoid using Dynamics, right?

Well for any users who have tried working with serialized reflection data, they've likely come across the biggest use
case for `FromReflect`: deserialization.

When reflected data is deserialized, all of it is placed in a Dynamic type[^2].
This means that, even if we know the Real type ahead of time, we're still stuck with the Dynamic type instead.
Like in the example above, we might know we want `Foo`, but deserializing it using `ReflectDeserializer` will
give us a `DynamicStruct`.

This is why `FromReflect` is so useful: it allows us to convert that Dynamic into our desired Real type.
So with it, we can convert the returned `DynamicStruct` into a Real instance of `Foo`.

## Motivation

So where's the issue?

The main problem is in the ergonomics of this whole system for the typical Bevy developer as well as the consistency of
it across the Bevy ecosystem.

To be specific, there are three areas that could be improved:

1. **Derive Complexity** - A way to cut back on repetitive or redundant code.
2. **Dynamic Conversions** - A way to use `FromReflect` in a purely dynamic way, where no Real types are known.
3. **Deserialization Clarity** - A way to make it clear to the user exactly what they are getting back from the
   deserializer.

Please note that these are not listed in order of importance but, rather, order of understanding.

## User-facing explanation

### Derive Complexity

> Removing redundancy and providing sane defaults

Bevy's reflection model is powerful because it not only allows type metadata to be easily traversed, but it also allows
for (de)serialization without the need for any `serde` derives.

This is great because nearly every reflectable type likely wants to be readily serializable as well.
This is so they can be supported by Bevy's scene format or a maybe custom config format.
This can be used for sending serialized data over the network or to an editor.

Chances are, if a type implements `Reflect`, it should also be implementing `FromReflect`.
Because deserialization is so common in these situations, `FromReflect` is now automatically derived alongside
`Reflect`.

```rust
// Before:
#[derive(Reflect, FromReflect)]
struct Foo;

// After:
#[derive(Reflect)]
struct Foo;
```

Not only does this cut back on the amount of code a user must write, but it helps establish some amount of consistency
across the Bevy ecosystem.
Many plugin authors, and even Bevy itself (here's one [example](https://github.com/bevyengine/bevy/blob/4c5e30a9f83b4d856e58d23a4e89d662abf476ad/crates/bevy_sprite/src/rect.rs#L7-L7)),
can sometimes forget to consider users who might want to use their type with `FromReflect`.

#### Opting Out

Again, in most cases, a type that implements `Reflect` is probably fine implementing `FromReflect` as well— if not for
themselves, then for another user/plugin/editor/etc.
However, there are a few cases where this is not always true— often for runtime identifier types, or other types never
meant to be serialized or manually constructed.

Take Bevy's [`Entity`](https://github.com/bevyengine/bevy/blob/4c5e30a9f83b4d856e58d23a4e89d662abf476ad/crates/bevy_ecs/src/entity/mod.rs#L101-L101)
type as an example:

```rust
#[derive(Clone, Copy, Hash, Eq, Ord, PartialEq, PartialOrd)]
pub struct Entity {
    pub(crate) generation: u32,
    pub(crate) id: u32,
}
```

The `generation` and `id` of each entity is generated at runtime (not good for general-purpose serialization).
Additionally, the fields are only `pub(crate)` because this struct should almost never be manually generated outside
the crate it's defined (not good for `FromReflect`).

But say we wanted to make it so that we could reflect an entity to get its current `id`. 
To do this without implementing `FromReflect` requires us to *opt-out* of the automatic derive:

```rust
#[derive(Reflect)]
#[from_reflect(auto_derive = false)]
pub struct Entity {
    pub(crate) generation: u32,
    pub(crate) id: u32,
}
```

By marking the container with `#[from_reflect(auto_derive = false)]`, we signal to the derive macro that this type
should not be constructible via `FromReflect` and that it should not be automatically derived.

### Dynamic Conversions

> Adding `ReflectFromReflect` type data

`FromReflect` is great in that you can easily convert a Dynamic type to a Real one. 
Oftentimes, however, you might find yourself in a situation where the Real type is not known. 
You need to call `<_>::from_reflect`, but have no idea what should go in place of `<_>`.

In these cases, the `ReflectFromReflect` type data may be used. 
This can be retrieved from the type registry using the `TypeID` or the type name of the Real type. 

For example:

```rust
let rfr = registry
.get_type_data::<ReflectFromReflect>(type_id)
.unwrap();

let real_struct: Box<dyn Reflect> = rfr.from_reflect( & dyn_struct).unwrap();
```

Calling `rfr.from_reflect` will return an `Option<Box<dyn Reflect>>`, where the `Box<dyn Reflect>` contains the Real
type registered with `type_id`.

#### More Derive Complexity

This feature also pairs nicely with the changes listed in **[Derive Complexity](#derive-complexity)**, since otherwise
this would require the following macros at minimum:

```rust
#[derive(Reflect, FromReflect)]
#[reflect(FromReflect)]
struct Foo;
```

Again, listing all of that out on almost every reflectable type just adds noise.

### Deserialization Clarity

> Making the output of deserialization less confusing

This RFC also seeks to improve the understanding of `ReflectDeserializer`. 
For newer devs or those not familiar with Bevy's reflection have likely been confused why they can't do something like 
this:

```rust
let some_data: Box<dyn Reflect> = reflect_deserializer.deserialize( & mut deserializer).unwrap();
let real_struct: Foo = some_data.take().unwrap(); // PANIC!
```

Again, this is because `ReflectDeserializer` always returns a Dynamic[^2]. 
A user must then use `FromReflect` themselves to get the Real value.

Instead, `ReflectDeserializer::deserialize` now performs the conversion automatically before returning. 
Not only does this cut back on code needed to be written by the user, but it also cuts back on the need for them to
understand the complexity and internals of deserializing reflected data. 
This means that the above code should no longer panic (assuming the Real type *actually is* `Foo`).

If any type in the serialized data did not register `ReflectFromReflect`, this should result in an error being returned
indicating that `ReflectFromReflect` was not registered for the given type. 
With the changes listed in **[Derive Complexity](#derive-complexity)**, this should be less likely to happen, since all
types should register it by default.
If it does happen, the user can choose to either register said type data, or forgo the automatic conversion with
**dynamic deserialization**.

#### Dynamic Deserialization

There may be some cases where a user has data that doesn't conform to `FromReflect` but is still serializable.
For example, we might not want to make `Entity` constructible, but we could still pass along data that *represents*
an `Entity` so it can be constructed in some Bevy-controlled way.

These types require manual reconstruction of the Real type using the deserialized data. 
As such, they should just deserialize to a Dynamic and leave the construction part up to the user. 
The `ReflectDeserializer::deserialize` method, before this RFC, would do just that and return a Dynamic for *all* types.

We can re-enable this behavior on the deserializer by disabling automatic conversion:

```rust
// Disable automatic conversions
reflect_deserializer.set_auto_convert(false);

// Use the deserializer as normal
let some_data: Box<dyn Reflect> = reflect_deserializer.deserialize( & mut deserializer).unwrap();
let dyn_struct: DynamicStruct = some_data.take().unwrap(); // OK!
```

## Implementation strategy

### Implementing: Derive Complexity

Implementation for **[Derive Complexity](#derive-complexity)** is rather straightforward: call the `FromReflect` derive
function from the `Reflect` derive function.
They both take `&ReflectDeriveData`, which should make doing this relatively painless.

Note that we are not removing the `FromReflect` derive. 
There may be cases where a manual implementation of `Reflect` is needed, such as for other proxy types akin to the
Dynamics. 
To make things easier, we can still expose the standard `FromReflect` derive on its own.

The biggest change will be the inclusion of the opt-out strategy. 
This means we need to be able to parse `#[from_reflect(auto_derive = false)]` and use it to control whether we call the
`FromReflect` derive function or not.
This attribute is a `syn::MetaNameValue` where the path is `auto_derive` and the literal is a `Lit::Bool`.

Essentially we can do something like:

```rust
pub fn derive_reflect(input: TokenStream) -> TokenStream {
    // ...
    let from_reflect_impl = if is_auto_derive {
        match derive_data.derive_type() {
            DeriveType::Struct | DeriveType::UnitStruct => Some(from_reflect::impl_struct(&derive_data)),
            // ...
        }
    } else {
        None
    };

    return quote! {
    #reflect_impl
    
    #from_reflect_impl
  };
}
```

> As an added benefit, we only need to parse the `ReflectDeriveData` once, as opposed to twice for both reflect derives.

### Implementing: Dynamic Conversions

Implementing **[Dynamic Conversions](#dynamic-conversions)** mainly involves the creation of a `ReflectFromReflect` 
type.
This was previously explored in https://github.com/bevyengine/bevy/pull/4147, however, it was abandoned because of
the issues this RFC aim to fix.

This type data should automatically be registered if `FromReflect` is automatically derived.

### Implementing: Deserialization Clarity

**[Deserialization Clarity](#deserialization-clarity)** can readily be implemented by adding a private field
to `ReflectDeserializer`:

```rust
pub struct ReflectDeserializer<'a> {
    registry: &'a TypeRegistry,
    auto_convert: bool // <- New
}
```

This should default to `true` and be changeable using a new method:

```rust
impl<'a> ReflectDeserializer<'a> {
    pub fn set_auto_convert(&mut self, value: bool) {
        self.auto_convert = value;
    }
}
```

Once the Dynamic is deserialized by the visitor, it will check this value to see whether it needs to perform the
conversion. 
This is done using the same method as shown in **[Dynamic Conversions](#dynamic-conversions)**. 
However, rather than unwrapping, we will return a `Result` that gives better error messages that point to `FromReflect`
and/or `ReflectFromReflect`.

## Drawbacks

* Unidiomatic. One of the biggest drawbacks of this change is that it is not very standard to have opt-out mechanisms
  for derives. They mainly act in an additive fashion. Thus, it could be confusing.

* Increased code generation. By automatically deriving `FromReflect` we increase the amount of code that needs to be
  generated per type— namely for reflectable types that *really* don't need `FromReflect` (such as for a single binary
  that doesn't need serialization and is not meant to be a library).

  <details>
  <summary>Rebuttal</summary>  

  In most cases, however, this is code that *should* be generated as explained in
  **[Derive Complexity](#derive-complexity)**. 
  Additionally, the code generation is rather small with each field in a struct only generating around 1–4 lines of 
  code.

  </details>

## Rationale and alternatives

The biggest alternative is doing nothing. Again, most of these are ergonomics and ecosystem wins. 
The engine will work pretty much the same with or without these changes. 
The hope is that we can make reflection less intimidating, confusing, and frustrating, while making it simpler,
more intuitive, and more consistent across crates.

One concern about integrating the `FromReflect` derive into the `Reflect` derive is that we might want to support
something like a `FromWorldReflect`.
However, using the opt-out method this should still be possible.

## Unresolved questions

- Should `ReflectFromReflect` be replaced by a method on `TypeRegistration`, such as `TypeRegistration::construct`?
- Any other pros/cons to these changes?

<br />

[^1]: This only applies to *active* fields, or fields not marked as `#[reflect(ignore)]`. For *inactive* fields, it instead requires a `Default` impl (unless given a specialized default value using `#[reflect(default = "some_function")]`)
[^2]: Actually, not *all* data is deserialized into a Dynamic. Primitive types, like `u8`, are deserialized into their Real type. This is a confusing disparity addressed by https://github.com/bevyengine/bevy/pull/4561.
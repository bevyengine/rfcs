# Feature Name: `reflect_generic_id`

## Summary

Make type identification in the Reflection system generic, allowing multiple identification data types to coexist.

## Motivation

Various use cases of reflection have different requirements for dynamic type identification methods and different types have different ups and downs.

## User-facing explanation

> Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:
> 
> - Introducing new named concepts.
> - Explaining the feature, ideally through simple examples of solutions to concrete problems.
> - Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
> - If applicable, provide sample error messages, deprecation warnings, or migration guidance.
> - If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

Comparisons of previously used identification methods and their pos and cons:
- A UUID would be universally unique, but require manually specifying one for each reflected type in source code. Automatically creating one at compile time results in equivalency with TypeId, if a random one is generated. If some information about a type such as name and fields/variants is used as randomness seed, it would be somewhat equivalent to type name or type path.
- A type name can be automatically derived but can cause collisions when types from multiple modules are combined in a registry.
- A `core::any::TypeId` identifies a type uniquely, but only for a single compiled binary, meaning it could change when source code is changed or in dynamically loaded binaries.

## Implementation strategy

> This is the technical portion of the RFC.
> Try to capture the broad implementation strategy,
> and then focus in on the tricky details so that:
> 
> - Its interaction with other features is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
> 
> When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.
> 
> When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):
> 
> - **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
> - **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
> - **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

First, add a trait combining traits needed for a key to be used in a registry:

```rs
pub trait TypeKey: Copy + Hash + Eq {}
```

This example is for use with hash maps.

Second, make `TypeRegistry` generic over the key type:

```rs
pub struct TypeRegistry<K: TypeKey> {
    registrations: HashMap<K, TypeRegistration>
}

impl<K: TypeKey> TypeRegistraty<K> {
    pub fn add_registration(&mut self, key: K, registration: TypeRegistration) {
        self.registrations.insert(key, registration);
    }

    pub fn register<T: RegisterType<K>>(&mut self) {
        T::register_type(self);
    }
}
```

Third, introduce new trait, `RegisterType<K>`:

```rs
pub trait RegisterType<K: TypeKey>: GetTypeRegistration {
    fn register_type(registry: &mut TypeRegistry<K>) {
        let key = Self::get_key();
        let registration_value = Self::get_type_registration();
        registry.add_registration(key, registration_value);
    }
    fn get_key() -> K;
}
```

This trait could be implemented multiple times with different `K` types, such as `TypeId`, `&'static str`, `Uuid` and so on:

```rs
struct Foo;

impl GetTypeRegistration for Foo {
    // ...
}

// Implementation with a `Uuid`, some boilerplate could be moved into a derive macro and an attribute, like a former `TypeUuid` trait
impl RegisterType<Uuid> for Foo {
    fn get_key() -> Uuid {
        uuid!("8d17d298-5132-4739-9a13-96e93e4b5034")
    }
}
```

This trait could also be implemented in a derive macro or a blanket implementation:

```rs
// Implementation using `core::any::TypeId`
impl<T> RegisterType<TypeId> for T
where
    T: core::any::Any,
{
    fn get_key() -> TypeId {
        TypeId::of::<T>()
    }
}

// Using a type name, ideally it would be a newtype to indicate that this is a type name and not just some arbitrary string
impl<T> RegisterType<&'static str> for T {
    fn get_key() -> &'static str {
        core::any::type_name::<T>()
    }
}
```

## Drawbacks

> Why should we *not* do this?

This introduces a concept of multiple type identifications which might have no connection between them. There would be no singular central type registry, but instead a number of registries, each with a different id type and for a different purpose.

## Rationale and alternatives

> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not choosing them?
> - What objections immediately spring to mind? How have you addressed them?
> - What is the impact of not doing this?
> - Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

> Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

> - Does this feature exist in other libraries and what experiences have their community had?
> - Papers: Are there any published papers or great posts that discuss this?

> This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

> Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

A `StableTypeId` has been discussed in an issue: https://github.com/bevyengine/bevy/issues/32

## Unresolved questions

> - What parts of the design do you expect to resolve through the RFC process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
> - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

> Think about what the natural extension and evolution of your proposal would
> be and how it would affect Bevy as a whole in a holistic way.
> Try to use this section as a tool to more fully consider other possible
> interactions with the engine in your proposal.
> 
> This is also a good place to "dump ideas", if they are out of scope for the
> RFC you are writing but otherwise related.
> 
> Note that having something written down in the future-possibilities section
> is not a reason to accept the current or a future RFC; such notes should be
> in the section on motivation or rationale in this or subsequent RFCs.
> If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.

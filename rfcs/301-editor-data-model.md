# Feature Name: `editor_data_model`

## Summary

This RFC describes the data model and API that the Editor uses in memory, and the basis for communicating with the Game process and for serializing that data to disk (although the serialization format to disk is outside of the scope of this RFC). The data model is largely inspired by the one of [Our Machinery](https://ourmachinery.com/), as described in [this blog post](https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/).

## Glossary

In this RFC, the following terms are used:

- **Game** refers to any application made with Bevy, which encapsulates proper games but also any other kind of Bevy application (CAD, architecture software, _etc._).
- **Author** refers to any person (including non-technical) building the Game with the intent of shipping it to end users.
- **Editor** refers to the software used to edit Game data and prepare them for runtime consumption, and which this RFC describes the data model of.
- **Runtime** refers to the Game or Editor process running time, as opposed to compiling or building time. When not otherwise specified, this refers to the Game's runtime. In general it's synonymouns of a process running, and is opposed to _build time_ or _compile time_.

## Motivation

We need a data model for the Bevy Editor. We want to make it decoupled from the runtime data model that the Game uses, which is based on compiled Rust types (`World`, _etc._) with a defined native layout from a fixed Rust compiler version (as most types don't use `#[repr(C)]`), so we can handle data migration across Rust compiler versions and therefore across both Editor and Game versions. We also want to make it _transportable_, that is serializable with the intent to transport it over any IPC mechanism, so that the Editor process can dialog with the Game process to enable real-time Game editing ("Play Mode").

## User-facing explanation

The _data model_ of the Editor refers to the way the Editor manipulates editing data, their representation in memory while the Editor executable is running, and the API to do those manipulations. The Editor at its core defines a centralized data model, and all systems use the same API to access that data model, making it the unique source of truth while the Editor process is running. This unicity prevents desynchronization between various subsystems, since they share a same view of the editing data. It also enables global support for common operations such as copy/paste and undo/redo without requiring each subsystem to implement their own variant.

A Bevy application (the Game) uses Rust types defined in one of the `bevy_*` crates, any other third-party plugin library, or the Game itself (_custom types_). The data is represented in memory at runtime by _instances_ (allocated objects) of those Rust types. This representation is optimal in terms of read and write access, but is _static_, that is the type layout is defined during compiling and cannot be changed afterward.

In contrast, the Editor needs to support type changes to allow hot-reloading custom components from the Game, in order to enable fast iteration for the Authors. To enable this, the Editor data model is based on _dynamic types_ defined exclusively during Editor execution. The data model defines an API to instantiate objects from dynamic types, manipulates the data of those instances, but also modify the types themselves while objects are instantiated. This latter case is made possible through a set of migration rules allowing to transform an object instance from a version of a type to another modified version of that type.

The general workflow is as follows:

- The Author starts the Editor executable and create a new project or loads an existing project.
- The Editor launches the Game executable in _edit mode_, whereby the `EditorClientPlugin` is enabled and allows the Editor and the Game to communicate in real time.
- The Editor queries the Game for all its custom types, including components and resources.
- The Editor follows the same process for any third-party library, to enable extending Bevy with new functionalities.
- The Editor builds a database of all (dynamic) types, merging the built-in Bevy types (the ones defined in any `bevy_*` official crate), any third-party library, and the Game.
- The Author creates new object instances from any of those types, and manipulates the data of those instances by setting the value of the properties exposed by the type of the object.
- Optionally, the Author saves the project, which makes the Editor serialize the data model to disk for later reloading.

The data model supports various intrinsic types: boolean, integral, floating-point, strings, arrays, dictionaries, enums and flags, and objects (references to other types). Objects can be references to entities or to components. Dynamic types extend those by composition; they define a series of _properties_, defined by their name and a type themselves. Conceptually, a property can look like:

```txt
type "MyPlayerComponent" {
  property "name"  : string
  property "health": float32
  property "target": entity
  property "weapon": type "MyWeaponComponent"
}
```

The data model offers an API to manipulate object instances:

- get the type of an object instance
- get and set property values of a type instance
- lock an object for exclusive write access; this allows making a set of writes transactional (see below for details)

This API is most commonly used to build a User Interface (UI) to allow the Authors to edit the data via the Editor itself. It is also used by Editor systems to manipulate object instances where needed.

The data model also offers an API to manipulate the types themselves:

- get a type's name
- enumerate the properties of a type
- add a new property
- remove an existing property

This API is most commonly used when reloading the Game after editing its code and rebuilding it, to update the Editor's dynamic types associated with the custom Game types.

The data model guarantees the integrity of its data by allowing concurrent reading of data but ensuring exclusive write access. This allows concurrent use of the data, for example for remote collaboration, so long as write accesses to objects are disjoints. Each object for which write access was given is locked for any other access. This is similar in concept to the Rust borrow model, and to that respect the Editor data model plays the role of the Rust borrow checker.

<!-- Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one. -->

## Implementation strategy

### Names

Being data-heavy, and forfeiting the structural access of Rust types, the data model makes extensive use of strings to reference items. In order to make those operations efficient, we define a `Name` type to represent those strings. The implementation of `Name` is outside of the scope of this RFC and left as an optimization, and should be considered as equivalent to a `String` wrapper:

```rust
struct Name(pub String);
// -or-
type Name = String;
```

The intent is to explore some optimization strategies like string interning for fast comparison. Those optimizations are left out of the scope of this RFC.

### Built-in types

The data model supports a number of built-in types, corresponding to common Rust built-in types.

```rust
enum SignedIntegralType {
    Int8, Int16, Int32, Int64, Int128,
}

enum UnsignedIntegralType {
    Uint8, Uint16, Uint32, Uint64, Uint128,
}

enum FloatingType {
    Float32, Float64,
}

enum SimpleType {
    Bool,
    Int8, Int16, Int32, Int64, Int128,
    Uint8, Uint16, Uint32, Uint64, Uint128,
    Float32, Float64,
}

enum BuiltInType {
    Bool,
    Int8, Int16, Int32, Int64, Int128,
    Uint8, Uint16, Uint32, Uint64, Uint128,
    Float32, Float64,
    Enum(UnsignedIntegralType), // C-style enum, not Rust-style
    String,
    Array(BuiltInType),
    Dict,
    ObjectRef,
}
```

_Note_: We explicitly handle all integral and float bit variants to ensure tight value packing and avoid wasting space both in objects (`DataInstance`; see below) and collections (arrays, _etc._).

### Data world

The data model uses dynamic types, so it can be represented purely with data, without any build-time types. Bevy already has an ideal storage for data: the `World` ECS container. This RFC introduces a new struct to represent the data model, the `DataWorld`:

```rust
struct DataWorld(pub World);
```

Since the underlying `World` already offers guarantees about concurrent mutability prevention, via the Rust language guarantees themselves, those guarantees transfer to the `DataWorld` allowing safe concurrent read-only access or exclusive write access to all data of the data model.

**TODO - Explain where the `DataWorld` lives. Idea was as a resource on the Editor's `World`, but that means regular Editor systems cannot directly write queries into the `DataWorld` ergonomically. See how extract does it for the render world though.**

### Data types

The data world is populated with the dynamic types coming from the various sources described in [User-facing explanation](#user-facing-explanation).

This RFC introduces a `DataType` component that stores the description of a single dynamic type. We use the name `DataType` to hint at the link with the `DataWorld` and avoid any confusion with any pre-existing dynamic object from the `bevy_reflect` crate (_e.g._ `DynamicScene` or `DynamicEntity`).

```rust
#[derive(Component)]
struct DataType {
    name: Name,
    instances: Vec<Entity>,
}
```

The `DataType` itself only contains the name of the type. Other type characteristics depend on the kind of type (object, enum, _etc._) and are defined in other kind-specific components attached to the same `Entity`; see [sub-types](#sub-types) below for details.

The data type contains an array of entities defining all the instances of this type for easy reference, since this information is not encoded in any Rust type (see [Data instances](#data-instances)) so cannot be queried via the usual `Query` mechanism.

#### Type origin

Some marker components are added to the same `Entity` holding the `DataType` itself, to declare the _origin_ of the type:

```rust
// Built-in type compiled from the Bevy version the Game is built with.
#[derive(Component)]
struct BevyType;

// Type from the Game of the currently open project.
#[derive(Component)]
struct GameType;
```

_Note that the Editor itself may depend on a different Bevy version. The built-in Bevy types of that version are not present in the `DataWorld`; only types from the Game are._

This strategy allows efficiently querying for groups of types, in particular to update them when the Game or a library is reloaded.

```rust
fn update_game_type_instances(
    mut query: Query<
        &DataType,
        (With<GameType>, Changed<DataType>)
    >) {
    for data_type in query.iter_mut() {
        // apply migration rules to all instances of this type
        for instance in &mut data_type.instances {
            // [...]
        }
    }
}
```

### Sub-types

Various kinds of types have additional data attached to a separate component specific to that kind. This is referred to as _sub-types_.

The sub-type components allow, via the `Query` mechanism, to batch-handle all types of a certain kind, like for example listing all enum types:

```rust
fn list_enum_types(query: Query<&DataType, With<EnumType>>) {
    for data_type in query.iter() {
        println!("Enum {}", data_type.name);
    }
}
```

#### Simple types

_Simple types_ are built-in types defined by the data model itself, and corresponding to the basic types found in the Rust language. They constitute building blocks to creating more complex types via aggregation (see [Objects](#objects)).

**FIXME - naming conflicting with SimpleType enum above**

```rust
#[derive(Component)]
struct SimpleType {
    value: SimpleTypeEnum,
}
```

#### Enums and flags

Enums are sets of named integral values called _entries_. They are conceptually similar to so-called "C-style" enums. An enum has an underlying integral type called its _storage type_ containing the actual integral value of the enum. The storage type can be any integral type, signed or unsigned, with at most 64 bits (so, excluding `Int128` and `Uint128`). The enum type defines a mapping between that value and its name. An enum value is stored as its storage type, and is equal to exactly one of the enum entries.

**TODO - think about support for Rust-style enums...**

Flags are like enums, except the value can be any bitwise combination of entries, instead of being restricted to a single entry. The storage type is always an unsigned integral type.

The `EnumEntry` defines a single enum or flags entry with a `name` and an associated integral `value`. The value is encoded in the storage type, then stored in the 8-byte memory space offered by `value`. This means reading and writing the value generally involves reinterpreting the bit pattern.

```rust
struct EnumEntry {
    name: Name,
    value: u64,
}
```

The `EnumType` component defines the set of entries of an enum or flags, the storage type, and a boolean indicating whether the type is an enum or a flags.

```rust
#[derive(Component)]
struct EnumType {
    entries: Vec<EnumEntry>,
    storage_type: IntegralType,
    is_flags: bool,
}
```

#### Arrays

Arrays are dynamically-sized homogenous collections of _elements_. The `ArrayType` defines the element type, which can be any valid data type.

```rust
#[derive(Component)]
struct ArrayType {
    elem_type: Entity, // with DataType component
}
```

#### Dictionaries

TODO...

Dictionaries are dynamically-sized homogenous collections mapping _keys_ to _values_, where each key is unique in the instance. The key type is always a string. The `DictType` defines the value type.

```rust
#[derive(Component)]
struct DictType {
    value_type: Entity, // with DataType component
}
```

#### Objects

An _object_ is an heterogenous aggregation of other types (like a `struct` in Rust). The `ObjectType` component contains the _properties_ of the object.

```rust
#[derive(Component)]
struct ObjectType {
    properties: Vec<Property>, // sorted by `byte_offset`
}
```

Properties themselves are defined by a name and the type of their value. The type also stores the byte offset of that property from the beginning of an object instance, which allows packing property values for an object into a single byte stream, and enables various optimizations regarding diff'ing/patching and serialization. Modifying any of the `name`, `value_type`, or `byte_offset` constitute a type change, which mandates migrating all existing instances.

```rust
struct Property {
    name: Name,
    value_type: Entity, // with DataType component
    byte_offset: u32,
    validate: Option<Box<dyn Validate>>,
}
```

The _type layout_ refers to the collection of property offsets and types of an `ObjectType`, which together define:

- the order in which the properties are listed in the object type (the properties are sorted in increasing byte offset order), and any instance value stored in a `DataInstance`.
- the size in bytes of each property value, based on its type.
- eventual gaps between properties, known as _padding bytes_, depending on their offsets and sizes.

Unlike Rust, the Editor data model guarantees that **padding bytes are zeroed**. This property enables efficient instance comparison via `memcmp()`-like byte-level comparison, without the need to rely on individual properties inspection nor any equality operator (`PartialEq` and `Eq` in Rust). It also enables efficient copy and serialization via direct memory reads and writes, instead of relying on getter/setter methods.

The `Property` struct contains also an optional validation function used to validate the value of a property for things like range and bounds check, nullability, _etc._ which is used both when the user (via the Editor UI) attempts to set a value, and when a value is set automatically (_e.g._ during deserialization).

```rust
trait Validate {
    /// Check if a value is valid for the property.
    fn is_valid(&self, property: &Property, value: &Value) -> bool;

    /// Try to assign a value to a property of an instance.
    fn try_set(&mut self, instance: &mut DataInstance, property: &Property, value: &[u8]) -> bool;
}
```

### Data instances

Instances of any type are represented by the `DataInstance` component:

```rust
#[derive(Component)]
struct DataInstance {
    data_type: Entity, // with DataType component
    values: Vec<u8>,
}
```

The type of an instance is stored as the `Entity` holding the `DataType`, which can be accessed with the usual `Query` mechanisms:

```rust
fn get_data_instance_type(
    q_instance: Query<&DataInstance, [...]>,
    q_types: Query<&DataType>)
{
    for data_inst in q_instances.iter() {
        let data_type = q_types
            .get_component::<DataType>(&data_inst.data_type).unwrap();
        // [...]
    }
}
```

The property values are stored in a raw byte array, at the property's offset from the beginning of the array. The size of each property value is implicit, based on its type. Any extra byte (padding byte) in `DataInstance.values` is set to zero.

_Note: The Rust compiler, as per Rust language rules, will reorder fields of any non-`#[repr(C)]` type to optimize the type layout and avoid padding in the Game process. This native type layout of each Game type is then transferred to the Editor to create dynamic types. We expect as a result minimal padding bytes, and therefore do not attempt to further tightly pack the values ourselves, for the sake of simplicity. This has the added benefit to make the Editor type layout binary-compatible with the Game type native layout._

### Migration rules

The _migration rules_ are a set of rules applied when transforming a `DataInstance` from one `DataType` to another `DataType`. The rules define if such conversion can successfully take place, and what is its result.

Given a setup where a migration of a `src_inst: DataInstance` is to be performed from a `src_type: DataType` to a `dst_type: DataType` to produce a new `dst_inst: DataInstance`, the migration rules are:

1. If `src_type == dst_type`, then set `dst_inst = src_inst`.
2. Otherwise, create a new empty `dst_inst` and then for each property present in either `src_type` or `dst_type`:
   - If the property is present on `src_type` only, ignore that property (so it will not be present in `dst_inst`).
   - If the property is present on `dst_type` only, add a new property with a default-initialized value to `dst_inst`.
   - If the property is present on both, then apply the migration rule based on the type of the property.
3. A value migrated from `src_type` to `dst_type == src_type` is unmodified.
4. A value is migrated from `src_type` to `dst_type != src_type` by attempting to coerce the value into `dst_type`:
   - For the purpose of coercion rules, boolean values act like an integral type with a value of 0 or 1.
   - Any integral or floating-point value (numeric types) is coerced to the destination property type according to the Rust coercion rules, with possibly some loss. This means there's no guarantee of round-trip.
   - Numeric types are coerced into `String` by applying a conversion equivalent to `format!()`.
   - String types are coerced into numeric types via the `ToString` trait.
   - Enums are coerced into their underlying unsigned integral type first, then any integral coercion applies.
   - Integral types can be coerced into an enum provided an enum variant with the same value exists and the integral type can be coerced into the underlying unsigned integral type of the enum.
   - A single value is coerced into a single-element array.
   - An array of values is coerced into a single value by retaining the first element of the array, if and only if this element exists and has a type that can be coerced into the destination type.
   - A single value is coerced into a dictionary with a key corresponding to the coercion to `String` of its value, and a value corresponding to the coercion to the dictionary value type. If any of these fail, the entire coercion to dictionary fails.
   - A dictionary with a single entry is coerced into a value by extracting the single-entry value and coercing it, discarding the entry key. If the dictionary is empty, or has more than 1 entry, the coercion fails.
   - Coercions from and to `ObjectRef` are invalid.

_Note: the migration rules are written in terms of two distinct `src_inst` and `dst_inst` objects for clarity, but the implementation is free to do an in-place modification to prevent an allocation, provided the result is equivalent to migrating to a new instance then replacing the source instance with it._

<!-- This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected. -->

## Drawbacks

<!-- Why should we *not* do this? -->

### Abstraction complexity

The data model adds a layer of abstraction over the "native" data the Game uses, which makes most operations indirected and more abstract. For example, setting a value involves invoking a setter method (or equivalent binary offset write), possibly after locking an object for write access, as opposed to simply assigning the value to the object (`c.x = 5;`).

## Rationale and alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate? -->

## Prior art

### Unity3D

The editor of the [Unity3D](https://unity.com/) game engine (or "Unity" for short) is closed-source software, so we can only speculate on how its data model works.

Some hints (C# attributes mainly) tell us that built-in components are implemented in native C++ while others built-in and third-party ones are implemented in C#. For the native part, we have little information, so we focus here on describing the C# handling.

For C# components, the editor embeds a custom .NET runtime host based on Mono, which allows it to load any user assembly and extract from it metadata about the user types, and in particular user components. This enables the editor to display in its Inspector window the user components with editing UI widgets to edit component instances.

Data is serialized and saved as YAML files. Although this process is outside of the scope of this RFC, we can guess from it that the Editor mainly operates on _dynamic types_, and type instances are likely blobs of binary data associated with the fields of those dynamic types. This guess is reinforced by the fact that:

- the editor is known to be written in C++ and the user code is C# so even if the editor instantiated .NET objects within its Mono runtime it would have to access them from C++ so would need a layer of indirection;
- the editor hot-reloads assemblies on change, so is unlikely to have concrete instantiation of those types, even inside its .NET runtime host, otherwise it would have to pay the cost of managing those instances (destroy and recreate on reload) which is likely slow;
- the on-disk serialized format (YAML) shows types and fields referenced by name and by UUID to the file where the type is defined, but does not otherwise contain any more type information;
- the immediate-mode editor UI customization API has some `SerializedProperty` type with generic getter/setter functions for all common data types (bool, string, float, _etc._), and that interface gives access to most (all?) instances of the data model via a unified data-oriented API.

Unity notably supports hot-reloading user code (C# assemblies) during editing and even while the Game is playing in Play Mode inside the Editor. It also supports editing on the fly data while the Game is running in that mode, via its Inspector window. This is often noted as a useful features, and hints again at a data-oriented model.

## Our Machinery

[Our Machinery](https://ourmachinery.com/) has a few interesting blog posts about the architecture of the engine. Notably, a very detailed and insightful blog post about their data model.

<https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/>

The data model is heavily data-oriented using dynamic types:

> The Truth is our toungue-in-cheek name for a centralized system that stores the application data. It is based around IDs, objects, types and properties. An object in the system is identified by a uint64_t ID. Each object has a set of properties (bools, ints, floats, strings, …) based on its type.

The data model explicitly handles _prefabs_ as well as change notifications.

> In addition to basic data storage, The Truth has other features too, such as sub-objects (objects owned by other objects), references (between objects), prototypes (or “prefabs” — objects acting as templates for other objects) and change notifications.

A core goal is atomicity of locking for multi-threaded data access; writing only locks the object being written, and other objects can continue being accessed in parallel for read operations.

They list the following things the data model helps with:

- Dependency tracking via object references and IDs.
- Unified copy/paste for all systems, since all systems read/write through this central data model hub ("The Truth").
- Unified undo/redo for the same reason.
- Real-time collaboration since it's pure data and can be serialized easily, so can be networked.
- Diffs/merges easily with VCS.

They also list some bad experiences with the Bitsquid data model (a predecessor engine from the same devs), which led them to design their current centralized in-memory data model:

- disk-based data model like JSON files can't represent undo/redo and copy/paste, leading to duplicated implementations for each system.
- string-based object references are not semantically different from other strings in a JSON-like format, so cannot be reasoned about without semantic context, and internal references inside the same file are ill-defined.
- file-based models play poorly with real-time collaboration.



<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC. -->

## Unresolved questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

## \[Optional\] Future possibilities

<!-- Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it. -->

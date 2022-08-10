# Feature Name: `editor_data_model`

## Summary

This RFC describes the data model and API that the Editor uses in memory, and the basis for communicating with the Game process and for serializing that data to disk (although the serialization format to disk is outside of the scope of this RFC). The data model is largely inspired by the one of [Our Machinery](https://ourmachinery.com/), as described in [this blog post](https://web.archive.org/web/20220727114600/https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/).

## Glossary

In this RFC, the following terms are used:

- **Game** refers to any application made with Bevy, which encapsulates proper games but also any other kind of Bevy application (CAD, architecture software, _etc._).
- **Author** refers to any person (including non-technical) building the Game with the intent of shipping it to end users.
- **Editor** refers to the software used to edit Game data and prepare them for runtime consumption.
- **Runtime** refers to the Game process running time, as opposed to compiling or building time. In general it's synonymouns of a process running, and is opposed to _build time_ or _compile time_.
- **Project** refers to the Editor's representation of the Game that data is being created/edited for, and all the assets, metadata, settings, or any other data the Editor needs and which is associated to that specific Game. There's a 1:1 mapping between a Game and an Editor project.

## Motivation

The Bevy Editor needs a _data model_, that is a way to organize the data it manipulates, and more precisely here the Game data that it creates and edits for the purpose of transforming it into a final shippable form to be loaded by the Game at runtime. The data model defines the structure of this data, and the APIs to manipulate it.

We want to make the Editor data model decoupled from the runtime data model that the Game uses to avoid the Editor depending on the Game. This avoids having to rebuild the Editor, or even restart its process, when the Game code changes.

We also want to make that data structured in the Editor data model format _transportable_, that is serializable with the intent to transport it over any inter-process communication (IPC) mechanism, so that the Editor process can dialog with the Game process.

## User-facing explanation

The Bevy Editor creates and edits Game data to build scenes and other assets, and eventually organize all those Game assets into some cohesive game content.

### Type exchange

The Editor and the Game have different data models. The Game data is baked into a format optimized for runtime consumption (fast loading from persistent storage and minimal transformation before use), often with platform-specific choices. In contrast, the Editor manipulates a higher-level format independent of the Game itself or the target platform. Therefore the two must exchange Game types and data to enable conversions between the data encoded in the two data models. Because the Game and the Editor are separate processes, this exchange occurs through an inter-process communication (IPC) mechanism.

During development, the Game links and uses the `EditorClientPlugin`. This is a special plugin provided as part of the Editor ecosystem, and which enables the Game and the Editor to communicate via IPC. This plugin is not linked in release builds, and therefore does not ship with the Game. The Game only loads the `EditorClientPlugin` when launched into a special _edit mode_ via a mechanism outside of the scope of this RFC (possibly some environment variable, or command-line argument, or any other similar mechanism).

When writing the Game code, the Author(s) define new Rust types to represent components, resources, _etc._ specific to that Game (for example, `MainPlayer` or `SpaceGun`). Because the Editor does not depend on the Game, those Game types are sent at runtime to the Editor via IPC. This allows the Editor to understand the binary format of the Game objects and convert them into its data model and back.

### Editing workflow

The general editing workflow is as follows:

- The Author starts the Editor executable and create a new project or loads an existing project.
- The Editor launches the Game executable in _edit mode_ to use the `EditorClientPlugin` for Game/Editor communication.
- The Editor queries via IPC the Game for all its Game types, including components and resources.
- The Editor builds a database of all the (dynamic) types received, and make them available through the type API of its data model.
- The Author creates new object instances from any of those types, and manipulates the data of those instances via the various Editor functionalities, themselves using the data model API of the Editor.
- Optionally, the Author saves the project, which makes the Editor serialize the data model to persistent storage for later Editor sessions on the same project.

### Data model

The _data model_ of the Editor refers to the way the Editor manipulates this Game data, their representation in memory while the Editor executable is running, and the API to do those manipulations. The Editor at its core defines a centralized data model, and all the ECS systems of the Editor use the same API to access that data model, making it the unique source of truth while the Editor process is running. This unicity prevents desynchronization between various subsystems, since they share a same view of the editing data. It also enables global support for common operations such as copy/paste and undo/redo, since all ECS systems will observe the same copy/paste/undo/redo changes to that unique source of truth.

When writing the Game code, the Author(s) define new Rust types to represent components, resources, _etc._ specific to that Game (for example, `MainPlayer` or `SpaceGun`). At runtime, those Rust types are used to allocate _instances_ (allocated objects). The representation of the instances in memory (_e.g._ `SpaceGun.fireSpeed` field takes 4 bytes starting at 28 bytes from the start of the object instance), also called the _layout_, is optimal in terms of read and write access speed by the CPU, but it is _static_, that is the type layout is defined during compilation and cannot be changed afterward.

In contrast, we want to enable the Editor to observe changes in the Game code without rebuilding or even restarting the Editor process, in order to enable fast iteration for the Authors. This feature, referred to as _hot-reloading_, requires the Editor to support type changes, which is not possible with static types like the Game uses. Therefore, to allow this flexibility, the Editor data model is based on _dynamic types_ defined exclusively during Editor execution. The data model defines an API to instantiate objects from dynamic types, manipulates the data of those instances, but also modify the types themselves while objects are instantiated. This latter case is made possible through a set of migration rules allowing to transform an object instance from a version of a type to another modified version of that type.

The data model supports various intrinsic types: boolean, integral, floating-point, strings, arrays, dictionaries, enums and flags, and objects (references to other types). Objects can be references to entities or to components. Dynamic types extend those by composition; they define a series of _properties_, defined by their name and a type themselves. Conceptually, a property can look like (illustrative pseudo-code):

```txt
type "MyPlayerComponent" {
  property "name"  : string
  property "health": float32
  property "target": entity
  property "weapon": type "MyWeaponComponent"
}
```

The data model offers an API (the _object API_) to manipulate object instances:

- get the type of an object instance
- get and set property values of a type instance
- lock an object for exclusive write access; this allows making a set of writes transactional (see below for details)

The object API is most commonly used to populate a User Interface (UI) with data. That UI allows the Authors to edit the data via the Editor itself. It's also used by Editor systems to manipulate object instances where needed.

The data model also offers an API (the _property API_) to manipulate the types themselves:

- get a type's name
- enumerate the properties of a type
- add a new property
- remove an existing property

The property API is most commonly used when reloading the Game after editing its code and rebuilding it, to update the Editor's dynamic types associated with the custom Game types.

The data model guarantees the integrity of its data by allowing concurrent reading of data but ensuring exclusive write access. This allows concurrent use of the data, for example for remote collaboration, so long as write accesses to objects are disjoints. Each object for which write access was given is locked for any other access. This is similar in concept to the Rust borrow model, and to that respect the Editor data model plays the role of the Rust borrow checker.

<!-- Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one. -->

## Implementation strategy

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
    Name,
    Enum(UnsignedIntegralType), // C-style enum, not Rust-style
    Array(AnyType),
    Dict(AnyType), // key type is Name
    ObjectRef(Option<Entity>),
}
```

_Note_: We explicitly handle all integral and floating-point bit variants to ensure tight value packing and avoid wasting space both in structs (`GameObject`; see below) and collections (arrays, _etc._).

`ObjectRef` is a reference to any object, custom or built-in. This allows referencing components built-in inside Bevy or the Editor (TBD depending on design of that part). The reference needs to be valid (non-null) when the data gets processed for baking, but can temporarily (including while serialized to disk and later reloaded) be left invalid (`None`) for editing flexibility. Nullable references require a `NullableRef` component to be added to mark the reference as valid even if null.

```rust
#[derive(Component)]
struct NullableRef;
```

This allows filtering out nullable references, and collect invalid `ObjectRef` instances easily via a `Query`, for example to emit some warning to the user.

The `AnyType` is defined as either a built-in type or a reference to a custom Game type (see [Game types](#game-types)).

```rust
enum AnyType {
    BuiltIn(BuiltInType),
    Custom(Entity), // with GameType component
}
```

### Game world

The data model uses dynamic types, so it can be represented purely with data, without any build-time types. Bevy already has an ideal storage for data: the `World` ECS container. This RFC introduces a new struct to represent the data model, the `GameWorld`:

```rust
struct GameWorld(pub World);
```

Since the underlying `World` already offers guarantees about concurrent mutability prevention, via the Rust language guarantees themselves, those guarantees transfer to the `GameWorld` allowing safe concurrent read-only access or exclusive write access to all data of the data model.

**TODO - Explain where the `GameWorld` lives. Idea was as a resource on the Editor's `World`, but that means regular Editor systems cannot directly write queries into the `GameWorld` ergonomically. See how extract does it for the render world though.**

### Game types

The game world is populated with the dynamic types coming from the various sources described in [User-facing explanation](#user-facing-explanation).

This RFC introduces a `GameType` component that stores the description of a single dynamic type from the Game. We use the name `GameType` to hint at the link with the `GameWorld`, and avoid the term "dynamic" to prevent any confusion with any pre-existing dynamic object from the `bevy_reflect` crate (_e.g._ `DynamicScene` or `DynamicEntity`).

```rust
#[derive(Component)]
struct GameType {
    name: Name,
    instances: Vec<Entity>,
}
```

The `GameType` itself only contains the name of the type. Other type characteristics depend on the kind of type (struct, enum, _etc._) and are defined in other kind-specific components attached to the same `Entity`; see [sub-types](#sub-types) below for details.

The game type contains an array of entities defining all the instances of this type for easy reference, since this information is not encoded in any Rust type (see [Data instances](#data-instances)) so cannot be queried via the usual `Query` mechanism.

#### Type origin

Some marker components are added to the same `Entity` holding the `GameType` itself, to declare the _origin_ of the type:

```rust
// Built-in type compiled from the Bevy version the Game is built with.
#[derive(Component)]
struct BevyType;

// Type from the Game user code of the currently open project.
#[derive(Component)]
struct UserType;
```

_Note that the Editor itself may depend on a different Bevy version. The built-in Bevy types of that version are not present in the `GameWorld`; only types from the Game are._

This strategy allows efficiently querying for groups of types, in particular to update them when the Game or a library is reloaded.

```rust
fn update_game_type_instances(
    mut query: Query<
        &GameType,
        (With<UserType>, Changed<GameType>)
    >) {
    for game_type in query.iter_mut() {
        // apply migration rules to all instances of this type
        for instance in &mut game_type.instances {
            // [...]
        }
    }
}
```

### Sub-types

Various kinds of types have additional data attached to a separate component specific to that kind. This is referred to as _sub-types_.

The sub-type components allow, via the `Query` mechanism, to batch-handle all types of a certain kind, like for example listing all enum types:

```rust
fn list_enum_types(query: Query<&GameType, With<EnumType>>) {
    for game_type in query.iter() {
        println!("Enum {}", game_type.name);
    }
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

Arrays are dynamically-sized homogenous collections of _elements_. The `ArrayType` defines the element type, which can be any valid game type.

```rust
#[derive(Component)]
struct ArrayType {
    elem_type: AnyType,
}
```

Array values store their elements contiguously in the byte storage (see [Game objects](#game-objects)), without padding between elements.

#### Dictionaries

Dictionaries are dynamically-sized homogenous collections mapping _keys_ to _values_, where each key is unique in the instance. The key type is always a string (`Name` type). The `DictType` defines the value type.

```rust
#[derive(Component)]
struct DictType {
    value_type: AnyType,
}
```

Dictionary values store their (key, value) pairs contiguously in the byte storage (see [Game objects](#game-objects)), key first followed by value, without padding.

#### Structs

A _struct_ is an heterogenous aggregation of other types (like a `struct` in Rust). The `StructType` component contains the _properties_ of the struct.

```rust
#[derive(Component)]
struct StructType {
    properties: Vec<Property>, // sorted by `byte_offset`
}
```

Properties themselves are defined by a name and the type of their value. The type also stores the byte offset of that property from the beginning of a struct instance, which allows packing property values for a struct into a single byte stream, and enables various optimizations regarding diff'ing/patching and serialization. Modifying any of the `name`, `value_type`, or `byte_offset` constitute a type change, which mandates migrating all existing instances.

```rust
struct Property {
    name: Name,
    value_type: AnyType,
    byte_offset: u32,
    validate: Option<Box<dyn Validate>>,
}
```

The _type layout_ refers to the collection of property offsets and types of an `StructType`, which together define:

- the order in which the properties are listed in the struct type (the properties are sorted in increasing byte offset order), and any instance value stored in a `GameObject`.
- the size in bytes of each property value, based on its type.
- eventual gaps between properties, known as _padding bytes_, depending on their offsets and sizes.

Unlike Rust, the Editor data model guarantees that **padding bytes are zeroed**. This property enables efficient instance comparison via `memcmp()`-like byte-level comparison, without the need to rely on individual properties inspection nor any equality operator (`PartialEq` and `Eq` in Rust). It also enables efficient copy and serialization via direct memory reads and writes, instead of relying on getter/setter methods.

The `Property` struct contains also an optional validation function used to validate the value of a property for things like range and bounds check, nullability, _etc._ which is used both when the user (via the Editor UI) attempts to set a value, and when a value is set automatically (_e.g._ during deserialization).

```rust
trait Validate {
    /// Check if a value is valid for the property.
    fn is_valid(&self, property: &Property, value: &[u8]) -> bool;

    /// Try to assign a value to a property of an instance.
    fn try_set(&mut self, instance: &mut GameObject, property: &Property, value: &[u8]) -> bool;
}
```

### Game objects

Instances of any type are represented by the `GameObject` component:

```rust
#[derive(Component)]
struct GameObject {
    game_type: Entity, // with GameType component
    values: Vec<u8>,
}
```

The type of an instance is stored as the `Entity` holding the `GameType`, which can be accessed with the usual `Query` mechanisms:

```rust
fn get_data_instance_type(
    q_instance: Query<&GameObject, [...]>,
    q_types: Query<&GameType>)
{
    for data_inst in q_instances.iter() {
        let game_type = q_types
            .get_component::<GameType>(&data_inst.game_type).unwrap();
        // [...]
    }
}
```

The property values are stored in a raw byte array, at the property's offset from the beginning of the array. The size of each property value is implicit, based on its type. Any extra byte (padding byte) in `GameObject.values` is set to zero. For example:

```rust
#[repr(C)] // for the example clarity only
struct S {
    f: f32,
    b: u8,
    u: u64,
}
```

has its values stored in `GameObject.values` as:

```txt
[0 .. 3 | 4  | 5 ... 7 | 8 .. 11 ]   bytes
[  f32  | u8 | padding |   u64   ]   values
```

_Note: The Rust compiler, as per Rust language rules, will reorder fields of any non-`#[repr(C)]` type to optimize the type layout and avoid padding in the Game process. This native type layout of each Game type is then transferred to the Editor to create dynamic types. We expect as a result minimal padding bytes, and therefore do not attempt to further tightly pack the values ourselves, for the sake of simplicity._

### Migration rules

The _migration rules_ are a set of rules applied when transforming a `GameObject` from one `GameType` to another `GameType`. The rules define if such conversion can successfully take place, and what is its result.

Given a setup where a migration of a `src_inst: GameObject` is to be performed from a `src_type: GameType` to a `dst_type: GameType` to produce a new `dst_inst: GameObject`, the migration rules are:

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

### Reflection extraction

The reflection data needed to create a `GameType` is extracted from the Game process launched by the Editor, via the help of the `EditorClientPlugin`. The plugin allows communicating with the Editor process to send it the data needed to create the `GameType`s, which is constituted of the list of types of the Game, with for each type:

- its kind (sub-type)
- any additional kind-specific data like the list of properties for a struct

For properties, the plugin extracts the byte offsets of all properties by listing the struct fields and using a runtime `offsetof` implementation like the one proposed by @TheRawMeatball:

```rust
macro_rules! offset_of {
    ($ty:ty, $field:ident) => {
        unsafe {
            let uninit = MaybeUninit::<$ty>::uninit();
            let __base = uninit.as_ptr();
            let __field = std::ptr::addr_of!((*__base).$field);
            __field as usize - __base as usize
        }
    };
}
```

See also the [rustlang discussion](https://internals.rust-lang.org/t/pre-rfc-add-a-new-offset-of-macro-to-core-mem/9273) on the matter. There is currently no Rust `offset_of()` macro, and not custom implementation known to work in a `const` context with the `stable` toolchain. However we only intend to use that macro at runtime, so are not blocked by this limitation.

The details of:

1. how to connect to the Game process from the Editor process via the `EditorClientPlugin`,
2. what format to serialize the necessary type data into to be able to create `GameType`s,

are left outside the scope of this RFC, and would be best addressed by a separate RFC focused on that interaction.

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

This design is thought to be the best in space because it adapts to the Bevy architecture the design of Our Machinery, which was built by a team of experts with years of experience and 2 shipped game engines. Our Machinery is also a relatively new game engine, so is not constrained by legacy choices which would make its design arguably outdated or not optimal.

The impact of not doing this is obvious: we need to do _something_ about the Editor data model, so if not this RFC then we need another design anyway.

The RFC is targeted at the Bevy Editor, so the change is to be implemented within that crate (or set of crates), with any additional changes to Bevy itself to catter for possible missing features in reflection. The data model constitutes the core of the Bevy Editor so implementation in an ecosystem crate is not suited.

- **Alternative**: Use the current `Reflect` architecture as-is, with `Path`-based property manipulation. This is the most simple way, as it avoids the complexity of type layouts and byte stream values (and the `unsafe` associated). The drawback is that calling a virtual (trait) method for each get and set of each value is reasonable for manual edits in the Editor (execution time much smaller than user reaction time) but might become a performance issue when automated edits come into play, like for an animation system with a timeline window allowing to scroll current time and update all animated properties of all objects in the scene at once.

## Prior art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC. -->

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

### Our Machinery

[Our Machinery](https://ourmachinery.com/) has a few interesting blog posts about the architecture of the engine. Notably, a very detailed and insightful blog post about their data model.

<https://web.archive.org/web/20220727114600/https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/>

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

### Guerrilla Games

[🎞 _GDC 2017 - Creating a Tools Pipeline for Horizon: Zero Dawn_](https://www.youtube.com/watch?v=KRJkBxKv1VM)

Guerrilla Games started to rebuild their entire tools pipeline during production of _Horizon: Zero Dawn_. The aim was to replace a set of disparate tools, including a custom Maya plugin embedding the renderer of _Decima_ (their proprietary game engine); they state Maya is not suited for level editing.

#### General architecture

Given their various constraints (game already in production, rebuilding everything from scratch, wanted to reuse Decima's libraries), they decided to go for a tools framework built on top of the Decima code. In particular, they reuse the existing Decima RTTI (reflection) system. Their focus is also on a limited subset of a full-featured editor: object creation and placement. Nonetheless, they approached the design with the intent to eventually replace all other tools, therefore their design decisions are worth considering.

In detail, the following choices were made:

1. Integrate limited sub-systems OR the full game

   - Limited sub-systems is purpose built, controllable, fast to build, but adds a new codepath separate from the game and encourages visual and behavioral disparities.
   - Full game makes available all game systems under a single unified codepath. In their case, it also adds for free their in-game DebugUI system. However there is an overhead to running all systems of an entire game all the time for any editing action only concerned by a few of them.

   Due to time constraints, they decided to integrate the full game inside the Editor.

2. In-process OR out-of-process

   - In-process (game and editor share the same execution process) is fast to setup, allows direct and synchronous access to game data from the editor. The drawbacks are instabilities (game crash = editor crash = loss of work) and a high difficulty to enforce code separation.
   - Out-of-process (game process and editor process run separately) enables stability, cleaner separated code, and more responsive UI (game performance has no impact on the editor UI, provided CPU not starved). However the inital development is more complex, and it's harder to debug 2 asynchronous processes.

   In the end, they decided to go for in-process due to time constraints, but with a design that allows future out-of-process by enforcing a very strict code separation between Editor and Game. To that end, we can consider they conceptually went with an out-of-process solution.

#### Editor model

The editor supports play-time editing, whereby the game is paused and some edits are made to a level, before the game is resumed on that newly-edited level. However the game uses baked data (called _converted_ in the presentation), and baking is a destructive process which loses some information necessary for editing (like, objects hierarchies of static objects before it's flattened, to allow local-space editing). To work around this issue, they introduce an Editor Node, which contains the edit-time representation of (and maps 1:1 with) a runtime (baked) game object. This defines their data model for the editor, with a one-way relationship Editor -> Game. So Game code doesn't have any Editor-specific code. They build 2 separate hierarchies of Editor Nodes, one for the Editor to manipulate, and one tied with the Game, to enforce code separation (and later allow out-of-process).

They mention some specific non-goals:

- No proxy representation of Game objects.
- No string-based accessors to Game objects.
- No Editor-specific boilerplate.
- No code generation step in the build.

Instead they aimed at:

- Working directly with conrete objects.
- Directly modifying object members.
- Re-using the editing and baking tools already existing to create/transform data.

The data model allows object changes to the Game by generating _transactions_ applied to the Node Tree of the Editor itself, which then broadcasts those changes to the Node Tree of the Game, which applies them to the actual runtime game objects. This allows making _e.g._ a position change in local space (relative to parent object) in the Editor, where the hierarchy is known, which is transformed into a world-space transaction sent to the Game which apply them to baked that (which don't have any hierarchy info left).

When the change originates from the Game itself, like object placement / vegetation painting / terrain scuplting inside the rendered viewport, the Game sends back an _intention_ that is sent to the Editor. Then the Editor applies it like any other modification. This keeps the Editor authority over its Node Tree, keeps the flow of changes from Editor to Game, and prevents infinite loops (Game modified Editor data, which applies them to Game, _etc._).

For newly-added objects, they go into details on how they generate a patch file for the Game then apply it while the Game is paused. This feature is named _GameSync_ and has the following reported properties:

- ➕ Works on all changes throughout the Editor
- ➕ Works on target platform (console)
- ➕ No game restart required
- ➖ Complex change analysis
- ➖ Performance issues
- ➖ Crashes, inconsistencies
- ➖ Low level of confidence from users

From those conclusions, including the statement that they wish to rewrite GameSync in the future, we can conclude this feature represents a challenging task.

Finally, the Editor data model allows them to expose a Python API to build new tools easily.

#### Undo system

They also specifically list requirements for an Undo system:

- Simple and reliable.
- No virtual undo/redo functions:
  - Didn't want the possibility of undo not being symmetric to redo
  - Didn't want to have to write new undo/redo functions for each new feature

The ideal undo system they aimed at can automatically track changes and how to revert them. They achieve this with manual tagging for start and end of modification. Change detection is based on reflection (RTTI) and builds a list of commands (add object, set value, _etc._) that can be easily reverted (remove object, set old value, _etc._). The system includes a notification mechanism for any system to observe changes to objects, which helps system decoupling and prevent duplicated functionalities. They explicitly mention a Model-View-Controller (MVC) pattern.

They list some pros and cons of this approach:

- ➕ Reliable system for arbitrarily complex changes
- ➕ No distinction between undo and redo (all commands)
- ➕ Stable code that changed very little once written
- ➖ Manual code tagging easy to forget, hard to debug
- ➖ No filesystem operations support

## Unresolved questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

1. How do we store string instances? Storing a `String` requires an extra heap allocation per string and makes `GameObject` non-copyable. Interning and using an ID would solve this.

## Future possibilities

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

Because the Editor data model provides a centralized source of truth for all editing systems, this enables building a unique undo/redo system and a unique copy/paste system shared by all other editing systems, without the need for each of those editing systems to design their own.

I expect the following future RFCs to build on top of this one:

- Game/Editor communication (IPC)
- Undo/redo system
- Copy/paste system
- Live editing (update Game data in real-time while editing in Editor)
- Remote collaboration (multiple users editing the same project collaboratively from multiple remote locations via networking)

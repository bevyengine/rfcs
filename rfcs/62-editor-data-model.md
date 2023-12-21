# Feature Name: `editor_data_model`

## Summary

This RFC describes the data model and API that the Editor uses in memory,
and the basis for communicating with the Game process
and for serializing that data to disk
(although the serialization format to disk is outside of the scope of this RFC).
The data model is largely inspired by the one of [Our Machinery](https://ourmachinery.com/),
as described in [this blog post](https://web.archive.org/web/20220727114600/https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/).

## Glossary

In this RFC, the following terms are used:

- **Game** refers to any application made with Bevy, which encapsulates proper games but also any other kind of Bevy application (CAD, architecture software, _etc._).
- **Author** refers to any person (including non-technical) building the Game with the intent of shipping it to end users.
- **Editor** refers to the software used to edit Game data and prepare them for runtime consumption.
- **Runtime** refers to the Game process running time, as opposed to compiling or building time. In general it's synonymouns of a process running, and is opposed to _build time_ or _compile time_.
- **Project** refers to the Editor's representation of the Game that data is being created/edited for, and all the assets, metadata, settings, or any other data the Editor needs and which is associated to that specific Game.

## Motivation

### Editor MVP

The Bevy Editor is an application focusing on visual editing of Bevy scenes,
which are collections of entities and components.
As an MVP (Minimal Valuable Product), this is its primary function.

There are currently two major classes of approaches being considered for the Editor:

- embedding the Editor into the Game itself, for example as a `Plugin`; and
- having a separate Editor application independent from the Game.

This RFC assumes the Editor is a standalone application.
The motivating argument is that an Editor for a game engine
is typically a standalone application downloaded and installed by the user,
and reused multiple times to create many different Games.
This is the case for all the major game engines (Unity3D, UnrealEditor, Godot).
The data model described by this RFC could be use with an embedded Editor,
although some of its rationales might be less relevant in that scenario.

In the context of a standalone Editor application,
the ability to quickly iterate on saving the scene
and reloading it inside the (separate) runtime Game application
is also generally considered part of an MVP.
This is to ensure some decent productivity for the Editor user.

In any case however, editing a scene includes the ability to:

- Create new empty scenes
- Save to disk and reload the scene
- Add entities and components to the scene
- Visualize the list of existing entities and their components
- Inspect the data associated with components, modify that data with a UI
- Visual placement in 3D (for entities with a `Transform` component)
  is generally considered an MVP feature too
- Undo / redo actions

This RFC argues that the latter point (undo/redo) is critical for an MVP,
as editing without Undo can quickly become as painful as building a scene from code.
A workaround could be to reload from disk, but an MVP won't focus on load speed,
so this can quickly become a time sink.

### Undo / redo

Undo / redo systems can take many forms.
In its simplest form,
an _ad hoc_ system allows undos and redos for a specific type of editing.
For example, adding an entity or component can have a simple undo system
which deletes that item.
For redo, things are slightly more complex,
as the system needs to remember how to re-create the entity or component.
Again, there are various options here,
but they all require some form of serializing of the data to save it for a later redo.

Ad hoc systems generally have the advantage of being simpler,
as they're focused on a single type of editing.
For example, an undo / redo system for adding and removing components
only requires being able to manipulate components,
and doesn't care for example about manipulating audio clips or animation tracks.
However, the multiplying of those specialized systems means that
the same kind of code is written again and again with slight variations,
as all undo / redo systems need some form of diffing for example.

An alternative design to undo / redo solving this issue
is to create a global undo system for any kind of user action in the Editor.
This offers several advantages:

- Single code to maintain and debug
- Consistent user experience (key binding, undo UI, _etc._)
- Adding support for a new kind of edits is vastly simpler,
  as it requires only an adapter to the global undo / redo system,
  but most other functions are already shared and available.
- Ability to have a single unified _command palette_,
  which shows all actions (edits) available to the user,
  and is as such a great tool for learning.

This global approach does bear however the inconvenient of an initial larger effort
to build such global system and its abstraction of all user actions.

Because of the advantages mentioned above,
this RFC proposes that a global unified undo / redo system be adopted for the Editor.

### Data Model

The _data model_ defines the abstraction that the Editor uses
to represent the user actions and manipulate the scene data as a result.

An abstraction is desirable for several reasons:

- As a separate process, the Editor doesn't have direct knowledge of the Game types.
  This means for example that it doesn't know any custom Game component.
  However the user still expects being able to edit their own components,
  in addition to the Bevy built-in ones.
  So those custom components need to be abstracted in a way the Editor can edit them
  without having to load and execute the actual Game code inside the Editor process,
  which would pose many issues with Rust compiler version, dynamic loading, _etc._.
- The undo / redo system should ideally be able to work with any kind of action/edit,
  without having to know the underlying types those actions apply to.
  This ability greatly simplifies writting such a system,
  and make it automatically extensible to new actions in the future.
- Similarly, any system working globally, like a command palette,
  becomes automatically extensible by using a single shared abstraction.
- Looking forward, having an abstraction which allows serializing user actions
  enables _transporting_ those actions and executing them somewhere else.
  In the context of an Editor talking to a live Game process,
  the abstaction enables building some form of inter-process communication (IPC) mechanism,
  which allows live editing, both locally and remotely (server, console devkit, ...).

Based on this abstraction, the overall editing process for users becomes:

1. Users generate actions (click, _etc._) to edit the scene data.
1. Those actions produce _edits_, with associated editing data.
   For example, adding a new component generates an "Add Component" edit,
   which contains the type of component to add and the target entity to add it to.
1. Edits are optionally recorded by the undo / redo system for later undo.
   That system can also act as an edit producer during redo.
1. Edits are finally consumed by the Editor and applied to the scene data to modify it.

Note that this data model abstraction stragegy is the same strategy adopted by `serde`
to ensure different incompatible formats of serialization and deserialization
can all work together and manipulate the same data,
without requiring all formats to understand each other.
See the [Serde data model](https://serde.rs/data-model.html) for reference.
For Bevy, we already have the `Reflect` API and the `DynamicScene` type,
which allow manipulating objects in a generic way
without actually knowing at build time the types being edited.

## User-facing explanation

<!-- Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one. -->

The Editor data model is the API by which the various Editor parts edit the Game data.
It's a global abstraction of the various data sources (scenes, assets, ...),
and the various editing systems (3D view, `Parent`-based hierarchy, asset inspector, ...) manipulating them.

### Overview

A _data source_ is some Game data which the Editor can create and modify. The Editor supports a few built-in data sources corresponding to common Bevy engine types:

- Scenes are collections of entities and components.
  They correspond to the `Scene` Bevy type.
- Components are ECS `Component`s`, structured collections of individual fields.
- Assets are any other type of reusable data referenced by the Game,
  like meshes, images, fonts, audio clips, animation tracks, ...
  They correspond to the `Asset` Bevy trait.

Thanks to the data source abstraction,
the Editor can handle custom components and assets defined in Game code.

An _editing system_ is any data consumer
using the data model API to query and modify data from one or more data sources.
Typical examples include:

- a 3D view allowing the user to place objects in space,
  which requires editing the `Transform` of entities.
- an asset inspector to list the various `Asset`s in use by the Game,
  and edit their import settings (`.meta` files).
- a hierarchy view to group and reorganize entities based on their `Parent` hierarchy.

This abstracted editing model offers many advantages:

- Robust: the sharing of common steps like edit diffing means less code to maintain,
  and less code to test and debug.
- Extensible: new data sources and new editing systems can easily be added,
  without the need for changing existing ones.
- Observable: an editing system can list all possible registered editing actions
  (command palette),
  or observe all data changes, and record and replay them
  (undo / redo system, automation / scripting system).
  This replay can also be remote (live editing).

### Game type discovery

In order to allow the user to edit custom Game types (components, assets, ...),
the Editor needs to discover those types.

There are two main strategies for this:

1. The Editor builds the Game code and extracts from it metadata about the types.
   This requires some tooling to access the type system of a Rust build.
   However, this provides the best user experience, as discovery is automated
   and exhaustive.

2. The Editor runs the Game in a separate process, and communicates with it,
   requesting a serialized dump of all registered types.
   This is somewhat easier to implement that the previous strategy,
   in particular because Bevy requires registering all types before starting the app,
   so an `EditorPlugin` could automatically read the `TypeRegistry` once the app runs.
   However, from a user experience perspective,
   this means types have to be actively registered before the Editor can "see" them.

Once the types are gathered, they are serialized and sent to the Editor.
This makes them available to the data model API
to create instances of those types, and edit them.

### Data sources

The three built-in Editor data sources are:

- Scenes
- Components
- Asset import settings (`.meta` files)

Scenes are the main editing unit, and contain a collection of entities and components.
Bevy already supports scenes with types unknown at build time
via the `DynamicScene` type and related API.
That API is largely based on the Bevy reflection (`Reflect`) API.

Components are structured collections of fields, implementing the `Component` trait.
They can be accessed via the reflection API.

Assets are serialized to disk accompanied with their import settings.
Those settings are written in `.meta` files alongside the assets.
The import settings are also accessed via the reflection API.

### Editing systems

The Editor is composed of a variety of editing systems.
Each editing system manipulates some Game data via the data model API.
They provide the basic building blocks for the user to edit the Game data.

Each editing system uses the same basic workflow.
Let's take the example of a hierarchy panel,
which shows the list of entities in a scene as a hierarchy
based on the relationship defined by the `Parent` component.
The hierarchy panel allows the user to edit this hierarchy by reparenting entities.
To achieve this, the hierarchy panel uses the data model API to:

- query the list of entities in the current scene;
- for each entity, query the data of its `Parent` component (if any);
- set new data to the `Parent` component when the user edits the hierarchy.

For each user action, the editing system generates an _edit diff_,
which represents the difference between the "before" and "after" state of an object.
For our example, the diff contains the old and new values of the `Parent` component data,
which are the old and new parent entities.
Next, the editing system asks the Editor to apply this diff to a target object,
like for example the `Parent` component of entity `3v0`.
Conceptually, we can represent the edit diff as:

```txt
EditDiff {
    target: 3v0/Parent
    old: 18v0,
    new: 34v1,
}
```

This diff is used to mutate the actual scene being edited.
It's also recorded by the undo system, to allow the user to undo their action. Because the scene is mutated via diffs only,
there's no concern about ordering multiple Bevy systems mutating it,
which prevents complex coordination issues between various plugins
and simplifies writing robust third-party plugins to extend the Editor.

Editing components and asset import settings,
or any other kind of data source,
follows the exact same scheme.

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

### Edit diffs

Edit diffs representing the difference between two versions of a same object.
they allow a set of basic operations:

- calculate the difference between two `Reflect` objects,
- apply a diff onto a `Reflect` object to obtain another `Reflect` object,
- serialize and deserialize the diff itself,

such that calculating the edit diff between an old and a new object,
and applying that diff to the old object, yields the new object.

Diffs in details are out of the scope of this RFC,
which assumes they already exist for both `Reflect` and `DynamicScene`.
See for example [#5563](https://github.com/bevyengine/bevy/issues/5563).

In the following, we assume the existence of an `EditDiff` type representing the diff itself,
and an `EditEvent` type representing the diff and a specific target.

```rust
struct EditDiff {
    // [...]
}

impl EditDiff {
    /// Make a new diff from an old and new objects.
    fn make(old: &dyn Reflect, new: &dyn Reflect) -> EditDiff;
    /// Apply the current diff to a target, yielding a new object.
    fn apply(&self, target: &mut dyn Reflect) -> Box<dyn Reflect>;
    /// Apply the current diff by in-place mutating a target object.
    fn apply_inplace(&self, target: &mut dyn Reflect);
}

#[derive(Event)]
struct EditEvent {
    target: Entity,
    diff: EditDiff,
}
```

### Data model API

#### `EditScene`

The primary editing object is the `EditScene`.
Each `EditScene` represents a single scene being edited.
From a user's perspective, this corresponds to an "open" scene document.
The `EditScene` is a Bevy component, and is manipulated with the ECS API;
spawning an entity with an `EditScene` opens a particular scene file,
while despawning it closes the scene.

The `EditScene` internally wraps a `DynamicScene` describing its content,
and an `AssetPath` defining where that scene is saved as an asset.
It's not an `Asset` itself however,
because the data model needs to control and observe mutating operations
and handle load from and save to disk.

```rust
#[derive(Component)]
pub struct EditScene {
    path: Option<AssetPath>,
    data: DynamicScene,
}

impl EditScene {
    /// Create a new empty scene.
    pub fn new() -> Self;
    /// Get the asset path where the scene is saved to, if any.
    pub fn path(&self) -> Option<&AssetPath>;
    /// Set the asset path where the scene is saved to.
    pub fn set_path(&mut self, path: AssetPath);
    /// Get read-only access to the scene content.
    pub fn data(&self) -> &DynamicScene;
    /// Apply a diff to mutate the scene content.
    pub fn apply_diff(&mut self, diff: EditDiff);
}

// Example: create a new empty scene with a given path
fn new_scene(mut commands: Commands) {
    let mut scene = EditScene::new();
    scene.set_path("scenes/level1.scene");
    commands.spawn(scene);
}

// Example: enumerate all open scenes
fn list_scenes(q_scenes: Query<&EditScene>) {
    for scene in q_scenes {
        info!("Scene: {} resources, {} entities, path={:?}",
            scene.data().resources.len(),
            scene.data().entities.len(),
            scene.path());
    }
}

// Example: save and close a scene
fn save_and_close(
    q_scenes: Query<&EditScene>,
    mut commands: Commands,
    mut events: EventReader<SaveAndCloseSceneEvent>,
) {
    for ev in events.read() {
        let scene_entity = ev.0;
        let Ok(scene) = q_scenes.get(scene_entity) else {
            continue;
        };
        if scene.save().is_ok() {  // saves to scene.path()
            // Close the scene by destroying the EditScene
            commands.entity_mut(scene_entity).despawn_recursive();
        }
    }
}
```

As usual, ECS change detection allows a system to be notified
when a scene has been mutated.

#### `EditEvent`

Applying an `EditDiff` to an `EditScene` is done via _events_.
An edit system sends an `EditEvent` with the `EditDiff` to apply
and the target entity owning the `EditScene` to mutate.
The event is sent as usual, typically with `EventWriter<EditEvent>`.

```rust
#[derive(Event)]
struct EditEvent {
    target: Entity,
    diff: EditDiff,
}
```

Internally, the Editor runs a system to consume those events
and apply the diffs to the various `EditScene` targets.
This ensures changes to a scene are sequential and ordered.
Careful ordering of editing systems through ECS schedules
allows prioritizing some edits over others.

```rust
fn apply_scene_edits(
    mut q_scenes: Query<&mut EditScene>,
    mut reader: EventReader<EditEvent>,
) {
    for ev in reader {
        let Ok(mut scene) = q_scenes.get_mut(ev.target) else {
            continue;
        };
        scene.apply_diff(ev.diff);
    }
}
```

By leveraging the lower level functionalities `Events`,
and the ordering of ECS schedules,
an edit system can also observe some edits,
for example to record them (undo system),
and rewrite them or inject new edits (redo system).

------------
v TODO from here v


```rust
pub struct DataModel {
    // [...]
}

impl DataModel {
    pub fn mutate_scene(&mut self, )
}
```

```rust
pub trait Action {
    fn exec(&mut self, scene: &EditScene) -> EditDiff;
}

pub trait DataModel {
    // Scenes
    fn new_scene(&mut self) -> EditScene;
    fn open_scene(&mut self, path: AssetPath) -> Result<EditScene, OpenSceneError>;
    fn close_scene(&mut self, path: AssetPath);
    fn scenes(&self) -> impl Iterator<Item = &EditScene>;
    fn scenes_mut(&mut self) -> impl Iterator<Item = &mut EditScene>;

    // Actions
    fn register_action(&mut self, action: Action, name: String);
    fn unregister_action(&mut self, name: &str);
}
```

## Drawbacks

<!-- Why should we *not* do this? -->

### Abstraction complexity

The data model adds a layer of abstraction over the "native" data the Game uses,
which makes most operations indirected and more abstract.
For example, setting the `Transform` of a component involves creating an edit
and associating the value the user wants to assign,
then letting the Editor apply that edit to the actual component instance,
as opposed to simply accessing the component directly through the `World`.

## Rationale and alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate? -->

This design is thought to be the best in space because it adapts to the Bevy architecture the design of Our Machinery,
which was built by a team of experts with years of experience and 2 shipped game engines.
Our Machinery is also a relatively new game engine,
so is not constrained by legacy choices which would make its design arguably outdated or not optimal.

The impact of not doing this is obvious: we need to do _something_ about the Editor data model,
so if not this RFC then we need another design anyway.

The RFC is targeted at the Bevy Editor, so the change is to be implemented within that crate (or set of crates),
with any additional changes to Bevy itself to catter for possible missing features in reflection.
The data model constitutes the core of the Bevy Editor so implementation in an ecosystem crate is not suited.

- **Alternative**: Use the current `Reflect` architecture as-is,
with `Path`-based property manipulation.
This is the most simple way, as it avoids the complexity of type layouts and byte stream values (and the `unsafe` associated).
The drawback is that calling a virtual (trait) method for each get and set of each value
is reasonable for manual edits in the Editor (execution time much smaller than user reaction time)
but might become a performance issue when automated edits come into play,
like for an animation system with a timeline window
allowing to scroll current time and update all animated properties of all objects in the scene at once.

## Prior art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC. -->

### Unity3D

The editor of the [Unity3D](https://unity.com/) game engine (or "Unity" for short) is closed-source software,
so we can only speculate on how its data model works.

Some hints (C# attributes mainly) tell us that built-in components are implemented in native C++
while others built-in and third-party ones are implemented in C#.
For the native part, we have little information, so we focus here on describing the C# handling.

For C# components, the editor embeds a custom .NET runtime host based on Mono,
which allows it to load any user assembly and extract from it metadata about the user types,
and in particular user components.
This enables the editor to display in its Inspector window the user components with editing UI widgets
to edit component instances.

Data is serialized and saved as YAML files.
Although this process is outside of the scope of this RFC, we can guess from it that the Editor mainly operates on _dynamic types_,
and type instances are likely blobs of binary data associated with the fields of those dynamic types.
This guess is reinforced by the fact that:

- the editor is known to be written in C++ and the user code is C#
so even if the editor instantiated .NET objects within its Mono runtime
it would have to access them from C++ so would need a layer of indirection;
- the editor hot-reloads assemblies on change,
so is unlikely to have concrete instantiation of those types,
even inside its .NET runtime host,
otherwise it would have to pay the cost of managing those instances
(destroy and recreate on reload) which is likely slow;
- the on-disk serialized format (YAML) shows types and fields referenced by name
and by UUID to the file where the type is defined,
but does not otherwise contain any more type information;
- the immediate-mode editor UI customization API has some `SerializedProperty` type
with generic getter/setter functions for all common data types (bool, string, float, _etc._),
and that interface gives access to most (all?) instances of the data model
via a unified data-oriented API.

Unity notably supports hot-reloading user code (C# assemblies) during editing
and even while the Game is playing in Play Mode inside the Editor.
It also supports editing on the fly data while the Game is running in that mode,
via its Inspector window.
This is often noted as a useful features, and hints again at a data-oriented model.

### Our Machinery

[Our Machinery](https://ourmachinery.com/) has a few interesting blog posts about the architecture of the engine.
Notably, a very detailed and insightful blog post about their data model.

<https://web.archive.org/web/20220727114600/https://ourmachinery.com/post/the-story-behind-the-truth-designing-a-data-model/>

The data model is heavily data-oriented using dynamic types:

> The Truth is our toungue-in-cheek name for a centralized system that stores the application data. It is based around IDs, objects, types and properties. An object in the system is identified by a uint64_t ID. Each object has a set of properties (bools, ints, floats, strings, …) based on its type.

The data model explicitly handles _prefabs_ as well as change notifications.

> In addition to basic data storage, The Truth has other features too, such as sub-objects (objects owned by other objects), references (between objects), prototypes (or “prefabs” — objects acting as templates for other objects) and change notifications.

A core goal is atomicity of locking for multi-threaded data access;
writing only locks the object being written,
and other objects can continue being accessed in parallel for read operations.

They list the following things the data model helps with:

- Dependency tracking via object references and IDs.
- Unified copy/paste for all systems, since all systems read/write through this central data model hub ("The Truth").
- Unified undo/redo for the same reason.
- Real-time collaboration since it's pure data and can be serialized easily, so can be networked.
- Diffs/merges easily with VCS.

They also list some bad experiences with the Bitsquid data model (a predecessor engine from the same devs),
which led them to design their current centralized in-memory data model:

- disk-based data model like JSON files can't represent undo/redo and copy/paste, leading to duplicated implementations for each system.
- string-based object references are not semantically different from other strings in a JSON-like format, so cannot be reasoned about without semantic context, and internal references inside the same file are ill-defined.
- file-based models play poorly with real-time collaboration.

### Guerrilla Games

[🎞 _GDC 2017 - Creating a Tools Pipeline for Horizon: Zero Dawn_](https://www.youtube.com/watch?v=KRJkBxKv1VM)

Guerrilla Games started to rebuild their entire tools pipeline during production of _Horizon: Zero Dawn_.
The aim was to replace a set of disparate tools,
including a custom Maya plugin embedding the renderer of _Decima_ (their proprietary game engine);
they state Maya is not suited for level editing.

#### General architecture

Given their various constraints
(game already in production, rebuilding everything from scratch, wanted to reuse Decima's libraries),
they decided to go for a tools framework built on top of the Decima code.
In particular, they reuse the existing Decima RTTI (reflection) system.
Their focus is also on a limited subset of a full-featured editor:
object creation and placement.
Nonetheless, they approached the design with the intent to eventually replace all other tools,
therefore their design decisions are worth considering.

In detail, the following choices were made:

1. Integrate limited sub-systems OR the full game

   - Limited sub-systems is purpose built, controllable, fast to build,
   but adds a new codepath separate from the game and encourages visual and behavioral disparities.
   - Full game makes available all game systems under a single unified codepath.
   In their case, it also adds for free their in-game DebugUI system.
   However there is an overhead to running all systems of an entire game all the time
   for any editing action only concerned by a few of them.

   Due to time constraints, they decided to integrate the full game inside the Editor.

2. In-process OR out-of-process

   - In-process (game and editor share the same execution process) is fast to setup,
   allows direct and synchronous access to game data from the editor.
   The drawbacks are instabilities (game crash = editor crash = loss of work)
   and a high difficulty to enforce code separation.
   - Out-of-process (game process and editor process run separately) enables stability,
   cleaner separated code,
   and more responsive UI (game performance has no impact on the editor UI, provided CPU not starved).
   However the inital development is more complex, and it's harder to debug 2 asynchronous processes.

   In the end, they decided to go for in-process due to time constraints,
   but with a design that allows future out-of-process by enforcing a very strict code separation between Editor and Game.
   To that end, we can consider they conceptually went with an out-of-process solution.

#### Editor model

The editor supports play-time editing, whereby the game is paused and some edits are made to a level,
before the game is resumed on that newly-edited level.
However the game uses baked data (called _converted_ in the presentation),
and baking is a destructive process which loses some information necessary for editing
(like, objects hierarchies of static objects before it's flattened, to allow local-space editing).
To work around this issue, they introduce an Editor Node,
which contains the edit-time representation of (and maps 1:1 with) a runtime (baked) game object.
This defines their data model for the editor, with a one-way relationship Editor -> Game.
So Game code doesn't have any Editor-specific code.
They build 2 separate hierarchies of Editor Nodes, one for the Editor to manipulate,
and one tied with the Game, to enforce code separation (and later allow out-of-process).

They mention some specific non-goals:

- No proxy representation of Game objects.
- No string-based accessors to Game objects.
- No Editor-specific boilerplate.
- No code generation step in the build.

Instead they aimed at:

- Working directly with conrete objects.
- Directly modifying object members.
- Re-using the editing and baking tools already existing to create/transform data.

The data model allows object changes to the Game by generating _transactions_
applied to the Node Tree of the Editor itself,
which then broadcasts those changes to the Node Tree of the Game,
which applies them to the actual runtime game objects.
This allows making _e.g._ a position change in local space (relative to parent object) in the Editor,
where the hierarchy is known,
which is transformed into a world-space transaction sent to the Game
which apply them to baked that (which don't have any hierarchy info left).

When the change originates from the Game itself,
like object placement / vegetation painting / terrain scuplting inside the rendered viewport,
the Game sends back an _intention_ that is sent to the Editor.
Then the Editor applies it like any other modification.
This keeps the Editor authority over its Node Tree, keeps the flow of changes from Editor to Game,
and prevents infinite loops (Game modified Editor data, which applies them to Game, _etc._).

For newly-added objects, they go into details on how they generate a patch file for the Game
then apply it while the Game is paused.
This feature is named _GameSync_ and has the following reported properties:

- ➕ Works on all changes throughout the Editor
- ➕ Works on target platform (console)
- ➕ No game restart required
- ➖ Complex change analysis
- ➖ Performance issues
- ➖ Crashes, inconsistencies
- ➖ Low level of confidence from users

From those conclusions, including the statement that they wish to rewrite GameSync in the future,
we can conclude this feature represents a challenging task.

Finally, the Editor data model allows them to expose a Python API to build new tools easily.

#### Undo system

They also specifically list requirements for an Undo system:

- Simple and reliable.
- No virtual undo/redo functions:
  - Didn't want the possibility of undo not being symmetric to redo
  - Didn't want to have to write new undo/redo functions for each new feature

The ideal undo system they aimed at can automatically track changes and how to revert them.
They achieve this with manual tagging for start and end of modification.
Change detection is based on reflection (RTTI)
and builds a list of commands (add object, set value, _etc._)
that can be easily reverted (remove object, set old value, _etc._).
The system includes a notification mechanism for any system to observe changes to objects,
which helps system decoupling and prevent duplicated functionalities.
They explicitly mention a Model-View-Controller (MVC) pattern.

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

1. How do we store string instances? Storing a `String` requires an extra heap allocation per string
and makes `GameObject` non-copyable.
Interning and using an ID would solve this.

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

Because the Editor data model provides a centralized source of truth for all editing systems,
this enables building a unique undo/redo system and a unique copy/paste system shared by all other editing systems,
without the need for each of those editing systems to design their own.

I expect the following future RFCs to build on top of this one:

- Game/Editor communication (IPC)
- Undo/redo system
- Copy/paste system
- Live editing (update Game data in real-time while editing in Editor)
- Remote collaboration (multiple users editing the same project collaboratively from multiple remote locations via networking)

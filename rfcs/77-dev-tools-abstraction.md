# Feature Name: `dev-tools-abstraction`

Key questions:

- what are the defining features of a dev tool?
  - eases Bevy development experience, typically be presenting information about the app or its function
  - not intended for end-user use
    - must be able to be disabled at compile time
  - contextually useful based on encountered weirdness in the app
    - must be able to be toggled on and off at runtime without a reboot
  - should not interfere with normal operation of the app
  - can be customized to meet the needs of both the app and the developer
- what are clear examples of dev tools that we might want?
  - system graph visualizer
  - bevy_inspector_egui
  - fly camera
  - UI node visualizer
  - FPS display
  - resetting a level
  - toggling god mode
- who creates dev tools?
  - Bevy itself: FPS display
  - third-party crates: bevy_rapier collider overlay
  - end users: toggle god mode
- who creates dev tool consumers?
  - Bevy itself: scene editor
  - third-party crates: dev console
  - end users: custom level editors
- how might dev tools be toggled and consumed?
  - Quake-style dev console
  - unified set of hot keys
  - menu bar from scene editor
- how might dev tools be displayed?
  - screen overlay
  - pop-out window
  - embedded panel widget
  - cli
- how might dev tools be configured?
  - color palettes
  - fonts
  - font size
  - screen location
  - dev tool specific features: camera speed, entities to ignore etc
- why do dev tools need to be aware of each other?
  - overlays can clash: toggle off current overlay before enabling next
- how can we make it easy for third-party consumers and producers to play nice together?
  - Cart's suggestion: let each consumer decide which tools it wants to use
    - creates a quadratic problem: must wire up plumbing separately for each tool and consumer
  - define a standard API for available dev tools and essential operations
    - existing tools will have to conform to this in some way

Out of scope:

- [where should dev tools live](https://github.com/bevyengine/bevy/pull/12354)?
  - where the primitives used are defined? e.g. bevy_gizmos
  - where they're used for debugging? e.g. bevy_sprite
  - in bevy_dev_tools?
  - in dedicated crates, saving bevy_dev_tools for higher level abstractions?
- should dev tools be embedded in each application or should testing be done through an editor which controls these dev tools?

## Summary

One paragraph explanation of the feature.

## Motivation

[`bevy_dev_tools`](https://github.com/bevyengine/bevy/tree/main/crates/bevy_dev_tools) was recently added to Bevy, giving us a home for first-party tools to ease the developer experience.
However, since its inception, it has been plagued by debate over ["what is a dev tool?"](https://github.com/bevyengine/bevy/issues/12358)
and every **tool** within both Bevy and its ecosystem follows its own ad hoc conventions about [how it might be enabled and configured](https://github.com/bevyengine/bevy/issues/12368).

This is frustrating to navigate as an end user: every tool has its own way to configure it, with no rhyme, reason or way to set the same property to the same value across tools.
It also makes the creation of **toolboxes**, interfaces designed to collect multiple dev tools in a single place (such as a Quake-style dev console)
needlessly painful, requiring manual glue work by either the toolbox creator or application developer for every new tool that's added to it.

This problem is particularly nasty when combined with the orphan rule: Rust's restriction that traits can only be implemented in crates that own either the type or the trait.
Suppose that `bevy_xr_dev_console` (a hypothetical promising new toolbox crate!) requires its own `XrConsoleCommand` trait on the config for each of the dev tools that they want to add to their XR game.
If we want to integrate this with beloved existing tools like `bevy_inspector_egui`, we need to special-case either the tool or the toolbox in the third-party crates.
There's no path forward for the end users who wants to create the simple unified interface they want.
Instead, for every new tool, they have to newtype the existing configuration, and then add systems to carefully synchronize the settings.
By providing a standard interface in Bevy itself, we can ensure the ecosystem plays nicely together.

In some cases, tools can actively interfere with the function of other tools.
A prime example of this are text-based overlays, like you might see in an FPS meter.
Toggling one tool might completely cover (or distort the layout) of another, and so some level of coordination is required for a smooth user experience.

## User-facing explanation

**Dev tools** are an eclectic collection of features used by developers to inspect and manipulate their game at runtime.
These arise naturally to aid debugging and testing: setting up test scenarios.
Some examples include:

- displaying the FPS or other performance characteristics
- an entity inspector like `bevy_inspector_egui`, to understand the state of the `World`
- a UI node visualizer, to debug issues with layout
- a fly camera, to examine the environment around the player avatar
- system stepping, to walk through the evaluation of system logic
- adding items to the player's inventory
- warping to a given level
- toggling god mode

Some tools, while useful for development, *don't* match this pattern!
For example:

- a system graph visualizer like [`bevy_mod_debugdump`](https://github.com/jakobhellermann/bevy_mod_debugdump): this doesn't rely on information about a running game
- the [`perfetto`](https://ui.perfetto.dev/) performance profiler or a time travel debugger like [rerun-io's Revy](https://github.com/rerun-io/revy/tree/main): these relies on logs created, although in-process equivalents could be built
- a scene editor, pixel art tool or audio workstation: these are used to create assets, and do not need to be accessible at runtime
- gizmos: while these are commonly used for *making* dev tools, they don't inherently provide a way to inspect or manipulate the game world

As you can see, these tools are currently and will continue to be created by the creators of indidvidual games, third-party crates in Bevy's ecosystem, and Bevy itself.
Looking at the usage examples more closely, we can classify them into two patterns: **modal dev tools** and **dev commands**.

- Modal dev tools (like an FPS meter): can be toggled off and on, and want persistent configuration.
- Dev commands (like adding items): immediately alter the game world

As a result, we require two distinct (but inter-related) abstractions to handle these patterns.

Regardless of this classification, dev tools are:

- not intended to be shipped to end users
  - as a result, they can be enabled or disabled at compile time, generally behind a single `dev` flag in the application
- highly application specific: the tools needed for a 3D FPS, a 2D tilemap game, or a puzzle game are all very different
  - adding these to your game should be done granularly at compile time
- enabled or operated one at a time, on an as-needed basis, when problems arise
  - toggling dev tools needs to be done at runtime: without recompiling or restarting the game
- intended to avoid interfering with the games ordinary operation, especially when disabled
  - keybindings must be configurable to avoid conflicting with actual gameplay
  - while off, performance costs should be low or zero
- expansive and ad hoc: it's common to end up with dozens of assorted dev tools in a finished game, added to help solve specific problems as they occur
  - some structure is generally needed to present these options to the developer, and avoid ovewhelming them with dozens of chorded hotkeys

In order to manage the complexity of an eclectic collection of one-off debugging tools, dev tools are commonly organized into a **toolbox**: an abstraction used to provide a unified interface over the available dev tools.
This might be:

- an in-game dev console (like in Quake)
- special developer commands entered into the chat box, commonly with a `/` to denote their non-textual effect
- cheat codes
- a unified set of hotkeys
- game editor widgets

Regardless of the exact interface used, toolboxes have a few shared needs:

- turn on and off various modes
- ensure that the active modes don't interfere with each other
- execute one-time commands (like spawning enemies or setting the player's gold)
- list the various options for the user, ideally with help text

### Toggling modes: `ModalDevTool`

In order to facilitate the creation of toolboxes, Bevy provides the `ModalDevTool` trait.

```rust
/// Modal dev tools are used by developers to inspect their application in a toggleable way,
/// such as an FPS meter or a fly camera.
/// 
/// Their configuration is stored as a resource (the type that this trait is implemented for),
/// and they can be enabled, disabled and reconfigured at runtime.
/// 
/// The documentation on this struct is reflected, and can be read by toolboxes to provide help text to users.
trait ModalDevTool: Resource + Reflect {
    /// Turns this dev tool on (true) or off (false).
    fn set_enabled(&mut self, enabled: bool);

    /// Is this dev tool currently enabled?
    fn is_enabled(&self) -> bool;

    /// Enables this dev tool.
    fn enable(&mut self) {
        self.toggle(true);
    }

    /// Disables this dev tool.
    fn disable(&mut self) {
        self.toggle(false);
    }

    /// Enables this dev tool if it's disabled, or disables it if it's enabled.
    fn toggle(&mut self){
        if self.is_enabled(){
            self.disable();
        } else {
            self.enable();
        }
    }
}
```

Modal dev tools are registered via `app.init_modal_dev_tool::<D>()` (to use default config based on the `FromWorld` implementation) or via `app.insert_modal_dev_tool(config: D)`.
This adds them as a resource (just like ), but also registers their `ComponentId` with the central `DevToolsRegistry`, which can be consumed by toolboxes to get a list of the available dev tools.

To build your own modal dev tool, simply create a configuration resource, implement the `ModalDevTool` trait, and then register it in the app when the correct feature flag is enabled.

```rust
/// A flying camera controller that lets you disconnect your camera from the player to freely explore the environment.
/// 
/// When this mode is disabled
#[derive(Resource, Reflect)]
struct DevFlyCamera {
    enabled: bool,
    /// How fast the camera travels forwards, backwards, left, right, up and down, in world units.
    movement_speed: f32,
    /// How fast the camera turns, in radians per second.
    turn_speed: f32,
}

impl ModalDevTool for DevFlyCamera {
    fn set_enabled(&mut self, enabled: bool) {
        self.enabled = enabled;
    }

    fn is_enabled(&self) -> bool {
        self.enabled
    }
}
```

To configure a dev tool at runtime, simply access the resource, and either mutate or overwrite the `ModalDevTools` struct.

### Immediate dev tools

Immediate dev tools modify the world a single time, and can be called with arguments.
To model this, we leverage Bevy's existing `Command` trait, which exists to perform complex one-off modifications to the world.

```rust
/// Dev commands are used by developers to modify the `World` in order to easily debug and test their application.
/// 
/// Dev commands can be called with arguments to specify the exact behavior: if you are creating a toolbox, parse the provided arguments 
/// to construct an instance of the type that implements this type, and then send it as a `Command` to execute it.
/// 
/// The documentation on this struct is reflected, and can be read by toolboxes to provide help text to users.
trait DevCommand: Command + Reflect;

/// `DevCommand` uses a blanket implementation over all appropriate commands, as it has no special requirements.
impl <C: Command + Reflect> DevCommand for C {}
```

Creating your own dev command is simple! Create a struct for any config, implement `Command` for it and then register it using `app.register_dev_command::<C>()`, making it available for various toolboxes to inspect and send.

```rust
/// Sets the player's gold to the provided value.
#[derive(Reflect)]
struct SetGold {
    amount: u64,
}

impl Command for SetGold {
    fn apply(&self, world: &mut World){
        let mut current_gold = world.resource_mut::<Gold>();
        current_gold.0 = self.amount;
    }
}
```

### Building toolboxes

TODO

### Conventions for building dev tools

Not everything can or should be defined by a trait! To supplement the `DevTool` trait, we recommend that Bevy and its ecosystem follow the following conventions:

1. Dev tools can be toggled at compile time.
2. Dev tools can be toggled at run time.
3. Dev tools implement the `DevTool` trait, and if the corresponding feature is enabled, are added to the app via `.init_dev_tool` or `.insert_dev_tool`.
   1. This can be done either in the main plugin or a dedicated dev tools plugin.
   1. If configuration is required, splitting this into a dedicated dev tools plugin is preferred.
4. Dev tools are disabled by default, both at compile and run-time.
5. Each dev tools should be configured independently via a single resource.

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

### What is a `DevToolsRegistry`?

TODO

## Drawbacks

1. Third-party and end user dev tools will be pushed to conform to this standard. Without the use of a toolbox, this is added work for no benefit.
2. This abstraction may not fit all possible tools and toolboxes. The manual wiring approach is more flexible, and so if our abstraction is overly prescriptive, it may not work correctly.

## Rationale and alternatives

### Why should we use commands rather than one-shot systems for immediate dev tools?

Immediate dev tools commonly require input from the developer to take effect: what level to warp to, what item to spawn, what to set the player's gold to.
This pattern is very straightforward with commands, and quite complex to do with one-shot systems, requiring an associated type for the input.

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Related but out-of-scope questions:

- [where should dev tools live](https://github.com/bevyengine/bevy/pull/12354#issuecomment-1982284605)?
  - where the primitives used are defined? e.g. bevy_gizmos
  - where they're used for debugging? e.g. bevy_sprite
  - in bevy_dev_tools?
  - in dedicated crates, saving bevy_dev_tools for higher level abstractions?
- should dev tools be embedded in each application or should testing be done through an editor which controls these dev tools?
  - this is one of the [key questions](https://github.com/bevyengine/bevy_editor_prototypes/discussions/1) for the bevy_editor efforts

Future possibilities:

1. Over time, we can extend the `ModalDevTool` trait with optional methods like `set_font`, enabling gradual unification of more complex shared configuration.

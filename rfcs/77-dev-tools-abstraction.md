# Feature Name: `dev-tools-abstraction`

## Summary

Dev tools are an essential but poorly-defined part of the game development experience.
Designed to allow developers (but generally not end users) to quickly inspect and manipulate the game world,
they accelerate debugging and testing by allowing common operations to be done at runtime, without having to write and then delete code.
To support a robust ecosystem of dev tools,
`bevy_dev_tools` comes with two core traits: `ModalDevTool` (for tools that can be toggled) and `DevCommand` (for operations with an immediate effect on the game world).
These are registered in the `DevToolsRegistry` via methods on `App`, allowing tool boxes (such as a dev console) to easily identify the available options and interact with them without the need for ad-hoc user glue code.

## Motivation

[`bevy_dev_tools`](https://github.com/bevyengine/bevy/tree/main/crates/bevy_dev_tools) was recently added to Bevy, giving us a home for first-party tools to ease the developer experience.
However, since its inception, it has been plagued by debate over ["what is a dev tool?"](https://github.com/bevyengine/bevy/issues/12358)
and every **tool** within both Bevy and its ecosystem follows its own ad hoc conventions about [how it might be enabled and configured](https://github.com/bevyengine/bevy/issues/12368).

This is frustrating to navigate as a Bevy user: every tool has its own way to configure it, with no rhyme, reason or way to set the same property to the same value across tools.
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
- a scene editor, pixel art tool, asset preprocessing solutions or audio workstation: these are used to create assets, and do not need to be accessible at runtime
- gizmos: while these are commonly used for *making* dev tools, they don't inherently provide a way to inspect or manipulate the game world

As you can see, these tools are currently and will continue to be created by the creators of individual games, third-party crates in Bevy's ecosystem, and Bevy itself.
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
- a command palette in an editor
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
trait ModalDevTool: Resource + Reflect + FromReflect + FromStr<Err=DevToolParseError> + Debug {
    /// The name of this tool, as might be supplied by a command line interface.
    fn name() -> &'static str {
        Self::type_name().to_snake_case()
    }
    
    fn short_description() -> Option<&'static str>;

    /// The metadata for this modal dev tool.
    fn metadata() -> DevToolMetaData {
        DevToolMetaData {
            name: Self::name(),
            type_id: Self::type_id(),
            type_info: Self::type_info(),
            // A function pointer, based on the std::str::from_str method
            from_str_fn: <Self as FromStr>::from_str,
            short_description: Self::short_description()
        }
    }

    /// Turns this dev tool on (true) or off (false).
    fn set_enabled(&mut self, enabled: bool);

    /// Is this dev tool currently enabled?
    fn is_enabled(&self) -> bool;

    /// Enables this dev tool.
    fn enable(&mut self) {
        self.set_enabled(true);
    }

    /// Disables this dev tool.
    fn disable(&mut self) {
        self.set_enabled(false);
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
This adds them as a resource (just like the familiar `.init_resource` or `insert_resource`), but also registers their `ComponentId` with the central `DevToolsRegistry`, which can be consumed by toolboxes to get a list of the available dev tools.

To build your own modal dev tool, simply create a configuration resource, implement the `ModalDevTool` trait, and then register it in the app when the correct feature flag is enabled.

```rust
/// A flying camera controller that lets you disconnect your camera from the player to freely explore the environment.
/// 
/// When this mode is disabled
#[derive(Resource, Reflect, Debug)]
struct DevFlyCamera {
    enabled: bool,
    /// How fast the camera travels forwards, backwards, left, right, up and down, in world units.
    movement_speed: Option<f32>,
    /// How fast the camera turns, in radians per second.
    turn_speed: Option<f32>,
}

impl Default for DevFlyCamera {
    fn default() -> Self {
        DevFlyCamera {
            enabled: false,
            movement_speed: 3.,
            turn_speed: 10.
        }
    }
}

impl ModalDevTool for DevFlyCamera {
    fn set_enabled(&mut self, enabled: bool) {
        self.enabled = enabled;
    }

    fn is_enabled(&self) -> bool {
        self.enabled
    }
}

impl FromStr for DevFlyCamera {
    fn from_str(s: &str) -> Result<Self, DevToolParseError>{
        let parts = s.split_whitespace();
        let name = parts.iter.next()?;
        if name != self.name() {
            return Err(DevToolParseError::InvalidName);
        }

        let movement_speed: f32 = match parts.iter.next() {
            Some(string) = parse(string)?,
            None => Self::default().movement_speed,
        };

        let turn_speed: f32 = match parts.iter.next() {
            Some(string) = parse(string)?,
            None => Self::default().turn_speed,
        };

        Ok(DevFlyCamera {
                enabled: true,
                movmeent_speed,
                turn_speed,
            }
        )
    }
}
```

To configure a dev tool at runtime, simply access the resource, and either mutate or overwrite the `ModalDevTools` struct.

### Dev commands

Dev commands modify the world a single time, and can be called with arguments.
To model this, we leverage Bevy's existing `Command` trait, which exists to perform complex one-off modifications to the world.

```rust
/// Dev commands are used by developers to modify the `World` in order to easily debug and test their application.
/// 
/// Dev commands can be called with arguments to specify the exact behavior: if you are creating a toolbox, parse the provided arguments 
/// to construct an instance of the type that implements this type, and then send it as a `Command` to execute it.
/// 
/// The documentation on this struct is reflected, and can be read by toolboxes to provide help text to users.
trait DevCommand: Command + Reflect + FromReflect + FromStr<Err=DevToolParseError> + Debug + 'static {
    /// The name of this tool, as might be supplied by a command line interface.
    fn name() -> &'static str {
        Self::type_name().to_snake_case()
    }

    /// The metadata for this dev command.
    fn metadata() -> DevCommandMetadata {
        DevCommandMetadata {
            name: self.name(),
            type_id: Self::type_id(),
            type_info: Self::type_info(),
            // A function pointer, based on the std::str::from_str method
            from_str_fn: <Self as FromStr>::from_str
        }
    }
}
```

Creating your own dev command is straightforward! Create a struct for any config, implement `Command` for it and then register it using `app.register_dev_command::<C>()`, making it available for various toolboxes to inspect and send.

```rust
/// Sets the player's gold to the provided value.
#[derive(Reflect, FromReflect, Debug)]
struct SetGold {
    amount: u64,
}

impl Command for SetGold {
    fn apply(&self, world: &mut World){
        let mut current_gold = world.resource_mut::<Gold>();
        current_gold.0 = self.amount;
    }
}

impl DevCommand for SetGold {}

impl FromStr for SetGold {
    fn from_str(s: &str) -> Result<Self, DevToolParseError>{
        let parts = s.split_whitespace();
        let name = parts.iter.next()?;
        if name != self.name() {
            return Err(DevToolParseError::InvalidName);
        }

        let amount_string = parts.iter.next()?;
        let amount = amount_string.parse()?;

        Ok(SetGold {amount} )
    }
}
```

### Conventions for building dev tools

Not everything can or should be defined by a trait! To supplement the `ModalDevTool` trait, we recommend that Bevy and its ecosystem follow the following conventions:

1. Dev tools can be toggled at compile time.
2. Dev tools can be toggled at run time.
3. Dev tools implement the `ModalDevTool` trait, and if the corresponding feature is enabled, are added to the app via `.init_dev_tool` or `.insert_dev_tool`.
   1. This can be done either in the main plugin or a dedicated dev tools plugin.
   1. If configuration is required, splitting this into a dedicated dev tools plugin is preferred.
4. Dev tools are disabled by default, both at compile and run-time.
5. Each dev tools should be configured independently via a single resource.
6. If a meaningful defaults exist for fields on dev commands, that field should be modelled as an `Option`.
   1. This allows CLI-style toolboxes to more easily populate your struct.

### Building toolboxes

Let's take a look at how this all fits together by attempting to build a Quake-style dev console, much like [`bevy-console`](https://github.com/RichoDemus/bevy-console).
Obviously, this is too big to cover properly in an RFC example, so we're going to hand-wave the user input side of things by showing how you might deal with the parsed events, and simplify output by just logging to `println!`.

Our first task is getting the list of dev tools.

```rust
#[derive(Event)]
struct ListDevTools;

fn list_dev_tools(mut event_reader: EventReader<ListDevTools>, dev_tools_registry: Res<DevToolsRegistry>){
    // Clear the events, and act if at least one is found
    if event_reader.drain().next().is_some() {
        println!("Available modal dev tools:");
        for (_component_id, modal_dev_tool) in dev_tools_registry.iter_modal_dev_tools() {
            println!("{}", modal_dev_tool.type_name());
            println!("{}", modal_dev_tool.docs());

            for field in modal_dev_tools.fields(){
                println!("{}", field.type_name());
                println!("{}", field.docs())
            }
        }
    }
}
```

Next, we want to be able to toggle each modal tool by name.

```rust
#[derive(Event)]
struct ToggleDevTool{
    name: String,
}

fn toggle_dev_tools(world: &mut World){
    // Move the events out of the world, clearing them and avoiding a persistent borrow
    let events = world.resource_mut::<Events<ToggleDevTool>>().drain();

    // Use a resource scope to allow us to access both the dev tools registry and the rest of the world at the same time
    world.resource_scope(|(world, dev_tools_registry: Mut<DevToolsRegistry>)|{
        for event in events {
            // This gives us a mutable reference to the underlying resource as a `&mut dyn ModalDevTool`
            let Some(dev_tool) = dev_tools_registry.get_tool_by_name(world, event.name) else {
                warn!("No dev tool was found for {}). Did you forget to register it?", event.name);
                continue;
            };

            // Since we know that this object always implements `ModalDevTool`,
            // we can use any of the methods on it, or the traits that it requires (like `Reflect`)
            dev_tool.toggle();
        }
    })
}


```

Next, we want to be able to configure modal dev tools at run time.

```rust
#[derive(Event)]
struct ConfigureDevTool{
    tool_string: String,
};

fn configure_dev_tools(world: &mut World){
    // Move the events out of the world, clearing them and avoiding a persistent borrow
    let events = world.resource_mut::<Events<ToggleDevTool>>().drain();

    // Use a resource scope to allow us to access both the dev tools registry and the rest of the world at the same time
    world.resource_scope(|(world, dev_tools_registry: Mut<DevToolsRegistry>)|{
        for event in events {
            // Check the implementation details for information about how this works!
            let result = dev_tools_registry.parse_and_insert_tool(event.tool_string);
            if let Err(error) = result {
                warn!(error);
            }
        }
    })}
```

Finally, we want to be able to pass in a user supplied string, parse it into a dev command and then evaluate it on the world.

```rust
#[derive(Event)]
struct DevCommandInput(String);

fn parse_and_run_dev_commands(world: &mut World){
    // Move the events out of the world, clearing them and avoiding a persistent borrow
    let events = world.resource_mut::<Events<ToggleDevTool>>().drain();

    // Use a resource scope to allow us to access both the dev tools registry and the rest of the world at the same time
    world.resource_scope(|(world, dev_tools_registry: Mut<DevToolsRegistry>)|{
        for event in events {
            // This gives us access to the metadata needed to inspect the dev command and construct a new one.
            let Some(dev_command_metadata) = dev_tools_registry.get_command_metadata_by_name(world, event.name) else {
                warn!("No dev tool was found for {}). Did you forget to register it?", event.name);
                continue;
            };

            // Create a concrete instance of our dev command from the supplied string.
            let Ok(command) = dev_command_metadata.from_str(&event.0) else {
                warn!("Could not parse the command from the supplied string");
                continue;
            }

            // Now we can run the command directly on the `World` using `Command::apply`!
            command.apply(world);
        }
    })
}
```

While a number of other features could sensibly be added to this API (a `--help` flag, saving and loading config to disk, managing compatibility between dev tools),
this MVP should be sufficient to prove out the viability of the core architecture.

## Implementation strategy

### What metadata do toolboxes need?

We can start simple:

- the type name of the dev tool
- any doc strings on the dev tool struct
- the value and types of any configuration that must be supplied

By relying on simple structs to configure both our modal dev tools and commands, we can extract this information automatically, via reflection.
Additional dev tool specific metadata (such as a classification scheme) can be added to the traits on an opt-in basis in the future.

Optional methods on both `ModalDevTool` and `DevCommand` will allow us to override the supplied defaults if needed.

We also need access to one other critical piece of information: a function pointer that allows us to construct a new value of this type from a string.

As a result our `DevToolMetadata` looks like:

```rust
struct DevToolMetadata {
   name: String,
   type_info: TypeInfo,
   from_str: StringConstructorFn,
}
```

The tricky bit comes when we get to `StringConstructorFn`: we need to be able to store a function pointer that will take us from a string, and return an object that implements our core traits.

TODO: how can this be done?

The definition for `DevCommandMetadata` is effectively identical to start, but over time we should expect these to diverge: splitting them from the beginning will ease migrations going forward.

### What do the registries for our dev tools look like?

Keeping track of the registered dev tools without storing them all in a dedicated collection is quite challenging!
How can we keep track of it under the hood?

```rust
#[derive(Resource)]
struct DevToolsRegistry {
    /// The stored collection of modal dev tools, tracked in a type-erased way using [`ComponentId`]
    /// 
    /// The key is the `name()` provided by the `ModalDevTool` trait.
    modal_dev_tools: HashMap<String, ToolMetaData>,
    /// The metadata for all registered dev commands.
    /// 
    /// The key is the `name()` provided by the `DevCommand` trait.
    dev_commands: HashMap<String, DevCommandMetaData> 
}
```

There are a few key operations that we will need to perform:

```rust
impl DevToolsRegistry {
    /// Gets a reference to the specified modal dev tool by `ComponentId`.
    fn get_tool_by_id(world: &World, component_id: ComponentId) -> Option<&dyn ModalDevTool> {
        let resource = world.get_resource_by_id(component_id)?;
        resource.downcast_ref()
    }

    /// Gets a mutable reference to the specified modal dev tool by `ComponentId`.
    fn get_tool_mut_by_id(mut world: &mut World, component_id: ComponentId) -> Option<&mut dyn ModalDevTool> {
        let resource = world.get_resource_mut_by_id(component_id)?;
        resource.downcast_mut()
    }

    /// Gets the `DevCommandMetadata` for a given dev tool by name.
    /// 
    /// The supplied name should match the `DevCommand::name()` method.
    fn get_command_metadata(name: &str) -> Option<&DevCommandMetadata> {
        self.dev_commands.get(name)
    }

    /// Gets the `DevToolMetadata` for a given dev tool by name.
    /// 
    /// The supplied name should match the `DevCommand::name()` method.
    fn get_tool_metadata(name: &str) -> Option<&DevCommandMetadata> {
        self.modal_dev_tools.get(name)
    }

    /// Looks up the `ComponentId` associated with the given name, as supplied by the `ModalDevTool` trait.
    fn lookup_tool_component_id(&self, name: &str) -> Option<ComponentId> {
       *self.get_tool_metadata(name)?.component_id
    }

    /// Gets a reference to the specified modal dev tool by type name, as supplied by the `ModalDevTool` trait.
    fn get_tool_by_name(&self, world: &World, name: &str) -> Option<&dyn ModalDevTool> {
        let component_id = self.lookup_tool_component_id(name)?;
        DevToolsRegistry::get_by_id(world, component_id)
    }

    /// Gets a mutable reference to the specified modal dev tool by name.
    fn get_tool_mut_by_name(&self, mut world: &mut World, component_id: ComponentId) -> Option<&mut dyn ModalDevTool> {
        let component_id = self.lookup_tool_component_id(name)?;
        DevToolsRegistry::get_mut_by_id(world, component_id)
    }
    
    /// Iterates over the list of registered modal dev tools.
    fn iter_tools(&self, world: &World) -> impl Iterator<Item = (ComponentId, &dyn ModalDevTool)> { 
        self.modal_dev_tools.iter().map(|&id| (id, self.get(world, id)))
    }

    /// Mutably iterates over the list of registered modal dev tools.
    fn iter_tools_mut(&self, mut world: &mut World) -> impl Iterator<Item = (ComponentId, &mut dyn ModalDevTool)> { 
        self.modal_dev_tools.iter_mut().map(|&id| (id, self.get(world, id)))
    }

    /// Iterates over the list registered dev commands, returning their name and `DevCommandMetadata`.
    fn iter_commands(&self) -> impl Iterator<Item = (&str, `DevCommandMetadata)> {
        self.dev_commands.iter()
    }
}
```

Once the user has a `dyn ModalDevTool`, they can perform whatever operations they'd like: enabling and disabling it, reading metadata and so on.
There's no need to add further duplicate API: users building toolboxes are generally sophisticated, and the number of calls should be quite small.

While we can use the [dynamic resource APIs](https://docs.rs/bevy/latest/bevy/ecs/world/struct.World.html#method.get_resource_by_id) and trait casting to get references to the list of modal dev tools, we cannot do the same for the dev commands!
While this may seem unintuitive, the reason for this is fairly simple: modal configuration is always present, while the configuration for each dev command is generated when it is sent.
As a result, we can only access its metadata: the arguments it takes, its doc strings and so on.

### Parsing and inserting modal dev tools

One method on `DevToolsRegistry` is worth special attention: `parse_and_insert_tool`.
This tool takes a common pattern, parsing the configuration for a dev tool from a provided string, and provides a user friendly, safe API over it.

```rust
impl DevToolsRegistry {
    /// Parses the given str `s` into a modal dev tool corresponding to its name if possible.
    ///
    /// For a typed equivalent, simply use the `FromStr` trait that this method relies on directly.
    fn parse_and_insert_tool(&self, world: &mut World, s: &str) -> Result<(), DevToolParseError>{
        // Parse out the name
        let name = s.clone().split_whitespace.next()?;
        
        // Get the associated `ComponentId`, so we can use it to insert a resource of a dynamic type
        let component_id = self.lookup_tool_component_id(name);
        // Look-up the existing resource to get access to get access to the metadata we need
        let tool_metadata: &dyn DevTool = self.get_tool_metadata(component_id)?;   
        // Parse the string into a new copy of the tool using the stored function pointer
        let new_tool = tool_metadata.from_str(s);
        // Construct an `OwningPointer` so we can dynamically insert the resource we just made
        OwningPointer::make(new_tool, |owning_pointer|{
            // SAFETY: the value referenced by value is valid for the given `ComponentId` of this world,
            // as the component id is cached upon initialization of the resource / dev tool.
            unsafe {
                world.insert_resource_by_id(component_id, owning_pointer);
            }
        });

        Ok(())
    }
}
```

## Drawbacks

1. Third-party and end user dev tools will be pushed to conform to this standard. Without the use of a toolbox, this is added work for no benefit.
2. This abstraction may not fit all possible tools and toolboxes. The manual wiring approach is more flexible, and so if our abstraction is overly prescriptive, it may not work correctly.
3. The `from_str` methods are currently quite heavy on boilerplate. A `clap`-inspired derive macro would be lovely here. Perhaps a crate already exists?
4. How can we store type-erased `StringConstructorFn`s in our metadata?

## Rationale and alternatives

### Why should we use commands rather than one-shot systems for immediate dev tools?

Immediate dev tools commonly require input from the developer to take effect: what level to warp to, what item to spawn, what to set the player's gold to.
This pattern is very straightforward with commands, and quite complex to do with one-shot systems, requiring an associated type for the input.

### Why should we store the configuration in resources?

As [Cart pointed out](https://github.com/bevyengine/bevy/pull/12427#issuecomment-2005328288), using a dedicated collection to store information about the state of various dev tools has several drawbacks:

- unidiomatic: these are currently configured via stand-alone resources, and changing that would require either synchronization or the imposition of this pattern on every tool that could be used as a dev tool
- breaks the granularity of change detection: we can't determine *which* dev tool's config has changed
- accessing information about the current config of a dev tool requires an added layer of indirection, and will simply fail if the dev tool was not registered, even in cases where no toolbox is being used

While this makes our internals more complex, that's a trade-off we're willing to accept.

### Why not use trait queries?

If we need to access a set of objects, all of which implement the same trait, why not simple use [trait queries](https://github.com/bevyengine/rfcs/pull/39).

There are two arguments against this:

1. While [`bevy_trait_query`] exists,  it is currently third-party, and upstreaming it would be a major initiative.
2. This config is idiomatically stored in resources, not singleton entities.
3. This still doesn't solve the problem of how to track the set of registered dev commands.

### Why not add a `Display` bound to `ModalDevTool` and `DevCommand`?

We want to be able to directly display their value to users in a generic way: why not rely on the `Display` trait?

There are three arguments against this:

1. Implementing `Display` is relatively tedious due to string formatting.
2. We can get good enough results to print via `Reflect` and `Debug`.
3. Toolkits will typically rely on their own custom UI for formatting this information, and need more structured data than a simple string.

### Why not add information about layout to our traits?

This sort of information is really helpful for toolboxes like editors to ensure that dev tools don't overlap each other: allowing them to be laid out beside each other or enabled one at a time when necessary.

However:

1. This RFC is complex enough: keeping its scope small will make it easier to review and implement.
2. Unlike with a CLI-style interface the correct abstractions aren't immediately obvious: we should prototype more toolkits and tools to explore the needs of the space further.
3. Not all dev tools need layout information: we would need to use optional methods and/or subtraits to fill this need, adding further complexity.

## Unresolved questions

1. Are the provided methods (along with those on `Reflect`) sufficient to allow toolkits to populate and run a wide range of dev tools without requiring an additional trait and manual registration?
2. Is the use of `downcast_ref` / `downcast_mut` correct in `DevToolsRegistry::get`?
3. Are `ModalDevTool` and `DevCommand` the best names?
4. How, precisely, can we achieve the sort of API shown in [Building toolboxes] using reflection?
   1. A prototype would be great here.
5. Does the function pointer approach used by `DevCommandMetadata` compile and work successfully?
   1. Again, prototype please!

## Future possibilities

Related but out-of-scope questions:

- [where should dev tools live](https://github.com/bevyengine/bevy/pull/12354#issuecomment-1982284605)?
  - where the primitives used are defined? e.g. bevy_gizmos
  - where they're used for debugging? e.g. bevy_sprite
  - in bevy_dev_tools?
  - in dedicated crates, saving bevy_dev_tools for higher level abstractions?
- should dev tools be embedded in each application or should testing be done through an editor which controls these dev tools?
  - this is one of the [key questions](https://github.com/bevyengine/bevy_editor_prototypes/discussions/1) for the bevy_editor efforts

Future possibilities:

1. Over time, we can extend the `ModalDevTool` and `DevCommand` traits with more methods (mandatory and optional), enabling:
   1. Gradual unification of more complex shared configuration (e.g. a `set_font` method).
   2. Serializing and deserializing structs from a `.bsn` file, rather than simple CLI-style strings, allowing toolboxes to build abstractions for persistent config.
2. Add support for a trait-queries-on-resources operation, and stop storing modal dev tools in the `DevToolsRegistry`.
3. A derive macro for `DevCommand`, that aligns with `clap`'s calling conventions.
4. Add specialized types of dev tools, likely each with their own trait to establish conventions and resolve conflicts between them.
    1. For example, we may want to coordinate between various UI elements or overlays to avoid positioning conflicts, dev tools that change the materials of objects, or camera controllers. See [Discussion #12589](https://github.com/bevyengine/bevy/discussions/12589) for some initial exploration.
    2. This could be queried via Opt-in metadata about the type of tool they are (e.g. an overlay, a camera controller, a 2D vs. 3D tool), allowing toolkits to seamlessly organize the options.

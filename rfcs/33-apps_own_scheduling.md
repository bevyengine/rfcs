# Feature Name: `apps_own_scheduling`

## Summary

All scheduling information is attached directly to systems, and ultimately owned and configurable by the central app (or hand-written `bevy_ecs` equivalent).
To ensure that this can be done safely, plugins specify invariants about the ordering of their systems which must be respected.
Hard ordering constraints (where systems must be both ordered and separated by a hard sync point) are introduced as an assert-style protection to guard against a serious and common form of plugin misconfiguration.

## Motivation

The current plugin model works well in simple cases, but completely breaks down when the central consumer (typically, an `App`) needs to configure when and if their systems run. The main difficulties are:

- 3rd party plugins can have "unfixable" ordering ambiguities or interop issues with other 3rd party plugins
- the systems from plugins can not be added to states, and run criteria cannot be added to them
- plugins can not be incorporated into apps and games that use a non-standard control flow architecture
- end users who care about granular scheduling control for performance optimization have no way to change when systems run
- worse, plugins can freely insert their own stages, causing sudden and uncontrollable bottlenecks
- plugins cannot be used by standalone `bevy_ecs` users

See [Bevy #2160](https://github.com/bevyengine/bevy/issues/2160) for more background on this problem.

By moving the ultimate responsibility for scheduling configuration to the central app, users can comfortably combine plugins that were not originally designed to work together correctly and fit the logic into their particular control flow.

## User-facing explanation

### Plugins 101

**Plugins** are cohesive, drop-in units of functionality that you can add to your Bevy app.
Most of what plugins do is the result of **systems** that they add to your app.
Plugins can also initialize **resources**, which store data outside of the entity-component data store in a way that can be accessed by other systems.

The `Plugin` struct stores this data in a straightforward way:

```rust
pub struct Plugin {
  pub systems: HashMap<Box<dyn SystemLabel>, SystemSet>,
  pub resources: HashSet<Box<dyn Resource>>,
  // Other plugin-configuration settings ommitted here
}
```

Plugins can be added to your app using the `App::add_plugin` method.

```rust
use bevy::prelude::*;
use bevy_hypothetical_tui::prelude::*;

fn main(){
  App::minimal().add_plugin(tui_plugin()).run();
}
```

The best way to pass `Plugins` to the apps that consume them is by exporting a function that returns a `Plugin` struct with the desired system sets and resources from your crate or module.

```rust
fn combat_plugin() -> Plugin {
    Plugin::default()
      .add_systems("Combat", SystemSet::new().with(attack).with(death));
      .add_system("PlayerMovement", player_movement)
      .init_resouce::<Score>()
}

fn main(){
  App::default().add_plugin(combat_plugin()).run();
}
```

When designing plugins for a larger audience, you may want to make them configurable in some way to accommodate variations in logic that users might need.
Doing so is quite straightforward: you can simply add parameters to the plugin-creating function.
If you want the types that your plugin operates on to be configurable as well, add a generic type parameter to these functions.

```rust
// We may want our physics plugin to work under both f64 and f32 coordinate systems
trait Floatlike;

fn physics_plugin<T: Floatlike>(realistic: bool) -> Plugin {
  let mut plugin = Plugin::default().add_system(collision_detection::<T>);

  if realistic {
    plugin.add_system(realistic_gravity);
  } else {
    plugin.add_system(platformer_gravity);
  }

  plugin
}

fn main(){
  App::default()
    .add_plugin(physics_plugin::<f64>(true))
    .run();
}
```

### System ordering in Bevy

Bevy schedules are composed of stages, which have two phases: parallel and sequential.
The parallel phase of a stage always runs first, then is followed by a sequential phase which applies any commands generated in the parallel phase.

During **sequential phases**, each system can access the entire world freely but only one can run at a time.
Exclusive systems can only be added to the sequential phase while parallel systems can be added to either phase.

During **parallel phases**, systems are allowed to operate in parallel, carefully dividing up access to the `World` according to the data accesses requested by their system parameters to avoid undefined behavior.
By default, the only rule that their order follows is that a **waiting** system cannot be started if any **active** (currently running) systems are **incompatible** (cannot be scheduled at the same time) if they have conflicting data access.
This is helpful for performance reasons, but, as the precise order in which systems start and complete is nondeterministic, can result in logic bugs or inconsistencies.

To help you eliminate these sort of difficulties, Bevy lets you specify several types of **system ordering constraints**.
Ordering constraints can be applied directly to systems, on system sets, or to labels, but are ultimately stored and worked with on a per-system basis.
However, ordering constraints can only be applied *relative* to labels, allowing users to replace core parts of the engine (or other dependencies) by substituting new systems with the same public label.

- **Strict ordering:** Systems from set `A` cannot be started while any system from set `B` are waiting or active.
  - Simple and explicit.
  - Use the `before(label: impl SystemLabel)` or `after(label: impl SystemLabel)` methods to set this behavior.
- **If-needed ordering:** A given system in set `A` cannot be started while any incompatible system from set `B` is waiting or active.
  - Usually, if-needed ordering is the correct tool for ordering groups of systems as it avoids unnecessary blocking.
  - If systems in `A` use interior mutability, if-needed ordering may result in non-deterministic outcomes.
  - Use `before_if_needed(label: impl SystemLabel)` or `after_if_needed(label: impl SystemLabel)`.
- **At-least-once separation:** Systems in set `A` cannot be started if a system in set `B` has been started until at least one system with the label `S` has completed. Systems from `A` and `B` are incompatible with each other.
  - This is most commonly used when commands created by systems in `A` must be processed before systems in `B` can run correctly. `CoreLabels::ApplyCommands` is the label used for the exclusive system that applies queued commands that is typically added to the beginning of each sequential stage.
  - Use the `between(before: impl SystemLabel, after: impl SystemLabel)` method.
  - The order of arguments in `between` matters; if 
  - This methods do not automatically insert systems to enforce this separation: instead, the schedule will panic upon initialization as no valid system execution strategy exists.

### Configuring plugins

When you're creating Bevy apps, you can read, add, remove and configure both systems and resources stored within a plugin that you're using, allowing you to fit it into your app's needs using the standard `HashMap` and `HashSet` APIs.
The methods shown above (such as `.add_system`) are simply convenience methods that manipulate these public fields.

Note that each system set that you include requires its own label to allow other plugins to control their order relative to that system set, and allows the central `App` to further configure them as a unit if needed.

System sets don't store simple `Systems`: instead they store fully configured `SystemDescriptor` objects which store information about when and if the system can run. This includes:

- which stage a system runs in
- any system ordering dependencies a system may have
- any run criteria attached to the system
- which states the system is executed in
- the labels a system has

Without any configuration, these systems are designed to operate as intended when the Bevy app is configured in the standard fashion (using `App::default()`). Each system will belong to the standard stage that the plugin author felt fit best, and ordering between other members of the plugins and Bevy's default systems should already be set up for you.
For most plugins and most users, this strategy should just work out of the box.

When creating your `App`, you can add additional systems and resources directly to the app, or choose to remove system sets or resources from the plugins you are using by directly using the appropriate `HashSet` and `HashMap` methods.
Removing resources in this way can be helpful when you want to override provided initial values, or if you need to resolve a conflict where two of your dependencies attempt to set a resource's value in conflicting way.

More commonly though, you'll be configuring the system sets using the labels that they were given by their plugin.
When system sets are added to a `Plugin`, the provided label is applied to all systems within that set, allowing other systems to specify their order relative to that system set. Note that *all* systems that share a label must complete before the scheduler allows their dependencies to proceed.

The `App` can also configure systems provided by its plugins using the `App::configure_label` method.
There are four important limitations to this configurability:

1. Systems can only be configured as a group, on the basis of their public labels.
2. Systems can only be removed from plugins as a complete system set.
3. `configure_label` cannot remove labels from systems.
4. `configure_label` cannot remove ordering dependencies, only add them.

These limitations are designed to enable plugin authors to carefully control how their systems can be configured in order to avoid subtly breaking internal guarantees.
If you change the configuration of the systems in such a way that ordering dependencies cannot be satisfied (by creating an impossible cycle or breaking a hard system ordering rule by changing the stage in which a system runs), the app's schedule will panic when it is run.
Duplicate and missing resources will also cause a panic when the app is run.

### Writing plugins to be configured

When writing plugins (especially those designed for other people to use), try to define constraints on your plugins logic so that your users could not break it if they tried:

- if some collection of systems will not work properly without each other, make sure they are in the same system. This prevents them from being split by either schedule location or run criteria.
- if one system must run before the other, use granular strict system ordering constraints
- if a broad control flow must be follow, use if-needed system ordering constraints between labels
- if commands must be processed before a follow-up system can be run correctly, use at-least-once separation ordering constraints
- if a secondary source of truth (such as an index) must be updated in between two systems, use at-least-once separation ordering constraints

Exposing system labels via `pub` allows anyone with access to:

- set the order of their systems relative to the exposed label
- add new constraints to systems with that label
- change which states systems with that label will run in
- add new run criteria to systems with that label
- apply the labels to new systems

As such, only expose labels that apply to systems

### Standalone `bevy_ecs` usage

Even if you're just using `bevy_ecs`, plugins can be a useful tool for enabling code reuse and managing dependencies.

Instead of adding plugins to your app, independently add their system sets to your schedule and their resources to your world.

## Implementation strategy

Small changes:

1. As plugins no longer depend on `App` information at all, they should be moved into `bevy_ecs` directly.
2. To improve ergonomics, `App::add_system_set` should be changed to `App::add_systems`, and `SystemSet::with_system` should be shortened to `SystemSet::with`.
3. If plugins can no longer configure the `App` in arbitrary ways, we need a new, ergonomic way to set up the default schedule and runner for standard Bevy games. The best way to do this is to create two new builder methods on `App`: `App::minimal` and `App::default`, which sets up the default stages, sets the runner and adds the required plugins.
4. Plugin groups are replaced with simple `Vec<Plugin>>` objects when needed.

### Stageless architecture

1. System ordering between exclusive and parallel systems is defined, as is system ordering between systems that run in different stages.
2. All system scheduling information must be stored in the system descriptor (which are then stored within system sets) in order to enable a unified approach to configuration. See [Bevy #2736](https://github.com/bevyengine/bevy/pull/2736) for a nearly complete PR on this point.
3. `Commands` application is re-architected, and is now handled explicitly in an exclusive system.
This is by default the first system in each sequential stage.
This system is now labeled so systems can be before or after command application.
Users can freely add more copies of this system to their schedule.
4. Systems cannot run "between stages" anymore, and the current "should an exclusive system run at the start of the stage, before commands, or after commands" is removed in favor of simple ordering.

### System ordering mechanics

Under the hood, all three of the system ordering constraints collapse to the standard `.before`-style edges that we know and love in Bevy 0.5.

If-needed edges are evaluated and inserted on the basis of the existing entities at the start of each stage.

At-least-once-separation forces the `before` label to be before the intervening system, and the `after` system to be after the intervening system.
However, instead of counting the number of instances of that system with that label, only a single dependency is added to the [`dependency_count`.](https://github.com/bevyengine/bevy/blob/9eb1aeee488684ed607d249f883676f3e711a1d2/crates/bevy_ecs/src/schedule/executor_parallel.rs).

### Resource initialization

All resources are added to the app before the schedule begins. The app should panic if duplicate resources are detected at this stage; app users can manually remove them from one or more plugins in a transparent fashion to remove the conflict if needed.

### Label properties

Users must be able to configure systems by reference to a particular label. This proposal has previously been called **label properties**.

## Drawbacks

1. Plugins are no longer quite as powerful as they were. This is both good and bad: it reduces their expressivity, but reduces the risk that they mysteriously break your app.
2. Apps can directly manipulate the system ordering of their plugins. This is essential for ensuring that interoperability and configurability issues are resolved, but risks breaking internal guarantees.
3. Replacing existing resource values is more verbose, requiring either a startup system or explicitly removing the resource from the pre-existing plugin. This is much more explicit, but could slow down some common use cases such as setting window size.

## Rationale and alternatives

### Don't plugins need to be able to mutate the app in arbitrary ways?

After extensive experience with Bevy and review of community-made plugins, adding systems and adding resources seem to be the only functionality commonly used in plugins.

Direct world access is the other pattern commonly used. These usages can be directly replaced by startup systems which perform the same task in more explicit and controllable fashions.

If schedule and runner manipulation is ultimately required, this functionality can and should be added to exclusive systems instead.

### Why should we use a structured approach to plugin definition?

Taking a structured approach improves things by:

1. Enforcing principled rules about label exporting.
2. Clearly communicating to users how plugins should be used.
3. Decoupling the app and plugin API.
4. Avoids surprising and non-local effects caused by plugins.
5. Moves all top-level app-configuration into the main app-building logic, to encourage discoverable, well-organized code.

### Why are system sets the correct granularity to expose?

If we accept that apps must be allowed to configure when and if the systems in their plugins run, there are four possible options for how we could expose this:

1. Individual systems. These are too granular, and risk ballooning complexity and further breaking of plugin-internal guarantees.
2. System sets. The most flexible of options: allowing plugin authors to obscure or expose details as desired in order to encourage shared configuration. Configuration applied to a system set can only be applied to all systems in the set at once.
3. Stages. This heavily discourages system parallelism, and forces a global and linear view of the schedule in order to enforce the desired ordering constraints.
4. All systems. This is far too coarse, and will prevent users from changing which stage systems runs in as any reasonable configuration API will apply the same operation to every system in a group.

System sets are also the most flexible of these options: they can be lumped or split to fit the particular needs of the users and invariants required by the plugin's logic.

### Why are plugins forced to export a unique label for each system set?

One of the driving principles behind this design is that plugin authors cannot and should not predict every possible user need for their plugins.
This leads to either limited configurability, which forces dependency "vendoring" (maintaining a custom fork of the dependency) or internally developing the functionality needed, or excessive configurability, which bloats API surface in ways that are irrelevant to almost all users.

Exported system sets are the atomic unit of plugin configurability, and configuration cannot occur effectively without labels.
By forcing a one-to-one relationship between exported labels and system sets plugin authors have a clear standard to build towards that simplifies their decisions about what to make public.

### Why should we allow users to add dependencies to external plugin systems?

From a design perspective, this is essential to ensure that two third-party plugins can co-exist safely without needing to know about each other's existence in any way.

More humorously, we could *already* induce dependencies between labelled third-party systems.
Suppose we have two labelled systems: `A` and `C`, and want `A` to run before `C`.
The existing model is designed to prevent this level of meddling, and provides no way to configure the ordering of either system by any direct means.
However, if we insert a system `B` (which may do literally nothing!), and state that `B` runs after `A` and before `C`, we induce the desired dependency as system ordering is necessarily transitive.

There is no reasonable or desirable way to prevent this: instead we should embrace this by designing a real API to do so.

## Unresolved questions

1. Should we rename `SystemDescriptor` to something more user-friendly like `ConfiguredSystem`?

### Plugin type semantics

There are several decent options for how plugins should be defined.
Let's review two different sorts of plugins, one with no config and one with config, and see how the ergonomics compare.

All of these assume a standard `Plugin` struct, allowing us to have structured fields.

### Crates define functions that return `Plugin`

This is the simplest option, but looks quite a bit different from our current API.

```rust
mod third_party_crate {
  pub fn simple_plugin() -> Plugin {
    Plugin::default()
  }
  
  pub struct MyConfig(pub bool);

  pub fn configured_plugin(my_config: MyConfig) -> Plugin {
    Plugin::default().insert_resource(MyConfig(my_config))
  }
}

fn main(){
  App::new()
    .add_plugin(simple_plugin())
    .add_plugin(configured_plugin(MyConfig(true)))
    .run();
}
```

### Crates define structs that implement `Into<Plugin>`

Slightly more convoluted, but lets us pass in structs rather than calling functions.

If a user cares about allowing reuse of a particular plugin-defining struct, they can `impl Into<Plugin> for &ConfiguredPlugin`.

```rust
mod third_party_crate {
  pub struct SimplePlugin;

  impl Into<Plugin> for SimplePlugin {
    fn into(self) -> Plugin {
      Plugin::default()
    }
  }

  pub struct MyConfig(pub bool);

  pub struct ConfiguredPlugin {
    pub my_config: MyConfig, 
  }

  impl Into<Plugin> for ConfiguredPlugin {
    fn into(self) -> Plugin {
      Plugin::default().insert_resource(self.my_config)
    }
  }

  fn third_party_plugin() -> Plugin {
    Plugin::default()
  }
}

fn main(){
  App::new()
    .add_plugin(third_party_plugin())
    .run();
}
```

### Crates define structs that implement `MakePlugin`

Compared to the `Into<Plugin>` solution above, this would enable us to write our own derive macro for the common case of "turn all my fields into inserted resources".

On the other hand, this is less idiomatic and more elaborate.

```rust
mod third_party_crate {
  pub struct SimplePlugin;

  impl MakePlugin for SimplePlugin {
    fn make(self) -> Plugin {
      Plugin::default()
    }
  }

  pub struct MyConfig(pub bool);

  pub struct ConfiguredPlugin {
    pub my_config: MyConfig, 
  }

  impl MakePlugin for ConfiguredPlugin {
    fn make(self) -> Plugin {
      Plugin::default().insert_resource(self.my_config)
    }
  }

  fn third_party_plugin() -> Plugin {
    Plugin::default()
  }
}

fn main(){
  App::new()
    .add_plugin(third_party_plugin())
    .run();
}
```

## Future possibilities

This is a simple building block that advances the broader scheduler plans being explored in [Bevy #2801](https://github.com/bevyengine/bevy/discussions/2801).

This could be extended and improved in the future with:

1. Enhanced system visualization tools.
2. Hints to manually specify which hard sync points various systems run between.
3. A scheduler / app option could be added to automatically infer and insert systems (including command-processing systems) on the basis of at-least-once separation constraints.
4. At-least-once separation constraints can be used to solve the cache invalidation issues that are blocking the implementation of indexes, as discussed in [Bevy #1205](https://github.com/bevyengine/bevy/discussions/1205).

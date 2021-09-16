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

**Plugins** are cohesive, drop-in units of functionality that you can add to your Bevy app.
Plugins can initialize **resources** to store data to, or add **system sets**, used to add coherent batches of logic to your app.

Plugins can be added to your app using the `App::add_plugin` method.
If you're new to Bevy, third-party plugins added in this way should immediately work without further configuration.

```rust
use bevy::prelude::*;
use bevy_hypothetical_tui::prelude::*;

fn main(){
  App::minimal().add_plugin(tui_plugin()).run();
}
```

You can write your own plugins by exporting a function that returns a `Plugin` struct with the desired system sets and resources.
This can be provided as a standalone function or, more commonly, as a method on a simple `MyPlugin` struct:

```rust
fn combat_plugin() {
    Plugin::default()
      .add_systems("Combat", SystemSet::new().with(attack_system).with(death_system));
      .add_system("PlayerMovement", player_movement_system)
      .init_resouce::<Score>()
}

fn main(){
  App::default().add_plugin(combat_plugin()).run();
}
```

Under the hood, plugins are very simple, and their systems and resources can be created and configured directly.

The `Plugin` struct stores data in a straightforward way:

```rust
pub struct Plugin {
  pub systems: HashMap<Box<dyn SystemLabel>, SystemSet>,
  pub resources: HashSet<Box<dyn Resource>>,
}
```

When you're creating Bevy apps, you can read, add, remove and configure both systems and resources stored within a plugin that you're using, allowing you to fit it into your app's needs using the standard `HashMap` and `HashSet` APIs.
The methods shown above (such as `.add_system`) are simply convenience methods that manipulate these public fields.

Note that each system set that you include requires its own label to allow other plugins to control their order relative to that system set, and allows the central `App` to further configure them as a unit if needed.

### Hard system ordering rules

In addition to `.before` and `.after`, `.hard_before` and `.hard_after` can be used to specify that two systems must be separated by a hard sync point in the schedule (generally the end of the stage), in addition to following the standard ordering constraint.

This is critical when `Commands` from the first system must complete (e.g. spawning or despawning entities) before the second system can validly run.

Note that hard system ordering rules have no special schedule-altering powers!
They are merely an assertion to be checked at the time of schedule construction.
Their purpose is to ensure that you (or the users of your plugin) do not accidentally break the required logic by shuffling when systems run relative to each other.

### Configuring plugins

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

When writing plugins (especially those designed for other people to use), group systems that should be thought about in one coherent "unit of function" together into a single system set.

Be sure to use hard system ordering dependencies when needed to ensure that downstream consumers cannot accidentally break your system sets which require command processing by moving systems into the same stage.
**Systems which must live in separate stages due to hard system ordering rules must be in separate system sets in order to allow users to add them to their own stages freely.**

### Standalone `bevy_ecs` usage

Even if you're just using `bevy_ecs`, plugins can be a useful tool for enabling code reuse and managing dependencies.

Instead of adding plugins to your app, independently add their system sets to your schedule and their resources to your world.

## Implementation strategy

### Code organization

As plugins no longer depend on `App` information at all, they should be moved into `bevy_ecs` directly.

### System sets

The changes proposed are simple ergonomic renaming to reflect the increased prominence of system sets.
`App::add_system_set` should be changed to `App::add_systems`, and `SystemSet::with_system` should be shortened to `SystemSet::with`.

### Resources

All resources are added to the app before the schedule begins. The app should panic if duplicate resources are detected at this stage; app users can manually remove them from one or more plugins in a transparent fashion to remove the conflict if needed.

### Schedule overhaul prerequisites

1. Stage boundaries must be weakened. This API assumes that e.g. system sets can span hard sync points to allow them to be inserted into a given state and that ordering dependencies operate across hard sync points.
2. All system scheduling information must be stored in the system descriptor (which are then stored within system sets) in order to enable a unified approach to configuration. See [Bevy #2736](https://github.com/bevyengine/bevy/pull/2736) for a nearly complete PR on this point.
3. Users must be able to configure systems by reference to a particular label. This proposal has previously been called **label properties**.

### Default stages and runner

If plugins can no longer configure the `App` in arbitrary ways, we need a new, ergonomic way to set up the default schedule and runner for standard Bevy games.

The best way to do this is to create two new builder methods on `App`: `App::minimal` and `App::default`, which sets up the default stages, sets the runner and adds the required plugins.

### Plugin groups

Plugin groups now require the simple `IntoPluginGroup` trait with a `into` method that returns a `Vec<Plugin>`.

Alternatively, these could just be completely deprecated due to the move towards `App::default`, which was overwhelmingly their most common use.

## Drawbacks

1. Plugins are no longer quite as powerful as they were. This is both good and bad: it reduces their expressivity, but reduces the risk that they mysteriously break your app.
2. Apps can directly manipulate the system ordering of their plugins. This is essential for ensuring that interoperability and configurability issues are resolved, but risks breaking internal guarantees.
3. Users can accidentally break the requirement for a hard sync point to pass between systems within a plugin as encouraged by the plugin configuration by moving system sets into the same stage. Hard ordering dependencies briefly discussed in the *Future Work* section of this RFC could ensure that this is impossible.
4. Replacing existing resource values is more verbose, requiring either a startup system or explicitly removing the resource from the pre-existing plugin. This is much more explicit, but could slow down some common use cases such as setting window size.

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
2. What should `hard_before` and `hard_after` be called?
3. Should `PluginGroup` and `App::add_plugins` be deprecated?

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
    .add_pluging(simple_plugin())
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
3. Stages could (optionally?) be replaced with automatically inferred hard sync points on the basis of hard system ordering dependencies.
4. Provide a more global, staged analysis of system and resource initialization to reduce or eliminate order-dependence of app initialization, as raised in [Bevy #1255](https://github.com/bevyengine/bevy/issues/1255).
5. If-needed ordering constraints ([Bevy #2747](https://github.com/bevyengine/bevy/discussions/2747)) will significantly improve the ability to control ordering between plugin system sets without inducing excessive blocking.

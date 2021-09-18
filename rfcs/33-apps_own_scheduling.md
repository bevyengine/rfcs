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

Plugins can be added to your app using the `App::add_plugin` method.

```rust
use bevy::prelude::*;
use bevy_hypothetical_tui::prelude::*;

fn main(){
  App::minimal().add_plugin(TuiPlugin).run();
}
```

The standard pattern for defining plugins is to export a `MyCrateNamePlugin` struct that implements the `Plugin` trait.

```rust
struct CombatPlugin;

impl Plugin for CombatPlugin {
  fn build(plugin: &mut PluginState) {
    plugin
      .add_systems(SystemSet::new().with(attack).with(death));
      .add_system(player_movement)
      .init_resouce::<Score>()
  }
}

fn main(){
  App::default().add_plugin(CombatPlugin).run();
}
```

When designing plugins for a larger audience, you may want to make them configurable in some way to accommodate variations in logic that users might need.
To ensure that plugins can be properly controlled by the editor and customized by the users, all plugin configuration must be done using resources.

```rust
// We may want our physics plugin to work under both f64 and f32 coordinate systems
trait Floatlike;

// We can make our plugins generic and specify trait bounds
pub struct PhysicsPlugin<T: Floatlike> {
  // Remember to add a phantom data field to satisfy the compiler
  _phantom: PhantomData<T>,
};

struct RealisticGravity(bool);

fn realistic_gravity_criteria(realistic_gravity_setting: Res<RealisticGravity>) -> ShouldRun {
  if RealisticGravity.0 {
    ShouldRun::Yes
  } else {
    ShouldRun::No
  }
}

fn platformer_gravity_criteria(realistic_gravity_setting: Res<RealisticGravity>) -> ShouldRun {
  if RealisticGravity.0 {
    ShouldRun::No
  } else {
    ShouldRun::Yes
  }
}

impl Plugin for PhysicsPlugin<T> {
  fn build(plugin: &mut PluginState) {
    plugin
    .insert_resource(RealisticCollisionDetection)
    .add_system(collision_detection::<T>.label("Collision detection"))
    // These systems share a label to ensure that other systems will be scheduled correctly
    // regardless of the configuration chosen
    .add_system(realistic_gravity::<T>.set_run_criteria(realistic_gravity_criteria).label("Gravity"))
    .add_system(platformer_gravity::<T>.set_run_criteria(platformer_gravity_criteria).label("Gravity"));
    plugin
  }
}

fn main(){
  App::default()
    .add_plugin(PhysicsPlugin::<f64>)
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
  - The order of arguments in `between` matters; the labels must only be separated by a cleanup system in one direction, rather than both.
  - This methods do not automatically insert systems to enforce this separation: instead, the schedule will panic upon initialization as no valid system execution strategy exists.

### Configuring systems by their label

Systems do not exist in a vacuum: in order to perform useful computations, they must follow certain **constraints**.
The `SystemDescriptor` type captures this information, storing the underlying function and any labels, ordering constraints, run criteria and state information that may be attached to a particular system.

Note that a single function can be reused, and turned into multiple distinct systems, each with their own `SystemDescriptor` object which is owned by a `Schedule`.

In many cases, systems will be configured locally, either on their own or as part of a system set.

```rust
struct CombatPlugin;

impl Plugin for CombatPlugin {
  fn build(plugin: &mut PluginState) -> Plugin {
    plugin
      .add_systems(SystemSet::new().with(attack).with(death).label("Damage"));
      .add_system(player_movement.after("Damage"))
      .init_resouce::<Score>()
  }
}
```

However, the `App` can also configure systems by their label, applying the same configuration to all systems that share that label.
In addition to explicitly given labels, each plugin automatically assigns a shared label to every system that it includes.

```rust
fn main(){
  App::default()
    .add_plugin(CombatPlugin)
    // This uses the SystemLabel trait to define operations on the manually specified label
    .configure_label("Damage".after(CoreSystem::Time))
    // This label is automatically applied based on the type_id of the Plugin struct provided to add_plugin
    .configure_label(CombatPlugin.get_label().set_run_criteria(FixedTimeStep::(0.5)))
    .run();
}
```

### Configuring systems added by plugins

Carefully configuring the systems added by your plugins can be critical for fitting the logic defined by third-party crates to fit the unique needs and control flow of your app. You might want to:

1. Make sure that one of your systems run before a third-party system.
2. Control the relative order of two conflicting third-party systems.
3. Add a run criteria to disable a plugin's system when certain criteria are met.
4. Change which state(s) a plugins' systems runs in.
5. Add a plugin's systems to a nested schedule that runs repeatedly each frame.
6. Optimize the schedule strategy of a large schedule containing both first and third-party plugins.

However, Bevy enforces **plugin privacy rules**, and does not allow you to freely dissect and reconfigure every systems provided by third-party plugins. These rules are:

1. System ordering constraints and labels cannot be removed, although you can add new ones. Schedules will panic if ordering constraints cannot be satisfied.
2. The app can only configure plugins' systems by applying configuration changes to every system in that label.
3. Not all labels are public: you can only apply a label to new systems, or configure systems by that label, if the label is publicly exported.

You can think of these rules as an extra layer of safety-through-constraints provided by the creators of the plugins you are using, much like Rust's type system.
If you're coming to Bevy from a traditional game engine, these constraints may feel dangerous and limiting to you.
However, there are very good reasons that they exist:

1. System ordering constraints are powerful, flexible and minimal. If these constraints are violated, you are likely to get serious, hard-to-detect, non-deterministic bugs in your code that you will not discover until the night before launch.
2. Tying your design to individual systems, rather than carefully exposed sets of systems, makes it much harder to upgrade your code base as the plugin you are depending on evolves. This is problematic for plugin authors too, as this tight coupling makes re-architecting their code (even to split, merge or rename systems) expensive as they need to worry about breaking their users' apps.
3. Systems in Bevy are generally rather small and granular due to the benefits for code modularity and parallelizability that it offers. Separating closely-interlocked systems (with distinct run criteria, or by moving them into different states or schedules) will tend to result in complete but bizarre failure.
4. Manually specifying the precise order and stage arrangement increases the maintenance costs of your code base in a quadratic fashion. Each time you organize your schedule manually (for performance or to fix a bug), you risk creating new problems and create an undocumented, unenforced dependency that future maintainers cannot break.
5. Rust's own privacy rules will not (sanely) allow Bevy to force all system label types to be public, even if we tried.

There are also some powerful but subtle tools to work around these limitations:

1. You can remove an existing group of systems by their public label, and then replace them with a new set of your own systems that perform the same job.
2. System ordering constraints (usually) cease to have any effect if no systems with that label exist in the schedule. You can remove (or disable) any system by its label.
3. Every system has at least one label (other than those added directly to the app that you control), which is automatically assigned to it by its `Plugin` when it is added to the app.

Now, let's take a look at how we can use `App::configure_label` to tackle those use cases we mentioned at the top of this section:

TODO: Demonstrate how these examples work.

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

As such, try to only expose labels that apply to systems that can safely be enabled or disabled as a group.

### Standalone `bevy_ecs` usage

Even if you're just using `bevy_ecs`, plugins can be a useful tool for enabling code reuse and managing dependencies.

Instead of adding plugins to your app, independently add their system sets to your schedule and their resources to your world.

## Implementation strategy

Small changes:

1. As plugins no longer depend on `App` information at all, they should be moved into `bevy_ecs` directly.
2. If plugins can no longer configure the `App` in arbitrary ways, we need a new, ergonomic way to set up the default schedule and runner for standard Bevy games. The best way to do this is to create two new builder methods on `App`: `App::minimal` and `App::default`, which sets up the default stages, sets the runner and adds the required plugins.

### Plugin architecture

The ergonomics and semantics here are quite constrained. The following design:

- exports a type for users to consume
- allows users to pass in the struct to `.add_plugin`
- ensures each system has at least one label
- creates a structured API that limits the power of plugins
- allows us to later add more required fields to plugins (mostly without breaking the existing ecosystem)

Machinery:

```rust
// This trait must be implemented for all Plugins
pub trait Plugin: 'static {
  // This method is automatically called when plugins are added to the app
  pub fn build(plugin: &mut PluginState);

  /// Automatically generates a unique label based on the type implementing Plugin
  pub fn get_label() -> PluginLabel {
    PluginLabel(TypeId::of::<Self>())
  }
}

/// A label corresponding to the type of the Plugin is automatically added
/// to each system added by that plugin when `App::add_plugin` is called
#[derive(SystemLabel, Debug, Clone, PartialEq, Eq, Hash)]
pub struct PluginLabel(TypeId);

#[derive(Default)]
pub struct PluginState {
    systems: Vec<SystemDescriptor>,
    system_set: Vec<SystemSet>,
    resources: Vec<Box<dyn Resource>>,
    /* other plugin config fields here */
}

impl PluginState {
    pub fn new<P: Plugin>() -> Self {
        Self {
            system_sets: Vec::new(),
            systems: Vec::new(),
            label: PluginLabel::of::<P>(),
        }
    }
    pub fn apply(self, app: &mut App) {
        for system in self.systems {
            app.add_system(system.label(self.label.clone()))
        }

        for system_set in self.system_sets {
            app.add_system_set(system_set.label(self.label.clone()))
        }
    }

    /* other plugin builder functions here */
}
```

Example:

```rust
pub struct CustomPlugin;

impl Plugin for CustomPlugin {
    // Most existing Bevy code should be easily portable;
    // only the function signature of `build` will change
    fn build(plugin: &mut PluginState) {
        plugin
            .add_system(hello_world)
            .add_system_set(SystemSet::new().with_system(velocity).with_system(movement))
            .init_resource::<Thing>()
    }
}

fn main (){
  App::default()
    .add_plugin(CustomPlugin)
    .configure_label(CustomPlugin.get_label().after(CoreSystem::Time))
}
```

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

### Why do we need to automatically assign a shared label to systems added by a plugin?

The basic principle here is that **each system must always have at least one publicly visible label.**

If a system does not have any labels that are visible to the `App`, **unresolvable system ordering** ambiguities can occur.
The path to this is straightforward:

1. A third-party plugin defines some component or resource that they make `pub`.
2. They operate on this data in one of their systems, that they do not make `pub` and do not assign a `pub` label to.
3. The user defines another system that operates on the data.

In this realistic scenario, the user has now created a gameplay-relevant system order ambiguity that they have no way of fixing (short of vendoring the dependency).
The plugin author could not have foreseen this *particular* interaction.

In order to fix gameplay bugs (and ensure determinism for the use cases that care about it), this ambiguity must be resolved.
In a stage-centric architecture, this could be done by ensuring that the stage that the user's system is in isn't the same as the stage that the plugin's system is in.

However, this has some serious drawbacks:

1. The stage separation is not actually necessary, pointlessly limiting parallelism.
2. The fact that a system must live in a particular stage to avoid bugs must be manually maintained through comments and tests.
3. There may not be a suitable existing stage to satisfy all of the existing constraints, forcing the user to create a special stage *just* to satisfy this requirement.

In an inferred-hard-sync-points architecture, even this escape hatch is gone.
By providing at least one public label per system, we can avoid this problem without giving the end user unfettered access to the plugin internals.

As a nice benefit, this also allows us to use labels as our primary tool for customizing plugins, without having to use system sets as yet another top-level tool for grouping systems.

### Why is only the `App` allowed to configure systems by their labels?

The principle here is similar to that of orphan rules in Rust (and the reason why a structured `Plugin` API is a good idea): if you allow arbitrary dependencies to change unrelated things in complex ways, things start to break in horrible fashions once your dependency tree grows.
This is particularly bad under the working model where constraints are write-only.

By contrast, the `App` has context on both the global control flow and is end-user controlled, allowing you to centralize your rules in a single place.

Plugins should almost always be able to configure their own systems correctly without this tool, as they own the `SystemDescriptors` directly.
In the very rare cases where they require app-level permissions to set some global rule, they can instruct the user to do so in their README or examples.

### Why should we let the `App` remove systems by their label?

If the `App` is allowed to set arbitrary run criteria on systems, they could just add a run criteria that always return `ShouldRun::No`.

This is dumb, so we should support it properly (but with warnings about how there's a high chance that this breaks things).

### Why should we allow users to add dependencies between external plugin systems?

From a design perspective, this is essential to ensure that two third-party plugins can co-exist safely without needing to know about each other's existence in any way.

More humorously, we could *already* induce dependencies between labelled third-party systems.
Suppose we have two labelled systems: `A` and `C`, and want `A` to run before `C`.
The existing model is designed to prevent this level of meddling, and provides no way to configure the ordering of either system by any direct means.
However, if we insert a system `B` (which may do literally nothing!), and state that `B` runs after `A` and before `C`, we induce the desired dependency as system ordering is necessarily transitive.

There is no reasonable or desirable way to prevent this: instead we should embrace this by designing a real API to do so.

## Why should plugins be configured by setting a resource?

Using resources to configure plugins provides a clear, unified tool for setting plugin configuration.
This will be particularly valuable when working on editor support, or when building complex trees of plugin dependencies.

It also cleans up the ergonomics of the `Plugin` machinery substantially.

## Unresolved questions

1. Should we rename `SystemDescriptor` to something more user-friendly like `ConfiguredSystem`?
2. How do we handle [plugins that need to extend other resources at initialization](https://github.com/bevyengine/rfcs/pull/33#discussion_r709654419)?
3. How do we ensure that [adding run criteria won't break invariants](https://github.com/bevyengine/rfcs/pull/33#discussion_r709655879)?
4. Do we still need plugin groups at all? Can we replace them with a simple standard collection? The ergonomics matter much less with `App::default()` initialization.

## Future possibilities

This is a simple building block that advances the broader scheduler plans being explored in [Bevy #2801](https://github.com/bevyengine/bevy/discussions/2801).

This could be extended and improved in the future with:

1. Enhanced system visualization tools.
2. A scheduler / app option could be added to automatically infer and insert systems (including command-processing systems) on the basis of at-least-once separation constraints.
3. At-least-once separation constraints can be used to solve the cache invalidation issues that are blocking the implementation of indexes, as discussed in [Bevy #1205](https://github.com/bevyengine/bevy/discussions/1205).
4. `ShouldRun` could be cleaned up and moved back to simple binary logic. More helpers could be added for common plugin use cases.
5. Systems could be fully added and removed from the schedule based on run-time information, keeping the schedule clean when configuring plugins with resources.

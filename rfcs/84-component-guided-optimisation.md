# Feature Name: `component-guided-optimisation`

## Summary

Enable fine-grained compile-time dead code elimination in Bevy applications by automatically detecting which components are actually used and conditionally compiling only the systems that operate on those components. This allows the compiler to remove unused systems, reducing binary size and runtime overhead for applications.

## Motivation

Current Bevy applications include significant amounts of unused code that cannot be eliminated by traditional compiler optimizations. This creates several problems:

- Increased CPU usage
- Scheduler overhead
- RAM consumption
- Longer compilation time
- Larger executable binaries

Bevy already has "feature" level compilation features, to be able to compile out large part of the engine that are not used: 2d, 3d, ... This high level features are not enough to remove everything that is unused. For example, an application that only uses `SpotLight` would still have systems running for `PointLight` and `DirectionalLight`. Unless we manually add many features, it is not possible to be fine grained in conditional compilation.

## User-facing explanation

### For Bevy users

There is a new feature that you can enable to reduce Bevy footprint to only what you use! By enabling the `component_guided_optimization` feature and enabling for each component you're using, Bevy will be able to remove any built in system that isn't used by your game, reducing the engine to the minimum that you need.

This will improve compile time, binary size and runtime performance! Identifying the component-features to enable is easy, and with one command you'll be able to get all the benefits.

This comes in addition to the existing features: they will help at both dev time and for releases, while Component Guided Optimization is targeted for release builds.

### For Bevy Engine developpers

By annotating systems with the components they use, you enable users to remove systems that they don't actually need.

### For Bevy Plugins developpers

Like usually for Bevy, engine features are also available to you! You can annotate your systems to allow disabling them if they're not used.

Additionaly, you should enable the Bevy components you need with a `component_guided_optimization` feature on your crate, and add your own components as features, with the Bevy components they depend on.

## Implementation strategy

Everything for this is behind a `component_guided_optimisation` feature. Additional features for each component, `component_Xxxx`, are created, and not documented publicly.

### Part 1: Conditional System Compilation

Systems are annotated with a `#[cfg(...)]` attribute that:

- enable the system if `component_guided_optimization` is disabled
- disable the system if `component_guided_optimization` is enabled but one of its component is not
- enable the system if `component_guided_optimization` is enabled and all its components are

For example, a system using the `Transform` component:

```rust
pub fn transform_system(mut query: Query<&mut Transform>) {
    // system logic
}

app.add_system(Update, transform_system);
```

Would be annotated with:

```rust
pub fn transform_system(mut query: Query<&mut Transform>) {
    // system logic
}

#[cfg(any(not(feature = "component_guided_optimization"), feature = "component_Transform"))]
app.add_system(Update, transform_system);
```

First annotations can be done manually to enable progress on the other parts in parallel.

### Part 2: Automatic Component Feature Declaration

The `#[Derive(Component)]` macro annotates components declaration with an attribute to compile them out when not enabled.

```rust
#[derive(Component)]
struct MyComponent;
```

becomes

```rust
struct MyComponent;

#[cfg(any(not(feature = "component_guided_optimization"), feature = "component_MyComponent"))]
impl Component for MyComponent {
    //...
}
```

Engine / plugin developpers have to declare feature dependencies following required components.

### Part 3: Component Usage Identification

By enabling the `component_guided_optimization` feature, user code will stop building and fail for each component used. The user will have to enable features corresponding to each component used.

Users wanting to use Component Guided Optimization shouldn't have to manually identify the features needed. To help identify needed features, Bevy must provide tooling.

There are a few paths to investigate:

- Static Analysis 1: Find components used in commands and enable them
- Static Analysis 2: Build without the feature enabled and analyse the build log to find which components to enable
- Dynamic Analysis: Similar to Profile Guided Optimisation, add an helper to identify components used during a playthrough, and enable features from this "profile"

### Part 4: Better System Conditional Compilation

A new `const` boolean on the `IntoScheduleConfigs` trait would allow disabling adding a system to a schedule at compile time.

```rust
pub trait IntoScheduleConfigs<T: Schedulable<Metadata = GraphInfo, GroupMetadata = Chain>, Marker>:
    Sized
{
    const ENABLED: bool = true;

    // ...
}
```

if `ENABLED` is `false`, ignore this system when calling `add_systems`.

With a new attribute for systems, control the `ENABLED` value based on features:

```rust
#[ConditionalSystem("component_Transform")]
pub fn transform_system(mut query: Query<&mut Transform>) {
    // system logic
}
```

becomes (with some type shortcuts):

```rust
#[cfg(any(not(feature = "component_guided_optimization"), feature = "component_Transform"))]
pub fn transform_system(mut query: Query<&mut Transform>) {
    // system logic
}

#[cfg(not(any(not(feature = "component_guided_optimization"), feature = "component_Transform")))]
pub fn transform_system() {}

#[cfg(not(any(not(feature = "component_guided_optimization"), feature = "component_Transform")))]
impl IntoScheduleConfigs for transform_system {
    const ENABLED: bool = false;

    // ...
}
```

### Part 5: System Conditions Based On SystemParam

A new `const` boolean on the `SystemParam` trait would allow disabling adding a system to a schedule at compile time, with a default value at `true`.

```rust
pub unsafe trait SystemParam: Sized {
    const ENABLED: bool = true;

    // ...
}
```

if `ENABLED` is `false`, ignore the system using this system param when calling `add_systems`.

Same for the `QueryData` trait:

```rust
pub unsafe trait QueryData: WorldQuery {
    const ENABLED: bool = true;

    // ...
}
```

If `ENABLED` is `false`, any query as a system parameter of a system using this query has its `ENABLED` as `false`. They are combined according to how they are used in a query.

```rust
#[derive(Component)]
struct MyComponent;
```

becomes:

```rust
struct MyComponent;

#[cfg(any(not(feature = "component_guided_optimization"), feature = "component_MyComponent"))]
impl Component for MyComponent {
    //...
}


#[cfg(not(any(not(feature = "component_guided_optimization"), feature = "component_MyComponent")))]
impl QueryData for MyComponent {
    const ENABLED: bool = false;

    //...
}
```

Not all `SystemParam` in a system are equal: one could drive system execution by looping over the entities in a query, while another could only be used as a check to trigger additional behaviour. We need to be able to mark which `SystemParam` should control system compilation.

```rust
#[ConditionalSystem]
pub fn transform_system(
    #[conditional(controller)]
    mut query: Query<(Entity, &mut Transform)>,
    check: Query<&Visibility>,
) {
    for (entity, mut transform) in &mut query {
        // do something
        if check.get(entity).is_some() {
            // do something additional
        }
    }
}
```

This would check the `query` system parameter for system addition to the schedule, while ignoring `check`.

## Drawbacks

- Multiplication of features
- Tooling required for maintenance and use

## Rationale and alternatives

- Add more fine grained features manually
  - Pain to maintain and test
- Profile Guided Optimisation (PGO)
  - Doesn't actually compile out unused systems

## Unresolved questions

- Do we want to do it also for assets?
- Should the `component_Xxxx` features be available on the `bevy` crate, or only on individual crates?

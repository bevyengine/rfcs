# Feature Name: `xr-openxr-integration`

## Summary

This RFC details an API for XR integration in bevy, and focuses on OpenXR as a first backend. The implementation tightly integrates into the core bevy codebase.

## Motivation

XR brings immersive gaming and interaction technologies which are getting more and more popular. XR support is requested by a sizeable userbase of bevy. OpenXR as a first backend provides support for a wide range of VR and AR devices.

## User-facing explanation

The proposed XR API mirrors OpenXR functionality and semantics, but with a tight integration with bevy ECS and rendering systems, to minimize boilerplate. Some quality-of-life abstractions are introduced, like a generic virtual controller that corresponds to a polyfill between vendor-specific controllers.

### Terms & Definitions

* XR: Extended reality, which is the junction of VR (virtual reality) and AR (augmented reality). It is a set of technologies for immersive interaction with digital content (mainly audio-visual), for example though a head-mounted display or a handheld device used as a portal.
* OpenXR: standard XR API for games or apps.
* 3DOF: 3 degrees of freedom. A 3DOF device can only track its rotation.

### Usage examples

XR mode must be enabled in `Cargo.toml` with the feature `bevy_openxr`. There ia a separate flag `xr` which is automatically enabled with `bevy_openxr` and it should be used if the user provides its own xr backend plugin.

App creation requires a `XrConfig` resource to enable and configure XR. `OpenXrConfig` is optional and used to define backend-specific behavior. `vendor_bindings` is mainly used to define new functionality linked to vendor-specific controllers supported by the OpenXR backend. `vendor_bindings` can also be used to override the behavior of the general XR API, like the source of pose data and the source of `GenericControllerPairButtons` event data if using specific `OpenXrBindingDesc::name`s. `OpenXrActionPath` defines the coordinates for OpenXR to identify the source or destination of the interactions; `profile` and `path` are plugged directly into OpenXR's API, so the user can make use of the official OpenXR documentation for reference.

```rust
App::build()
    .insert_resource(XrConfig {
        mode: XrMode::Display {
            viewer: ViewerType::PreferHeadMounted,
            blend: BlendMode::PreferVR,
        },
        enable_generic_controllers: true,
    })
    .insert_resource(OpenXrConfig {
        vendor_bindings: vec![
            OpenXrBindingDesc {
                name: MY_ACTION,
                paths: vec![OpenXrActionPath {
                    profile: VALVE_INDEX_PROFILE,
                    path: LEFT_SQUEEZE_FORCE_PATH,
                }],
                action_type: OpenXrActionType::FloatInput,
            },
        ],
    })
```

Interaction is done through events and a `XrState` resource. If generic controllers are enabled, the user can read button events from `GenericControllerPairButtons`. The button set is modeled on the Oculus Touch controllers. Buttons of vendor-specific controllers are mapped to the ones of this virtual controller: extra buttons are ignored, and missing buttons will never generate events or stay at a fixed value. Given the diversity of VR controllers, the mapping is a "best effort" but the behavior can be overridden with `OpenXrConfig::vendor_bindings`.  
`GenericControllerPairButtons` groups together events from multiple buttons, for implementation convenience. Two-state button events have a `value` and `toggled` property. Continuous (float) state buttons (like trigger and joystick) do not have a `toggled` property.

```rust
fn input(mut events: EventReader<GenericControllerPairButtons>) {
    for e in events.iter() {
        if e.left_hand.primary_click.value && e.left_hand.primary_click.toggled {
            println!("You pressed the primary button on the left controller");
        }
    }
}
```

The generic controllers support only `GenericControllerVibration` as an output event.

```rust
fn output(mut events: EventWriter<GenericControllerVibration>) {
    if <touched some virtual object with left hand> {
        events.send(GenericControllerVibration {
            hand: HandType::Left,
            action: Vibration::Apply {
                duration: XrDuration::from_nanos(100_000_000),
                frequency: 1000_f32,
                amplitude: 0.5_f32,
            },
        });
    }
}
```

Pose is polled from `XrState` resource, provided by the XR plugin. Pose inputs are special and cannot be provided as an event. To provide the lowest perceived latency for head and hand motions, poses must be calculated (extrapolated) for a specific time in the future, corresponding to the target display time. This time could be one or more frames ahead. For simple update loops, the user should poll hand poses with `.hand_motion(...)` methods. For multi-frame update loops, users should use `.hand_motion_at_time(...)` where the time is calculated with `.predicted_display_time()` and `.predicted_display_period()`.

```rust
fn tracking(xr_state: Res<XrState>) {
    let left_pose = xr_state
        .hand_motion(HandType::Left, HandAction::Grip)
        .pose
        .to_mat4();
}
```

`XrState` provides the `set_tracking_reference_mode` method for setting its tracking mode. The tracking reference defines the origin of the virtual world with respect to the real world and is used to calculate the head and hand poses. Some tracking modes might not be supported (for example `Stage` for 3DOF headsets).

```rust
fn update(xr_state: Res<XrState>) {
    if <wants to change tracking mode> {
        let mode_supported = xr_state.set_tracking_reference_mode(TrackingReferenceMode::GravityAligned);
    }
}
```

OpenXR backend can be configured to support custom interactions in `OpenXrConfig`. Input and output actions are mapped to bevy events and grouped by action type. Events can be identified by its `name` field.

```rust
fn input(mut float_events: EventReader<OpenXrVendorInput<f32>>) {
    for e in events.iter() {
        if e.name == MY_ACTION {
            println!("You interacted with {}", e.name);
        }
    }
}
```

### Errors

The app may panic during initialization, setup or the first update loop, if misconfigured. The panic messages explain the cause of the crash and how to fix the root problem.  
The app panics if:

* `xr`or `bevy_openxr` feature is enabled but `XrConfig` resource has not been added before `DefaultPlugins`.
* The user sets `blend: BlendMode::AR` but the XR device has no AR pass-through capabilities. Not having a camera pass-through when it was expected changes the way the experience was intended to be consumed.
* Using `bevy_openxr` backend, the user sets `mode: XrMode::TrackingOnly`.
* The user specifies an invalid `OpenXrBindingDesc`: the profiles or paths are invalid or `OpenXrActionType` is not compatible with the profiles and paths.
* Using `bevy_openxr` backend, if the host PC does not have a default OpenXR runtime set.

In case of other types of invalid usage, the errors should be handled gracefully:

* `ViewerType { PreferHeadMounted, PreferHandHeld }`: The backend is allowed to change the user preference if not supported.
* `BlendMode { PreferVR, AR }`: The backend is allowed to switch from VR to AR if VR mode is not supported.
* `XrState::set_tracking_reference_mode(...)` can return false if a tracking mode is unsupported (the only case for this should be `Stage` when a device supports only 3DOF tracking).
* If the user sets `enable_generic_controllers` and the backend does not support motion controllers or the XR device has no controllers connected, `XrState::hand_motion*(...)` should return a `Motion` struct with pose with `None` position and `None` orientation. 

## Implementation strategy

**Note: Here are explained the plugins creation and input system. Rendering and lifecycle management are postponed after the rendering rewrite is completed.**

A reference implementation is provided [here](https://github.com/zarik5/bevy).

A `bevy_xr` crate is added as a surface level API with minimal amount of logic. `bevy_openxr` crate is added as a backend for `bevy_xr`, providing a subset of the functionality. The functionality that `bevy_openxr` can't provide must lead to a crash or must be handled gracefully -- functionality should not be faked.

TODO: finish

## Drawbacks

?

## Rationale and alternatives

It has been decided to add XR as a core functionality in bevy because:

* The promise of XR is to become as ubiquitous as flat-screen gaming. Bevy should stay ahead of the curve and provide XR support as a first-class citizen.
* Interoperability of OpenXR with `bevy_render` and `bevy_wgpu` is possible only if they are mutually aware of each other (this is true in particular for the Vulkan backend, where it is OpenXR that creates the Vulkan instance, device and texture handles).

`bevy_xr` is loosely modeled on OpenXR API because of its excellent design and it makes it easier to add more OpenXR features in the future. Some things like vendor-specific binding identification through string paths is confined only to `bevy_openxr` API, since it does not map well to other backends that may be added in the future, like WebXR, ArKit and ARCore. Only WebXR API has been consulted while developing `bevy_xr`, which is already very similar to OpenXR API.

A unified input system has been considered, which would allow to set mappings between mouse, keyboard, gamepads and VR motion controllers. The idea has been discarded for a few reasons:

* implementation complexity;
* difficulty of providing a sensible default mapping between such diverse types of inputs;
* the user can easily provide mappings functionality with ECS systems.

## Prior art

@blaind's [proof of concept](https://github.com/blaind/xrbevy), which proved the feasibility of integration of OpenXR with bevy. The code is written "ad-hoc" and cannot be integrated directly into bevy.

## Unresolved questions

The rendering code in bevy is going though a complete rewrite and the integration of OpenXR swapchain into bevy is postponed. This should be addressed before the completion of this RFC.

`Pose` and `Motion` are modeled to reflect 1:1 data obtained from OpenXR. If no pose data is available, this is represented by a `Pose` instance with `position: None` and `orientation: None`. This could be handled more ergonomically if `orientation` is not an `Option` and the absence of pose data is represented by `Option<Pose>::None`. The problem is that this opinionated decision cannot be made until we test the majority of XR devices and check their set of supported features in different contexts.

## Future possibilities

`bevy_xr` is designed to support and be extended for AR backends like ARCore and ARKit. While some features of these APIs are already provided by OpenXR (viewer tracking and camera pass-through) ARCore and ARKit provide features like face tracking, and might be supported where OpenXR is not.

The current RFC implementation does not include support for multi-view and hand tracking -- they are not essential but are very important respectively for performance and interaction expressiveness.

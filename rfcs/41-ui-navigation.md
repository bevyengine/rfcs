# Feature Name: Generic UI Navigation API

## Summary

Introduce [`bevy-ui-navigation`] into the bevy tree.

By default this, amounts to replacing `Interaction` with `Focusable` in `bevy_ui`.
On top of the current behavior, this adds the following capabilities:
- Focus management for **gamepads**
- Decouples input from focus management
- Decouples focus management from `bevy_ui`
- User-specified input handling for UI interaction
- An optional menu tree management system

## Terminology

- _menu tree_: the access hierarchy of menus.
- _activating_ a focusable: Sending the `Action` event while this focusable is focused.
- _focused menu_: The menu in which the currently focused `Focusable` is.
- _child menu_ (of a focusable): The menu accessed from this focusable.
- _transitive_: If `A` is _foo_ of `B` (swap _foo_ with anything such as
  _child_, _left_ etc.)
  Then `A` is _transitive foo_ of `B` if there is any chain of `C`, `D`,
  `E` etc. such as `A` is _foo_ `C`, `C` is _foo_ `D`, ... , `E` is _foo_ `B`.

## Motivation

`bevy_ui` interaction story is crude.
The only way to interact with the UI currently
is through the `Interaction` and `FocusPolicy` components.
`Interaction` only supports mouse or touch-based input.
We want gamepad supports and more game-oriented functionalities,
such as menu trees and movement between menus.

The current UI focus system is limited to `bevy_ui` and
can't be swapped out for something the user deems more appropriate.

Decoupling navigation from input and graphics also provides a handy API
for third party integration such as accessibility tools.

## User-facing explanation

The [bevy-ui-navigation README][`bevy-ui-navigation`] is a good start.

The API exposed in this RFC is relatively "low level".
Ideally, we provide default wrapping APIs to simplify using the focus system.

To summarize, here are the headlines:
- The navigation system can only be interacted with with events:
  `NavRequest` as input and `NavEvent` as output.
- `NavEvent::FocusChanged` contains the list of entities that are traversed
  to go from the last focus position to the new one.
- ui-navigation provides default input systems, 
  but it's possible to disable them and instead use custom ones.
  This is why we restrict interactions with navigation to events.
- It is possible to create isolated menus by using the `NavMenu` component.
- All `Focusable` children in the `NavMenu` entity's tree will be part
  of this menu, and moving using a gamepad from one focusable in a menu
  can only lead to another focusable within the same menu.
  You can specify a focusable in any other menu that will lead to this
  one once activated.

Following, a detailed tour.

### Using a `Focusable`

See `bevy-ui-navigation` [`Focusable`] doc.

The `Interaction` component in `ButtonBundle` is replaced by `Focusable`.

"Using" a `Focusable` is as easy as simply spawning a button using
`ButtonBundle`.
- All navigation input systems are added to the default plugins.
  Including mouse and gamepad.
- The resolution algorithm takes care of picking up an initial focused element
  if none exist yet.
  \
  To specify which focusable to focus initially, spawn a `ButtonBundle` with
  `focusable: Focusable::prioritized()`.
  \
  See [implementation section about initial focus](#what-to-focus-first).

### Reacting to button activation & cursor changes

See `bevy-ui-navigation` [`NavEvent`] doc,
[and example](https://github.com/nicopap/ui-navigation#simple-case).

The basic "change color of button based on focus state" is very similar to the
current focus management:

```rust
fn focus_color(mut interaction_query: Query<(&Focusable, &mut UiColor), Changed<Focusable>>) {
    for (focusable, mut material) in interaction_query.iter_mut() {
        if let FocusState::Focused = focusable.state() {
            *material = Color::ORANGE_RED.into();
        } else {
            *material = Color::DARK_GRAY.into();
        }
    }
}
```

Note that the `focus_color` system should be added `.after(NavRequestSystem)`
for same-frame updates.

For more advanced behaviors, the user should listen to the [`NavEvent`] event
reader.

The default way of hooking your UI to your code in [`bevy-ui-navigation`] is
a bit rough:
* Mark your UI elements with either marker components or an enum
* listen for [`NavEvent`] in a UI handling system,
* filter for the ones you care about (typically `NavEvent::NoChanges`),
* check what [`NavRequest`] triggered it,
* retrieve the focused entity from the event,
* check against your own queries what entity it is,
* write code for each case you want to handle

```rust
use bevy::prelude::*;
use bevy_ui_navigation::events::{NavEvent, NavRequest};

#[derive(Component)]
enum MainMenuButton { Start, Options, Exit }
/// Marker component
#[derive(Component)] struct ActiveButton;

fn handle_ui(
  mut events: EventReader<NavEvent>,
  buttons: Query<&MainMenuButton, With<ActiveButton>>,
) {
  // iterate NavEvents
  for event in events.iter() {
    // Check for a `NoChanges` event with Action
    if let NavEvent::NoChanges { from, request: NavRequest::Action } = event {
      // Get the focused entity (from.first()), check the button for it.
      match buttons.get(*from.first()) {
        // Do things when specific button is activated
        Ok(MainMenuButton::Start) => {}
        Ok(MainMenuButton::Options) => {}
        Ok(MainMenuButton::Exit) => {}
        Err(_) => {}
      }
    }
  }
}
```

It could be useful to provide a higher level API on top of [`NavEvent`].
[`bevy-ui-navigation`] has a module dedicated to it: [`event_helpers`].
Using [`event_helpers`] would reduce the previous code to:
```rust
fn handle_ui(mut button_events: NavEventQuery<&MainMenuButton, With<ActiveButton>>) {
  // NOTE: this will silently ignore multiple navigation event at the same frame.
  // It should be a very rare occurrence.
  match button_events.single_activated().ignore_remaining() {
    // Do things when specific button is activated
    Some(MainMenuButton::Start) => {}
    Some(MainMenuButton::Options) => {}
    Some(MainMenuButton::Exit) => {}
    None => {}
  }
}
```


### Moving the cursor

See `bevy-ui-navigation` [`NavRequest`] doc.

All navigation input systems are added to the default plugins.
Including mouse and gamepad.

However, if the user's game needs it,
the user can change how input interacts with navigation.
It is easy to remove the default input systems
to replace them with your own custom ones.

Use `NavRequest` to control the navigation state.
- For mouse picking, use `NavRequest::FocusOn(entity)` to move focus
  to the exact entity you want.
- For gamepad or any other kind of directional input, use the
  `NavRequest::Move` event.
- Standard "action" and "cancel", see the [`NavRequest`] doc for details

An input system should run before the request handling system using the
system label mechanism:
```rust
  .add_system(my_input.before(NavRequestSystem))
```

The exact mechanism for disabling default input is still to be designed.
[See relevant implementation section](#input-customization).


### Creating a menu

See the `bevy-ui-navigation` [`NavMenu`] doc,
[and example](https://github.com/nicopap/ui-navigation/blob/master/examples/menu_navigation.rs).

To create a menu, use the newly added `MenuBundle`,
this collects all [`Focusable`] children of that node into a single menu.

Note that this counts for **transitive** children of the menu entity.
Meaning you don't have to make your [`Focusable`]s direct children of the menu.

```rust
#[derive(Bundle)]
struct MenuBundle {
  #[bundle]
  pub node_bundle: NodeBundle,
  pub menu: MenuSeed,
}
```

See [relevant implementation section](#spawning-menus) for details on `MenuSeed`.

[`bevy-ui-navigation`] supports menus and inter-menu navigation out-of-the-box.
Users will have to write code to hide/show menus,
but not to move the cursor between them.

A menu is:
- An isolated piece of UI where you can navigate between focusables
  within it.
- It's either
  - a _root_ menu, the default menu with focus,
  - or _reachable from_ a [`Focusable`].
- To enter a menu, you have to either
  - _activate_ the [`Focusable`] it is _reachable from_,
  - _cancel_ while focus is in one of its sub-menus.

Menus have two parameters:
- Whether they are a "scope" menu: a "scope" menu is like a
  browser tab and can be directly navigated through hotkeys when
  focus is within a transitive submenu.
- Whether directional navigation is wrapping (e.g. going leftward from the
  leftmost `Focusable` focuses the rightmost `Focusable`)

Here again, a higher-level API could benefit users,
by automating the process of hiding and showing menus
that are focused and unfocused.


### Creating a custom widget

This is a case study of how [Warlock's Gambit implemented
sliders][gambit-slider] using [`bevy-ui-navigation`].

Warlock's Gambit has audio sliders, built on top of `bevy_ui`
and [`bevy-ui-navigation`].

We used the [locking mechanism](#locking) to disable navigation when starting to
drag the slider's knob, when the player pressed down the left mouse button.
This is to prevent other [`Focusable`]s from stealing focus while dragging
around the knob with the mouse.

We then use mouse movement to move around the knob,
and update the audio level based on the knob's position.

We send a `NavRequest::Free` when the player release's the left mouse button.


### Custom directional navigation

Beyond just changing the inputs that generate [`NavRequest`]s,
it's possible to customize gamepad-style directional input generated by
`NavRequest::Direction`.
The `NavigationPlugin` is generic over directional input.
It accepts a `M: MenuNavigationStrategy` type parameter.

The default implementation, the one for `bevy_ui`, uses `GlobalTransform`
to resolve position.

See [relevant implementation section](#MenuNavigationStrategy).


## Implementation

The nitty-gritty code… is already available!
The [`bevy-ui-navigation`] repo contains the code.
[this is a good start for people interested in the architecture][ui-nav-arch].

### Internal representation

The navigation tree is a set of relationships between entities with either the
[`Focusable`] component or the [`TreeMenu`] component.

The hierarchy is specified by both the native bevy hierarchy
and the `focus_parent` of the [`TreeMenu`] component.

In the following screenshot, `Focusable`s are represented as circles, menus
as rectangles, and the `focus_parent` by blue arrows
(menu points to its parent).

![A screenshot of a RPG-style menu with the navigation tree overlayed](https://user-images.githubusercontent.com/26321040/141671969-ea4a461d-0949-4b98-b060-c492d7b896bd.png)

Note that gold buttons are `FocusState::Active` while the orange-red button
("B") is `FocusState::Focused`.

Since the "tabs menu" doesn't have an outgoing arrow, it is the root menu.

The active buttons are a breadcrumb of buttons to activate to reach
the current menu with the `Focused` focusable from the root menu.

To move the focus from the "soul menu" to the "ABC menu",
you need to send `NavRequest::Action` while the button "abc" is focused
(i.e.: _"activating"_ the button).

Such a move would generate the following `NavEvent`:
```rust
NavEvent::FocusChanged {
  to: [<abc entity>, <B entity>],
  from: [<abc entity>],
}
```

If there were a "KFC menu" (currently hidden) [child of](#Terminology)
the "kfc" button, then [activating](#Terminology)
the "kfc" button would send the focus to
the prioritized focusable within the "KFC menu".

To navigate from "ABC menu" back to "soul menu",
you would send `NavRequest::Cancel`.
Such a move would generate the following `NavEvent`:
```rust
NavEvent::FocusChanged {
  to: [<abc entity>],
  from: [<abc entity>, <B entity>],
}
```
The "tabs menu" is defined as a "scope menu", which means that
by default, the `LT` and `RT` gamepad buttons will navigate
the "tabs menu" regardless of the current focus position.

Pressing `RT` while "B" if focused, would generate the following `NavEvent`:
```rust
NavEvent::FocusChanged {
  to: [<body entity>, <prioritized button in body menu>],
  from: [<soul entity>, <abc entity>, <B entity>],
}
```

There is no [child menus](#Terminology) of the "B" button
and "B" is of type `FocusAction::Normal`,
therefore, sending `NavRequest::Action` while "B" is highlighted
will do nothing and generate the following `NavEvent`:
```rust
NavEvent::NoChanges {
  request: NavRequest::Action,
  from: [<soul entity>, <abc entity>, <B entity>],
}
```

See [relevant implementation section](#NavEvent) for details on `NavEvent`.


### Exposed API

The crux of `bevy-ui-navigation` are the following types:
- [`Focusable`](#Focusable) component [on docs.rs][`Focusable`]
- [`NavMenu`](#NavMenu) enum [on docs.rs][`NavMenu`]
- `NavRequest` event [on docs.rs][`NavRequest`]
- [`NavEvent`](#NavEvent) event [on docs.rs][`NavEvent`]
- [`MenuNavigationStrategy`](#MenuNavigationStrategy) trait

### `Focusable`

See `bevy-ui-navigation` [`Focusable`] doc.

The `Focusable` component holds state about what happens when it is activated
and what focus state it is in. (focused, active, inactive, etc.)


#### Focusable state

See `bevy-ui-navigation` [`FocusState`] doc.

The focus state encodes roughly the equivalent of `Interaction`,
it can be accessed with the `state` method on `Focusable`.

**Hovering state is not specified here**, since it is orthogonal to
a generic navigation system. (see [dedicated section](#hovered-is-not-covered))

* **Prioritized**: 
  An entity that was previously `Active` from a branch of the menu tree that is
  currently not focused. When focus comes back to the `NavMenu` containing this
  `Focusable`, the `Prioritized` element will be the `Focused` entity.
* **Focused**:
  The currently highlighted/used entity, there is only a single focused entity.
  \
  All navigation requests start from it.
  \
  To set an arbitrary `Focusable` to focused, you should send a `NavRequest::FocusOn`
  request.
* **Active**:
  This Focusable is on the path in the menu tree to the current Focused entity.
  \
  `FocusState::Active` focusables are the `Focusable`s from previous menus that
  were activated in order to reach the `NavMenu` containing the currently focused
  element.
  \
  It is one of the "breadcrumb" of buttons to reach the current focused
  element.
* **Inert**:
  None of the above: This Focusable is neither Prioritized, Focused or Active.

#### Focusable action types

See `bevy-ui-navigation` [`FocusAction`] doc.

A `Focusable` can execute a variety of `FocusAction` when receiving
`NavRequest::Action`, the default one is `FocusAction::Normal`.

* **Normal**:
  Acts like a standard navigation node.
  \
  Goes into relevant menu if any [`NavMenu`] is
  `reachable_from` this `Focusable`.
* **Cancel**:
  If we receive `NavRequest::Action` while this `Focusable` is
  focused, it will act as a `NavRequest::Cancel` (leaving submenu to
  enter the parent one).
* **Lock**:
  If we receive `NavRequest::Action` while this `Focusable` is
  focused, the navigation system will freeze until `NavRequest::Free`
  is received, sending a `NavEvent::Unlocked`.
  \
  This is useful to implement widgets with complex controls you don't
  want to accidentally unfocus, or suspending the navigation system while
  in-game.

### `NavMenu`

See `bevy-ui-navigation` [`NavMenu`] doc.

The public API has `NavMenu`, but we use internally [`TreeMenu`],
this prevents end users from breaking assumptions about the menu trees.
[More details on this decision](#spawning-menus).

A menu that isolate children [`Focusable`]s from other
focusables and specify navigation method within itself.

A [`NavMenu`] can be used to:
* Prevent navigation from one specific submenu to another
* Specify if 2d navigation wraps around the screen.
* Specify "scope menus" such that a
  `NavRequest::ScopeMove` emitted when
  the focused element is a [`Focusable`] nested within this `NavMenu`
  will navigate this menu.
* Specify _submenus_ and specify from where those submenus are reachable.
* Specify which entity will be the parents of this [`NavMenu`], see
  `NavMenu::reachable_from` or `NavMenu::reachable_from_named` if you don't
  have access to the `Entity`
  for the parent [`Focusable`]

If you want to specify which [`Focusable`] should be
focused first when entering a menu, you should mark one of the children of
this menu with `Focusable::prioritized`.

#### Invariants

**You need to follow those rules  to avoid panics**:
1. A menu must have **at least one** [`Focusable`] child in the UI
   hierarchy.
2. There must not be a menu loop. I.e.: a way to go from menu A to menu B and
   then from menu B to menu A while never going back.

#### Panics

Thankfully, programming errors are caught early and you'll probably get a
panic fairly quickly if you don't follow the invariants.
* Invariant (1) panics as soon as you add the menu without focusable
  children.
* Invariant (2) panics if the focus goes into a menu loop.

#### Variants

* **Bound2d**: Non-wrapping menu with 2d navigation.
  It is possible to move around this menu in all cardinal directions, the
  focus changes according to the physical position of the
  [`Focusable`] in it.
  \
  If the player moves to a direction where there aren't any focusables,
  nothing will happen.
* **Wrapping2d**: Wrapping menu with 2d navigation.
  It is possible to move around this menu in all cardinal directions, the
  focus changes according to the physical position of the
  [`Focusable`] in it.
  \
  If the player moves to a direction where there aren't any focusables,
  the focus will "wrap" to the other direction of the screen.
* **BoundScope**: Non-wrapping scope menu
  Controlled with `NavRequest::ScopeMove`
  even when the focused element is not in this menu, but in a submenu
  reachable from this one.
* **WrappingScope**: Wrapping scope menu
  Controlled with `NavRequest::ScopeMove` even
  when the focused element is not in this menu, but in a submenu reachable from this one.

#### Going back to a previous menus

`Focusable`s have a `prioritized` state that is set when they go from
active/focused to not active.
This is a form of memory that allows focus to go back to the last focused
element within a menu when it is re-visited.
This is also the mechanism used to let the user decide which focusable to focus
when none are focused yet.

#### Spawning menus

Menus store a reference to their parent,
but that parent is not necessarily their hierarchical parent.
The parent is just a button in another menu.

It is inconvenient to have to pre-spawn each button to acquire their `Entity` id
just to be able to spawn the menu you'll get to from that button.
[`bevy-ui-navigation`] uses a proxy component holding both the parent
and the menu state info.

That proxy component is `MenuSeed`,
you can initialize it either with the `Name` or `Entity` of the parent focusable of the menu.
A system will take all `MenuSeed` components and replace them with a `TreeMenu`.

This also enables front-loading the check of whether the parent focusable
`Entity` is indeed a `Focusable`.
Before turning the `MenuSeed` into a `TreeMenu`,
we check that the alleged parent is indeed `Focusable`, and panic otherwise.

If we couldn't assume parents to be focusable,
the focus resolution algorithm would trip.

This also completely hides the `TreeMenu` component from end-users.
This allows safely changing internals of the navigation system without
breaking user code.

This introduces a single frame lag on menu spawn,
but it improves ergonomics to users.
I thought it was a pretty good trade-off,
since spawning menus is not time critical, unlike input.


### `NavEvent`

See `bevy-ui-navigation` [`NavEvent`] doc.

Events emitted by the navigation system.

(Please see the docs.rs page, the rest of this section refers to terms
explained in it)

Note the usage of `NonEmpty` from the `non_empty_vec` crate.

In most case, we only care about a single focusable in `NoChanges` and
`FocusChanged` (the `.first()` one).
Furthermore, we _know_ those lists will never be empty (since there is always
one focusable at all time).
We also really don't want to provide a clunky API where you have to `unwrap` or
adapt to `Option`s when it is not needed.
So we provide the list of changed focusable as an `NonEmpty`.

I vetted several "non empty vec" crates, and `non_empty_vec`
had the cleanest and easiest to understand implementation.
This is why, I chose it.


#### cancel events

When entering a menu, you want a way to leave it and return to the previous menu.
To do this, we have two options:
- A `NavRequest::Cancel`
- _activating_ a `FocusAction::Cancel` focusable.

This will change the focus to the parent button of the focused menu.

Since "cancel" or "go back to previous menu" buttons are a very common
occurrence, I thought it sensible to integrate it fully in the navigation
algorithm.

#### Moving into a menu

When the `Action` request is sent while a `Focusable` with a child menu is focused,
the focus will move to the child menu,
selecting the focusable to focus
with a heuristic similar to the initial focused selection.

### `MenuNavigationStrategy`

This trait is used to pick a focusable in a provided direction.
- The `NavigationPlugin` is generic over a `T` that implement this trait
- `DefaultNavigationPlugins` provides a default implementation of this trait

This decouples the UI from the navigation system.
For example, this allows using the navigation system with 3D elements.

```rust
/// System parameter used to resolve movement and cycling focus updates.
///
/// This is useful if you don't want to depend on bevy's [`GlobalTransform`]
/// for your UI, or want to implement your own navigation algorithm. For example,
/// if you want your ui to be 3d elements in the world.
///
/// See the [`UiProjectionQuery`] source code for implementation hints.
pub trait MenuNavigationStrategy {
    /// Which `Entity` in `siblings` can be reached from `focused` in
    /// `direction` if any, otherwise `None`.
    ///
    /// * `focused`: The currently focused entity in the menu
    /// * `direction`: The direction in which the focus should move
    /// * `cycles`: Whether the navigation should loop
    /// * `siblings`: All the other focusable entities in this menu
    ///
    /// Note that `focused` appears once in `siblings`.
    fn resolve_2d<'a>(
        &self,
        focused: Entity,
        direction: events::Direction,
        cycles: bool,
        siblings: &'a [Entity],
    ) -> Option<&'a Entity>;
}
```


### Locking

To ease implementation of widgets, such as sliders,
some focusables can "lock" the UI, preventing all other form of interactions.
Another system will be in charge of unlocking the UI by sending a `NavRequest::Free`.

The default input sets the escape key to send a `NavRequest::Free`,
but the user may chose to `Free` through any other mean.

This might be better served to a locking component that block navigation
of specific menu trees,
but I didn't find the need for such fine-grained controls.


### Input customization

[See the `bevy-ui-navigation` README example](https://github.com/nicopap/ui-navigation#simple-case).

The `NavigationPlugin` can be added to the app in two ways:
1. With `NavigationPlugin`
2. With `DefaultNavigationPlugins`

(1) only inserts the navigation systems and resources,
while (2) also adds the input systems generating the `NavRequest`s
for gamepads, mouse and keyboard.

This enables convenient defaults
while letting users insert their own custom input logic.

More customization is available:
[`MenuNavigationStrategy`](#MenuNavigationStrategy) allows customizing
the spacial resolution between focusables within a menu.


### What to focus first?

Gamepad input assumes an initially focused element
for navigating "move in direction" or "press action".
At the very beginning of execution, there are no focused element,
so we need fallbacks.
By default, it is any `Focusable` when none are focused yet.
But the user can spawn a `Focusable` as `prioritized`
and the algorithm will prefer it when no focused nodes exist yet.

```rust
self.focusables
    .iter()
    // The focused focusable.
    .find_map(|(e, focus)| (focus.state == Focused).then(|| e))
    // Prioritized focusable within the root menu (if it exists)
    .or_else(root_prioritized)
    // Any prioritized focusable
    .or_else(any_prioritized)
    // Any focusable
    .or_else(fallback)
```


## Prior art

I've only had a cursory glance at what already exists in the domain.

Beside [godot](https://github.com/godotengine/godot-docs/blob/master/tutorials/ui/gui_navigation.rst), I found the [Microsoft
`FocusManager` documentation](https://docs.microsoft.com/en-us/windows/apps/design/input/focus-navigation-programmatic)
to be very close to what I am currently doing ([additional Microsoft
resources](https://docs.microsoft.com/en-us/windows/apps/design/input/focus-navigation))

Our design differs from Microsoft's UI navigation in a few key ways.
We have game-specific assumptions. For example: we assume a "trail" of active elements
that leads to the sub-sub-submenu we are currently focused in, we rely on that
trail to navigate containing menus from the focused element in the submenu.

Otherwise, given my little experience in game programming, I am probably overlooking some
standard practices. My go-to source ("Game Engine Architecture" by Jason
Gregory) doesn't mention ui navigation systems.

## Major differences with the current focus implementation

### `Hovered` is not covered

Since navigation is completely decoupled from input and ui lib,
it is impossible for it to tell whether a focusable is hovered.
To keep the hovering state functionality,
it will be necessary to add it as an independent component
separate from the navigation library.

A fully generic navigation API cannot handle hover state.
This doesn't mean we have to entirely give up hovering.
It can simply be added back as a component independent
from the focus system.

### Interaction tree

In this design, interaction is not "bubble up" as in the current `bevy_ui`
design.

This is because there is a clear and exact distinction
in what can be "focused", what is a "menu", and everything else.
Focused element do not share their state with their parent.
The breadcrumb of menus defined [in the `NavMenu` section](#NavMenu)
is a very different concept from marking parents as focused.

As such, `FocusPolicy` becomes useless and is removed.

## Open questions

### (Solved, but worth asking) Game-oriented assumptions

This was not intended during implementation, but in retrospect, the `NavMenu`
system is relatively opinionated.

Indeed, we assume we are trying to build a game UI with a series of "menus" and
that it is possible to keep track of a "trail" of buttons from a "root menu" to
the deepest submenu we are currently browsing.

The example of a fancy editor with a bunch of docked widgets seems to break
that assumption. However, I think we could make it so each docker has a tab,
and we could navigate between the tabs. I think such an implementation strategy
encourages better mouseless navigation support and it might not be a
limitation.

![Krita UI screenshot](https://user-images.githubusercontent.com/26321040/141671939-24b8a7c3-b296-4fd4-8ae0-3bbe7fe4c9a3.png)

In this example, we can imagine the root `NodeBundle` having a `NavMenu`
component, and the elements highlighted in yellow have a `Focusable` component.
This way we create a menu to navigate between dockers. This is perfectly
consistent with our design.

### (Solved, but worth asking) Moving UI camera & 3d UI

The completely decoupled implementation of the navigation system
enables user to implement their own UI.
It is perfectly possible to add a `Focusable` component to a 3d object
and, for example, provide a [mouse picking] based input system
sending the `NavRequest`,
while using `world_to_viewport` in `MoveParam` for gamepad navigation.
(or completely omitting gamepad support by providing a stub `MoveParam`)


## Drawbacks and design limitations

### How does this work with the spawn/despawn workflow?

The current design imposes to the user that all UI nodes such a buttons and
menus must be already loaded, even if not visible on screen.
This is to support re-focusing the last focused element when moving between
menus.
The design of this RFC _hinges on_ the whole tree been loaded in ECS while
executing navigation requests, and would work poorly if menus were spawned and
despawned dynamically.
From what I gather from questions asked in the `#help` discord channel,
I think the design most people come up with naturally is to spawn and despawn
dynamically the menus as they are traversed, which is incompatible with
this design.

In practice, this should be done by setting the `style.display` of
non-active menus to `None` and change the style when focus enters them.
This also fixes the already-existing 1 frame latency issue with the
spawn/despawn design.

Something that automates that could be a natural extension of the exposed API.
It may also help end users go with the best design first.

### User upheld invariants

I think the most problematic aspect of this implementation is that we push on
the user the responsibility of upholding invariants. This is not a safety
issue, but it leads to panics. The invariants are:
1. Never create a cycle of menu connections
2. Each `NavMenu` must at least have one contained `Focusable`

The upside is that the invariants become only relevant if the user _opts into_
`NavMenu`. Already, this is a choice made by the user, so they can be made
aware of the requirements in the `NavMenu` documentation.

If the invariants are not upheld, the navigation system will simply panic.

It may be possible to be resilient to the violation of the invariants, but I
think it's better to panic, because violation will most likely come from
misunderstanding how the navigation system work or programming errors.

[mouse picking]: https://github.com/aevyrie/bevy_mod_picking/
[`bevy-ui-navigation`]: https://github.com/nicopap/ui-navigation
[ui-nav-arch]: https://github.com/nicopap/ui-navigation/blob/30c828a465f9d4c440d0ce6e97051a5f7fafa425/src/resolve.rs#L1-L33
[`event_helpers`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/event_helpers/index.html
[gambit-slider]: https://github.com/team-plover/warlocks-gambit/blob/3f8132fbbd6cce124a1cd102755dad228035b2dc/src/ui/main_menu.rs#L59-L95
[`NavRequest`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/events/enum.NavRequest.html
[`NavEvent`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/events/enum.NavEvent.html
[`Focusable`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/struct.Focusable.html
[`FocusState`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/enum.FocusState.html
[`FocusAction`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/enum.FocusAction.html
[`NavMenu`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/enum.NavMenu.html
[`UiProjectionQuery`]: https://github.com/nicopap/ui-navigation/blob/17c771b7f752cfd604f21056f9d4ca6772529c6f/src/resolve.rs#L378-L429
[`TreeMenu`]: https://github.com/nicopap/ui-navigation/blob/30c828a465f9d4c440d0ce6e97051a5f7fafa425/src/resolve.rs#L201-L216

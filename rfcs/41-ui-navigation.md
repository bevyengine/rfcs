# Feature Name: Generic UI Navigation API

## Summary

Introduce [`bevy-ui-navigation`] into the bevy tree.

[PR Status][this RFC's PR].

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
- It is possible to create isolated menus by using the `MenuSetting` component.
- All `Focusable` children in the `MenuSetting` entity's tree will be part
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

The default way of hooking the UI to code in [`bevy-ui-navigation`] is
a bit rough:

* Mark the UI elements with either marker components or an enum
* listen for [`NavEvent`] in a UI handling system,
* filter for the ones cared about (typically `NavEvent::NoChanges`),
* check what [`NavRequest`] triggered it,
* retrieve the focused entity from the event,
* check against other queries what entity it is,
* write code for each case to handle

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

To make it easier to react to "activation" events, we provide a trait extension
to `EventReader<NavEvent>`. This adds the `nav_iter` method, which returns a
wrapper struct that understands the concept of "activation":

```rust
fn handle_ui(
  mut events: EventReader<NavEvent>,
  buttons: Query<&MainMenuButton, With<ActiveButton>>,
) {
  for button in events.nav_iter().activated_in_query(&buttons) {
      match button {
        // Do things when specific button is activated
        MainMenuButton::Start => {}
        MainMenuButton::Options => {}
        MainMenuButton::Exit => {}
      }
  }
}
```

The current implementation is missing a method to handle mutable queries.

A previous design added a custom `SystemParam` that the user could call
directly. The trait extension allows the user to seamlessly go from the wrapper
struct to the actual `EventReader` and vis-versa, which is a plus.

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

[`bevy-ui-navigation`]'s menu system is purely optional,
and it is likely that a large section of users will ignore it
and content themselves with `Focusable`s, which is capable enough.

The menu system is also the source of most of the complexity and pitfalls.

However, having such a system available out of the box integrating seamlessly
with the existing focus system is just great user experience.

See the `bevy-ui-navigation` [`MenuSetting`] doc,
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
  pub menu: MenuBuilder,
}
```

See [relevant implementation section](#spawning-menus) for details on
`MenuBuilder`.

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

We send a `NavRequest::Unlock` when the player release's the left mouse button.


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

`Focusable`s are just components,
if the user doesn't ever spawn a `MenuBundle`,
then there is really not that much to describe.

Outside of menus,
most of the complexity resides in handling input and the event responses.
Which is already implemented and will be re-used by [`bevy-ui-navigation`].

For menus, to work, however, we design around a _navigation tree_.

The navigation tree is a set of relationships between entities with either the
[`Focusable`] component or the [`TreeMenu`] component.

There are two relationships backed in UI navigation:

1. A fully private one defining the "parent" of a menu in [`TreeMenu`]
2. A fully public one dependent on bevy_hierarchy Parent/Children
   used to tell which focusables is in a menu.
   A focusable is in a menu if it is a descendent of a [`TreeMenu`]
   (with children of `Focusable`s and `TreeMenu`s within that menu culled)

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
  to: [<B entity>, <abc entity>],
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
  from: [<B entity>, <abc entity>],
}
```
The "tabs menu" is defined as a "scope menu", which means that
by default, the `LT` and `RT` gamepad buttons will navigate
the "tabs menu" regardless of the current focus position.

Pressing `RT` while "B" if focused, would generate the following `NavEvent`:
```rust
NavEvent::FocusChanged {
  to: [<prioritized button in body menu>, <body entity>],
  from: [<B entity>, <abc entity>, <soul entity>],
}
```

There is no [child menus](#Terminology) of the "B" button
and "B" is of type `FocusAction::Normal`,
therefore, sending `NavRequest::Action` while "B" is highlighted
will do nothing and generate the following `NavEvent`:
```rust
NavEvent::NoChanges {
  request: NavRequest::Action,
  from: [ <B entity>, <abc entity>, <soul entity>],
}
```

See [relevant implementation section](#NavEvent) for details on `NavEvent`.


### The resolve algorithm

This is just a plain-english description of the `resolve` function in
`resolve.rs`:

The `TreeMenu::focus_parent` (the private relationship)
allows to define "links" between menus.

Menus usually are not children of other menus,
so it can't use the Parent/Child relation

Internally, what happens is:

- `FocusOn(Entity)`: pick the current focused and the target entity.
  Build the breadcrumbs from them to the root (this is easy with focus_parent),
  then trim the common tail, update the relevant focusable's state.
- `Action`, check if any menu has its field `focus_parent` = current focused,
  then set focused to that menu's `active_child`.
- `Cancel`, find this focusable's parent menu, set the new focused to its focus_parent
- `Move(Direction)` just call the `MenuNavigationStrategy`'s resolve_2d
  with a list of all focusables within the focused's menu

The navigation tree is "just there" in the ECS,
and traversed based on various entry points
depending on the `NavRequest` it received.
The direction of tree traversal is basically always from leaf to root.

### Exposed API

The crux of `bevy-ui-navigation` are the following types:
- [`Focusable`](#Focusable) component [on docs.rs][`Focusable`]
- [`MenuSetting`](#MenuSetting) enum [on docs.rs][`MenuSetting`]
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
  currently not focused. When focus comes back to the `MenuSetting` containing this
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
  were activated in order to reach the `MenuSetting` containing the currently focused
  element.
  \
  It is one of the "breadcrumb" of buttons to reach the current focused
  element.
* **Inert**:
  None of the above: This Focusable is neither Prioritized, Focused or Active.
* **Blocked**:
  Prevents all interactions with this Focusable.
  \
  This is equivalent to removing the Focusable component from the entity,
  but without the latency.

#### Focusable action types

See `bevy-ui-navigation` [`FocusAction`] doc.

A `Focusable` can execute a variety of `FocusAction` when receiving
`NavRequest::Action`, the default one is `FocusAction::Normal`.

* **Normal**:
  Acts like a standard navigation node.
  \
  Goes into relevant menu if any [`MenuSetting`] is
  `reachable_from` this `Focusable`.
* **Cancel**:
  If we receive `NavRequest::Action` while this `Focusable` is
  focused, it will act as a `NavRequest::Cancel` (leaving submenu to
  enter the parent one).
* **Lock**:
  If we receive `NavRequest::Action` while this `Focusable` is
  focused, the navigation system will freeze until `NavRequest::Unlock`
  is received, sending a `NavEvent::Unlocked`.
  \
  This is useful to implement widgets with complex controls you don't
  want to accidentally unfocus, or suspending the navigation system while
  in-game.

### `MenuSetting`

See `bevy-ui-navigation` [`MenuSetting`] doc.

The public API has `MenuSetting`, but we use internally [`TreeMenu`],
this prevents end users from breaking assumptions about the menu trees.
[More details on this decision](#spawning-menus).

A menu that isolate children [`Focusable`]s from other
focusables and specify navigation method within itself.

A [`MenuSetting`] can be used to:
* Prevent navigation from one specific submenu to another
* Specify if 2d navigation wraps around the screen.
* Specify "scope menus" such that a
  `NavRequest::ScopeMove` emitted when
  the focused element is a [`Focusable`] nested within this `MenuSetting`
  will navigate this menu.
* Specify _submenus_ and specify from where those submenus are reachable.
* Specify which entity will be the parents of this [`MenuSetting`], see
  `MenuSetting::reachable_from` or `MenuSetting::reachable_from_named` if you don't
  have access to the `Entity`
  for the parent [`Focusable`]

If you want to specify which [`Focusable`] should be
focused first when entering a menu, you should mark one of the children of
this menu with `Focusable::prioritized`.

#### Limitations

Menu navigation relies heavily on the bevy hierarchy being consistent.
You might get inconsistent state under the folowing conditions:

- You despawned an entity in the `FocusState::Active` state
- You changed the parent of a [`Focusable`] member of a menu to a new menu.

The navigation system might still work as expected,
however, `Focusable::state` may be missleading
for the length of one frame.

#### Panics

**Menu loops will cause a panic**.
A menu loop is a way to go from menu A to menu B and
then from menu B to menu A while never going back.

Don't worry though, menu loops are really hard to make by accident,
and it will only panic if you use a `NavRequest::FocusOn(entity)`
where `entity` is inside a menu loop.

#### Fields

```rust
pub struct MenuSetting {
    /// Whether to wrap navigation.
    ///
    /// When the player moves to a direction where there aren't any focusables,
    /// if this is true, the focus will "wrap" to the other direction of the screen.
    pub wrapping: bool,
    /// Whether this is a scope menu.
    ///
    /// A scope menu is controlled with [`NavRequest::ScopeMove`]
    /// even when the focused element is not in this menu, but in a submenu
    /// reachable from this one.
    pub scope: bool,
}
```

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
[`bevy-ui-navigation`] uses a proxy holding both the parent
and the menu state info.

That proxy component is `MenuBuilder`,
you can initialize it either with the `Name` or `Entity` of the parent focusable of the menu.
A system will take all `MenuBuilder` components and replace them with a `TreeMenu`.

This also enables front-loading the check of whether the parent focusable
`Entity` is indeed a `Focusable`.
Before turning the `MenuBuilder` into a `TreeMenu`,
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
Another system will be in charge of unlocking the UI by sending a `NavRequest::Unlock`.

It is also possible to send lock requests with `NavRequest::Lock`.

The default input sets the escape key to send a `NavRequest::Unlock`,
but the user may chose to `Unlock` through any other mean.

This might be better served to a locking component that block navigation
of specific menu trees,
but I didn't find the need for such fine-grained controls.


### Input customization

As a default bevy plugin, letting user change defaults is a bit more tricky,
here is how it is done is the current implementation:

```rust
App::new()
    .add_plugins_with(DefaultPlugins, |group| {
        group
            // Add your own cursor navigation system
            // by using `NavigationPlugin::<MyOwnNavigationStrategy>::new()`
            // See the [`bevy_ui_navigation::MenuNavigationStrategy`] trait.
            //
            // You can use a custom gamepad directional handling system if you want to.
            // This could be useful if you want such navigation in 3d space
            // to take into consideration the 3d camera perspective.
            //
            // Here we use the default one provided by `bevy_ui` because
            // it is already capable of handling navigation in 2d space
            // (even using `Sprite` over UI `Node`)
            .add(BevyUiNavigationPlugin::new())
            // Prevent `UiPlugin` from adding the default input systems for navigation.
            // We want to add our own mouse input system (mouse_pointer_system).
            .add(UiPlugin {
                default_navigation: false,
            })
    })
    // Since gamepad input already works for Sprite-based menus,
    // we add back the default gamepad input handling from `bevy_ui`.
    // default_gamepad_input depends on NavigationInputMapping so we
    // need to also add this resource back.
    .init_resource::<NavigationInputMapping>()
    // can manually add back the systems defined in bevy_ui_navigation
    .add_system(default_gamepad_input.before(NavRequestSystem))
    // And add user-defined one as well. In this example,
    // we removed the default mouse input system and replaced it with our own.
    .add_system(mouse_pointer_system.before(NavRequestSystem))
```

### What to focus first?

Gamepad input assumes an initially focused element
for navigating "move in direction" or "press action".
At the very beginning of execution,
or after having despawned the focused element,
there are no focused element, so we need fallbacks.

By default, it is any `Focusable` when none are focused yet.
But the user can spawn a `Focusable` as `prioritized`
and the algorithm will prefer it when no focused nodes exist yet.

```rust
// The focused focusable if it exists.
focused
    // Any focusable in the "active" menu, if there is such a thing
    .or_else(any_in_active)
    // Any focusable marked as prioritized if there is one
    .or_else(any_prioritized)
    // Any focusable in the root menu, if there is one
    .or_else(any_in_root)
    // Just any focusable at all, if there is one
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

## Closed, interesting questions

### Game-oriented assumptions

This was not intended during implementation, but in retrospect, the `MenuSetting`
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

In this example, we can imagine the root `NodeBundle` having a `MenuSetting`
component, and the elements highlighted in yellow have a `Focusable` component.
This way we create a menu to navigate between dockers. This is perfectly
consistent with our design.

### Moving UI camera & 3d UI

The completely decoupled implementation of the navigation system
enables user to implement their own UI.
It is perfectly possible to add a `Focusable` component to a 3d object
and, for example, provide a [mouse picking] based input system
sending the `NavRequest`,
while using `world_to_viewport` in `MoveParam` for gamepad navigation.
(or completely omitting gamepad support by providing a stub `MoveParam`)

#### How does this work with `FocusPolicy`?

`FocusPolicy` is a way to prevent or allow mouse pointer "hover"
to go to an UI element below the one in front of the camera.
Think of it as a way to make "transparent" or "opaque" to mouse cursor cast
queries selectively UI elements.

Since navigation is orthogonal to cursor pointing, we keep the `FocusPolicy`.
If the policy is set to `Capture`, the element will be focused by hover.
If the policy is set to `Pass`, the pointing algorithm tries to find another
element behind it.

There is a major difference though:
There could be multiple objects under the cursor,
therefore several could be "active" at a time.
The navigation design described in this RFC disallows
multiple focused elements at a time.
This is a semantic change to `Interaction`, which is removed,
the user would have needed to migrate to `Focusable` already,
so the impact of this change is nil.

### Panics on menu loop

It is possible to define a cycle of menu connections,
and this will cause a panic at run time.

However, it's hard to define a menu cycle by accident.
Because the `Entity` used to access a menu is a property of the menu itself,
So it can only have a single parent. Which makes it harder to define a loop.

It is still possible to define a loop, and this will cause a panic the moment
the loop is entered.

## Future possibilities

### New and better mouse picking!

Since the focus algorithm does not handle input, it is not relevant
to how we chose which entity is focused.

The input methods can be interchanged without having to change or update
the navigation plugins.

### Multiple cursors

A non-so-uncommon pattern of video games (especially split-screen)
is to have two different players handle two different cursors on screen.

The current design assumes a unique cursor navigating the menus
and can't account for multi-cursor.
However, it seems not fundamentally impossible to add multiple cursors.

It would require modifying the `NavRequest` to include a sort of ID
to identify which cursor originated the request,
and keeping track in the `FocusState` which cursor the `Focused`, `Activated`
and `Prioritized` variants are for.

### Higher level menu wrapper

A visual wrapper for menu navigation that handles `display` `Style`s of menus
when they are entered and left.

### Tab navigation

We should add an optional navigation system
that allows navigating through `Focusable`s using keyboard, such as tab.

### User placement of the `TreeMenu` insertion systems

It should be possible for the user to add the insertion system wherever they
want, this would allow lower latency in spawning the UI.

However, those systems currently depend on private types,
this requires a design rethinking to allow to expose those systems without
exposing internals that will blow up in the user's face.

### Optimization

I took care to avoid quadratic behaviors in the navigation algorithm,
but the linear constant factor is fairly high:

- Multiple iteration of the list of `Focusable`s at times
- a few allocations per `NavRequest`
- multiple recursive exploration of hierarchy tree at times

### More robustness

As mentioned in [a future section](#Menu-hierarchy-invariants),
the menu system is frail to changes to the hierarchy,
mostly affecting `Active` menus.

However, I'm not sure how bad the hierarchy changes affect navigation,
and it would be a mistake to "not panic at all cost."
A panic is an opportunity to teach the user a better way of doing things.
Not panicking might result in making ui-navigation harder to use.


## Drawbacks and design limitations

### Major differences with the current focus implementation

#### `Hovered` is not covered

Since navigation is completely decoupled from input and ui lib,
it is impossible for it to tell whether a focusable is hovered.
To keep the hovering state functionality,
it will be necessary to add it as an independent component
separate from the navigation library.

A fully generic navigation API cannot handle hover state.
This doesn't mean we have to entirely give up hovering.
It is simply added back as a component independent
from the focus system.

Note that the new `Hover` component does not comply with `FocusPolicy`.
This allows reducing the complexity of the focus code.

#### `Clicked` is not a focus state

Compared to the current `Interaction` system, the new focus system
separates semantics of "action" events and "UI focus state."

Meaning that `Interaction::Clicked`'s equivalent in the new navigation system
is the `NavEvent::NoChanges` event.

This is important, as code relying on the "Clicked" focus state
will need to be replace with reaction to a `NavEvent`.

`Clicked` also allowed to change the button style
when the mouse button is held down.

We somewhat emulate that by first providing focus on pressing down
and sending `NavRequest::Action` on mouse up.

However, the corresponding `NavEvent` for button activation will only be
sent after the `NavRequest::Action` is sent.
Which means, activation happens when before `Interaction`
went from `Clicked` to `Hovered`.

Looking at the code changes in the existing UI examples in [this RFC's PR]
is a good indicator at how the code will need to be changed.

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

### Menu hierarchy invariants

For menus to be optimally ergonomic to declare and use,
we chose which `Focusable` is in a menu by looking at the bevy hierarchy.

This has downsides, mainly, that the user can at any time completely change the
hierarchy: despawn entities, change their parent, remove their parents etc.

It is impossible to anticipate how the user will change the menu hierarchy,
yet we rely on it.

This problem only shows up when using menus, therefore,
it will likely only be a concern for more complex games with navigable menus.

In addition, because it's an _opt in_ feature,
we can advertise the dangers of manipulating the hierarchy of menus
in the documentation for the components and bundles used to create menus.

The failure mods are unclear,
the implementation might in fact be resilient to hierarchy changes.

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
[`MenuSetting`]: https://docs.rs/bevy-ui-navigation/latest/bevy_ui_navigation/enum.NavMenu.html
[`UiProjectionQuery`]: https://github.com/nicopap/ui-navigation/blob/17c771b7f752cfd604f21056f9d4ca6772529c6f/src/resolve.rs#L378-L429
[`TreeMenu`]: https://github.com/nicopap/ui-navigation/blob/30c828a465f9d4c440d0ce6e97051a5f7fafa425/src/resolve.rs#L201-L216
[this RFC's PR]: https://github.com/bevyengine/bevy/pull/5378

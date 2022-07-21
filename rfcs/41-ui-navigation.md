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

## Motivation

`bevy_ui` interaction story is crude.
The only way to interact with the UI currently
is through the `Interaction` and `FocusPolicy` components.
`Interaction` only supports mouse or touch-based input.
We want gamepad supports and more game-oriented functionalities,
such as menu trees and movement between menus.

The current UI focus system is limited to `bevy_ui` and
can't be swapped out for something the user deems more appropriate.

This is in contradiction with the general philosophy of bevy,
which is of providing to the user all the tools necessary to create something
that fits their needs optimally.

This RFC proposes to resolve all those questions.
[`bevy-ui-navigation`] provides a highly customizable ECS-based navigation engine.

## User-facing explanation

The [bevy-ui-navigation README][`bevy-ui-navigation`] is a good start.
To summarize, here are the headlines:
- The navigation system can only be interacted with with events:
  `NavRequest` as input and `NavEvent` as output.
- `NavEvent::FocusChanged` contains the list of entities that are traversed
  to go from the last focus position to the new one.
- ui-navigation provides default input systems, 
  but it's possible to disable them and instead use custom ones.
  This is why we restrict interactions with navigation to events.
- ui-navigation provides a system to declare buttons as "leading to menu X"
  in order to create menu trees.
- It is possible to create isolated menus by using the `NavMenu` component.
- All `Focusable` children in the `NavMenu` entity's tree will be part
  of this menu, and moving using a gamepad from one focusable in a menu
  can only lead to another focusable within the same menu.
  You can specify a focusable in any other menu that will lead to this
  one once activated.

## Implementation

Since `bevy-ui-navigation` already exists,
we will discuss its design, it's functionalities and their uses.

Implementation details are already covered by the implementation,
which btw is fully documented (including internal architecture).

### Exposed API

The crux of `bevy-ui-navigation` are the following types:
- `Focusable` component
  - `lock`: activating it will lock the navigation engine and discard all `NavRequest`.
  - `cancel`: activating it will do the equivalent of receiving a `Cancel` request.
  - `dormant`: this will be the preferred entity to focus when entering a menu.
- `NavMenu` enum
  - `Wrapping**`/`Bound**`: navigation can wrap
    if directional sticks are pressed in a direction there are no focusables in
  - `**Scope`/`**2d`: A scopped menu is tab menu navigable with special hotkeys.
- `NavRequest` event
  - `Action`: equivalent to left click or `A` button on controller
  - `Cancel`: backspace or `B` on controller
  - `FocusOn`: request to move the focus to a specific `Entity`
  - `Move(Direction)`: directional gamepad or keyboard input
  - `ScopeMove(ScopeDirection)`: tab menu hotkey
- `NavEvent` event
  - `InitiallyFocused`: We went from 0 focused to 1 focused element
  - `NoChanges`: The `NavRequest` did not change the focus state.
  - `FocusChanged`: The `NavRequest` changed the focus state,
    has the list of items that were made inactive
    and items made active.
  - `Locked`: The `NavRequest` caused the focus system to lock.
  - `Unlocked`: Something unlocked the focus system.
- `MoveParam` trait
  - Used to pick a focusable in a provided direction
  - The `NavigationPlugin` is generic over the `MoveParam`
  - `DefaultNavigationPlugins` provides a default implementation of `MoveParam`

### `Focusable`

A `Focusable` is an entity that can be focused.
For gamepad-based navigation,
this requires a way to pick a focusable in a provided direction.
This is delegated to the UI library implementation
through the `MoveParam` trait.
In the case of bevy, `bevy_ui` is the UI library implementation,
and the `GlobalTransform` is used to locate focusables in regard to each-other.

The `Focusable` component holds state about what happens when it is activated
and what focus state it is in. (focused, active, inactive, etc.)

#### Note on hovering state

Since navigation is completely decoupled from input and ui lib,
it is impossible for it to tell whether a focusable is hovered.
To keep the hovering state functionality,
it will be necessary to add it as an independent component
separate from the navigation library.

### Input customization

`ui-navigation` can be added to the app in two ways:
1. With `NavigationPlugin`
2. With `DefaultNavigationPlugins`

(1) only inserts the navigation systems and resources,
while (2) also adds the input systems generating the `NavRequest`s
for gamepads, mouse and keyboard.

This enables convenient defaults
while letting users insert their own custom input logic.


### What to focus first?

Gamepad input assumes an initially focused element
for navigating "move in direction" or "press action".
At the very beginning of execution, there are no focused element,
so we need fallbacks.
By default, it is any `Focusable` when none are focused yet.
But the user can spawn a `Focusable` as `dormant`
and the algorithm will prefer it when no focused nodes exist yet.

```rust
self.focusables
    .iter()
    // The focused focusable.
    .find_map(|(e, focus)| (focus.state == Focused).then(|| e))
    // Dormant focusable within the root menu (if it exists)
    .or_else(root_dormant)
    // Any dormant focusable
    .or_else(any_dormant)
    // Any focusable
    .or_else(fallback)
```


### Menu navigation handling

The public API has `NavMenu`, but we use internally `TreeMenu`,
this prevents end users from breaking assumptions about the menu trees.

We define the `TreeMenu` component.
All `Focusable` direct or indirect children of a `TreeMenu` are part of that menu.
A `TreeMenu` has a `Focusable` focus parent,
when that `Focusable` is focused and the `Action` request is received,
the focus changes to the `TreeMenu`

In short, the navigation tree is built on two elements:
- The `reachable_from` that a `NavMenu` was spawned with
- The `Focusable` within the hierarchy of a `NavMenu`

This is a tree in the ECS.

In the following screenshot, `Focusable`s are represented as circles, and menus
as rectangles.

![A screenshot of a RPG-style menu with the navigation tree overlayed](https://user-images.githubusercontent.com/26321040/141671969-ea4a461d-0949-4b98-b060-c492d7b896bd.png)

A `Focusable` used to go from the root menu
to the menu in which the currently focused entity is is _active_.
The current focus position is represented as a trail of breadcrumb active
focusables and the currently focused focusable.

This is reflected into the `NavEvent::FocusChanged` event.
When going from a focusable to another,
the navigation system emits a `FocusChanged` event,
the event contains two lists of entities:
- The list of entities that after the navigation event are no more focused
  or active
- The list of entities that are newly active or focused

For the simplest case of navigating from one focusable to another
in the same menu, the two list have a single element,
so it is capital that it is easy to access those elements.
This is why we use a `NonEmpty`, from the `non_empty_vec` crate.
A `NonEmpty<T>` has a `first` method that returns a `T`,
which in our case would be the last and new focused elements.
I chose this crate over similar ones,
because the code is dead simple and easy to vet.

#### Going back to a previous menus

`Focusable`s have a `dormant` state that is set when they go from
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
`ui-navigation` uses a proxy component holding both the parent and the menu state info.
That proxy component is `MenuSeed`,
you can initialize it either with the `Name` or `Entity` of the parent focusable of the menu.
A system will take all `MenuSeed` components and replace them with a `TreeMenu`.

This introduces a single frame lag on menu spawn,
but it improves ergonomics to users.
I thought it was a pretty good trade-off,
since spawning menus is not time critical, unlike input.


#### cancel events

When entering a menu, you want a way to leave it and return to the previous menu.
To do this, we have two options:
- The `Cancel` request
- The `cancel` focusable

This will change the focus to the parent button of the focused menu.

#### Moving into a menu

When the `Action` request is sent while a `Focusable` with a child menu is focused,
the focus will move to the child menu,
selecting the focusable to focus
with a heuristic similar to the initial focused selection.

### Locking

To ease implementation of widgets, such as sliders,
some focusables can "lock" the UI, preventing all other form of interactions.
Another system will be in charge of unlocking the UI by sending a `NavRequest::Free`.

This may be better served by keeping the `FocusPolicy` component,
in situations where a global lock prevents other uses.


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

## Open questions

### `NavRequest` system

I wanted to defer the raw input processing to the user. The current
implementation provides default systems to manage ui input, with a
`InputMapping` resource to customize a little bit the system's behaviors. 

### Game-oriented assumptions

This was not intended during implementation, but in retrospect, the `NavMenu`
system is relatively opinionated.

Indeed, we assume we are trying to build a game UI with a series of "menus" and
that it is possible to keep track of a "trail" of buttons from a "root menu" to
the deepest submenu we are currently browsing.

The example of a fancy editor with a bunch of dockered widgets seems to break
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
However, in practice, in my own games, I worked around it by expanding the
whole UI tree beyond the UI camera view and moving around the camera.
An alternative would simply to set the `style.display` of non-active menus to
`None` and change the style when focus enters them.
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

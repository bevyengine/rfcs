# Feature Name: Generic UI Navigation API

## Summary

Modification of the `Interaction` component to support keyboard & controller
based navigation of UI on top of mouse-oriented navigation. In the rest of this
RFC, consider the already existing "`Interaction`" bevy component as being a good
candidate for being replaced by what we describe as "`Focusable`".

The new navigation system by default enables unrestricted navigation between
all `Focusable` entities. But the developer may chose to opt in to a more
complex navigation scheme by adding `NavMenu`, encapsulating groups of
`Focusable`s into their own menus. The `NavMenu` component can also express
the navigation scheme within the menus.

The goal is to make the easiest navigation scheme work out of the box and the
more complex schemes easy or at least possible to implement.

## Terminology

* A **`NavRequest`** is the sum of all possible ui-related user inputs. It's what
  the algorithm takes as input, on top of the _navigation tree_ to figure out
  the next focused element.
* A **`Focusable`** is an Entity that can be navigated-to and highlighted (focused)
  with a controller or keyboard
* A **`NavMenu`** adds "scope" to `Focusable`s. Navigation between `Focusable`
  entities will be constrained to other `Focusable` that are children of the
  same `NavMenu`. It creates a self-contained menu. **A `NavMenu` may therefore
  be better described as a _menu_**. Note that the `Focusable`s do not need to
  be direct children of the `NavMenu`, only transitive children[^1]
* The **menu tree** (sometimes _navigation tree_) is the logical relationship
  between all `NavMenu`s.
* There is a single **focused** Entity, which is the currently highlighted/used
  Entity.
* There can be several **active** elements, they represent the "path" through the
  _navigation tree_ to the current _focused_ element. Which focusable in the
  previous menus were activated in order to reach the menu we are in currently.
* A **dormant** element is a previously active element from a branch of the
  menu tree that is currently not focused.
* The navigation algorithm **resolves** a `NavRequest` by finding which `Entity`
  to next focus to
* **focus memory** is what, for example, makes it possible in your text editor
  to tab to a different file, edit it, and tab back to the file you were
  previously editing and find yourself at the exact location where you left it.
  It is the ability for the software to keep track where you were at.
  **dormant** elements are necessary to track what was previously focused and
  come back to it when needed

## Motivation

Currently, the developer must themselves write the relationship between
various buttons in the menus and specify which button press leads to other menu
items, etc. Or more simply, limit themselves to pointer-based navigation to
avoid having to think about menu navigation.

By including navigation-related components such as `Focusable` and `NavMenu`,
the developer can quickly get a controller-navigable menu up and running.

There already exists components with similar purposes in Bevy:
[`Interaction` and `FocusPolicy`](https://github.com/bevyengine/bevy/blob/main/crates/bevy_ui/src/focus.rs#L14).
However, it only supports a barebone mouse-oriented navigation scheme. And the
focus handling is not even implemented.

My proposal, on top of making it easy to implement controller navigation of
in-game UIs, enables keyboard navigation and answers accessibility concerns
such as the ones
[raised in this issue](https://github.com/bevyengine/bevy/issues/254).

## User-facing explanation

Examples and an implementation-oriented description is available in the
[ui-navigation plugin README](https://github.com/nicopap/ui-navigation/blob/master/Readme.md).

To interact in real-time with the UI, the developer emits `NavRequest`s events.
The navigation system responds with `NavEvent` events, such as
`NavEvent::FocusChanged` or `NavEvent::Caught`. 

The navigation system is the only one to handle the focus state of the UI. The
user interacts with it strictly by sending `NavRequest`s, reading `NavEvent` or
using the `Focusable` methods. Typically in a system with a
`Query<&Focusable, Changed<Focusable>>` parameter.

Rather than directly reacting to input, we opt to make our system reactive to
`NavRequest`, this way the developer is free to chose which button does what in
the game. The developer also has the complete freedom to swap at runtime what
sends or how those `NavRequest` are generated.

### Example

Let's start with a practical example, and try figure out how we could specify
navigation in it:

![Typical RPG tabbed menu](https://user-images.githubusercontent.com/26321040/140716885-eb089626-21d2-4711-a0c9-cf185bc0f19a.png)

The tab "soul" and the panel "abc" are highlighted to show the player what menu
they are down, they are **active** elements. The circle "B" is highlighted
differently to show it is the **focused** element.

You can move `UP`/`DOWN` in the submenu, but you can also use `LT`/`RT` to
go from one submenu to the other (the _soul_, _body_ etc tabs). In code, we
might react to focus changes (for example to change the color of
focused/unfocused elements) as follow:
```rust
fn setup_ui() {
  // setup UI like you would setup UI in bevy today
}
fn handle_nav_events(mut events: EventReader<NavEvent>, game: Res<Gameui>) {
    use bevy_ui_navigation::{NavEvent::Caught, NavRequest::Action};
    for event in events.iter() {
        match event {
            Caught { from, request: Action } if from.first() == game.start_game_button => {
                  // Start the game on "A" or "ENTER" button press
            }
            _ => {}
        }
    }
}
fn button_system(
    materials: Res<Materials>,
    mut focusables: Query<(&Focusable, &mut Handle<ColorMaterial>), Changed<Focusable>>,
) {
    for (focus_state, mut material) in focusables.iter_mut() {
        if focus_state.is_focused() {
            *material = materials.focused.clone();
        } else {
            *material = materials.inert.clone();
        }
    }
}
```

## Implementation strategy

### Multiple menus

The first intuition is that this screenshot shows three menus:
* The "tabs menu" containing _soul_, _body_, _mind_ and _all_ "tabs"
* The "soul menu" containing _gdk_, _kfc_ and _abc_ "panels"
* The "ABC menu" containing a, b, c, A, B, C "buttons"

We expect to be able to navigate the "tabs menu" **from anywhere** by pressing
`LT`/`RT`. (or `TAB`,`SHIFT+TAB` on keyboard) We expect to be able to move
between buttons within the "ABC menu" with arrow keys, same with the "soul menu".

Finally, we expect to be able to go from one menu to another by pressing `A`
when the corresponding element in the parent menu is _focused_ or go back to
the parent menu when pressing `B`. (`ENTER` and `BACKSPACE` on keyboard)

### Navigating within a single menu

Our game engine already knows the position of our UI elements. We can use this
as information to deduce the next _focused_ element when pressing an arrow key.

In fact, if we have only a single menu on screen, we could stop at this point,
since there is no restrictions on navigation between `Focusable`s.

### Specifying navigation between menus

But we have more than one menu. We can't magically infer what the developer
thinks is a menu, this is why we need `NavMenu`s.

In our example, we want to be able to go from "soul menu" to "ABC menu" with
`A`/`B` (Action, Cancel). `Action` let you enter the related submenu, while
`Cancel` does the opposite.

With `NavMenu`s the menu layout may look like this:

![The previous tabbed menu example, but with _menus_ and _focusables_ highlighted](https://user-images.githubusercontent.com/26321040/141671969-ea4a461d-0949-4b98-b060-c492d7b896bd.png)

The _menus_ are represented as semi-transparent squares; the _focusables_
are circles.

A `NavMenu` keeps track of the Entity `Focusable` that leads to it (in the
example, the "soul" tab entity is tracked by the "soul menu" `NavMenu`).

The `NavMenu` isolates the set of `Focusable`s of a single menu:

![Graph menu layout](https://user-images.githubusercontent.com/26321040/140716920-fd298afb-093b-47f9-8309-c4c354c3d40f.png)
(red-orange represents _focused_ elements, gold are _active_ elements)

In bevy, a `NavMenu` might be nothing more than a component to add to the
`NodeBundle` parent of the _focusables_ you want in your menu. On top of that,
it needs to track which _focusable_ you _activated_ to enter this menu:
```rust
#[derive(Component)]
struct NavMenu {
  parent_focusable: Option<Entity>,
}
```

### `NavMenu` settings

#### Scope menu

However, it's not enough to be able to go up and down the menu hierarchy, we
also want to be able to directly "speak" with the tabs menu and switch from the
"soul menu" to the "body menu" with a simple press of `RT`, not having to
press `B` twice to get to the tabs menu.

The tabs menu is a special sort of menu, with movement that can be triggered
from another menu reachable from it. We must add a `setting: NavSetting` field
to the `NavMenu` component to distinguish this sort of menu. In our
implementation we call them `scope menu`s.

The `NavRequest::ScopeMove` request associated with `RT`/`LT` (or anything the
developer sets) will climb up the _menu tree_ to find the enclosing
_scope menu_ and move focus within that menu.

#### Looping

We may also chose to lock or unlock the ability to wrap back right when going
left from the leftmost element of a menu, same with any other directions. For
example pressing `DOWN` when "c" is _focused_ could change focus to "a". We
should use the `setting` field to keep track of this as well.

### Navigation tree

All in all, the navigation tree is just the specification of which menu is
reachable from where. Not the relation between each individual focusable. This
drastically simplifies the structure of the tree. Because a game may have many
interactive UI elements, but not that many menus.

The previous graph diagram may have been a bit misleading. It's not the real
_navigation graph_ (that we introduced as _menu graph_ for good reasons).
The tree that the algorithm deals with and traverse is between menus
rather than `Focusable` elements. The graph as it is used in our code is the
following:

![RPG menu demo split between menus with pointers leading to the "parent
focusable"](https://user-images.githubusercontent.com/26321040/141672009-e023855e-10a7-4c50-a8dd-eff10acb2208.png)

We may also imagine an implementation where the non-selected menus are hidden
but loaded into the ECS. In this case, the navigation tree might look like
this:

![RPG menu graph without focusables, only
menus](https://user-images.githubusercontent.com/26321040/141672033-8bd43660-78af-4521-b367-03c7270c58c5.png)

### Algorithm

The algorithm is the `resolve` function in [ui-navigation](https://github.com/nicopap/ui-navigation/blob/master/src/lib.rs#L414).
The meat of it is 50 lines of code.

The current implementation has an example demonstrating all features we
discussed in this RFC. For a menu layout as follow:

![Example layout](https://user-images.githubusercontent.com/26321040/141671978-a6ef0e99-8ee1-4e5f-9d56-26c65f748331.png)

We can navigate it with a controller as follow:

![Example layout being navigated with a controller, button presses shown as
overlay](https://user-images.githubusercontent.com/26321040/141612751-ba0e62b2-23d6-429a-b5d1-48b09c10d526.gif)

### UI Benchmarks discussion

As mentioned in [this RFC](https://github.com/alice-i-cecile/rfcs/blob/ui-central-dogma/rfcs/ui-central-dogma.md),
we should benchmark our design against actual UI examples. I think that the
one used in this RFC as driving example (the RPG tabbed menu) illustrates well
enough how we could implement Benchmark 2 (main menu) using our system.
Benchmark 1 is irrelevant. Benchmark 3 could potentially be solved by just
marking the interactive element with the `Focusable` component and nothing
else, so that every element can be reached from anywhere.


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

We **requires** the user to explicitly add the input systems to their `App`
rather than adding them to our `NavigationPlugin` because we **need** the
flexibility to swap out the default input handling after a certain point in the
game development phase (as my personal practical experience shows).

Though this is a very basic solution that screams "temporary", and should
probably in the future be swapper in favor of something more thought out and
ergonomic.

How would that work? Is there already examples in bevy where we provide default
button mappings that can be changed?

I think we should think deeply about input handling in default plugins.
However, this is out of scope for this RFC.


### Mouse hover behavior

I went the easy route when implementing mouse support, of having the focus
follow the mouse. In certain circumstances this is not wishable, but currently
I don't see an alternative that doesn't need to completely reinvent a new
system on top of the current one.


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


## Drawbacks and design limitations

### User upheld invariants

I think the most problematic aspect of this implementation is that we push on
the user the responsibility of upholding invariants. This is not a safety
issue, but it leads to panics. The invariants are:
1. Never create a cycle of menu connections
2. There must be at most one root `NavMenu`
3. Each `NavMenu` must at least have one contained `Focusable`

The upside is that the invariants become only relevant if the user _opts into_
`NavMenu`. Already, this is a choice made by the user, so they can be made
aware of the requirements in the `NavMenu` documentation.

If the invariants are not upheld, the navigation system will simply panic. I
didn't test enough the current implementation to see if all cases of panics
lead to meaningful error messages.

It may be possible to be resilient to the violation of the invariants, but I
think it's better to panic, because violation will most likely come from
misunderstanding how the navigation system work or programming errors.

### Jargon

The end user will have to deal with new concepts introduced by this RFC:
* `dormant`
* `active`
* `focused`
* `NavRequest`
* `NavEvent`
* `NavMenu`
* `Focusable`
* `cycling`
* `scope menu`

This seems alright, although `dormant` may be misleading, given it's a synonym
to "inactive". It is technically an antonym to "active", but in the current
design, it is not really the case. Beside I have a tendency to type "doormat"
instead.

On the implementation side. We have `inert` which is synonym to `dormant` and
`inactive` but is just a word to fill the roll of "none of the above".

I think the one term that can be improved is `NavMenu`. Replacing it with
`NavMenu` constructs on pre-existing knowledge and is a good approximation of
what it represents.

### Performance

The navigation system might be slow with a very large amount of `Focusable`
elements. **I didn't benchmark it**. In fact the `ultimate_menu_navigation.rs`
example is laggy. Especially visible with mouse focus. I'm not sure why. It
seems to be because of the large number of `Focusable` and doing geometry
computation on all of them (which is necessary to find the next element to
focus or check which element is under the cursor)

Note that the current implementation only performs computation on `NavRequest`
events. The input systems might add their own processing. I tried to minimise
as much as possible the `default_mouse_input` runtime, but it still seem quite
laggy.

We might improve the performance with well designed 2d navigation caches. But
i'm not sure how this would look like.

[^1]: ie: includes children, grand-children, grand-grand-children etc.

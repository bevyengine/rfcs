# Feature Name: Generic UI Navigation API

## Summary

A set of components and a new system to define navigation nodes for the ECS UI.

## Motivation

On top of making it easy to implement controller navigation of
in-game UIs, it enables keyboard navigations and answers accessibility concerns
such as the ones
[raised in this issue](https://github.com/bevyengine/bevy/issues/254).

## User-facing explanation

### Terminology

* A **`NavRequest`** is the sum of all possible ui-related user inputs. It's what
  the algorithm takes as input, on top of the _navigation tree_ to figure out
  the next focused element.
* A **`Focusable`** is an Entity that can be navigated-to and highlighted (focused)
  with a controller or keyboard
* A **`NavFence`** adds "scope" to `Focusable`s. All `Focusable`s children of a
  `NavFence` will be isolated from other `Focusable` and will require special
  rules to access and leave.
* The **navigation tree** is the logical relationship between all `NavFence`s.
  `Focusables` are not really part of the tree, as you never break out of the
  `NavFence` without a special `NavRequest`
* There is a single `Focused` Entity, which is the currently highlighted/used
  Entity.
* There can be several **`Active`** elements, they represent the "path" through the
  _navigation tree_ to the current `Focused` element (there isn't an exact 1
  to 1 relationship, see following examples)
* The navigation algorithm **resolves** a `NavRequest` by finding which `Entity`
  to next focus to
* The **active trail** is all the `Active` elements from the outermost
  `NavFence` up to the innermost `NavFence` containing the `Focused` element.

### Game developer interactions with the system

We add a _navigation tree_ to the UI system, the tree will be used to enable
key-based (and joystick controllable) navigation of UI, by allowing users to
rotate between different UI elements.

Currently, the developer must themselves write the relationship between
various buttons in the menus and specify which button press leads to other menu
items, etc. Or more simply, limit themselves to pointer-based navigation to
avoid having to think about menu navigation.

By including navigation-related components such as `Focusable` and `NavFence`,
the developer can quickly get a controller-navigable menu up and running.

We could also add the `Focusable` component to standard UI widget bundles such
as `ButtonBundle` and a potential `TextField`. This way, the developer _doesn't
even need to specify the `Focusable` elements_.

To interact in real-time with the UI, the developer emits `NavRequest`s events.
The navigation system responds with `NavEvent` events, such as
`NavEvent::FocusChanged` or `NavEvent::Caught`. 

The navigation system doesn't handle itself the state of the UI, but instead
emits the `FocusChanged` etc. events that the developer will need to read to,
for example, play an animation showing the change of focus in the game.

The game developer may also chose to listen to `Changed<Focusable>` and read
the `Focusable.is_active` and `Focusable.is_focused` fields.

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
go from one submenu to the other (the _soul_, _body_ etc tabs). This seems to
be a reasonable API to interact with the focus change events:
```rust
fn setup_ui() {
  // TODO: add an example in the ui-navigation repo and link to it
}
fn ui_events_system(mut ui_events: EventReader<NavEvent>, game: Res<Game>) {
  for event in ui_events.iter() {
    match event {
      NavEvent::Unresolved(NavRequest::Action, button) if button == game.ui.start_game_button => {
        // Start the game
      }
      _ => {}
    }
  }
}
fn ui_draw_system(focus_changes: Query<(Entity, &Focusable), Changed<Focusable>>) {
  for (button, focus_state) in focus_changes.iter() {
    if focus_state.is_focused {
      // Draw the button as being focused
    } else if focus_state.is_active {
      // etc.
    }
    // etc.
  }
}
```


## Implementation strategy

### Basic physical navigation

Let's drop the idea to change submenu with `LT`/`RT` for a while. Let's focus on
navigating the "ABC menu".

Typical game engine implementation of menu navigation requires the developer to
specify the "next", "left", "right", "previous" etc.[^1] relationships between
focusable elements. This is just lame! Our game engine not only already knows
the position of our UI elements, but it also has knowledge of the logical
organization of elements through the `Parent`/`Children` relationship.

Therefore, with the little information we provided through this hand wavy
specification, we should already have everything setup for the navigation to
just work™.


### Dimensionality

Let's go back to our ambition of using `LT`/`RT` for navigation now. 

UI navigation is not just 2D, it's N-dimensional. The
`LT`/`RT` to change tabs is like navigating in a 3rd orthogonal dimension to the
dimensions you navigate with `UP` `DOWN` `LEFT` `RIGHT` inside the menus. And to go
from one menu to another you most often press `A`/`B`.

We should also be able to "loop" the menu, for example going `LEFT` of "soul"
loops us back to "all" at the other side of the tab bar.

### Specifying navigation between menus

We posited we could easily navigate with directional inputs the menu based on
the position on-screen of each ui elements. This is not
the case for the other dimensions of navigation. We can't magically infer the
intent of the developer: They need to specify the dimensionality of their menus.

This is where `NavFence`s become useful. We could specify a fully navigable
menu with any arbitrary **graphical layout** but without any **navigation
layout** by not bothering to insert any `NavFence` component. But a serious™
game oftentimes has deep nested menus with arbitrary limitations on the
navigation layout to help the player navigate easily the menu.
The tabbed menu example could be from such a game.

In our example, we want to be able to go from "soul menu" to "ABC menu" with
`A`/`B` (Action, Cancel). `Action` let you enter the related submenu, while
`Cancel` does the opposite.

`NavFence` specify the level of nestedness of each menu elements.

![The previous tabbed menu example, but with _fences_ and _focusables_ highlighted](https://user-images.githubusercontent.com/26321040/140716902-bd579243-9cfa-4bdf-a633-572344e15242.png)

The _fences_ are represented as semi-transparent squares; the _focusables_
are circles.

How this relates to dimensionality becomes obvious when drawing the
relationship between menus as a graph. Here we see that `Action` goes down the
navigation graph, while `Cancel` goes up:

![Graph menu layout](https://user-images.githubusercontent.com/26321040/140716920-fd298afb-093b-47f9-8309-c4c354c3d40f.png)
(orange represents `Focused` elements, gold are `Active` elements)

In bevy, a `NavFence` might be nothing else than a component:
```rust
#[derive(Component)]
struct NavFence;
```

### Navigating the tab bar

However, it's not enough to be able to go up and down the menu hierarchy, we
also want to be able to directly "speak" with the tab menu and switch from the
"soul menu" to the "body menu" with a simple press of `RT` without having to
press `B` twice to get to the tab menu.

To solve this, we add a field to our `NavFence`, 
```rust
  sequence_menu: bool,
```
With this, when we traverse upward the navigation tree, for the `Next` and
`Previous` `NavRequest`, we check for this field and try to move within that
`NavFence`.


### Loops

TODO: specify how we solve and deduce loops

### Algorithm

This is a lot of words to describe something that is actually quite simple. An
implementation is available [in this
repo](https://github.com/nicopap/ui-navigation), but the focus resolution
algorithm can be copied here:
```rust
/// Change focus within provided set of `siblings`, `None` if impossible.
fn resolve_within(
    focused: Entity,
    request: NavRequest,
    siblings: &[Entity],
    transform: &Query<&GlobalTransform>,
) -> Option<Entity>;

fn resolve(
    focused: Entity,
    request: NavRequest,
    queries: &NavQueries,
    mut from: Vec<Entity>,
) -> NavEvent {
    let nav_fence = match parent_nav_fence(focused, queries) {
      Some(entity) => entity,
      None => return NavEvent::Uncaught { request, focused },
    };
    let siblings = children_focusables(nav_fence, queries);
    let focused = get_active(&siblings, &queries.actives);
    from.push(focused);
    match resolve_within(focused, request, &siblings, &queries.transform) {
        Some(to) => NavEvent::FocusChanged { to, from },
        None => resolve(nav_fence, request, queries, from),
    }
}
```
The `NavEvent` can then be used to tell bevy to modify the `Focusable`,
`Active` and `Focused` component as needed.


### UI Benchmarks discussion

As mentioned in [this RFC](https://github.com/alice-i-cecile/rfcs/blob/ui-central-dogma/rfcs/ui-central-dogma.md),
we should benchmark our design against actual UI examples. I think that the
one used in this RFC as driving example (the RPG tabbed menu) illustrates well
enough how we could implement Benchmark 2 (main menu) using our system.
Benchmark 1 is irrelevant. Benchmark 3 could potentially be solved by just
marking the interactive element with the `Focusable` component and nothing
else, so that every element can be reached from anywhere.


## Prior art

I've only had a cursory glance at what already exists in the domain. Beside the
godot example, I found the [Microsoft
`FocusManager` documentation](https://docs.microsoft.com/en-us/windows/apps/design/input/focus-navigation-programmatic)
to be very close to what I am currently doing ([additional Microsoft
resources](https://docs.microsoft.com/en-us/windows/apps/design/input/focus-navigation))

Our design differs from Microsoft's classical UI navigation in a few key ways.
We have game-specific assumptions. For example: we assume a "trail" of active elements
that leads to the sub-sub-submenu we are currently focused in, we rely on that
trail to navigate containing menus from the focused element in the submenu.

Otherwise, given my little experience in game programming, I am probably overlooking some
standard practices. My go-to source ("Game Engine Architecture" by Jason
Gregory) doesn't mention ui navigation systems.

## Rationale and alternatives

### More holistic approach

This system relieves the game developer from implementing themselves an often
tedious UI navigation system. However, it still asks them to manage the
graphical aspect of UI such as visual highlights, a more holistic approach
probably solves this, at the cost of being more complex and less flexible. Such
a system might be explored in the future, but for the sake of
having an implementation to work with, I think aiming for a smaller scope is a
better idea.

### As plugin or integrated in the engine?

I want to first draft this proposition as a standalone plugin ([see on
github](https://github.com/nicopap/ui-navigation)). The current
design doesn't require any specific insight into the engine that cannot be
accessed by a 3rd party plugin. However as I already mentioned, this solves 
[bevy-specific concerns](https://github.com/bevyengine/bevy/issues/254), and
bevy would benefit from integrating a good ui navigation system.

## Drawbacks and design limitations

### Unique hierarchy

The proposed navigation tree has a surprising design. The navigation tree only
manages the path to the current active element and the siblings of the
fences you have to traverse from the root to get to the focused element.

This has several drawbacks:
* The user must be aware and understand the expectations of the navigation
  algorithm. It's easy to detect violations of the expectations, communicating
  the fix to the user less so. Potentially we could use a runtime error or
  warning.
* We don't have any memory beside the activation trail. A good example of focus
  memory is a classical text editor with multiple tabs. When you go from Tab 2
  to Tab 1, and back from Tab 1 to Tab 2, you will find the text cursor at the
  same location you left it. This isn't possible with this system, as the only
  memory is from the root of the window to the current cursor.


I think this is fixable. It's possible to store the entire navigation tree. We
could simply keep track of the active trail as boolean flags on each branches.
The focus algorithm only cares about the active path, so there would be very
little to change on that side.

The focus memory use case is not important for most games. However it will
be necessary for any complex editor.

### No concurrent submenus

You have to "build up" reactively the submenus as you navigate through them,
since **there can only be a single path through the hierarchy**.

We expect that on `FocusChanged`, the developer adds the components needed if
the focus change leads to a new sub-menu.

I'm implementing first to see how cumbersome this is, and maybe revise the
design with learning from that.

The only motivation for this is that it makes it easier to
implement and reason about the system. It's far easier to implement. In fact
the current implementation doesn't even use a tree data type, but a vector of
vectors where each row represents a layer of _focusables_.

It's possible to keep more than one `NavFence` within other fences.
The solution is that of [the previous section](#unique-hierarchy).


### Inflexible navigation

The developer might want to add arbitrary "jump" relations between various
menus and buttons. This is not possible with this design. Maybe we could add a
```rust
  bridges: HashMap<NavRequest, Entity>,
```
field to the `NavFence` to reroute `NavRequest`s that failed to change focus
within the menu. I think this might enable cycling references within the
hierarchy and break useful assumptions.

### Mouse support

We should add mouse support (pointer devices). Though it is not at all
mentioned in this RFC, it is self-evident that integrating the navigation-based
focus with pointer-based focus is the way forward.

## Isn't this overkill?

Idk. My first design involved even more concepts, such as `Fence`s and `Bubble`
where each `NavFence` could be open or closed to specific types of `NavRequest`s.
After formalizing my thought, I came up with the current design because it
solved issues with the previous ones, and on top of it, required way less
concepts.

For the use-case I considered, I think it's a perfect solution. It seems to
be a very useful solution to less complex use cases too. Is it overkill for
that? I think not, because we don't need to define relations at all, especially
for a menu without any hierarchy. On top of that, I'm not creative
enough to imagine more complex use-cases.

[^1]: See [godot documentation](https://github.com/godotengine/godot-docs/blob/master/tutorials/ui/gui_navigation.rst)
[^2]: ie: includes children, grand-children, grand-grand-children etc.

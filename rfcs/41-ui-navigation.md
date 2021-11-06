# Feature Name: Generic UI Navigation API

## Summary

A set of components and a new system to define navigation nodes for the ECS UI.

## Motivation

On top of making it easy to implement controller navigation of
in-game UIs, it enables keyboard navigations and answers accessibility concerns
such as the ones
[raised in this issue](https://github.com/bevyengine/bevy/issues/254).

## User-facing explanation

We add a _navigation tree_ to the UI system, the tree will be used to enable
key-based navigation of the specified UIs.

Currently, the developer must themselves write the relationship between
various buttons in the menus and specify which button press leads to other menu
items, etc. Or more simply, limit themselves to pointer-based navigation to
avoid having to think about menu navigation.

By including navigation-related components such as `Navigable` and `Container`,
the developer can quickly get a controller-navigable menu up and running. The
`Container` accepts various `Plane`s to fine-tune more precisely which keys
trigger which kind of navigations.

To interact in real-time with the UI, the developer emits `NavCommand`s events.
The navigation system responds with `NavEvent` events, such as
`NavEvent::FocusChanged` or `NavEvent::Caught`.

The navigation system doesn't handle itself the state of the UI, but instead
emits the `FocusChanged` etc. events that the developer will need to read to,
for example, play an animation showing the change of focus in the game.

Rather than directly reacting to input, we opt to make our system reactive to
`NavCommand`, this way the developer is free to chose which button does what in
the game.

What I want is being able to specify how to navigate a menu (however complex it is)
without headaches. A simple menu should be dead easy to specify, while a
complex menu should be possible to specify. Let's start by the simple one.

### Example

Let's start with a practical example, and try figure out how we could specify
navigation in it:

![Typical RPG tabbed menu](https://user-images.githubusercontent.com/26321040/140542742-0618a428-6841-4e64-94f0-d64f2d03d119.png)

You can move UP/DOWN in the submenu, but you see you can also use LT/RT to
go from one submenu to the other (the _soul_, _body_ etc tabs).

Here is a draft of how I could specify in-code how to navigate this menu.
`build_ui!` is a theoretical macro that builds a bevy UI without the verbosity
we currently are accustomed to.

Think of `horizontal`, `vertical`, `panel` etc. as a `NodeBundle`s presets, the
elements between `{}` are additional `Component`s added to the `NodeBundle` and
elements within parenthesis following a keyword are children `Entity` added with
`.with_children(|xyz| {...})`.

```rust
build_ui! {
  vertical {:container "menu"} (
    horizontal(
      tab {:navigable} ("soul")
      tab {:navigable} ("body")
      tab {:navigable} ("mind")
      tab {:navigable} ("all")
    )
    horizontal {:container "soul menu"} (
      vertical(
        panel {:navigable} (
          colored_circle(Color::BLUE)
          title_label("gdk")
        )
        panel {:navigable} (
          colored_circle(Color::GREEN)
          title_label("kfc")
        )
        panel {:navigable} (
          colored_circle(Color::BEIGE)
          title_label("abc")
        )
      )
      vertical {:container "abc menu"} (
        title_label("ABC")
        grid(
          circle_letter {:navigable} ("a")    circle_letter {:navigable} ("A")
          circle_letter {:navigable} ("b")    circle_letter {:navigable} ("B")
          circle_letter {:navigable} ("c")    circle_letter {:navigable} ("C")
        )
      )
    )
  )
}
fn react_events(events: ResMut<Event<NavEvent>>, game: Res<Game>) {
  for event in events.iter() {
    if matches!(event, NavEvent::Caught(NavCommand::Action, game.ui.start_button)) {
      // start game
    }
    //etc.
  }
}
```


## Implementation strategy

### Basic physical navigation

Let's drop the idea to change submenu with LT/RT for a while. We would change
submenus by navigating upward with the Dpad or arrow keys to the tab bar and
selecting the tab with the LEFT and RIGHT keys.

Typical game engine implementation of menu navigation requires the developer to
specify the "next", "left", "right", "previous" etc.[^1] relationships between
focusable elements. This is just lame! Our game engine not only already knows
the position of our UI elements, but it also has knowledge of the logical
organization of elements through the `Parent`/`Children` relationship. (in this
case, children are specified in the parenthesis following the element name,
look at `grid(...)`)

Therefore, with the little information we provided through this hand wavy
specification, we should already have everything setup for the navigation to
just work™.


### Dimensionality

Let's go back to our ambition of using LT/RT for navigation now. 

UI navigation is not just 2D, it's N-dimensional. The
LT/RT to change tabs is like navigating in a 3rd orthogonal dimension to the
dimensions you navigate with UP DOWN LEFT RIGHT inside the menus.

I'll call those dimensions `Plane`s because it's easier to type than
"Dimension", I'll limit the implementation to 3 planes. We can imagine truly
exotic menu navigation systems with more than 3 dimensions, but I'll not
worry about that. Our `Plane`s are:
* `Plane::Menu`: Use LT/RT to move from one element to the next/previous
* `Plane::Select`: Instead of emitting an `Action` event when left-clicking or
  pressing A/B, go up-down that direction
* `Plane::Physical`: Use the `Transform` positioning and navigate with Dpad or
  arrow keys

Each `NavCommand` moves the focus on a specific `Plane` as follow:
* `Plane::Menu`: `Previous`, `Next`
* `Plane::Select`: `Action`, `Cancel`
* `Plane::Physical`: `MoveUp`, `MoveDown`, `MoveLeft`, `MoveRight`

We should also be able to "loop" the menu, for example going LEFT of "soul"
loops us back to "all" at the other side of the tab bar.

### Specifying navigation dimensions

We posited we could easily infer the physical layout and bake a navigation map
automatically based on the `Transform` positions of the elements. This is not
the case for the dimensions of our navigation. We can't magically infer the
intent of the developer: They need to specify the dimensionality of their menus.

Let's go back to the menu. In the example code, I refer to `:container` and
`:navigable`. I've not explained yet what those are. Let's clear things up.

A _navigable_ is an element that can be focused. A _container_ is a node entity
that contains _navigables_ and 0 or 1 other _container_.

![The previous tabbed menu example, but with _containers_ and _navigables_ highlighted](https://user-images.githubusercontent.com/26321040/140542768-4fdd5f23-2c2e-43c1-9fa4-cc11fe67c619.png)

The _containers_ are represented as semi-transparent squares; the _navigables_
are circles.

In rust, it might look like this:
```rust
struct Container {
  inner: Option<Box<Container>>,
  siblings: NonEmptyVec<Navigable>,
  active: SiblingIndex,
  plane: Plane,
}
```
Note: this is the data structure for the navigation algorithm, the ECS
componenet called `Container` will probably look like this:
```rust
#[derive(Component)]
struct Container {
  plane: Plane,
  loops: bool,
}
```

For now, let's focus on navigation within a single _container_. The
_navigables_ are collected by walking through the bevy `Parent`/`Children` 
hierarchy until we find a `Parent` with the `Container` component. Transitive
children[^2] `Entity`s marked with `Navigable` are all _navigable siblings_.
When collecting sibling, we do not traverse _container_ boundaries (ie: the
_navigables_ of a contain**ed** _container_ are not the _navigables_ of the
contain**ing** _container_)

A `Container`'s plane field specifies how to navigate between it's contained
_navigables_. In the case of our menu, it would be `Plane::Menu`. By default it is
`Plane::Physical`.

So how does that play with `NavCommand`s?  For example: You are focused on "B"
navigable in the "abc menu" container, and you issue a `Next` `NavCommand` (press
RT). What happens is this:
1. What is the `plane` of "abc menu"? It is `Physical`, I must look up the
   containing `Container`'s plane.
2. What is the `plane` of "soul menu"? It is `Physical`, I must look up the
   containing `Container`'s plane.
3. What is the `plane` of "menu"? It is `Menu`!
4. Let's take it's current active _navigable_ and look which other _navigable_
   we can reach with our `NavCommand`

In short, if your focused element is inside a given _container_ and you emit a `NavCommand`, 
you climb up the container hierarchy until you find one in the plane of your
`NavCommand`, then lookup the sibling you can reach with the given `NavCommand`.

This algorithm results in three different outcomes:
1. We find a container with a matching plane and execute a focus change
2. We find a container with a matching plane, but there is no focus change,
   for example when we try to go left when we are focused on a leftmost
   element
3. We bubble up the container tree without ever finding a matching plane


### Navigation boundaries and navigation tree

The navigation tree is the entire container hierarchy, including all nodes (which
are always `Container`s) and leaves (`Navigable`s). For the previous example,
it looks like this:

![A diagram of the navigation tree for the tabbed menu example](https://user-images.githubusercontent.com/26321040/140542937-e28eed5e-70d5-4899-9c41-fb89b222469e.png)

Important: The navigation tree is linear: it doesn't have "dead end" branches,
it has as many nodes as there are depth levels.

The algorithm for finding the next focused element based on a `NavCommand` is:
```python
def change_focus(
    focused: Navigable,
    cmd: NavCommand,
    child_stack: List[ChildIndex],
    traversal_stack: List[Navigable],
) -> FocusResult:
  container = focused.parent
  if container is None:
    first_focused = traversal_stack.first() or focused
    return FocusResult.Uncaught(first_focused, cmd)

  next_focused = container.contained_focus_change(focused, cmd)
  if next_focused.is_caught:
    first_focused = traversal_stack.first() or focused
    return FocusResult.Caught(first_focused, container, cmd)

  elif next_focused.is_open:
    parent_sibling_focused = child_stack.pop()
    traversal_stack.push(focused)

    return change_focus(parent_sibling_focused, cmd, child_stack, traversal_stack)

  elif next_focused.is_sibling:
    first_focused = traversal_stack.first() or focused
    traversal_stack.remove_first()

    return FocusResult.FocusChanged(
      leaf_from= first_focused,
      leaf_to= next_focused.sibling,
      branch_from= traversal_stack,
    )
  else:
    print("This branch should be unreachable!")
```

### UI Benchmarks discussion

As mentioned in [this
RFC](https://github.com/alice-i-cecile/rfcs/blob/ui-central-dogma/rfcs/ui-central-dogma.md),
we should benchmark our design against actual UI examples. I think that the
one used in this RFC as driving example (the RPG tabbed menu) illustrates well
enough how we could implement Benchmark 2 (main menu) using our system.
Benchmark 1 is irrelevant and Benchmark 3 seems to only require a `Physical`
layer.


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

### Naming

I'm not comfortable with the names I chose for the various concepts I
introduced. (such a `Container`, `Plane`, `Physical`) Do not be afraid to
suggest name changes. Naming is important and I'm already not satisfied with
the nomenclature I came up with.

## Drawbacks and design limitations

### Unique hierarchy

If we implementation this with a cached navigation tree, it's going to be
easy to assume accidentally that we have a single fully reachable navigation
tree. This is not obvious, it's easy to just add the `Container` or `Navigable`
`Component` to an entity that doesn't have itself a `Container` parent more
than once.

This needs to be solved, I'm thinking that an error message would be enough in
this case. But it's worth considering design changes that makes this situation
a non-issue.

### No concurrent submenus

You have to "build up" reactively the submenus as you navigate through them,
since **there can only be a single path through the hierarchy**.

We expect that on `FocusChanged`, the developer adds the components needed if
the focus change leads to a new sub-menu.

I'm implementing first to see how cumbersome this is, and maybe revise the
design with learning from that.

Maybe it's possible to integrate more than one `Container` within other
containers. The only motivation for this currently is that it makes it easier to
implement and reason about the system.

### Inflexible navigation

The developer might want to add arbitrary "jump" relations between various
menus and buttons. This is not possible with this design. Maybe we could add a
```rust
  bridges: HashMap<NavCommand, Entity>,
```
field to the `Container` to reroute `NavCommand`s that failed to change focus
within the menu. I think this might enable cycling references within the
hierarchy and break useful assumptions.

## Unresolved questions

Building the physical navigation tree seems not-so trivial, I'm not sure how
feasible it is, although I really think it's not that difficult.


## Isn't this overkill?

Idk. My first design involved even more concepts, such as `Fence`s and `Bubble`
where each `Container` could be open or closed to specific types of `NavCommand`s.
After formalizing my thought, I came up with the current design because it
solved issues with the previous ones, and on top of it, required way less
concepts.

For the use-case I considered, I think it's a perfect solution. It seems to
be a very useful solution to less complex use cases too. Is it overkill for
that? I think not, because we don't need to define relations at all, especially
for a menu exclusively in the physical plane. On top of that, I'm not creative
enough to imagine more complex use-cases.

[^1]: See [godot documentation](https://github.com/godotengine/godot-docs/blob/master/tutorials/ui/gui_navigation.rst)
[^2]: ie: includes children, grand-children, grand-grand-children etc.

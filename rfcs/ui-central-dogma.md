# Feature Name: (fill me in with a unique ident, `ui-central-dogma`)

## Summary

`bevy_ui` is a Rust-only, data-driven, ECS-powered UI framework that is tightly integrated with the rest of Bevy.
Rather than being a monolithic "UI solution", `bevy_ui` creates UI by granting UI-like properties to ordinary entities and components, allowing you to pick and choose which behaviors you want to opt in to in a modular way.
This RFC lays out the motivation for this approach, describes the core data flow, connects the high-level components we'll need, and shows what complete vertical slices of complex UI will look like.

## Motivation

Bevy, like all other game engines, needs a UI solution.
By establishing a common, idiomatic approach to UI we can ensure that the Bevy experience remains accessible for developers of all skill levels and is easy to interact with,
regardless of how blurred the line between gameplay and interface becomes.

Bevy's existing UI framework (as of 0.5), is unequipped to handle complex designs due to the excessive boilerplate required and unopinionated dataflow,
doesn't fit well into a data-driven paradigm, and is generally short on features.

The fundamental challenge in both designing a UI solution, and using that solution in real applications is managing complexity.
By thinking about this problem space through the lens of data flow and orthogonal UI-like properties, we can create a modular solution.
This splits the problem into manageable chunks and ultimately makes our solution easier to learn, use and hack.

## User-facing explanation

Ultimately, user interfaces are designed to allow users to *interact* with your game or application.
In Bevy, user interfaces are modelled after a **central dogma** of data flow:

1. **Raw input events** are accepted from users.
2. These are **dispatched**, and converted to **action events**, which are read by **game systems** and individual **UI widgets**.
3. Widgets **emit events** and **store state**, and can be easily read by game systems.
4. **Game systems** use this information to modify the world's state.
5. Widgets can read the world's state to update their own state.

This data flow, from raw input to actions to widgets to game systems to world state is unidirectional,
giving us a paradigm to organize our user interfaces around.
The loop closes when our widgets' state is updated with information about the game world, allowing for them to change and respond.

### Properties of a User Interface

TODO: expand explanations of these properties.

Traditional user interfaces have many important but largely orthogonal properties:

1. Input-responsive.
2. Selectable.
3. Located with screen-space coordinates.
4. Drawn on top.
5. Subject to a layout-algorithm.
6. Style-able.
7. Data-driven.
8. Localizable.

Thinking about these properties and behaviors independently allows us to design these systems more modularly, in a way that fits into the trait-like architecture encouraged by the ECS,
and allows us to reuse these properties for other aspects of our games.

This design makes particular sense in games, where the lines between "UI" and "game elements" tend to blur.
Controls often need to integrate deeply with overhead displays and respond to inputs in the same way,
user interfaces in VR may not be defined in screen space,
we may need to localize assets and dialog, rather than just static UI and so on.

### UI Architecture

This framing allows us to break the UI problem space down into many smaller, loosely coupled parts, and build our UI behaviors through **composition**.

Each of these **UI building blocks** works together, answering the following questions:

1. **Input dispatching**
    1. How do we read input more than once per frame?
    2. How do we transform input streams into actions?
    3. How do we unify multiple input methods to the same actions?
    4. How do we ensure actions are handed off to the correct widget?
2. **Widget state management** ([Discussion: UI as a pure function of state](https://github.com/bevyengine/bevy/discussions/1996#discussioncomment-654221)m [RFC: System-driven polling](https://github.com/bevyengine/rfcs/pull/11))
   1. How do widgets respond to actions?
   2. How do we allow widgets to [respond to actions more than once per frame](https://discord.com/channels/691052431525675048/743663673393938453/836404717499449375)?
   3. How do make our UIs animate?
   4. How do we ensure our UIs can be easily configured to respond to actions in automatic ephemeral ways (such as when hovered)?
   5. How is the state of widgets stored?
   6. How do game systems read the state of widgets?
   7. How do widgets read the state of the world and change to account for it?
3. **Object selection**
   1. How do we know which widget (or unit or...) is selected?
   2. How do we select different widgets with the mouse? Keyboard? Joystick?
   3. How do we customize selection behavior to account for unique orders and nesting?
   4. How can we select multiple objects?
4. **Screen-space coordinates**
   1. How do we mark an entity as being defined in terms of screen-space rather than world coordinates?
   2. How do we ensure an object is drawn on top of other graphical elements?
5. **Layout**
   1. How do we define an object as being subject to our layout algorithm?
   2. Which layout algorithm are we using?
   3. How do we change the parameters for this layout algorithm?
   4. How do we dynamically account for diverse screen resolutions and variable space?
6. **Styling** ([RFC: compositional entity styling](https://github.com/bevyengine/rfcs/pull/1))
   1. How do we store and modify the various cosmetic attributes that determine the aesthetics of our UIs?
   2. How do we compose styles to produce a final result?
   3. How do we easily revert applied styles?
   4. How do we define the style of many widgets en-masse?
   5. How do we easily load and apply premade themes?
7. **Data-driven**
   1. How do we define user interfaces largely in terms of data?
   2. How does a data-driven approach to UIs coexist with a code-first approach?
   3. How is this data saved, stored and loaded?
   4. How are data-driven UIs connected to game systems?
   5. How do we interface with external tools, if at all?
8. **Localization** ([RFC: Project Fluent integration](https://github.com/bevyengine/rfcs/pull/14))
   1. How do we control which language or culture or game is configured for?
   2. How do we propagate these changes down to individual widgets?
   3. How do we store text data so then it can be easily replaced?

## Implementation strategy

Even more than other RFCs, this RFC is intended to be collaborative and exploratory.
If you feel something needs clarification, find an argument that could be stronger, have an example to add or see another path forward,
please do not hesitate to leave a comment or make a pull request to [this PR's branch](https://github.com/alice-i-cecile/rfcs/tree/ui-central-dogma).

Each of the pieces of the data-flow pipeline outlined above are intended to be modular enough to be considered *largely* in isolation.
Implementation details for those pieces belong in their corresponding RFCs.

As you build out your UI ideas, please feel free to prototype in third-part crates, or when needed, directly on a fork of Bevy itself.
Once your API is sufficiently developed,
please try to contribute standard paper prototypes (directly below) so we can get a sense of the end-to-end architecture and ergonomics of the various proposals.

## Examples

Unlike most RFCs, the examples here are (mostly) intended for paper prototyping purposes.
That means that, rather than being used to teach users how to use our crate,
they're intended to quickly communicate what solutions look like in practice to other designers.

These paper prototypes will not actually *work*, and are intended to be composed entirely of end-user code and/or data.

We've established a common set of **benchmark prototypes** below for designers to target.
These are designed to capture a wide-range of required behavior and use cases,
and range from very simple to frustratingly complex.

Each example's unique challenges are defined, and then a detailed list of requirements is given.
Below that, we collect examples of alternative paper prototypes, labelled with the RFCs / approaches used.

TODO: add more benchmark prototypes from [#1974](https://github.com/bevyengine/bevy/issues/1974).

### Benchmark 1: Cookie clicker

**Unique challenges:** As simple as it gets. How easy is it to do easy things?

1. There is a single button.
2. There is a `CookiesClicked` `Resource` that starts at 0.
3. When you click the button, the value of the `CookiesClicked` goes up by one.

### Benchmark 2: Main menu

**Unique challenges:** How easy is it to do common reasonably simple things? How do you support multiple input paradigms at once?

1. We have a main menu screen with the following options:
    1. New Game.
    2. Load Game.
    3. Settings.
    4. Quit.
2. These options are centered in the screen, with an adjustable amount of padding between each item.
3. When each of these items is selected, an appropriate event is emitted that can be listened for by ordinary Bevy systems.
4. You can navigate and select these options with:
   1. Mouse.
   2. Keyboard arrows.
   3. Keyboard hotkeys.
   4. Joystick.

### Benchmark 3: To-do list

**Unique challenges:** Nested widgets, text input, drag-and-drop, clipboard, undo-redo.

1. We have a simple one page to-do list.
2. At the top of our list is a title that we can edit.
3. Each item has an associated check box and editable text field.
4. There is always one blank item at the bottom of our list.
5. When we press `Enter` with a list item selected, a new blank item is inserted and selected just below our previous item.
6. We can navigate between items by:
   1. Using the up and down arrow keys.
   2. Using tab or shift-tab.
   3. Clicking on an item.
7. We can cut-copy-paste to and from these text fields.
8. We can mark items as complete, causing them to be reversibly moved to a separate `Completed` section.
9. We can delete items completely.
10. We can undo and redo changes, including completion and deletion.

## Drawbacks

By splitting the work in this way, we force a relatively decoupled design.
This may not be optimal for performance, and limits the designs we can consider.
In particular, doing so rules out integrating another complete solution.

See **Rationale and Alternatives** directly below for drawbacks of high-level architectural decisions.

## Rationale and alternatives

### Why do we want to decouple UI behaviors from each other?

TODO: explain in terms of composition over inheritance.
Discuss unique needs of games, hackability, and ecosystem crates.

### Why are we trying to couple the UI to the ECS?

TODO: write.

### Why not integrate an existing Rust UI library?

TODO: write.

### Why not use CSS + HTML for our style parameters?

TODO: write.

## Prior art

For the most part, prior art will be better covered in RFCs tackling specific UI dataflow elements.

Prior art on similar high-level organization would be welcome here though, if others have insight to add.

## Unresolved questions

1. Is this an appropriate division of parts?
2. Which solution are we going to use for each part?

## Future possibilities

While the following areas are out-of-scope of the current RFC, they're important enough to ensure that existing designs can accommodate them:

1. Automation, for testing and accessibility purposes.

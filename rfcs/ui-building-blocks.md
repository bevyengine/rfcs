# Feature Name: `ui_building_blocks`

## Summary

This RFC establishes a common vocabulary, and basic data structures for UI.
It is combined with a design pattern to manipulate UI styles ergonomically using Bevy's ECS, to tangibly demonstrate the feasibility of the approach.

## Motivation

UI is the next big challenge for Bevy, with a complex set of data structures that need to be stored and manipulated.
In particular, the ability to style elements (by setting) in a reusable fashion is central to a polished, easy-to UI, but there's no immediately obvious approach to handling them.
What makes a "style" in Bevy? How do we represent the individual style parameters? How are widgets represented? How are styles composed? How are they applied?

None of these questions are terribly challenging to implement on their own, but taking the first-seen path is particularly dangerous here, due to the risk of proliferation of non-local configuration and similar-but-not-identical style parameters, as seen in CSS.

We need to settle on a common standard, in order to enable interop between and establish basic building blocks that more complex UI decisions can build off of.

## Guide-level explanation

User interfaces in Bevy are made out of **widgets**, which are modified by the **styles** that are applied to them, and the aesthetic and functional behavior of is ultimately implemented through **UI systems**.

A **widget** is a distinct part of the UI that needs its own data: be that its position, local state or any style parameters.
Widgets in Bevy are represented by entities with a `Widget` marker component.

Each widget has an associated `Styles` component, which contains a `Vec<Entity>` which points to some (possibly 0) number of **styles**, represented as entities with a `Style` marker component.

Each style in that vector is applied, in order, to the widget, overriding the **style parameters** that already exist.
Style parameters are stored as components, and serve to control some element of its presentation, such as the font, background color or text alignment.
Style parameters can be reused across disparate widget types, with functionality achieved through systems that query for the relevant components.
The type of the style parameter components determines which behavior it controls,
while the value returned by `.get()` controls the final behavior of the widget.

Every style parameter has both a `MyStyle<Base>` and a `MyStyle<Final>` variant, stored together on each entity.
When creating a new style parameter, you must ensure that it implements `StyleParam`, typically achieved with `#[derive(StyleParam)]`.

When working on complex games or applications, you're likely to want to group your styles into **themes**,
automatically applying them to large groups of widgets at once.
In Bevy, themes are applied by adding a generic system that corresponds to that theme to your app's schedule.
Select a marker component type `W`, then add a theme system to your app with `W` as a type parameter:
`app.add_startup_system(solarized_theme::<Button>.system())` will then create an entity that stores the solarized theme
style parameters to your world, and adds the appropriate `Entity` reference to the end of each of the `Styles` components
on all entities with `Widget` that have the `Button` marker component on them.

Generally, you'll want to add these systems to a startup stage, but you may also find it useful to add them to various `State`s.
This can work well for quickly toggling between various themes, like you might see in a light-dark mode.

### Example: Building a Simple Widget

```rust
commands.spawn_bundle(ButtonBundle::default());
```

### Example: Building a Compound Widget

```rust
commands.spawn_bundle(ButtonBundle::default())
  .with_children(|parent| {
    parent.spawn_bundle(TextBundle::default())
  });
```

### Example: Stacking Styles

```rust
// Adds two styles to `SpecialWidget` entities
// SpecializedStyle will overwrite BaseStyle for any shared style parameters 
// 
// If we wanted to ensure that these styles were always associated with `SpecialWidget` 
// even after its identity changed, we'd need to write a `Removed` system as well
fn style_stacking(mut query: Query<&mut Styles, Added<SpecialWidget>>, 
  base_style: Res<BaseStyle>, 
  specialized_style: Res<SpecializedStyle>){
  for mut styles in query.iter_mut(){
    // styles.insert removes any matching style, then adds the style to the end of the vec
    styles.insert(base_style.entity);
    styles.insert(specialized_style.entity);
  }
}
```

### Example: Changing Style on Hover

```rust
struct Hovering(bool);

fn on_hover(mut query: Query<(&IsHovered, &mut Styles), (With<OnHover>, Changed<IsHovered>)>, 
  hover_style: Res<HoverStyle>){
  for (is_hovered, mut styles) in query.iter_mut(){
    match is_hovered.0 {
      // Adds the hover style to the widget
      true => styles.insert(hover_style.entity), 
      // Removes the hover style from the widget, restoring its original appearance
      false => styles.remove(hover_style.entity),
    }
  }
}
```

## Reference-level explanation

### Style data flow

At the heart of the styling design is a data flow that propagates style parameters from the style to the end widget.
Naively, you'd like to just overwrite the parameter in question, applying the first style's value if any, then the next and so on.
Unfortunately, this causes issues with dynamically applying and reverting styles, because the original value is lost completely.

In order to get around this, we need to somehow duplicate our data, storing both the original and final values.
This is done by creating two variants of each style parameter component: `Foo::Base` and `Foo::Final`.

This requires the `StyleParam` trait:

```rust
trait StyleParam {
  type Inner: Clone;

  type Base: Component + DerefMut + Deref<Target = Self::Inner>;
  type Final: Component + DerefMut + Deref<Target = Self::Inner>;
}
```

End users should use the #[style_param] attribute macro on `struct Foo(...)`, which implements the `StyleParam` trait on an ordinary-looking struct.
Under the hood, this creates the following boilerplate:

```rust
// PhantomData and the Inner type are used to ensure Foo is !Component
// This avoids a footgun for users trying to query for Foo directly
struct Foo(PhantomData<*mut u8>);

// Actual data goes here
struct FooInnerStyle(...)

struct FooBase(FooInnerStyle);
struct FooFinal(FooInnerStyle);

/* add impls for deref/mut for both FooBase and FooFinal */

impl StyleParam for Foo {
  // Both Base and Final should derefence to the same type of data
  // Allowing us to convert between them
  type Inner = FooInnerStyle;

  type Base = FooBase;
  type Final = FooFinal;
}
```

In order for the data to be propagated from our styles to our widgets, we need a set of **style propagation systems**, that work like so:

```rust
/// Rebuilds the style parameter `S` for the widget
///
/// Styles need to be update if either
/// a) the styles associated with the widget has changed or b) the style's style parameter values have changed
/// Styles are rebuilt from scratch, working from their base value each time the styles are changed
fn apply_styles<S:StyleParam>(mut widget_param: &mut S, style_params: Vec<Option<&S>>){  
  // If the style is set in that style, use style_param.set() to apply it
  // This replaces any previous values for the `final` field
}

/// Automatically updates the styling for all style parameter components of type `S` whose Styles changed
///
/// End users should register this system in your app using `app.add_style::<S>()
pub fn propagate_styles<S: StyleParam>(mut widget_query: Query<(&S::Base, &mut S::Final, &Styles), 
  (With<Widget, Without<Style>, Changed<Styles>>)>,
  style_query: Query<Option<&S::Base>, With<Style>>){
 
 for (s_base, s_final, styles) in widget_query.iter_mut(){
    let mut style_params = Vec<Option<&S::Base>>;

    for style in style.iter(){
      style_params.push(style_query.get_component::<S>());
    }

   *s_final = apply_styles(s_base, style_params);
 }
}


/// Automatically updates the styling for all style parameter components of type `S` whose underlying style entity changed
///
/// End users should register this system in your app using `app.add_style::<S>()
pub fn propagate_style_changes<S: StyleParam>(mut widget_query: Query<(&S::Base, &mut S::Final, &Styles), 
  (With<Widget, Without<Style>>)>,
  style_query: Query<Option<&S::Base>, (With<Style>, Changed<S::Base>>)){
 
 // Implementation note: we can narrow our queries by splitting this work into two systems, despite shared logic
 for (s_base, s_final, styles) in widget_query.iter_mut(){
    let mut style_params = Vec<Option<&S::Base>>;

    for style in style.iter(){
      if style_query.get(style).is_some(){
        style_params.push(style_query.get_component::<S::Base>());
      }
    }

   *s_final = apply_styles(s_base, style_params);
 }
}


```

Style propagation systems run every loop;
the `Changed<Styles>` filter will prevent us from doing unnecessary work.

When we call `app.add_style::<S::Base, S::Final>()`, the following steps occur:

1. We add a `maintain_style::<S::Base, S::Final>` system to `CoreStage::PreUpdate`, which adds and removes the `Base` and `Final` variants of `S` to entities as needed.
2. We add a `propagate_style_changes::<S::Base, S::Final>` system to `CoreStage::Update`.
3. We add a `propagate_style::<S::Base, S::Final>` system to `CoreStage::Update`, which runs after the corresponding `propagate_style_changes`.

This is done under the hood for all of the built-in style parameters as part of `DefaultPlugins`.

### Theming systems

Themes are applied to the app using **theming system**, which, when run, adds the appropriate style to the widgets with the correct component.
Users can control the priority of the themes by controlling the relative run order of their theming systems in the usual fashion.

These should be constructed ad hoc following a pair of simple examples.

```rust
pub MyTheme {
 pub style_entity: Entity
}

/// Adds an instantiated style to all widgets with the component `M`
pub fn my_theme<M: Component>(widget_query: Query<&mut Styles, (With<M::Final>, With<Widget>)>, theme: Res<MyTheme>){
  for styles in widget_query.iter_mut(){
    styles.push(theme.style_entity);
  }
}

/// Adds the style stored in the `style_scene` to all widgets with the component `M`
pub fn my_stored_theme<M: Component>(mut commands: Commands, 
  widget_query: Query<Entity, (With<Styles>, With<M::Final>, With<Widget>)>, 
  style_scene: Handle<DynamicScene>){

  let widget_entities = widget_query.iter().collect();
 
  // Loads in the style scene and create an entity out of it
  // Then, pushes that entity onto the end of the `Styles` component on each of those entities 
  commands.apply_stored_theme(style_scene, entities);
}
```

We can apply this to construct a simple light-dark mode toggle, by combining this with app-level states:

```rust
enum LightDarkMode{
  Light,
  Dark
}

fn main{
  App::build()
    // These are automatically constructed using a FromWorld implementation that creates the appropriate entity 
    // and stores a reference to the correct entity
    .init_resource::<LightTheme>()
    .init_resource::<DarkTheme>()
    // Controls the starting state
    .add_state(LightDarkMode::Dark)
    .add_system_set(SystemSet::on_enter(LightDarkMode::Light).with_system(toggle_light_dark::<Widget>.system()))
    .add_system_set(SystemSet::on_enter(LightDarkMode::Dark).with_system(toggle_light_dark::<Widget>.system()))
}

pub fn toggle_light_dark<M: Component>(
  widget_query: Query<&mut Styles, (With<M::Final>, With<Widget>)>, 
  light_theme: Res<LightTheme>,
  dark_theme: Res<DarkTheme>,
  state: Res<State<LightDarkMode>>
){
  let (searched, replace) = match state.active() {
    LightDarkMode::Light => (dark_theme.style_entity, light_theme.style_entity),
    LightDarkMode::Dark => (light_theme.style_entity, dark_theme.style_entity),
  }

  for styles in widget_query.iter_mut(){
    // The `Styles::replace` method replaces the specified style with a new style in the same position
    styles.replace(searched, replace);
  }
}
```

## Drawbacks

1. Applying stored themes relies on hand-crafted `Commands` magic due to timing issues, rather than being implementable in transparent vanilla Bevy.
2. Every style parameter component needs its own `propagate_style` and `maintain_style` systems.
3. The type-system macro magic around `#[style_param]` is complex and mildly cursed.
4. Implementing traits on `StyleParam` structs doesn't work in the obvious way.
Instead of `impl MyTrait for Foo { ... }`, users must do:

```rust
impl MyTrait for FooInnerStyle { /* ... */ }
impl MyTrait for FooBase { /* just call into FooInnerStyle */ }
impl MyTrait for FooFinal { /* just call into FooInnerStyle */ }
```

## Rationale and alternatives

### Why put your UI in the ECS?

UI is not terribly performance sensitive, and often involves complex hierarchies that can be a nuisance to fit into an ECS paradigm.
So why put it there?

1. UI is a complex, but reasonably well-understood problem domain.
By solving the problems presented for our ECS here, we can make it more useful for other gameplay patterns.
2. Bevy's ECS is a particularly ergonomic and familiar tool for accessing specific data. Why reinvent what works?
3. UIs will often need to change dynamically, and integrate with the game in complex ways that are hard to predict.
By making our UI out of vanilla ECS, we can ensure that interoperability and extension are easy.
4. The ECS provides a great tool for pre-made objects: Scenes.
Keeping the UI in the ECS ensures we can take advantage of its features and improvements,
enabling a data-driven workflow for UI designers and artists.

### Approaches to styling

Pick up a UI framework, and you'll find a new approach to styling.
So why *this* approach?
Let's look at some of the alternatives we should *not* take:

1. **In-line styles only:** you can only configure widgets locally.
This is, in effect, the current approach of `bevy_ui`.
It's very explicit, but also very tedious and impossible to synchronize across a large UI.
If we do not provide a styling solution, users will invent their own ad-hoc, forcing them to think through this design on their own and fragmenting the ecosystem.
2. **Inherited styles:** rather than allowing for multiple styles on each widget with overwriting, we could have a single style per widget,
and define styles in terms of inheritance from other styles.
This results in complex inheritance trees, and results in a proliferation of styles, each of which must be named and can be reused.
Fundamentally, this is a bad pattern because it obscures the underlying end-user logic of "like X, except...".
3. **Cascading styles:** a widget's styles could cascade down to all other child widgets.
This leads to more complex non-local behavior,
requires that all widgets live within a tree,
and is fragile if you want to move a widget out of the tree it currently lives in.
Users can also replicate this behavior easily enough with custom systems when it is desired.
4. **Global style priorities:** rather than defining which styles overwrite the others on each entity,
we could give each one a "priority", and always apply them in that order.
This is fiddly to tweak (the same challenge emerges with z-ordering in 2D applications), is harder to debug,
and can make it very hard to get the exact behavior in special cases.
Furthermore, this is very easily replicated by simply applying themes via systems that run in a specified order.
5. **Widget-to-widget inheritance:** we could allow widgets to mirror the styles of other widgets.
While conceptually simple, this will very quickly result in long and tangled web of inheritance.
This fragile and complex anti-feature, incidentally, is a large part of the value of the `Style` marker component: to ensure that users are not presented with easy tools to tangle themselves up.

In the end, this approach to styles allows for complex but ergonomic customization with the use of custom systems (ala the themes proposal),
but still allows for simple, local reasoning about the final results.
Simply examine the `Styles` on the entity you're attempting to debug, examine the style entities that it points to,
and you have a complete understanding of why your widget looks the way it does.
No endless chains or inheritance trees to walk!

### Storing original style parameter values

While storing old data about the style parameters complicates our design significantly,
it is critical for managing dynamic changes in style (such as light/dark mode or hover behavior).
Without automatically caching the value for our users on the component itself, the users (or the engine) are left to cache the state of the widgets before applying the theme on their own, then reset it.

Instead of our current approach, we could instead cache the previous value of the style parameter components in a resource or a component of its own,
using trait objects (`Box<dyn StyleParameter>`), then refer to these when restoring the original components.
This keeps our components simpler, but is more technically challenging and dramatically more magical for end users (who will no longer be able to simply check the original value on their own).

Instead of the generics pattern described in the *Reference-Level Explanation*,
we could use getter and setter methods on the `StyleParam` trait:

```rust
trait StyleParam: Component {
  /// The underlying value of the style parameter, unique to this particular widget
  ///
  /// Modifying this value is akin to applying an in-line style to your widget
  pub fn base(&self) -> Self;  
  /// The final value of the style parameter, which will be displayed in your app
  ///
  /// Obtained by succesively applying the styles found in this widget's `Styles` component,
  /// with later styles overwriting earlier styles
  pub fn final(&self) -> Self;  
  /// Sets the private `final` field to val
  ///
  /// In typical usage, this is done automatically for you in the corresponding [propagate_style] system,
  /// registered using `app.add_style::<S>()
  pub fn set(&mut self, val: Self);
}
```

This pattern is clear, and doesn't involve any fancy generics shenanigans.
However, it makes our components larger (reducing cache efficiency), and is a shift away from Plain Old Data,
which makes the components more complex to work with as an end user.

The only other obvious existing tools we have to do so are entity (or relevant component) cloning (very hard to implement, see [#1515](https://github.com/bevyengine/bevy/issues/1515)) or scenes.
Scenes require a large amount of boilerplate to get working right now, including on every component, and are likely to be relatively expensive.

## Unresolved questions

1. Can / should we use trait queries to reduce the boilerplate involved in adding new style parameters?
2. Does the `#[style_param]` magic work? Particularly with derive macros.
3. Should we just use two generics in `add_style`, `propagate_style` and `maintain_style` and spare the macro magic?
4. If we're using macro magic, do we care about the `Inner` trait type to avoid the querying footgun?
5. Do we want to hold off on implementing this until after `min_relations` to get a substantially cleaner solution?

## Future work

1. Eventually, we can use the `Widget` and `Style` marker components to enforce appropriate `Entity` keys in both engine and user code using [kinded entities](https://github.com/bevyengine/bevy/issues/1634).
2. Following the introduction of [archetype invariants](https://github.com/bevyengine/bevy/issues/1481),
an archetype invariant should be added as part of `add_style`,
which ensures that `Base` and `Final` variants of each component always coexist on `Widget` entities.
3. Once we have fairly advanced [relations](https://github.com/bevyengine/bevy/pull/1627) features, we should look at using them in place of `Vec<Entity>` in `Styles`.
This is not as trivial as it might appear: `Styles` very much wants an ordered list of entity references (rather than an unordered set or numbered priority list).
4. As alternative to the mildly cursed `#[style_param]` solution, we could use [relations](https://github.com/bevyengine/bevy/pull/1627) to duplicate the data.
The base version of the style parameter data will be a relation with a target of a unique dummy entity, whose identity is stored in a resource.
Then, the final version of the data is a vanilla component (a relation with no target) of the same type,
allowing end users to query for style parameter components directly to receive the current value after all styles have been applied.
5. Finally, in order to fully move forward on the second rendition of `bevy_ui`, we also need to solve the following other UI focus areas:

   1. Widget composition.
   2. Layout.
   3. Wiring: call-backs, message passing.

This RFC defines the underlying UI data structures, so should be settled first.

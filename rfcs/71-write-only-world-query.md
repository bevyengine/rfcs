# `WriteOnly` world query

## Summary

Add a `WriteOnly<T>` `WorldQuery`. `WriteOnly` acts like `&mut`, but tells bevy
that the value of component `T` is overwritten unconditionally by the system.
Add the `B0006` error. This tells users that they are modifying a component
that is later overwritten unconditionally.

## Motivation

When I write to the `Transform` of a UI node, it doesn't do anything
and I see no error messages.

Why this happens is fairly trivial: `Transform` is updated by the UI system
in `PostUpdate`, and overwrites the user-set value unconditionally.

This is also true of `AnimationClip`s. A system overwrites bone transforms
targeted by an animations in `PostUpdate`. Even if the `AnimationClip` is
paused!

## User-facing explanation

### A new `WorldQuery`

We introduce a new `WorldQuery`: `WriteOnly<T>`.

```rust
struct WriteOnly<'a, T>(Mut<'a, T>);

impl<'a, T> WriteOnly<'a, T> {
  pub fn set(&mut self, value: T) {
    *self.0 = value
  }
  /// Only **write** to the returned reference.
  /// If you have a `Vec`, for example, you should not leave any of the
  /// previously existing value.
  pub fn ref_mut(&mut self) -> Mut<T> {
    self.0.reborrow()
  }
  pub fn into_inner(self) -> Mut<'a, T> {
    self.0
  }
}

```

`WriteOnly<T>` is very much nothing more than a thin wrapper around `Mut`.
With one significant difference: semantically, it cannot be read.

We promise to the scheduler that all values in the `Query<WriteOnly<T>>` will
be overwritten, with no traces of its previous value.

### Write-only access

We add a new method to the `WorldQuery` trait.

```rust
    fn update_write_only_component_access(state: &Self::State, access: &mut FilteredAccess<ComponentId>) {}
```

Notice that — unlike `update_component_access` — it has a default implementation.
It is not unsound to erronously populate this `FilteredAccess`. It is just used
as a hint for triggering error messages.

This adds another `FilteredAccess` to the query. This `FilteredAccess`
is only written-to and read-from at app initialization when setting up the systems.

### Triggering the warning

Consider `ui_layout_system`:

https://github.com/bevyengine/bevy/blob/469a19c290861049ad121aa83d72f830d1ec9b40/crates/bevy_ui/src/layout/mod.rs#L215-L230

We would replace

```rust
    mut node_transform_query: Query<(Entity, &mut Node, &mut Transform, Option<&Parent>)>,
```

with the following:

```rust
    mut node_transform_query: Query<(Entity, &mut Node, WriteOnly<Transform>, Option<&Parent>)>,
```

Now, let's say Daniel — a bevy user — want to add a screenshake effect to their UI
(please never do this):

```rust
fn shake_ui(mut nodes: Query<&mut Transform, With<Node>>) {
  for tran in &mut nodes {
      tran.translation += random_offset;
  }
}
```

Daniel adds `shake_ui` to their game to spice things up:


```rust
app.add_plugins(DefaultPlugins::new())
  .add_systems(Update, shake_ui);
```

Daniel looks at their game. And *\*sad trombone\** nothing happens.
The UI stays fixed to the screen as if sticked with ultra glue.

**But lo**! What is this? An error in the bevy log output?

```
2023-04-20T11:11:11.696969Z ERROR bevy_ecs: [B0006] In `Update`, the system `shake_ui` writes the `Transform`, but `ui_layout_system` in `PostUpdate` later overwrites it unconditionally.
```

| 👆 **We will refer to this error as `B0006` for the rest of this RFC** |
|------------------------------------------------------------------------|

Daniel reads the error message, goes to <https://bevyengine.org/learn/errors/#b0006>
and learns that they should — I quote — "schedule `shake_ui` in `PostUpdate`,
and label it as `shake_ui.after(ui_layout_system)` to effectively update `Transform`".

Daniel updates their app schedule code:

```rust
app.add_plugins(DefaultPlugins::new())
  .add_systems(PostUpdate, shake_ui.after(ui_layout_system));
```

Daniel recompiles their game and looks at the screen…

It works now! Hurray! Without even having to ask for help.

## Implementation strategy

Ok, so how would we accomplish this feat of ECS user friendliness?

### Detecting overwrites

Let's consider first the naive approach: If a system writes to `&mut Transform`
and later a system that accesses `WriteOnly<Transform>`, then emit `B0006`.

Let's go back to our user Daniel.

They are making a game, so they also have this system:

```rust
fn move_player(player: Query<&mut Transform, With<Player>>) {
  // ...
}
//...
app.add_plugins(DefaultPlugins::new())
  .add_systems(Update, (shake_ui, move_player));
```

According to our naive logic `Transform` in `move_player`
is "later overwritten by `ui_layout_system`".

But this doesn't make sense! Entities with the `Player` component
**cannot** be overwritten by `ui_layout_system`, since `Player` is never present
on entities that also have a `Node` component!

We should only emit the error if the system changes components on entities that
will be written to by the `WriteOnly` system.

#### Significant component set

Yet, this is still not good enough! Daniel, having reasoned that they shouldn't
shake text — because otherwise it would be completely unreadable — has created
a component they add to all non-text UI entities:

```rust
#[derive(Component)]
struct ShakeUi;

fn setup(mut cmds: Commands) {
  cmds.spawn((ImageBundle { /* … */, ..default() }, ShakeUi));
}

// Previously: `With<Node>`
fn shake_ui(mut nodes: Query<&mut Transform, With<ShakeUi>>) // ...
```

Now, Daniel's `shake_ui` query _doesn't have a `Node`_! We can't use the previous
reasoning to detect that `shake_ui` only writes to a subset of entities that
will later be overwritten by `ui_layout_system`.

Ooof, yickes.

But bevy already exposes the information necessary to know that `ShakeUi` only
exists on entities with a `Node` component:

- We are aware of all currently existing archetypes in our world.
- All archetypes describe the components it has.

Here is the gold insight: 

Consider a component _C_ and a set of components _K_.

_K_ is the significant set of _C_ when:
∀ archetypes _Ai_,
if _C_ ∈ _Ai_ ⇒ _K_ ⊂ _Ai_.

In plain English: If an entity has the _C_ component, it necessarily also has
all components in _K_.

#### Definition

Let's narrow down the meaning of the words we use, so that we can do
mathematical reasoning on them:

- _system_: A set of _queries_ and a schedule slot
- _query_: three sets of _components_: read, write, write-only.
- _component_: TODO

Now let's define relations:

- query _Qx_ **writes to** component _Cy_ when: _Cy_ ∈ _Ax.write_
- Ditto for "_Qx_ **reads** component _Cy_"
  and "_Qx_ **writes-only to** component _Cy_"
- query _Qx_ **subset of** query _Qy_ when:
  - ∀ _Ci_ ∈ _Qx_, _Ci_ ∈ _Qy_. (ie: `Qx & Qy == Qx`)
- system _Sx_ **happens before** _Sy_ when the scheduler runs _Sx_ before _Sy_.
- system _Sx_ **overwrites** _Sy_ on _Cz_ when:
  - ∃ _Qx_ ∈ _Sx_, _Qx_ writes-only to _Cz_
  - **and** _Sy_ happens before _Sx_
  - **and** ∃ _Qy_ ∈ _Sy_ such as _Qy_ subset of _Qx_
  - **and** _Qy_ writes to _Cz_.

| ❗ **TODO, this definition is not complete** |
|----------------------------------------------|

- **happens before** should be replaced by "directly written-to afterward"
- add the significant component set to the definition

Now we have the mathematical foundations to detect overwrites.

But how would this look like in bevy? Where to check for the overwrites? When?
With which frequency?

| ❗ **TODO, no concrete implementation plans** |
|-----------------------------------------------|

#### False positives

Suppose Daniel writes the following system (they shouldn't!):

```rust
fn move_all(mut query: Query<&mut Transform>)
```

The `Transform` will later be overwritten by `ui_layout_system` for entities
that also happen to have a `Node` parameter. So this is bad.
Yet, Daniel probably didn't mean to write to `Node` entities.

So I think it is a fair to leave this as is.

Modifications to generic components should **always** be scopped to a category
of entities with an additional filter. If the scopped category shares entities
with a `WriteOnly` query, it's probably that something happens that the user
didn't intend.

Therefore, false positives only exist for bad practices patterns,
and are acceptable in this case.

#### False negatives

False negatives only occur when the `WriteOnly` access is poorly scopped.
See the [`animation_player`](#animation_player) section.

## Future works

An advantage of this approach is that we can confidently extend `WriteOnly`
to any kind of component.

We could extend this to `GlobalTransform`, or `ComputedVisibilty` and enable
writing to those, since now we are capable to emit an error message with
no more overhead if they are misused.

Since `WriteOnly` is publicly accessible and is a `WorldQuery` like any other,
it is also possible for end-users and plugin authors to use it in their own
systems, to extend the write-only error detection to their own components
and systems.

## Alternatives

Let's be honest and fully transparent: This is an extremely niche use-case.

- **What we solve**: Confusion when a system mutates a component that is later
  unconditionally overwritten
  - Currently only relevant to `Transform` in ui and animation

- **What we do** to solve it:
  - Add a new `WorldQuery` with special semantics
  - Add a new method to `WorldQuery` to register write-only components
  - Add more system buraucracy to manage the write-only components
  - Add logic in scheduler to verify that components accessed by a system are
    not overwritten by write-only queries.
  - Add a component to animated entities (more on this later)

Whether the complexity is worth the gain is a serious consideration.

In short, this is why this RFC exists. It's questionable whether we want to add
it to bevy, and I wanted feedback from the community before implementing it.

## Drawbacks

### Fine-grained write-only

Consider a system that only writes-only to `Transform.rotation`. We can't mark
`Transform` as `WriteOnly`, since writing to the `Transform.translation` won't
be overwritten.

But a system that writes to the `rotation` will have its effect overwritten.

No way to account for that.

### Inter-frame dependency

Now this is another tricky bit. If we had added `shake_ui` after `transform_propagate`,
it would have had the same issue. But in the schedule. It doesn't appear
before `ui_layout_system`.

I'm not proposing to solve this in this RFC.

### `animation_player`

As far as I can tell, current day bevy has two systems that might overwrite
unconditionally existing component values:

- `node_transform_query` in `bevy_ui`
- `animation_player` in `bevy_animation`

using `WriteOnly` in `animation_player` would lead to some issue. Let's look
at its signature:

https://github.com/bevyengine/bevy/blob/469a19c290861049ad121aa83d72f830d1ec9b40/crates/bevy_animation/src/lib.rs#L355-L364

Notice:

```rust
    transforms: Query<&mut Transform>,
```

Ouch! Converting this to a `WriteOnly<Transform>` would be catastrophic! It would
match every entity with a `Transform` component, hence it would raise the
error **for all systems accessing `Transform`**. Not acceptable!

I see two solutions:

- Insert a `Animated` component to entities targeted by a `Transform` animation
  clip, and limit it to `Query<WriteOnly<Transform>, With<Animated>>`.
- Do not convert the query to a `WriteOnly<Transform>`.


# Feature Name: Macro-derived Substates

## Summary

Support for nested states (substates) as an extension of `States` derive macro.

## Motivation

Currently Bevy support flat states very well, but using any sort of nested states is heavily limited.

## User-facing explanation

We need to tell apart states in the hierarchy:
- Root state - Top-level state object, stored directly in the world as a `State<S>` resource, can contain sub-states.
- Sub-state - Not top-level state object, stored as part of some root state or another sub-state (which is eventually stored in a root state), can contain sub-states.

Let's use a simple game as an example.
We have a menu and gameplay which can be paused.

```rs
#[derive(States, ...)]
enum AppState {
    MainMenu,
    Gameplay(GameplayState),
}

#[derive(States, ...)]
enum GameplayState {
    Running,
    Paused,
}
```

The problem we'll immediatelly run into is, when we change `GameplayState`, we will experience a bunch of transition events for `AppState`.

To fix this, I propose 2 major changes to the `States` derive macro:
1. It will use a sub-macro `#[substate]` for marking sub-state fields.
  The marked field's type has to implement `States`.
1. If the struct/enum contains the macro above, a new object will be derived.
  The new object will be the same as original, but without the marked fields.
  If no sub-state macro is used, a type alias will be created instead for API continuity.
  I will refer to the derivatory object as `Just<original name>`, for example `JustAppState`, since it contains no other states. *the name needs bikeshedding*

The previous example will now look as following:
```rs
#[derive(States, ...)]
enum AppState {
    MainMenu,
    Gameplay(#[substate] GameplayState),
}
// enum JustAppState {
//     MainMenu,
//     Gameplay,
// }

#[derive(States, ...)]
enum GameplayState {
    Running,
    Paused,
}
// type JustGameplayState = GameplayState;
```

Adding the root state is unchanged.
```rs
app.add_state::<AppState>();
```

We can now use `OnExit`, `OnTransition` and `OnEnter` using the derivatory `JustAppState`, which will not trigger if a sub-state field changes.
```rs
app.add_system(OnEnter(JustAppState::Gameplay), generate_level)
app.add_system(OnExit(JustAppState::Gameplay), delete_level)
```

Changing state is still done through `NextState<AppState>`.
```rs
fn update_state(state: ResMut<NextState<AppState>>) {
    state.set(AppState::Gameplay);
}
```

`From<GameplayState>` can be implemented for `AppState` to simplify the usage.
```rs
impl NextState<S: States> {
    fn set<T: Into<S>>(state: T);
}
// [...]
fn update_state(state: ResMut<NextState<AppState>>) {
    state.set(GameplayState::Paused);
}
```

## Implementation strategy

The derive `States` macro will require a major rework, it needs to:
- allow for `#[substate]` macro,
- create the derivatory type or alias,
- work on structs and enums,
- implement methods for running state transition logic.

The sub-state macro is very simple, it has to check whether the field is of type that implements `States`.

The derivatory type/alias is easy as well, it will copy the original type, possibly implement a new trait like `JustStates` for usage in transition schedules.
Additionaly, the original type has to allow for conversion to the derivatory type (e.g. `AppState` -> `JustAppState`).
```rs
pub trait States {
    fn just(&self) -> JustSelf;
    ...
}
```

Structs will be treated as described above, for enums, each variant will be trated as a standalone struct.

The hardest part is state transition logic, which runs transition schedules.
It has to be implemented through a set of trait methods, so it can run recursively over sub-states.
```rs
pub trait States {
    fn just(&self) -> JustSelf;
    fn run_transition(&self, next: &Self, world: &mut World);
    fn run_exit(&self, &mut World);
    fn run_enter(&self, &mut World);
}
```

`run_transition` has to look as following:
1. Check if the state actually changed, return if it didn't.
1. Check whether it was this state or it sub-states that changed, by comparing derivatory types.
1. If only sub-state changed
    1. Run this function for sub-state.
1. If this state changed
    1. Run exit for all old sub-states.
    1. Run exit for this state.
    1. Run transition for this state.
    1. Run enter for this state.
    1. Run enter for all new sub-states.

`run_exit` and `run_enter` will run respective `OnExit` and `OnEnter` for all sub-states recursively.

## Drawbacks

One complex macro.

The derivatory type can be confusing.

## Rationale and alternatives

For alternatives check [Prior art](#prior-art).

This solution combines a single point of entry root state, with simple API.

All complxeity is hidden in the derive macro.

## Prior art

[State pattern matching.](https://github.com/bevyengine/bevy/pull/10088)  
Solves the issue, but obfuscates states and heavily relies on function macros.

[Sub-states through system ordering.](https://github.com/MiniaczQ/bevy-design-patterns/blob/better-states/patterns/instant-states-hierarchy/README.md)  
Simple and similar to existing API, but sub-states are stored in separate resources.

Ideally we'd want a similar API to the existing one, because it's easier to migrate and the current API is very user-friendly.

We also don't want to store the sub-states separately, because it introduces a risk of desynchronizing them with their parent states.

## Unresolved questions

Using the same sub-state type multiple times. This can result in weird exit-enter transitions for that state.

The scope of this RFC is relatively well defined and atomic.
The RFC is complete when the implementation is done.

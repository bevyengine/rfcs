# Feature Name: Macro-derived Substates

## Summary

Support for nested states (substates) as an extension of `States` derive macro.

## Motivation

Currently Bevy support flat states very well, but using any sort of nested states is heavily limited.

In general, attempting to detect change to specific level of state hierarchy will fail, either due to changes in a substate or parent state.
In other words:
- Parent state transition schedules detect changes in substates,
- Substate transition schedules detect changes in parent states.

The goal of this RFC is to address those problems, while still keeping all substates stored in the root state.

Ideally we'd want a similar API to the existing one.
It's user-friendly and easier to migrate.

## User-facing explanation

Few definitions that are used through the RFC:
- **Root state** - Top-level state object, stored directly in the world as a `State<S>` resource, can contain substates.
- **Substate** - Not top-level state object, stored as part of some root state or another substate (which is eventually stored in a root state), can contain substates.
- **Leaf state** - State at the end of hierarchy, does not contain substates.
- **Substate-free** (**SSF**) - Type derived from a state type, with all substate fields removed.

Let's use the following states as an example.
```rs
#[derive(States, ...)]
enum AppState {
    MainMenu,
    Gameplay(#[substate] GameplayState),
}

#[derive(States, ...)]
enum GameplayState {
    Running,
    Paused,
}
```

The updated `States` derive macro will generate the following additional types.
```rs
enum SsfAppState {
    MainMenu,
    Gameplay,
}

// Possibly type alias, check unanswered questions section
enum SsfGameplayState {
    Running,
    Paused,
}
```
which have all the `#[substate]` marked fields removed.

Adding the root state is unchanged.
```rs
app.add_state::<AppState>();
```

`OnExit`, `OnTransition` and `OnEnter` schedules will now use the new SSF state type variants instead.
This allows us to target changes of particular level in the state hierarchy, rather than all of them at once.
```rs
app.add_system(OnEnter(SsfAppState::Gameplay), generate_level)
app.add_system(OnExit(SsfAppState::Gameplay), delete_level)
```

The same applies to any substate.
```rs
app.add_system(OnEnter(SsfGameplayState::Running), resume_music)
app.add_system(OnExit(SsfGameplayState::Running), stop_music)
```

Changing state is still done through `NextState<S>` of the root state. (`S` is `AppState` in our case)
```rs
fn update_state(mut state: ResMut<NextState<AppState>>) {
    state.set(AppState::Gameplay);
}
```

## Implementation strategy

The `States` macro needs to do the following:
- Allow for `#[substate]` statement before fields,
- Create the substate-free type variant,
- Implement core transition logic,
- Implement substate transition logic.

`States` trait will take over most of the transition logic from the transition system.
This allows recursive transition logic for all substates.

The transition system has to be adjusted to use the `States`-defined transitions.

### `#[substate]` statement

State fields can be annoted with `#[substate]` to signify they hold a substate.
Annoted field's type has to implement `States`.

Examples:
```rs
struct Foo {
    #[substate]
    bar: Bar,
}

struct Alice(#[substate] Bob);

enum Food {
    Pasta {
        #[substate]
        sauce: Sauce
    }
}

enum Language {
    English(#[substate] Dialect),
}
```

### Substate-free variant (SSF)

Each state needs to have an SSF variant so we can match schedules, without also matching values of substates.

```rs
struct AppState {
    #[substate]
    menu: MenuState,
    #[substate]
    game: GameState,
    paused: bool,
}

// MACRO GENERATED
struct SsfAppState {
    paused: bool
}
```

This is analogous for tuple structs and enums.
If no fields are left, the brackets should be collapsed.

The SSF variant has to satisfy at least `Eq`.
Additional trait `SsfStates` may be introduced to help filter types in `OnExit`, `OnTransition`, `OnEnter` schedules,
which currently use the `States` trait as their generic requirements.

Any state can be turned into it's SSF variant through a `ssf(&self)` method.
This is intended for internal usage, users should create SSF variants directly during schedule matching.

```rs
trait States {
    type Ssf: Eq + Hash + Clone,

    fn ssf(&self) -> Self::Ssf;

    // Rest of trait
}
```

### Core transition logic

Core transition logic code is substate independent, it's always the same for every `States` implementation.
It relies on `Eq`, SSF and it's `Eq` implementations to decide whether it was this state that changed, or some substate.
If it was a substate that changed, we recursively run core transition logic on all substates.
If it was this state that changed instead, we perform the following steps:
- exit all current substates,
- exit current state,
- transition from current to next state,
- enter next state,
- enter all next substates.

```rs
trait States {
    fn transition(&self, next: &Self, world: &mut World)
    {
        // Impl
    }

    // Rest of trait
}
```

### Substate transition logic

Substate transition logic has to be generated by the derive `States` macro.
It operates solely on `#[substate]` marked fields, as opposed to SSF.

This logic consists of 3 methods:
- Transition substates - Run core transition logic for all substate fields,
- Exit substates - Starting from leaf, run `OnExit` for all substate fields, then self,
- Enter substates - Starting from leaf, run `OnEnter` for all substate fields, then self.

```rs
trait States {
    fn transition_substates(&self, next: &Self, world: &mut World);
    fn exit_substates(&self, world: &mut World);
    fn enter_substates(&self, world: &mut World);

    // Rest of trait
}
```

## Drawbacks

Hidden complexity of `States` types.

Generation of a new type, this can be confusing to users.

Macro complexity, it has to create a new type and implement 4 functions.
All the rules can be very well defined, but it is quite a lot.

## Rationale and alternatives

For alternatives check [Prior art](#prior-art).

This solution combines a single point of entry root state, with simple API.

All complxeity is hidden in the derive macro.

## Prior art

[Prototype without the macro.](https://github.com/MiniaczQ/bevy-design-patterns/tree/better-states-alt/patterns/better-states-alt/src)  
Proof of concept where all logic is implemented by hand, without the macro.

[State pattern matching.](https://github.com/bevyengine/bevy/pull/10088)  
Solves the issue, but obfuscates states and heavily relies on function macros.

[Substates through system ordering.](https://github.com/MiniaczQ/bevy-design-patterns/blob/better-states/patterns/instant-states-hierarchy/README.md)  
Simple and similar to existing API, but substates are stored in separate resources.

## Unresolved questions

Using the same substate type multiple times.
This can result in weird transitions where we exit then immediately enter the same substate.

Should leaf states generate type aliases for SSF variants or not?

The scope of this RFC is atomic and well defined.
The RFC is complete when the implementation is done.

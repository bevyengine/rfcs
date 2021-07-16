# Feature Name: `centralized_run_criteria`

## Summary

This feature lifts run criteria and related machinery out of individual stages, and centralizes them in the schedule instead.

## Motivation

Currently, run criteria are stage-specific: it's not possible to use the same criteria across two stages without creating two copies of it. Code and behavior duplication is bad enough on its own, but here it also results in unintuitive quirks in any mechanism built on top of criteria ([bevy#1671](https://github.com/bevyengine/bevy/issues/1671), [bevy#1700](https://github.com/bevyengine/bevy/issues/1700), [bevy#1865](https://github.com/bevyengine/bevy/issues/1865)).

## User-facing explanation

Run criteria are utility systems that control whether regular systems should run during a given schedule execution. They can be used by entire schedules and stages, or by individual systems. Any system that returns `ShouldRun` can be used as a run criteria.

Criteria can be defined in two ways:
- Inline, directly attached to their user:
```rust
fn run_if_player_has_control(controller: Res<PlayerController>) -> ShouldRun {
    if controller.player_has_control() {
        ShouldRun::Yes
    } else {
        ShouldRun::No
    }
}

stage
    .add_system(player_movement_system.with_run_criteria(run_if_player_has_control))
    .add_system(
        player_attack_system.with_run_criteria(|controller: Res<PlayerController>| {
            if controller.player_can_attack() {
                ShouldRun::Yes
            } else {
                ShouldRun::No
            }
        }),
    );
```
- Globally, owned by a schedule, usable by anything inside said schedule via a label:
```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, RunCriteriaLabel)]
enum PlayerControlCriteria {
    CanAttack,
}

schedule
    .add_run_criteria(run_if_player_can_attack.label(PlayerControlCriteria::CanAttack))
    .add_system(player_attack_system.with_run_criteria(PlayerControlCriteria::CanAttack))
    .add_system(player_dmg_buff_system.with_run_criteria(PlayerControlCriteria::CanAttack))
```

Since run criteria aren't forbidden from having side effects, it might become necessary to define an explicit execution order between them. This is done almost exactly as with regular systems, the main difference is that the label must be unique (to facilitate reuse, and piping - covered elsewhere):
```rust
schedule
    .add_run_criteria(run_if_player_can_attack.label(PlayerControlCriteria::CanAttack))
    .add_run_criteria(run_if_player_can_move.after(PlayerControlCriteria::CanAttack))
```

It should be noted that criteria declared globally are required to have a label, because they would be useless for their purpose otherwise. Criteria declared inline can be labelled as well, but using that for anything other than defining execution order is detrimental to code readability.

Run criteria are evaluated once at the start of schedule, any criteria returning `ShouldRun::*AndCheckAgain` will be evaluated again at the end of schedule. At that point, depending on the new result, the systems governed by those criteria might be ran again, followed by another criteria evaluation; this repeats until no criteria return `ShouldRun::Yes*`. Systems with no specified criteria run exactly once per schedule execution.

## Implementation strategy

- Move all fields related to systems' criteria from `SystemStage` to `Schedule`.
- Move and adapt criteria processing machinery from `SystemStage` to `Schedule`.
- Handle inline-defined criteria in system descriptors handed directly to stages.
- Plumb criteria or their results from `Schedule` to `SystemStage` for execution.
- Change stages' own criteria to work with criteria descriptors.
- Rearrange criteria descriptor structs and related method bounds to make globally declared criteria require labels.

## Drawbacks

Stages will become unusable individually, outside of a schedule.

Performance of poorly-factored (wrt this feature) schedules might decrease. However, correctly refactoring them should result in a net increase.

## Rationale and alternatives

This feature is a step towards `first_class_fsm_drivers`, `stageless_api`, and `lifecycle_api` (TODO: link to RFCs). While these are attainable without implementing `centralized_run_criteria` first, they all would have to retread some amount of this work.

One alternative is to provide the new API as an addition to the existing stage-local criteria. This would maintain usability of stages outside of a schedule, but could prove to be considerably harder to implement. It will definitely complicate the API, and will likely exist only until `stageless_api`.

## Unresolved questions

- It's unclear what the better approach is: implement this feature followed by `stageless_api`, or both of them at once.
- Most of the implementation details are unknowable without actually starting on the implementation.
- Should there be any special considerations for e.g. schedules nested inside stages?

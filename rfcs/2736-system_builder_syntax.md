# Feature Name: (fill me in with a unique ident, `system_builder_syntax`)

## Summary

Rewrite the system creation workflow to remove the excess of `add_startup_system_to_stage`-like methods and instead use the `system.stage().label().startup()` builder-like system.

## Motivation

Mostly described in [#1978](https://github.com/bevyengine/bevy/issues/1978).

- Allows us to unify the existing `.label()`, `.after()`, etc. methods and the stage positioning of systems.
- App would only have 3 methods to add systems: `add_system`, `add_exclusive` and `add_system_set`.
- Arguably more intuitive design (somewhat backed up by those who commented on #1978).

## User-facing explanation

- `add_startup_system(system)` becomes `add_system(system.startup())`
- `add_system_to_stage(stage, system)` becomes `add_system(system.stage(stage))`
- `add_startup_system_to_stage(stage, system)` becomes `add_system(system.startup().stage(stage))`
- `add_system(system.exclusive_system())` becomes `add_exclusive(system.exclusive_system())`
- `add_startup_system(system.exclusive_system())` becomes `add_exclusive(system.exclusive_system().startup())`
- `add_startup_system(system.label(x).before(y))` becomes `add_system(system.startup().label(x).before(y))`

The rest should follow.

## Implementation strategy

This has already been implemented in pr [#2736](https://github.com/bevyengine/bevy/pull/2736).

Similar to what was suggested by @cart in #1978:

```
trait System {
  /* same functions as before */
  fn config(&mut self) -> &mut SystemConfig;
}

trait LabelSystem {
  fn label(self, label: impl SystemLabel) -> Self
}

impl<T: System> LabelSystem for T {
  fn label(self, label: impl SystemLabel) -> Self {
    self.config().add_label(label);
    self
  }
}
```

- Each system impl stores a `struct SystemConfig` which stores data such as stage labels, run criteria, insertion points, etc.
- A trait is created for each type of Config(urability) such as a `trait StageConfig` which has a `stage(StageLabel)` function that sets the given StageLabel in the system's config struct.
- All config types are added to the prelude so that no more imports are required in the examples at the very least.
- All legacy system adding functions are removed and replaced with the basic ones described in the motivation. They will handle everything the same as the original ones did but in fewer functions.

## Drawbacks

- Requires changes over a large surface area.
- Changes the way end users interact with the API, requires a (short) adjustment period.
- Removes the current system descriptor system (existing prs may need reworking)
- Adds a minor memory overhead to each system.
- Minor boilerplate increase for each system impl.
- "Forces systems to own "schedule config", which muddles the "dependency tree" a bit. Systems currently exist a level "below" shcedules, which means other schedule apis could theoretically be built on top. This "mixes" the levels a bit." - @cart #1978

## Rationale and alternatives

- Alternative: Somehow write this on top of the existing system descriptors.
    - Would be signifcantly less "clean", where clean is a mix of line count and code "layers".
- Not implementing this rfc would likely hinder future users and devs of bevy in the long run since this improvement is more modular and can be built upon easily.

## Unresolved questions

- Are we happy to fundamentally alter the end user experience of Bevy and basically make all existing docs out of date.

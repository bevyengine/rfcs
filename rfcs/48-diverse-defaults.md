# Feature Name: Different plugin groups to support different use cases (diverse-defaults for short)

## Summary

There are many different categories of Bevy applications. Each use case has distinct needs, and so we should provide a collection of `PluginGroups` to give our users good defaults.

## Motivation

Bevy would be doing this to support easier and faster 'getting started' user story for non-minimal bevy applications. It will be particularly helpful to beginners but would also be a nice ergonomics/UX win for experienced bevy devs to not have to configure every minute detail manually when setting up a new project that is in one of the above outlined application domains.

## User-facing explanation

Where there used to just be `DefaultPlugins` and `MiniminalPlugins` there will now be many different `DefaultPlugins` a user can choose from that exist for many common categories of bevy applications.

Bevy now provides you a number of convenient `PluginGroups` to help you get your application started with sensible defaults for your use case. The current list is:

- `MinimalPlugins`: The very basic internal structure required to ensure things like `Time` function properly.
- `GamePlugins`: Suitable for making both 2D and 3D real-time games with input handling, audio and rendering (note: previously `DefaultPlugins`).
- `2dGamePlugins`: Like `GamePlugins`, but limited to the systems and resources needed in a 2D game.
- `TuiPlugins`: Suitable for making TUI applications.
- `ApplicationPlugins`: Suitable for making relatively static non-game applications.
- `SimulationPlugins`: Plugins that make writing scientific simulations easier like enhanced determinism and ordering guarantees.

## Implementation strategy

Each of the above outlined `Plugins` may have some combination of the following divergent platform configuration needs:

- desktop
 - native
 - web
- tablet
 - ios native
 - android native
 - web
- phone
 - ios native
 - android native
 - web

This feature also touches on not-yet-implemented but planned bevy features that will have different implementations per use case.

For example work-minimizing scheduling for low-power consumption web and mobile applications, or desktop productivity apps are a very different configuration from high performance gaming and high performance scientific simulation.

2D vs 3D will have different built-in bevy selection methods/mechanisms and different cameras.

Building for touch vs mouse vs gamepad or other midi/advanced input controllers different use cases with very different requirements.

## Drawbacks

The primary drawback I can see is the need for bevy to maintain the multiple `DefaultPlugins`, however, I foresee the maintenance burden of this to be exceedingly minimal.

## Rationale and alternatives

Each of these use cases can be currently be targeted in a custom manner by the application developer. This is fine for advanced users, but is less than an ideal user story for bevy beginners who just want to get something working as fast as possible lest they decide to try a different easier to use engine/framework. Were bevy to provide sane defaults for each use case, new users would be able to get something working in non-standard (i.e. not minimal `DefaultPlugins`) different scenarios very easily. This can also be highlighted by different book chapters per use case for the purposes of user education and better documentation of this feature.

The alternative to this proposal is for bevy to continue providing only `DefaultPlugins` and `MinimalPlugins` and for users to customize the plugins themselves through writing their own or using 3rd-party plugins.

## Unresolved questions

Which default plugin sets do we want to include right now, and how precisely do they differ?

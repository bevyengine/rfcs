# Feature Name: `fluent-internationalization`

## Summary

Internationalization is the practice of adapting your app's user interface to support multiple languages.
This RFC proposes how we can create a first-class integration with the [`fluent-rs`](https://github.com/projectfluent/fluent-rs) crate.

## Motivation

Internationalization is an important feature for serious games and apps, allowing them to expand their audience beyond a single language.
The technical challenges are well-understood, and not insurmountable, but retrofitting existing designs to accommodate them is notoriously painful.

Even for unilingual games, a high-quality internationalization language provides excellent support for dynamically changing text:
interpolating names, adapting nearby words to the correct plurality and gender, date and time formatting and so on.
This is very useful for building polished UIs in games, where the details of what a widget or dialog box should display often vary dramatically based on player actions.

## Guide-level explanation

TODO: Write.

Language Resource.

Labels for text and audio.

Assets should use real text if possible.

## Reference-level explanation

TODO: Write.

## Drawbacks

1. The data model for unilingual text is necessarily more complex.
2. Our dependency tree will grow.
3. We're forced to confront challenges around variable text length, left-to-right text, non-Latin glyphs etc. sooner.

## Rationale and alternatives

### Why does Bevy itself need internationalization?

1. By establishing a single standard, we can ensure that ecosystem crates (for e.g. dialog, widgets) all interoperate correctly.
2. Bevy should have text-enabled widgets in `bevy_ui`. If we don't support localization on those, they will quickly be obsoleted by third-party equivalents.
3. Bevy itself is responsible for solving many of the ancillary problems induced by variable text contents, such as automatically rescaling UIs.
4. By enabling internationalization by default, games can seamlessly mature, rather than having to rip out all related prototyping work.
5. Existing internationalization solutions in other game engines are typically bolted-on and second-class. Excellent integration with a high-quality internationalization solution offers a serious competitive advantage for multilingual indie developers and serious commercial game developers considering Bevy.

### Why Fluent?

TODO: Write.

## Prior art

An existing Bevy integration with `fluent-rs`, [`bevy_fluent`](https://crates.io/crates/bevy_fluent) shows that basic integration is quite simple.

[Project Fluent](https://projectfluent.org/) has a great overview explaining the limitations of simpler approaches to localization.

### Internationalization in other game engines

#### Unity

TODO: Research.

https://medium.com/i18n-and-l10n-resources-for-developers/localizing-unity-games-with-the-official-localization-package-5e2f59779d45

#### Unreal

TODO: Research.

https://docs.unrealengine.com/en-US/ProductionPipelines/Localization/Overview/index.html

#### Godot

TODO: Research.

https://docs.godotengine.org/en/stable/tutorials/i18n/internationalizing_games.html

## Unresolved questions

1. How do we ensure that the "simple case" of code-only static text in a single language is still easy?
2. How do we ensure that translators have appropriate access to context?
3. How do we generalize over different asset types?

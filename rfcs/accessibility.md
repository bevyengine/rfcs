# Feature Name: Accessibility

## Summary

Accessibility features are needed for applications and games. This RFC covers very high-level needs, and will probably require subsequent sub-RFCs to hash out the design details of the many potential accessibility features.

## Motivation

We have an ethical (and legal) imperative to consider the needs of all end-users of the games and applications made with Bevy. To be used for anything more than toy projects, accessibility features are a requirement.

## Guide-level explanation

Some common terms and features we should consider:
- TTS: Text-to-speech
- Color blindness modes
- Text resizing (or respecting OS settings)
- Rebinding inputs
- Navigating UI without a mouse

## Reference-level explanation

TODO: The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks

It takes time and effort to develop these features.

## Rationale and alternatives

Even if we can't commit the resources or find the expertise to implement some or all of these desired features, at the bare minimum we should keep these features in mind as we work on the rest of the engine. We can still actively take steps to avoid making accessibilty features more difficult to implement in the future.

## \[Optional\] Prior art

- https://github.com/lightsoutgames/godot-accessibility
- https://crates.io/crates/tts
- https://github.com/emilk/egui/issues/167

## Unresolved questions

- What standards and laws should/must we comply with?
- What users can we work with to validate what we are doing solves their accessibility needs?

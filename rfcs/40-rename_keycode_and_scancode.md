# Feature Name: `rename_keycode_and_scancode`

## Summary

* Rename the `KeyCode` type to `LogicalKey`
* Rename the `key_code` field on `KeyboardInput` to `logical_key`
* Rename the `scan_code` field on `KeyboardInput` to `physical_key`

In order to fall in line with [Winit's proposed changes to their API](https://github.com/bevyengine/bevy/issues/2052#issuecomment-830910091),
and also to correct the existing confusing naming inconsistencies bevy currently has compared with other projects.


## Motivation

There's a longer & more complete discussion I produced some 3 months ago [bevy#2386](https://github.com/bevyengine/bevy/discussions/2386).


Long story short Bevy's current names for these things have some problems:

* They are inconsistent with the underlying winit API.
* They are inconsistent with every Platform, UI toolkit, Game Engine that I checked (see discussion post).
* The names directly collide with some of those Platforms, UI toolkits & Game Engines.


For these reasons the current names could cause some confusion for users resulting in difficult to debug problems.


This is a pretty foundational API for Bevy and it should be worked out sooner lest it become more disruptive to rectify later.


## User-facing explanation

A `LogicalKey` represents the semantic value of the key as the user understands it.
On an OS set to QWERTZ, this means that pressing the 6th alphanumeric key on the top row will result in a `LogicalKey` with the value 'Z'.
You may have heard `LogicalKey` referred to as "key symbol" in other contexts.


The `physical_key` field represents the physical location of the key the user pressed, as it would appear on a standard US QWERTY keyboard.
This means the 6th alphanumeric key on the top should almost always result in the KeyboardInput's `physical_key` field having value 'Y'.
You may have heard `PhysicalKey` referred to as "scan code" in other contexts.


Generally `LogicalKey`s are useful when users are typing text, while the `physical_key` value is useful for layout dependent mappings
\- such as helping to ensure that the 'WASD' cluster appears in the same physical location regardless of language settings.


## Implementation strategy

* Rename the `KeyCode` type to `LogicalKey`
* Rename the `key_code` field on `KeyboardInput` to `logical_key`
* Rename the `scan_code` field on `KeyboardInput` to `physical_key`


This can be completed with a careful but exhaustive search and replace.


Care needs to be taken to update comments, examples, documentation, and to add details to the migration guide.


## Drawbacks

This is a disruptive change. Users migrating to the new engine will have to update their code.


## Rationale and alternatives

- Why is this design the best in the space of possible designs?

  * Falling in line with Winit's API (a large dependency of Bevy) will reduce cognitive load and potentially simplify development of Bevy.

  * Winit's API is very good. The names they use are self-describing.

- What objections immediately spring to mind? How have you addressed them?

  * Changing this API could cause some pain for users migrating from a previous version of Bevy. Luckily the fix is a simple search & replace.

- What is the impact of not doing this?

  * Not changing this could cause small but noticed frustrations among new Bevy users as the engine's community grows in size.

- Could this wait?

  * Changing this too much later on could be quite disruptive as both Bevy's community may simply be larger and more mature
    and also because in the future Bevy users may come to expect more stability than they do now.

- What other designs have been considered and what is the rationale for not choosing them?

  * There is some exploration of other solutions in [bevy#2386](https://github.com/bevyengine/bevy/discussions/2386).
    None of the other solutions seem as cohesive as the one proposed here.


## Prior art

There is a fairly substantial analysis of prior art in [bevy#2386](https://github.com/bevyengine/bevy/discussions/2386).


The overarching observation is that Bevy's current names are not directly consistent with any of the naming schemes used by Operating systems, UI Toolkits, nor other game engines.


In fact, since 'scancode' and 'keycode' are synonymous in many other contexts, Bevy's naming choice for a LogicalKey as `keycode` directly conflicts with some of those other projects, and is a potential source of confusion for users.


## Unresolved questions

- Is addressing this problem worth the disruption this change will cause?


## Future possibilities

A related issue that [cart observed](https://github.com/bevyengine/bevy/issues/2052#issuecomment-829721338) is that "physical key/scan codes" might not necessarily represent the actual physical key location depending on the OS and keyboard,
though this *isn't supposed to be the case on modern USB keyboard* (as discussed in [bevy#2386](https://github.com/bevyengine/bevy/discussions/2386)).


There seems to be hope that Winit will address this problem (and send the 'corrected' value as `physical_key`),
and I suppose just not expose a genuine raw scancode at all(?)
\- this seems unlikely to me, but it could happen.


Alternatively, in the original discussion I mentioned that there might be ways to get fair dinkum "actual key position" information by analysing OS settings such as locale & language, or querying some OS API.
This could very well be out of scope, but if it might ever be a goal of Bevy some thought needs to be given to how the naming of such an entity would fit in with the proposed `physical_key` & `LogicalKey`.
Would it replace `physical_key`, or would there be a third value added to the `KeyboardInput` struct? What would it be called?


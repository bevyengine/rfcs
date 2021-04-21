# Feature Name: Fixed Point Support

## Summary

There should be the ability in bevy to switch all of the transforms and basic shapes and whatnot to use fixed point. Ideally, it would allow for the user to pick what fixed point version they want to use (22 by 10, 52 by 14, etc), but a standard fixed point number type is fine too.

## Motivation

No other engine I can think of has fixed point support, and yet it is extremely useful for networking purposes, due to the fact that fixed points produce the same results on different architectures. Most people who work in networking and need determinism for their games often have to resort to either building their own fixed point libraries, or using third parties libraries that often times aren't very good. Additionally, trying to integrate a fixed point game system into an existing engine often results in the developer having to put in as much effort as building their own engine anyways.

## Guide-level explanation

Fixed point would fundamentally just require replacing all of the transforms and primitive data types (floats) with fixed points.

- There are two ways to do fixed point - a standard value, and the ability to choose your fixed point type.
- The standard value is the easiest way to go about this, but this would sacrifice flexibility.
- Being able to pick the type of fixed point you want would be a bit harder, but much more flexible.

## Reference-level explanation

* Fixed point numbers are split into two halves
  * The upper (for example 22 by 10, the upper would be 22) is the amount of bits that represents the whole number portion of a fixed number
  * The lower (22 by 10, lower would be 10) is the amount of bits that represents the fractional portion of a fixed number
  * You have to make sure that the amount of bits that make up the upper and lower when added up is a multiple of 32, so that it could fit as a 32bit, 64 bit, or (possibly) a 128 bit fixed point number

## Drawbacks

* It is a lot of work to do this, but since bevy is so young, I don't think it would be as big of a mission as adding it to something as mature as Godot.
* There's also the issue of fixed point numbers being substantially slower than floats.

## Rationale and alternatives

- The reason for fixed point numbers over something like floats is that there is the issue of floating point rounding errors, which can accumulate overtime and can lead to desyncs in networked games, which prevents cross platform gaming.
- Of course, you could do the process of restricting floats to the same architecture, but that would result in a limitation of the potential audience you could reach with your game, and that also has the potential of breaking anyways.
- The reason why we should do this is because no other engine has done this, because its such a specific feature to networking (deterministic lockstep and the like)
- But Bevy is an ECS engine, a organization structure expressly made for networking, and so it makes sense to have a networking focused feature like fixed point numbers in the engine.

## \[Optional\] Prior art

I have done a sample project testing out existing fixed point systems in rust and they seem to work fine.

## Unresolved questions

- How exactly would we go about adding the option to have fixed point to everything? Would it be a build feature, or a crate?
- Could we integrate this into a possible bevy_physics crate when that comes around?



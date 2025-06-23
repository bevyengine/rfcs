# Feature Name: `bevy_audio_graph`

## Summary

A more-advanced graph-based audio system for Bevy, leveraging relationships to define the graph, a
separate schedule to build and drive the audio, and the Web Audio API to create parity between
desktop and web applications.

## Motivation

Right now, audio in Bevy is extremely simple. You add an `AudioPlayer` somewhere in your application,
which contains a handle to an encoded audio file, and the audio file will be decoded on the fly
and sent directly to the master mixbus. This works for simple cases, but it has a lot of issues. Below
I've enumerated some of the issues that I see.

#### Clipping

Overlapping audio, or audio played too loud, has the potential to clip the master bus. This is trivial
to solve if it is possible to add effects to the master bus, as you can simply add a
[limiter](https://en.wikipedia.org/wiki/Limiter). In Bevy, however, there is currently no way to do
this.

#### No control over audio

Once a sound has been queued for playback, you lose control over it. You can stop the sound by deleting
the `AudioPlayer` component, but there is no way to change the speed of playback, pause and resume,
set a new location to play from, et cetera.

#### No concept of channels

It is often very useful to set a maximum number of sounds of a certain type to play at the same time,
or a cooldown on sounds of a certain type. Maybe you have a lot of enemies and you want to prevent them
from all talking over one another, maybe you want to have story-relevant dialogue prevent random chatter
from being played, maybe you just want to have a maximum number of explosion sound effects to be played
at once. Unlike some other aspects of the system that I'm discussing here, this one _can_ be implemented
in terms of Bevy's existing system, but it would be quite nice for this functionality to be built-in.

#### No audio metadata

The most common kind of metadata tracked to audio is subtitles. While you can add subtitles to a Bevy
game, by triggering them to start playing at the same time as the audio, this is usually something that
you want built into the audio system itself. That way, subtitles can be affected by the speed of the
playing audio (even when it changes), it can reuse some of the same information used by a channel
system - e.g. importance.

#### Sample-accurate syncing is impossible

A very important aspect of modern games is dynamic soundtracks. This is a system that includes multiple
"layers" which can be affected independently, while staying in sync. As audio has no concept of one
another's current playback progress and no metadata is available - particularly timing data - it is
essentially impossible to do this right now.

#### No effects

There are kinda-nasty hacks that one can do to add effects to a single sound, but it is rarely the case
that you want a sound to have its own effects. You usually want multiple sounds to be mixed together,
and only then have effects applied. This should not be a strict heirarchy, but a graph. If you want to
modulate the volume of a sound independent from how much reverb is applied to it (a very common operation),
there is no way to do that with just a series of effects unless every sound has its own reverb node - which
is very expensive.

In my opinion, the most important effect to have is a low-latency limiter on the master bus. This prevents
the audio from clipping if too many sounds are playing simultaneously.

## User-facing explanation

Explain the proposal as if it was already included in the engine and you were teaching it to another Bevy user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how Bevy users should *think* about the feature, and how it should impact the way they use Bevy. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Implementation strategy

This RFC contains two parts - the API that will be interacted with by the user, and the internal
implementation details. As there are many ways to potentially handle the latter, I will give an example of
a possible API design first.

#### API design

The audiograph should be handled using entity relations, and it should be rebuilt into a more-efficient
form when extracted to the audio world. An example set of components is given below, roughly in order of
importance.

```rust
/// Mark that an `AudioSource` should have its audio sent to an `AudioSink`. Any intermediate processing
/// is done by an entity with both an `AudioSource` _and_ `AudioSink` trait, with the sink being used
/// for input and the source being used for output.
///
/// For the purposes of backwards-compatibility and simplicity, it is probably desirable to have a concept
/// of a "global sink", where an `AudioSource` that has no outbound connections is automatically connected
/// to a singleton sink which will output to the system's audio.
#[derive(Component, Relationship)]
struct AudioTo {
}

/// For each `AudioSink` + `AudioSource` effect node in an audio effect chain. This is not necessary for
/// building the audio graph, but it _is_ useful for conceptually grouping nodes into a single chain.
/// It could be that this is more-effectively represented with the `Parent` relationship, but that probably
/// messes with spatialization.
///
/// The concept is that a `AudioChain` can be treated as an `AudioSink` + `AudioSource`, with the "sink"
/// and "source" behavior being differing depending on if a node is in that chain or not. If a node
/// that is marked as "in that chain" tries to get a chain as a source it will read from the audio sent
/// to the `AudioSink`, but if a node outside the chain tries to get that chain as a source it will read
/// the _output_ of the chain, after FX processing. This allows the effects in a chain to be modified
/// without updating any audio components that are sending audio to/from that chain, and it also allows
/// reading all the effects that are in a single conceptual chain.
///
/// > NOTE: While it may be useful for components to send audio to nodes that aren't the chain root or other
///         FX within a chain, I think for now it's probably for the best to deny that. This allows us to
///         potentially cache the graph for a chain as a single unit, which gives us more leeway to do
///         expensive graph optimizations on when building - for example, JIT compiling multiple effects into
///         a single node. While those optimizations are well out-of-scope for now, I think it's best to
///         have more limitations now - they can be always be circumvented by developers by splitting a chain
///         up into multiple pieces.
#[derive(Component, Relationship)]
struct InChain {
    // Marker(?)
}

/// Marker component for a DSP processing chain. This makes it easier to add helper methods for creating
/// and modifying FX chains.
#[derive(Component)]
#[require(AudioSink, AudioSource)]
struct AudioChain {
    // Marker(?)
}

/// A component marking that an entity can have audio sent to it.
#[derive(Component)]
struct AudioSink {
    // Marker(?)
}

/// A component marking that an entity can have audio from it.
#[derive(Component)]
struct AudioSource {
    // Marker(?)
}

/// Exactly the same as the component as it exists in the current version of Bevy. I believe that Bevy
/// already has the limitation that sources need to just be a buffer of audio data (and cannot be
/// arbitrary iterators of samples) - at least for sounds played with the default `AudioPlugin`. This
/// makes the migration path much simpler.
#[derive(Component)]
#[require(AudioSource, AudioPlayback)]
struct AudioPlayer<Source = AudioSource>(pub Handle<Source>)
where
    Source: Asset + Decodable;

/// A component that tells the audio graph to handle the "ear location" of a sink. Exact method of
/// spatialization TBD, as it is probably useful to have something slightly more configurable than just
/// modulating the gain - developers may want to modify the graph routing to bucket sources into
/// different distance levels that send the audio to progressively more-aggressive reverb configurations,
/// they may want to modulate a filter based on distance, etc.
#[derive(Component)]
#[require(AudioSink)]
struct SpatialSink {
}

/// A marker component that tells the audio graph to calculate the distance from a `SpatialSink` that
/// this component is routed to.
#[derive(Component)]
#[require(AudioSource)]
struct SpatialSource {
}

// Effects should, ideally, _not_ be implemented fully generically as DSP processing nodes. They should
// mirror the Web Audio API, and so have a set of core components like a convolver, biquad filter, delay,
// etc. that all high-level effects are built in terms of using required components. It may be useful in
// the future to have a lower-level API that allows writing DSP nodes by hand, but for now this should
// not be implemented to reduce the amount that needs to be designed.
mod effects {
    #[derive(Component)]
    #[require(AudioSource, AudioSink)]
    struct ReverbNode {
        // ..fields..
    }

    #[derive(Component)]
    #[require(AudioSource, AudioSink)]
    struct DelayNode {
        // ..fields..
    }

    #[derive(Component)]
    #[require(AudioSource, AudioSink)]
    struct LimiterNode {
        // ..fields..
    }

    // etc
}
```

#### Internal design

Notably, this system requires moving away from Rodio. This is unfortunate, but necessary. Rodio simply
does not support the kind of processing that we want - particularly in a web context. The `web-audio-api`
crate (which provides a pure-Rust implementation of Web Audio) is probably the best option to migrate to,
as it allows us to provide a single interface that works for desktop applications but is guaranteed to
map to the audio system supported on web platforms. For the purpose of backwards-compatibility, it may
be desirable to still use the `Decodable` trait for `AudioPlayer`, but this should be the only aspect of
`rodio` that remains.

The Web Audio API (and thus, the `web-audio-api` crate) has a built-in way to handle graph construction,
which I propose that we use directly. It may be useful in the future to coalesce chains of effects into
something more-efficient in the future, but as this is a design for the audio engine of a game and not
a piece of audio editing software, I do not expect effects chains to be particularly long or complex and
I think that we should prioritize limiting the amount that needs to be implemented in Bevy itself. I
believe that `web-audio-api` does _not_ automatically use the actual Web Audio API when compiled for
browsers, so a wrapper will need to be created that abstracts over the pure Rust implementation and the
web implementation.

While not necessary, I believe that it would make the most sense to implement the audio processing in its
own schedule, with its own extraction step. This allows us to decouple audio from the rest of the system,
which allows us to have more control over the transformation from the form of the audiograph in the ECS
to the internal "compiled" form built using the Web Audio API. It also means that we can prevent the audio
thread from stalling if the main thread is overloaded, and it could also potentially allow developers to
specify that they want audio extracted multiple times a frame in order to reduce input-to-sound latency.

## Drawbacks

The design of this introduces some more complication into Bevy's audio system, as well as a new crate to
rely on. Conceptually a graph-based system is more difficult for developers to comprehend than a more
limited system, although this can easily be hidden away when a user does not need that level of control.

## Rationale and alternatives

- This design would allow external crates to introduce a multitude of new features into the audio system
- While I do not propose a precise method of implementing talkback in this RFC, it lays the groundwork
  for adding things like debugging gizmos for sets of sounds - you can send all the sounds you want to
  debug to a single `AudioSink` which then sends info to draw a debugging interface back to the main
  schedule via some talkback system.
- While an audiograph is conceptually complex when you need its full power, it is trivial to build simpler
  systems on top of. A user who is happy with the existing system will never know it exists - they will
  simply add an `AudioPlayer` just as before, which will be automatically connected to a singleton global
  sink.
- An alternative would be to use Kira - already implemented in `bevy-kira-audio`. While I believe that
  Kira is an interesting project, it unfortunately does not provide enough benefits over Bevy's existing
  system to justify the switch, in my opinion.

## Unresolved questions

- Is `web-audio-api` production-ready? Should we only expose a limited subset in order to reduce possible
  unpolished corners of this library?
- How do we ensure that this doesn't break external libraries that interact with Bevy's audio system but
  do not fully replace it?
  - I believe that Bevy's existing audio system is currently limited enough that this change is unlikely
    to break too many things
- Do we want to expose a Web Audio-like API surface, or do we want to have something more low-level even
  if it significantly reduces audio performance when compiling for web?

## Future possibilities

#### Talkback

```rust
/// Sent to the main schedule when a buffer has been received.
///
/// In the future, it may be useful to parameterize this by buffer type, as developers may want to
/// do some custom processing in the audio schedule that results in a type that is not an audio
/// buffer. For example, they may want to compress the buffer in order to send it over the network.
#[derive(Event)]
struct AudioBufferReceived<Buffer = web_audio_api::AudioBuffer> {
    pub from: Entity,
    pub buffer: Buffer,
}

#[derive(Component)]
#[require(AudioSink)]
struct Talkback {
    // .. some talkback configuration options, such as how often a buffer is sent ..
}
```

#### Playback control

```rust
/// Control the playback of an `AudioPlayer`.
struct AudioPlayback {
    /// Set the playback speed, with the default being 1. Future versions could add things like
    /// using pitch-independent retiming rather than just slowing down playback (which affects
    /// pitch).
    pub speed: f32,
    // .. fields ..
}

#[derive(Event)]
struct Play;

#[derive(Event)]
struct Pause;

enum AudioTime {
    /// This needs to be a `chrono::Duration` to handle negative time.
    Time(chrono::Duration),
    /// Number of samples.
    Sample(u64),
    /// A fraction of the total time of the sound.
    Fraction(f32),
}

enum PlaybackPoint {
    /// Set playback to a specific time from the current playback point of the sound.
    FromCurrent(chrono::Duration),
    /// Set playback to a specific time from the start of the sound.
    FromStart(chrono::Duration),
}

#[derive(Event)]
struct Skip {
    pub new_playback_point: PlaybackPoint,
}
```

#### Source priortization

As mentioned in Motivation, it can be useful to limit the number of inputs to a sink. This can be done
simply on a first-come-first-served basis, or we could have some system that allows users to control
importance both per-source and based on some set of properties of that source.

```rust
#[derive(Component)]
struct AudioImportance {
    pub importance: f32,
}

/// Control how to handle sounds that have
#[derive(Default, Component)]
enum IgnoreBehavior {
    /// Delete the sound immediately.
    #[default]
    Drop,
    /// Queue the sound to be played, optionally timing out after a certain period. This can be
    /// useful for UI sounds.
    Queue {
        timeout: Option<Duration>,
    },
    /// Play the sound, but ignore its output until a slot for it is found.
    Mute,
}

/// A trait used for limiting the number of inputs to a node.
#[derive(Component)]
#[require(IgnoreBehavior)]
struct MaxInputFilter {
    /// The maximum number of inputs allowed to this node at any one time.
    pub max_inputs: usize,
}

/// Multiply importance by a value based on the peak amplitude of a source.
#[derive(Component)]
#[require(MaxInputFilter)]
struct AmplitudeFilter {
    pub loudness_map: Box<dyn Curve<f32>>,
}

/// Multiply importance by a value based on the perceptual loudness (LUFS) of a source.
#[derive(Component)]
#[require(MaxInputFilter)]
struct LoudnessFilter {
    pub loudness_map: Box<dyn Curve<f32>>,
}

/// Multiply importance by a value based on the distance that a source is spatially from a listener.
/// Note that this is _not_ the same as modulating the gain of a source based on distance - it is
/// for ignoring sources that are too far away when many sounds are playing at once.
#[derive(Component)]
#[require(MaxInputFilter, SpatialSink)]
struct DistanceFilter {
    pub distance_map: Box<dyn Curve<f32>>,
}
```

#### Custom DSP

Another useful possibility is custom audio processing. This RFC only proposes a set of hard-coded effects
based on what Web Audio provides. Those are enough to implement a huge amount of different effects, as
WebAudio's primitives are very generic, but users may want to implement custom audio processing units.
In my opinion, this would probably best be handled by a system of traits like how the graphics processing
graph is implemented, but the precise design is out of scope for this RFC.

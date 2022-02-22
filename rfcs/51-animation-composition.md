# Feature Name: `animation-composition`

## Summary
Animation is particularly complex, with many stateful and intersecting
elements. This RFC aims to detail a methodlogy for composing multiple animations
together in a generic and composable way.

## Motivation
Animation is at the heart of modern game development. A game engine without an
animation system is generally not considered production-ready. To provide Bevy
users a complete animation suite, simply start, stop, and seek operations on
singular clips of animation is not sufficient to cover all of the animation
driven use cases in modern video games. An animation system should be able to
support composing individual animations in a generic and flexible way. This would
enable artists, animators, and game designers to create increasingly dynamic
animations without needing to resort to reauthoring multiple variants of the same
animation by hand.

## Scope
Animation is a huge area that spans multiple problem domains:

 1. **Storage**: this generally covers on-disk storage in the form of assets as
    well as the in-memory representation of static animation data.
 2. **Sampling**: this is how we sample the animation assets over time and
    transform it into what is applied to the animated entities.
 3. **Application**: this is how we applied the sampled values to animated
    entities.
 4. **Composition**: this is how we compose simple clips to make more complex
    animated behaviors.
 4. **Authoring**: this is how we create the animation assets used by the engine.

This RFC only aims to resolve problems within the domain of composition. Storage
and sampling is addressed in [RFC #49][primitives]. Application can be distinctly
decoupled from these earlier two stages, treating the sampled values as a black
box output, and authoring can be built separately upon the primitives provided by
this RFC and thus are explicit non-goals here.

## User-facing explanation
Individual animations describe the state of a hierarchy of entities at a given
moment in time. The values in a given animation clip can be blended with one or
more others to produce a composite animation that would otherwise need to be
authored by hand. These blends are usually weighted to allow certain clips to
have a stronger influence the final animation output, and these weights may
evolve over time.

A common example of this kind of blending is smoothing out a transition from one
animation clip to another. As a character stops walking and starts running, bones
do not immediately snap into place when starting a dash. Blending can help here
by smoothing out the transition. Upon beginning the transition, the blend is
weighted only towards the walk, but smoothly transitions over time to favor the
run. This removes any potential discontinuities and provides for a visually
appealing transition from one animation to another.

Another example allows developers to convey a range of game states to the player
via animation. For example, if a character is hurt, running may get progressively
harder and harder. To convey this to the player, developers may decrease the
overall movement speed of the character while blending the normal walk and run
animations with a limp animation. As the character gets increasingly damaged, the
blend becomes more and more weighted towards the limp animation. Unlike the
transition case, this is a persistent blend and is not driven by time but by
gameplay design.

The `AnimationGraph` component allows for storing and blending the clips that
drive animation for an `Entity` hierarchy. The graph itself defines a public
interface for creating a directed acyclic graph of blend nodes, with the terminal
nodes in the graph being individual animation clips.

## Implementation strategy
*Prototype implementation: https://github.com/HouraiTeahouse/bevy_prototype_animation*

### Graph Nodes
An animation graph is comprised of a network of connected graph nodes. A public
facing API allwos developers to create, get, and mutate nodes. Nodes can be of
either two types: blend node or clip node. The graph always has one single root
node of either type.

Blend nodes do not contain any animation of their own, but can have zero or more
input edges. Each of these edges has a `f32` weight associated with it and points
to a source node, which can be either clip node or another blend node. The weight
of parent to child edge affects the end weighting of all descendant clip nodes.

Clip nodes cannot have inputs, but contain a reference to an animation clip. A
animation graph can have multiple clip nodes referring to same underlying clip.

Edges can be disconnected without losing metadata on it's input. This is
functionally equivalent to setting a weight of 0, but the associated input node
and it's descendants will not be evaluated.

Every node has it's own time value. For clip nodes, this is the time at which the
underlying clip is sampled at. For blend nodes, this value doesn't do anything.
However, an optional setting will allow a blend node to propagate the time value
to all of it's immediate children. If this option is set for it's chilren, it
will continue to propagate downwards. This is disabled by default. Another option
for advancing the animation is to advance the entire graph's time in one go. This
will increment the time value on all nodes by a provided time delta.

### Graph Storage
Two main stores are needed internally for the graph: node storage and clip
storage. Nodes are stored in a flat `Vec<Node>`, and a newtyped `NodeId` is used
to keep a reference to a node within the graph. Due to this structure, nodes
cannot be removed from the graph. As a worksaround, a node can be effectively
removed by disconnecting all of it's incoming and outgoing edges. The root NodeId
is always 0.

Clip storage decomposes clips based on the properties each clip is animating.
Instead of using `Handle<AnimationClip>`, each clip is instead internally
assigned a auto-imcrementing `ClipId` much like `NodeId`. However, instead of
storing a `Vec<AnimationClip`, a map of property paths to `Track<T>` is stored.

`Track<T>` stores a contiguous sparse array of pointers to all curves associated
with a given property. Here's a rough outline of how such a struct might look:

```rust
struct Track<T> {
  curves: Vec<Option<Arc<dyn Curve<T>>>>,
}
```
The individual curves stored within are indexed by `ClipId`, and each pointer to
a curve is non-None if and only if the original clip had a curve for the
corresponding property.

A type erased version of `Track<T>` will be needed to store the track in a
concrete map. The generic type parameter of the sample function will be used to
downcast the type erased version into the generic version.

As mentioned in the prior primitives RFC, there will likely need to be a separate
clip storage dedicated to storing specialized `Transform` curves for the purposes
of skeletal animation that avoids both the runtime type casting and the dynamic
dispatch of `Arc<dyn Curve<T>>`.

### Graph Evaluation
To remove the need to evaluate the graph every time a property is sampled, an
influences map is constructed based on the current state of the graph. This is
effectively a `Vec<f32>` indexed on `ClipId` mapping clips and their respective
*cumulative* weights. To avoid constant reallocation, this influences map is
stored as a companion component and is cleared before every evaluation.

During evaluation, the graph is traversed along every connected edge and weights
are multiplicatively propagated downward. For example, if we have a simple graph
`A -> B -> C`, where A is the root and C is the final clip and the edges `A -> B`
and `B -> C` have the weights 0.5 and 0.25 respectively, the final weight for
clip C is 0.125. If multiple paths to a end clip are present, the final weight
for the clip is the sum of all path's weights.

After traversing the graph, the weights of all active inputs are normalized.

By default, a dedicated system will run before sampling every app tick that
evaluates every changed graph, relying on change detection to ensure that any
mutation to the graph results in a change in the influences. This behavior can be
disabled via a config option, requiring users to manually evaluate the graph
instead.

### Graph Sampling
Sampling a single value from the current state of the graph has the rough
following flow:

 - The corresponding track is located in clip storage based on the property
   path. Fail if not found.
 - The curves for the active clips is retrieved from the track's sparse array of
   curves.
 - Individual values are sampled from each active curve. Skipping curves with a
   weight of 0.
 - `Animatable::blend` is used to blend the sampled values together based on the
   weights in the influences map.

This approach has a number of performance related benefits:

 - It only requires one string hash lookup instead of one for every active clip.
 - The state of the graph does not need to be evaluated for every sampling.
 - Curves can be sampled in `ClipId` order based on the influence map. This
   parallel iteration should be cache friendly.
 - No intermediate storage is inherently required during sampling.
   (`Animatable::blend` may require it for some types).
 - This process is (mostly) branch free and can be accelerated easily with SIMD
   compatible `Animatable::blend` implementations.

## Drawbacks
TODO: Complete this section

## Rationale and alternatives
TODO: Complete this section

## Prior art
This proposal is largely inspired by Unity's [Playable][playable] API, which has
a similar goal of building composable time-sequenced graphs for animation, audio,
and game logic. Several other game engines have very similar APIs and features:

 - Unreal has [AnimGraph][animgraph] for creating dynamic animations in
   Blueprints.
 - Godot has [animation trees][animation-trees] for creating dynamic animations in
   Blueprints.

The proposed API here doesn't purport or aim to directly replicate the features
seen in these other engines, but provide the absolute bare minimum API so that
engine developers or game developers can build them if they need to.

Currently nothing like this exists in the entire Rust ecosystem.

[playable]: https://docs.unity3d.com/Manual/Playables.html
[animgraph]: https://docs.unrealengine.com/4.27/en-US/AnimatingObjects/SkeletalMeshAnimation/AnimBlueprints/AnimGraph/
[animation-trees]: https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html

## Unresolved questions
 - Is there a way to utilize change detection for graph evaluation without
   having a component exposed in userspace?

## Future possibilities
This RFC only details the lowest level interface for controlling the composition
and blending of multiple animations together and requires code to be written to
manually control the weight and time of every input going into the graph. This
provides signfigant flexibility, but isn't accessible to artists and animators
that don't have or need to interface at such a low level. One major potential
future extension is to expose an asset driven workflow for creating animation
graphs or state machines (i.e. Unity's Animation State Machine).

Another potential extension is to allow this graph-like composition structure for
non-animation purposes. Using graphs for low level composition of audio
immediately comes to mind, for example.

[primitives]: https://github.com/bevyengine/rfcs/pr/49

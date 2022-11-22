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

This RFC only aims to resolve problems within the domain of composition and
application. Storage and sampling is addressed in [RFC #49][primitives], and
authoring can be built separately upon the primitives provided by this RFC and
thus are explicit non-goals here.

[primitives]: https://github.com/bevyengine/rfcs/pr/49

## User-facing explanation
An `AnimationGraph` is a component that lives on the root of an entity hierarchy.
Users can add clips to it for it to play. Unlike the current `AnimationPlayer`,
it does not always exclusively play one active animation clip, allowing users to
blend any combination of the clips stored within. This produces a composite
animation that would otherwise need to be authored by hand.

Each clip has its on assigned weight. As a clips weight grows relative to the
other clips in the graph, it's influence on the final output grows. These weights
may evolve over time to allow a smooth transition between animations.

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

Each clip has its on assigned time. Some clips can advance faster or slower than
others. This allows more fine grained control over individual animations than
using the global time scaling feature to speed up or slow down animations. This
can be tied to some gameplay element to produce more believable effects. For
example, speeding up the run animation for a character when they pick up a speed
up can be a cheap and effective way to convey the power up without needing to
author a speed up manually.

### Graph Nodes
To help manage the blending of a large number of clips, an animation graph is
comprised of a network of connected graph nodes. A public facing API allows
developers to create, get, and mutate nodes. Nodes can be of either two types:
blend node or clip node. The graph always has one single root node of either
type.

Clip nodes cannot have inputs, but contain a reference to an animation clip. A
animation graph can have multiple clip nodes referring to same underlying clip.

Blend nodes mix all of the provided inputs using a weighted blend. Blend nodes do
not contain any animation of their own, but can have zero or more inputs. Each of
these edges has a `f32` weight associated with it and points to a source node,
which can be any other node, including another blend node. The weight
of parent to child edge affects the end weighting of all descendant clip nodes.

Every node has it's own time value. For clip nodes, this is the time at which the
underlying clip is sampled at. For blend nodes, this value doesn't do anything.
However, an optional setting will allow a blend node to propagate the time value
to all of it's immediate children. If this option is set for it's chilren, it
will continue to propagate downwards. This is disabled by default. Another option
for advancing the animation is to advance the entire graph's time in one go. This
will increment the time value on all nodes by a provided time delta multiplied by
a clip node's.

### Graph Edges
Edges can be disconnected without losing metadata on it's input. This is
functionally equivalent to setting a weight of 0, but the associated input node
and it's descendants will not be evaluated, which can cut down CPU time spent
evaluating that subgraph.

### Example Graph

```mermaid
flowchart LR
  root([Root])
  mixerA([Mixer A])
  mixerB([Mixer B])
  clip01([Walk])
  clip02([Run])
  clip03([Jump Start])
  clip04([Jump])
  clip05([Jump End])
  clip06([Punch])
  mixerA-->root
  mixerB-->root
  clip06-->root
  clip01-->mixerA
  clip02-->mixerB
  clip03-->mixerB
  clip04-->mixerB
  clip05-->mixerB
```

## Implementation strategy
*Prototype implementation: https://github.com/HouraiTeahouse/bevy_prototype_animation*

### Overview
There are several systems that the animation system uses to drive animation from
a graph:

 1. **Graph Evaluation**: This system evaluates the state of the graph to generate an
    influences map that determines how the clips stored within the graph are going
    to be blended. This can be done in parallel over every graph individually.
    This is also when the internal clocks of the graph are ticked.
 2. **Binding**: This system traverses the entity hierarchy starting at the
    graphs to find the corresponding entity for each bone animated by the graph.
    This can be done in parallel over every graph individually.
 3. **Graph Sampling**: This system samples the graph for values from the active
    clips, uses the generated influences map (from step 1) to blend them into
    their final values, and applies it to the bound bones (from step 2).

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
```rust
struct Influence {
    weight: f32,
    time: f32,
}

struct GraphInfluences {
    influence: Vec<Influence> // Indexed by BoneId
}
```

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

After traversing the graph, the weights of all active inputs are clamped and normalized.

By default, a dedicated system will run before sampling every app tick that
evaluates every changed graph, relying on change detection to ensure that any
mutation to the graph results in a change in the influences. This behavior can be
disabled via a config option, requiring users to manually evaluate the graph
instead.

### Binding

```rust
struct BoneId(usize);

#[derive(PartialEq, Eq, ParitalOrd, Ord)]
struct EntityPath {
    path: Vec<Name>,
}

struct GraphBindings {
    paths: BTreeMap<EntityPath, BoneId>,
    bones: Vec<Bone>,
}

struct Bone {
    properties: BTreeMap<FieldPath, Track>,
}

#[derive(Resource)]
struct BoneBindings(SparseSet<Entity, SmallVec<[(Entity, BoneId), 2]>>);
```

After graph evalution is building a global `BoneBindings` resource. This is a
newtype over a mapping between an animated entity (the key), and the
`AnimationGraph`s and corresponding `BoneId`s the entity is bound to.

`BoneBindings` allows us to safely mutate all animated entities in parallel. This
still requires use of `RefectComponent::reflect_unchecked_mut` or
`Query::get_unchecked`, but we can be assured that there is no unsound mutable
aliasing as each entity is only accessed by one thread at any given time.

`BTreeMap`s are used internally as they're typically more iteration friendly
than `HashMap`. Iteration on `BTreeMap` is `O(size)`, not `O(capacity)` like
HashMaps, and is array oriented which tends to be much more cache friendly than
the hash-based scan. It also provides a consistent iteration order for the
purposes of determinism.

Rough psuedo-code for building `BoneBindings`:

```rust
fn build_bone_bindings(
  graphs: Query<(Entity, &AnimationGraph), Changed<AnimationGraph>>,
  queue: Local<ThreadLocal<...>>, // Parallel queue
  mut bindings: ResMut<BoneBindings>,
) {
    graphs.par_for_each(|(root, graph)| {
        for (path, bone_id) in graph.bindings() {
            if let Ok(target) = NameLookup::find(root, path) {
                queue.push((target, (root, bone_id)))
            } else {
                warn!("Could not find bone at {}", path);
            }
        }
    });

    bindings.clear();
    update_bindings(&mut bindings, parallel_queue);
}
```

### Graph Sampling
All animated properties on an entity are sampled at the same time.
Sampling a single value from the current state of the graph has the rough
following flow:

 - All field paths and tracks for a given entity are fetched for a given bone
   from the associated graph. Fail if not found.
 - The curves for the active clips is retrieved from the track's sparse array of
   curves.
 - Individual values are sampled from each active curve. Skipping curves with a
   weight of 0.
 - `Animatable::blend` is used to blend the sampled values together based on the
   weights in the influences map.

This approach has a number of performance related benefits:

 - The state of the graph does not need to be evaluated for every sampling.
 - Curves can be sampled in `ClipId` order based on the influence map. This
   iteration should be cache friendly.
 - No allocation is inherently required during sampling.  (`Animatable::blend`
   may require it for some types).
 - This process is (mostly) branch free and can be accelerated easily with SIMD
   compatible `Animatable::blend` implementations.

Rough pseudo-code for sampling values with an exclusive system.

```rust
fn sample_animators(
  world: &mut World, // To ensure exclusive access
  bindings: Res<BoneBindings>,
) {
    let world: &World = &world;
    let type_registry = world.get_resource::<TypeRegistry>().clone();
    // Uses bevy_tasks::iter::ParallelIterator internally.
    bindings.par_for_each_bone(move |bone, bone_id, graph| {
        let graph = world.get::<AnimatorGraph>();
        for (field_path, track) in graph.get_bone(bone_id).tracks() {
            assert!(field_path.type_id() != TypeId::of::<AnimationGraph>());
            let reflect_component = reflect_component(
                field_path.type_id(),
                &type_registry);
            let component: &mut dyn Reflect = unsafe {
                reflect_component
                    .reflect_unchecked_mut(world, bone)
            };
            track.sample(
                &graph.influences(),
                component.path_mut(field_path));
        }
    });
}

impl<T: Animatble + Reflect> Track<T> {
    fn sample(&self, influences: &GraphInfluences, output: &mut dyn Reflect) {
        // Samples every track at the graph's targets
        let tracks = self.curves
            .zip(&influences)
            .filter(|(curve, _)| curve.0.is_some())
            .map(|(curve, influence)| BlendInput {
                value: curve.sample(influence.time),
                weight: influence.weight,
                ...
            });

        // Blends and uses Reflect to apply the result.
        let result = T::blend(inputs);
        output.apply(&mut result);
    }
}
```

## Drawbacks
The animation sampling system is an exclusive system and blocks all other systems
from running.

## Rationale and alternatives

### Transforms only
An alternative (easier to implement) version that doesn't use `Reflect` can be
implemented with just a `Query<&mut Transform>` instead of a `&World`, though
`Query::get_unchecked` will still need to be used. This could be used as an
intermediate step between the current `AnimationPlayer` and the "animate
anything" approach described by the implementation above.

### Generic Systems instead of Reflect
An alternative to the `Reflect` based approach is to use independent generic
systems that read from `BoneBindings`, but this requires registering each
animatable component and will not generically work with non-Rust components, it
will also not avoid the userspace `unsafe` unless the parallel iteration is
removed.

### Relatonal `BoneBinding` as a Component
Instead of using `BoneBindings` as a resource that is continually rebuilt every
frame it's changed, `BoneBinding` could be a relational component, much like
`Parent` or `Children`. This removes the need to scan the named hierarchy every
frame, and allows trivial parallel iteration via `Query::par_for_each`. However:

 - it does not avoid the need for userspace `unsafe`.
 - maintaining the binding components is going to be a nightmare if there are
   any hierarchy or name changes underneath an `AnimationGraph`.
 - Commands require applying buffers in between maintanence and use in queries,
   forces creation of a bottleneck sync point.
 - Using it as a component means there will be secondary archetype moves if a
   name or hierarchy changes, which increases archetype fragmentation.

### Combining Binding and Sampling Steps: Parallelizing on AnimationGraph
If we use the same `unsafe` tricks while parallelizing on `AnimationGraph`,
there's a risk of aliased mutable access if the same entity's components are
animated by two or more `AnimationGraph`s.


## Prior art
This proposal is largely inspired by Unity's [Playable][playable] API, which has
a similar goal of building composable time-sequenced graphs for animation, audio,
and game logic. Several other game engines have very similar APIs and features:

 - Unreal has [AnimGraph][animgraph] for creating dynamic animations in
   Blueprints.
 - Godot has [animation trees][animation-trees] for creating dynamic animations in
   Blueprints.

Within the Rust ecosystem, [Fyrox][fyrox] has it's own animation system that is
signifgantly higher level than this implementation. At time of writing, it
supports blending, but does not support arbitrary property animation, only
supporting transform based skinned mesh animation. Fyrox supports a high level
state machine that can be used to transition between different states and blend
trees, much like Unity's Mecanim animation system.

The proposed API here doesn't purport or aim to directly replicate the features
seen in these other engines, but provide the absolute bare minimum API so that
engine developers or game developers can build them if they need to. It's
entirely feasible to construct a higher level state machine animator like Unity's
Mecanim or Fyrox's animation system on top of the graph detailed here.

[playable]: https://docs.unity3d.com/Manual/Playables.html
[animgraph]: https://docs.unrealengine.com/4.27/en-US/AnimatingObjects/SkeletalMeshAnimation/AnimBlueprints/AnimGraph/
[animation-trees]: https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html
[fyrox]: https://fyrox-book.github.io/fyrox/animation/animation.html

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

### Potential Optimizations
[#4985](https://github.com/bevyengine/bevy/issues/4985) details one potential
optionization. `ReflectComponent::reflect_component_unchecked` calls `World::get`
internally, which requires looking up the entity's location for every component
that is animated. If an alternative that allows using a cached `Entity{Ref, Mut}`
can save that lookup from being used repeatedly. Likewise, caching the
`ComponentId` for a given `TypeId` when fetching the component can save time
looking up the type to component mapping from `Components`.

Using `bevy_reflect::GetPath` methods on raw strings requires string comparisons,
which can be expensvie when done repeatedly.
[#4080](https://github.com/bevyengine/bevy/issues/4080) details an option to
pre-parse the field path in to a sequence of integer comparisons instead. This
still incurs the cost of dynamic dispatch at every level, but it's signfigantly
faster than doing mutliple piecewise string comparisons for every animated field.

A direct path for `Transform` based animation that avoids dynamic dispatch might
also save on that very common use case for the system.

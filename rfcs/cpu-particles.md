# Feature Name: `cpu-particles`

Prototype Implementation: https://github.com/james7132/bevy_prototype_particles

## Summary
Provide a modular set of components and systems for authoring and controlling
CPU-simulated particle systems.

## Motivation
Games frequently require visual effects that are more fluid in nature, like fire,
lightining, or smoke. These effects are difficult, or impossible to simulate with
static or skinned meshes.

This RFC aims to detail an efficient CPU-bound implementation for modularly
creating, simulating, and rendering these kinds of effects.

## User-facing explanation
Each particle system is a collection of simple billboarded quads with the following
additional properties: position, 2D rotation, velocity, angular velocity, color,
texture UVs, and lifetime. When spawned, a particle is assigned a lifetime, which
ticks down frame by frame, until it's destroyed when it reaches 0.

Particles themselves are not entities, but are instead contained within a
`Particles` component, which stores the live state of all particles in a given
particle system as the general configuration for particle initialization.
Particles do not have a stable ID and attempting to store one is meaningless.

`ParticleMaterial` is a material component for rendering each particle. Main hook
for into the rest of the render graph. Adding this to a entity with `Particles`
will render the inner state of `Particles`. In this case, `Particles` does the
same job that a `Handle<Mesh>` component does in the normal rendering flows.

`ParticleEmitter` is a component for controlling the emission behavior of
particles, must be used in tandem with `Particles`. It controls the timing and
shape of how particles are spawned. One or more  `EmitterModifier` trait
object(s) can be added to an emitter to change initialization parameters for each
particle.

Additional optional components collectively known as simulation modifiers will
alter the post-initialization behavior of particles:

 - `VelocityOverLifetime` - Scales the speed of the particle over the course of
   it's lifetime.
 - `RotationOverLifetime` - Changes the rotation of the particle over the
   course of it's lifetime.
 - `SizeOverLifetime` - Scales the size of the particle over the course of it's
   lifetime.
 - `ColorOverLifetime` - Changes the color of the of the particle over the course
   of it's lifetime.
 - `RotationBySpeed` - Changes the rotation of the of the particle based on it's
   speed.
 - `SizeBySpeed` - Scales the size of the particle based on it's speed.
 - `ColorBySpeed` - Changes the color of the of the particle based on it's speed.
 - `ParticleNoise` - Adds noise to how particles move/rotate. Useful for creating
   turbulent systems.
 - `ForceOverLifetime` - Applies a constant force to all particles based on how
   long it has lived.

Critical to the "fluid" nature of the simulation, all of these components are
parameterizable using either ranges or animation curves to sample values from.

For utility, a `ParticleBundle` will be provided with a `Particles`,
`ParticleEmitter`, `ParticleMaterial`, and other required rendering components.

An example creating a round slow-burning fireball:

```rust
fn create_fireball(
  mut textures: Res<Assets<Textures>>,
  mut commands: Commands,
) {
  commands
    .spawn()
    .insert_bundle(ParticleBundle {
      particles: Particles::new(ParticleConfig {
        max_particles: 2500,
        ..Default::default(),
      }),
      emitter: PaticleEmitter::sphere()
        //
        .with_initial_velocity(0.0)
        .with_initial_size(0.25..0.5)
        .with_lifetime(0.75..2.0)
        .with_initial_color(Color::WHITE)
        // Fifty particles per frame
        .add_burst(50, Duration::from_millis(0))
        // Repeat every frame
        .repeat(),
      particle_material: ParticleMaterial {
        texture: textures.load("fire.png"),
      },
      ..Default::default(),
    })
    // Add color over lifetime to have it fade away
    .insert(ColorOverLifetime {
      // Fade out like a fire.
      color: CurveVariable::<Color>(
        vec![
          Color::WHITE,
          Color::YELLOW,
          Color::ORANGE,
          Color::RED,
          Color::MAROON,
        ]
    });
}
```

Just by changing the parameters a bit, it's possible to create a single burst
explosion:

```rust
fn create_fireball(
  mut textures: Res<Assets<Textures>>,
  mut commands: Commands,
) {
  commands
    .spawn()
    .insert_bundle(ParticleBundle {
      particles: Particles::new(ParticleConfig {
        // Only support 300 large
        max_particles: 300,
        ..Default::default(),
      }),
      emitter: PaticleEmitter::sphere()
        // Fast outward movement
        .with_initial_velocity(8.0..10.0)
        .with_initial_size(3.0)
        .with_lifetime(0.25..0.75)
        .with_initial_color(Color::WHITE)
        // Add one large burst at the start. Do not repeat.
        .add_burst(300, Duration::from_millis(10)),
      particle_material: ParticleMaterial {
        texture: textures.load("fire.png"),
      },
      ..Default::default(),
    })
    // Add color over lifetime modifier to have particles grow colder as they
    // age.
    .insert(ColorOverLifetime {
      // Fade out like
      color: CurveVariable::<Color>(
        vec![
          Color::WHITE,
          Color::YELLOW,
          Color::ORANGE,
          Color::RED,
          Color::MAROON,
        ]
    });
}
```

## Implementation strategy

### Particles

The heart and core of this system is `Particles`, a struct-of-arrays component
that contains all of the particle data:

```rust
#[derive(Clone, RenderResources)]
pub struct Particles {
  positions: Vec<Vec3>,
  rotations: Vec<f32>,
  sizes: Vec<f32>,
  colors: Vec<Vec4>,
  #[render_resources(ignore)]
  velociites: Vec<Vec3>,
  #[render_resources(ignore)]
  angular_velociites: Vec<f32>,
  #[render_resources(ignore)]
  remaining_lifetimes: Vec<f32>,
  #[render_resources(ignore)]
  starting_lifetimes: Vec<f32>,
}
```

This means that particles are not entities onto themselves, and thus cannot be
queried as entities. The tight locality of all of the fields should allow for
faster single-threaded iteration due to higher cache coherency and easier
auto-vectorization.

Particles are created by appending the associated data to the end of each field
of each Vec, and can be destroyed by calling `swap_remove` on the given
particle's index. This also can be extended to allow spawning/destroying batches
of particles efficiently.

When rendering, no local to world Mat4 needs to be computed CPU side, instead
opting to calculate the matrix on the GPU via TRS in the vertex shader. This
normally is considered wasteful when rendering normal meshes, but given the
number of particles with unique transforms, and the number of vertices per
particle (4), this approach may end up being faster. (TODO(james7132):
Benchmark). Separation into field buffers also makes it trivial to include each
buffer in GPU instanced draw calls.

For utility access `Particle<'a>`, `ParticleMut<'a>`, and `ParticleParams` are
added for read-only, mutable, and owned structs that hold particle fields.

### Particle Emitters
```rust
#[derive(Clone)]
pub struct ParticleEmitter {
  next_burst: Timer,
  bursts: Vec<EmitterBurst>,
  shape: EmitterShape,
  repeat: bool,
  modifiers: Vec<Box<dyn EmitterModifier>>,
}

pub struct EmitterBurst {
  count: Range<usize>,
  wait: Duration,
}

pub struct EmitterShape {
  Sphere { .. },
  Hemisphere { .. },
  Line { .. },
  ...
}

pub struct EmitterModifier: Send + Sync + 'static {
  fn modify(&mut self, params: &mut ParticleParams);
}
```

ParticleEmitters are used to track when the next burst of particles should be
generated.

The `EmitterBurst` vec details when and how many particles to spawn. When
`next_burst` evaluates to finished, the emitter will randomly select how many
particles to spawn, use `EmitterShape` to sample random points inside the shape,
create one or more `ParticleParams`, modify the initialization parameters with
all provided `EmitterModifiers`, and then use the `ParticleParams` to add new
particles to `Particles`.

### Simulation Modifiers
Simulation modifiers are simple POD components that apply some modiifer to
particle behavior via Bevy's systems, typically querying only for itself and a
`Particles` on the same entity.

These modifiers are optional and often not found on most particle systems. These
should use SparseSet storage.

### Randomness
Randomness is at the core of making visually appealling particle systems, and
almost all parameters in `ParticleEmitter`, `EmittterModifier` and simulation
modifiers should take some form of `Range<T>` as an imput for sampling random
values from.

If reproducible/deterministric particle systems are desirable, a main RNG
for the entire particle system must be exposed. Likewise, each particle should
contain its own random seed to avoid blocking on the global RNG.

### Parameterization
To best allow easier parameterization of particle behavior, a set of `Curve`s
are needed. More commonly used in animation, a simple curve allows sampling a
smoothed value based on an input time. This allows developers and designers to
define potentially arbitrary easings for every attribute of particles. For
example, the `SizeOverLifetime` modifier will normalize the lifetime of every
particle via `(start_lifetime - remaining_lifetime) / start_lifetime`, and
evaluate a curve to get the size of the particle at a given frame.

To allow for randomized parameterization, `Curve` should generally be replacable
with a `MinMaxCurve` which defines two separated curves as the minimum and
maximum values at a given point in time. This can be used to generate a range
from which a RNG can sample from or can be used as the start and end for linear
interpolation for values that require smooth flows from one to another.

If `Curve` is generic, it can be used to represent a gradient for changing colors
as well.

There already is a open PR for adding these types to Bevy:
https://github.com/bevyengine/bevy/pulls?q=1837.

## Drawbacks

## Rationale and alternatives
The main alternative comes in the form of GPU-based particle simulations, where
both the simualtion and rendering occur on the GPU instead. Compared to the
proposed CPU based design the main benefit is that much higher particle counts
can be used. Whereas the design proposed here may support several thousand or
several tens of thousands particles, GPU particle systems which use compute
shaders for simulation can support millions of particles per system. This enables
many more artistic workflows that are not available with just CPU particles.

The main drawbacks of a GPU particle system:

 - Compute shader support is needed, both in Bevy and the target platform. Most
   notably, WebGL does not have access to compute shaders, and WebGPU, it's
   eventual replacement, is not landing anytime soon.
 - Particles are difficult/impossible to access from the CPU, depending on the
   implementation.

When pipelined rendering lands, it will likely include compute shader passes,
upon which a GPU particle system can be built. It may be possible to reuse
multiple components from this design to drive compute shaders instead of running
them on the CPU. In these cases, an alternative storage for particle data will
likely be needed, as well as systems for triggering particle simulation, but all
of the emission and simulation modifiers can be reused.

However even when GPU particles are available, it makes sense to keep a CPU
implementation as a fallback for platforms that don't support compute shaders, as
well as provide a choice to the end-developer as for which fits their use case.

Another alternative is to not support particles systems as a first party engine
feature. This could very well be just another ecosystem crate; however, like
`bevy_pbr`, it's often better to have well established common convention
supported at a first-party level. Supporting it as a first party crate should
ensure that both end-developers, and third-party plugins target the same common
interface for working with these systems, as well as establish a common authoring
flow for artists when a fully-featured editor is ready.

### Particle Representation
Particles could be represented easily as Bevy entities and components. This could
be helpful as access to particle data would be as easy as normal components, and
provides more trivial multithreading via `par_iter(_mut)` and
`par_for_each(_mut)`. However, there are some notable pain points in this
approach:

  - Bevy does not curently support GPU-instancing in the general case for
    entities. The SoA construction make this notably easier to write
    single draw-call renderers for each particle system.
  - Destruction requires use of Commands, and must be done at a sync point. This
    may result in more work being done than necessary in frames where large
    counts of particles die. This may also be heavier than running `swap_remove`
    on a SoA.
  - Each component in a entity incurs the CPU and memory overhead of a change
    detection tick. With these systems, it's comon for particles to constantly be
    spawned, destoryed, and altered. Querying for a `Changed<ParticlePosition>`
    will likely return the entire set of particles.

### Particle Death
Other particle system implementations often keep a "alive list" of particles that
are alive vs dead instead of using something like `swap_remove` to
destory/recycle particles. This makes a particle destroy a simple bitflip instead
of a complete particle copy, but adds a O(n) search time when spawning new
particles, and a potential O(n) compaction step before rendering or a large number of
discards in the fragment shader. Note: in this case n is the maximum number of
supported particles, not the size of the alive set.

This can most easily be implemented with a `Vec<bool>`, but can be further
optimized by using a bitmask where each 1 maps to a live particle. This can be
further optimized using SIMD, where scanning 256 particles can be done with one
or two instructions (like SwissTable). Passing the "alive list" in this format as
a uniform to a shader will likely limit it to only 8192 per draw call.

## Prior Art
At time of writing, there are no public implementations of particle systems
within the bevy ecosystem that are meant for generic use.

Particle systems have some kind of implementation in almost every notable game
engine:

 * [Unity Particle Systems](https://docs.unity3d.com/Manual/ParticleSystems.html)
 * [Unity Visual Effect Graph](https://docs.unity3d.com/Packages/com.unity.visualeffectgraph@8.2/manual/index.html)
 * [Unreal Engine 4](https://docs.unrealengine.com/4.26/en-US/Resources/ContentExamples/EffectsGallery/1_A/)
 * [Godot](https://docs.godotengine.org/en/stable/classes/class_particles.html)
   ([2D](https://docs.godotengine.org/en/stable/tutorials/2d/particle_systems_2d.html))
 * [Source](https://developer.valvesoftware.com/wiki/Particle_System_Overview)

This RFC uses a CPU-particle simulation, and is heavily inspired by the public
interface from Unity's Shuriken particle system. Both Unity and Unreal offer
separate CPU and GPU particle implementations, the differences are discussed
above.

In the literature, one of the original papers on the topic is from Lucasfilm in
1983: http://graphics.cs.cmu.edu/courses/15-869-F07/Papers/Reeves-1983-PSA.pdf,
which this implementation replicates the main parts of it's particle definition.

## Unresolved questions

 - Benchmark the difference between SoA component vs Bevy entities.
 - The `swap_remove` approach is likely to break transparent rendering without
   some form of Z-sorting. Can we do this on the GPU?

## Future possibilities
Further extensions like additional emission and simulation modifiers can be added
for more complex particle behavior or visuals. This can either be added as first
party implementations or as a third-party ecosystem crate.

In particular, several modifiers that Unity's Shuriken particle system has that
is not supported by the RFC include particle trails (adding triangle strips that
follow particles during it's lifetime), collision with physical objects (perhaps
with Rapier), and particle force fields that apply a constant force to particles
within a bounded region of world space.

One optimization might be to provide slimmed down meshes for particles instead of
always using quads. This should drastically reduce the fill rate as there will be
a large number of overlapping transparent geometry in any parctical particle
system.

The easily instanced data model could be used for other high-repitiion,
low-impact rendering tasks like grass on terrain.

Following that same line of thought, providing the utility to specify what mesh
each particle uses might be useful in some use cases.

# Feature Name: non-unique-resource

## Summary

Introduce new builtin to Bevy ECS: non-unique resource.
Standard `Resource<T>` is unique per `<T>`, this new resource is not.
`NonUniqueResourceId<T>` is a low-level primitive, not usable as `SystemParam`.
But higher level tools can be built on top of it, in particular
by exposing resource as read/write systems which can be piped with other systems.

## Motivation

`Resource<T>` works as equivalent of rwlock in other concurrent systems,
where `Res<T>` is a read lock and `ResMut<T>` is a write lock.

Bevy limits one instance of such rwlock per `<T>` which makes impossible
to use it as generic rwlock to build complex systems on top of it.

So, why multiple instance of `Resource<T>` would be useful?
There are at least several reasons:

### Piping with dependencies

The issue is described in [Bevy issue #8857](https://github.com/bevyengine/bevy/issues/8857).

Current `.pipe()` implementation combines two systems into one, reducing concurrency.

We want to be able to do something like `system1.new_pipe(system2)` which can implemented as:
* allocate new `NonUniqueResourceId<T>` where `<T>` is the type of the result of `system1`
* `system1` is modified to write the result to the resource
* `system2` is modified to read the result from the resource
* `system2` is scheduled to run after `system1`

To make it work, we need to be able to allocate multiple instances of `Resource<T>`
because pairs of systems may have the same intermediate type `<T>`.

More complex piping scenarios should be possible, for example:

```rust
fn system_with_two_outputs() -> (u32, String) { /* ... */ }

let (pipe_32, pipe_str) = system_with_two_outputs.split_tuple2();

pipe_32.new_pipe(system1); // pipe to a system which takes u32 as input
pipe_str.new_pipe(system2); // pipe to a system which takes String as input
```

Possibilities of building such systems are endless.

### System autolinking

There's an idea of API like this:

```rust
fn system1() -> String { /* ... */ }
fn system2(input: In<String>) { /* ... */ }

// This should schedule `system2` after `system1` and pipe the result,
// like `system1.new_pipe(system2)` above does, except automatically.
app.add_system_auto_link(Update, system1);
app.add_system_auto_link(Update, system2);
```

We can _mostly_ implement it with resources, however, we should use different
resources for different schedules, so `NonUniqueResource<T>` would be useful here.

### New conditions/dynamic conditions

#### Possible to reimplement/simplify current conditions

Conditions can be rewritten as regular systems. Consider this code `system1.run_if(cond1)`.

We can reimplement it as:
* allocate new `NonUniqueResourceId<bool>`
* `cond1` writes the result to the resource
* `system1` reads the result from the resource, and if true, runs, otherwise skips
* `system1` is scheduled to run after `cond1`

This way we can remove support for conditions from scheduler greatly simplifying it.

#### Conditional conditions

But in addition to that, there are scenarios which are not possible to implement currently.
Consider this pseudocode:

```rust
fn run_if_opt(system: impl System, cond1: Option<Condition>) -> impl System {
    if let Some(cond1) = cond1 {
        system.run_if(cond1)
    } else {
        system
    }
}
```

This cannot not work, because `run_if` returns `SystemConfigs`, not `System`
(this is a strict requirement given how scheduler is implemented).
And user might want to get `run_if` because a user may want to continue working with system,
for example, pass the resulting system to this function again.

### Dynamically typed systems

Systems can built dynamically, or even bindings to dynamically typing languages can be used.
For example, for Python custom systems can be implemented like this:

```python
res = Resource()

@system(ResMut(res))
def system1(res):
    res.insert(10)

@system(Res(res))
def system2(res):
    print(res.get())
```

If it is dynamically typed, so all resources need to have the same type like `PyObject`.

### Reimplement resources

FWIW, regular resources can be reimplemented on top of non-unique resources.

## User-facing explanation

This section describes user-facing API.
Again, this is probably won't be used directly by most users.

### Model

Perhaps the best way to think about `NonUniqueResourceId<T>` as `Arc<RwLock<T>>`.

Note `RwLock` does not necessarily mean that the thread should be paused
if the resource is locked. For example, `tokio` version of `RwLock`
does not block the thread, but instead put away the task until the lock is released.
This is what Bevy schedule would do, except it would block before the system start
rather than during access to the resource (same way as it does for `Resource<T>`).

### API

Let's start defining the reference to the resource.
This is the reference (the identifier), the resource itself is stored in the world.
There's no lifetime parameter.

```rust
/// Reference to the resources.
/// As mentioned above, it cannot be used as `SystemParam` directly,
/// but can be used to build higher level tools.
struct NonUniqueResourceId<T> { /* ... */ }
```

How to create the resource:

* `NonUniqueResourceId<T>` can be either requested from `World`, like `world.new_non_unique_resource::<T>()`
* or maybe just lazily allocated on first access to the world. Like this:

```rust
impl<T> NonUniqueResourceId<T> {
    /// Generate unique resource id.
    /// The world will allocate the storage for the resource on first modification.
    fn new() -> Self { /* ... */ }
}
```

Raw API to read and write resource.

```rust
impl World {
    fn get_non_unique_resource<T>(&self) -> Option<&T> { /* ... */ }
    
    fn get_non_unique_resource_mut<T>(&mut self) -> Option<&mut T> { /* ... */ }
}
```

(Similarly, there might be more accessors here or in `UnsafeWorldCell` or in `App`
which are omitted here for brevity.)

For most advanced users, however, API provides "systems" which read or write resources:

```rust
impl NonUniqueResourceId<T> {
    /// Return a system which takes `T` as input and writes it to the resource.
    /// This system requests exclusive access to the resource.
    fn write_system(&self) -> impl System<In=T, Out=()> { /* ... */ }
    
    /// Return a system which takes `T` from the resource,
    /// panicking if the resource is not present.
    /// This system also requests exclusive access to the resource.
    fn read_system(&self) -> impl System<(), Out=()> { /* ... */ }
    
    /// Return a system which copied `T` from the resource,
    /// panicking if the resource is not present.
    /// This system only requests shared access to the resource.
    fn read_system_clone(&self) -> impl System<(), Out=T>
    where
        T: Clone,
    { /* ... */ }
    
    /// Like `read_system`, but returns `None` instead of panicking
    /// if the resource is not present.
    fn read_system_opt(&self) -> impl System<(), Out=Option<T>> { /* ... */ }
    
    // ... and several more similar operations.
}
```

System implementations provided by `NonUniqueResourceId<T>` do the heavy lifting
of configuring the concurrency of the resources.

### Example

Complete very simple system piping example:

```rust
fn new_pipe<T>(system1: impl System<In=(), Out=T>, system2: impl System<In=T, out=T>) -> SystemConfigs {
    let res = NonUniqueResourceId::new();
    let barrier = AnonymousSet::new();
    
    let system1 = system1.pipe(res.write_system());
    let system2 = res.read_system().pipe(system2);
    IntoSystemConfigs::into_configs((
        system1.before(barrier),
        system2.after(barrier),
    ))
}
```

## Implementation strategy

Simplified, the `World` gets a new field:

```rust
struct World {
    non_unique_resources: HashMap<NonUniqueResourceId<ErasedT>, ErasedT>,
    // ... plus some data to track changes.
}
```

PR prototyping this feature: [#10793](https://github.com/bevyengine/bevy/pull/10793).
The prototype implementation is incorrect, but it shows the general idea.

## Drawbacks

The is added feature, not modification of existing features.

So the main drawbacks come from extra complexity:
* more bugs
* maintenance burden
* harder to learn API

Another potential drawback is giving users too much power and flexibility
to design their systems. This can be viewed as a benefit, but also can be viewed
as a drawback, because for example, API of Bevy mods might be harder to understand.

## Rationale and alternatives

The reasons to have this feature are described above in the motivation section.

Alternative, considering we need more than one instance of given `Resource<T>`
there are strategies to achieve that with added complexity in given code.
For example, to `new_pipe` function described above,
we can pass extra `Id` type parameter to generate unique resource type
like `(T, [Id; 0])`. This may be not very ergonomic,
it increased code size and reduces compilation speed, but it is possible.

## Prior art

I'm not aware of these.

## Unresolved questions

- Naming. `NonUniqueResourceId<T>` is mouthful. `RwLock<T>`?
- `NonUniqueResourceId::new` or `World::new_non_unique_resource`?

## Future possibilities

Rewrite parts of Bevy on top of this new feature:
- Rewrite conditions as regular systems using `NonUniqueResourceId<bool>`
- Rewrite `Res<T>` and `ResMut<T>` systems using `NonUniqueResourceId<T>`
- Provide better piping API ([#8857](https://github.com/bevyengine/bevy/issues/8857))
- Implement system "autolinking" (automatically connect systems which have matching input/output types)

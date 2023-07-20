# Feature Name: `Subworlds`

## Summary

Currently, `bevy` contains a single _global_ `World` which holds all entities/components, which are operated on via `Queries` and `Systems`. However, there are uses for disjoint sets of entities that can operate independently from one another. This RFC proposes a concept of `Subworlds`, which are individually a distinct collection of entities and their components. In this model, `Queries` and `Systems` can operate on these `Subworlds` independently and concurrently. This provides the ability to encapsulate sets of entities into `Subworlds`, reducing some of the complexity and overhead that came with grouping entities in the _global_ `World` (via _marker-components_ for example). By excluding large sets of entities that live in other `Subworlds`, `Systems` and `Queries` can more efficiently perform their logic on a smaller, more concise set of components.  

## Motivation 🏃

The primary motivation for using `Subworlds` to allows systems and queries to operate on disjoint sets of entities and their components.

## Guide-level explanation 📝

### Terms & Definitions 📚

- `Subworld`: This refers to the original definition of a `World` in bevy 0.5. (essentially a collection of components which belong to a set of entities). 
  - _Note_: I may use `World` and `Subworld` interchangeably.
- `Worlds`: This can be thought of as a collection of `Subworlds`. 

### Example ⭐
- Your game is composed of levels. 
- The player is currently on level 1 and nearing the end, so you would like to begin loading level 2.
- Level 1 and 2 are completely disjoint and do not share any data (except perhaps the player themselves).
- The difficulty resides in the fact that entities from levels 1 and 2 contain similar components, but the entities themselves are disjoint.
  - In order to begin populating level 2's entities, something must exist to distinguish the two sets of entities, so that systems operating for level 1 (e.g. _physics_) do not operate on level 2's entities, even though they may contain physics components.
- Only having a single `World` requires that systems filter the entities based on what level they belong to, adding overhead and complexity to either the `Query`, the system, the components describing the entities, or a combination.
- `Subworlds` would provide a clean and efficient solution to this problem. 
  - The primary systems `[physics, render, etc...]` would only operate on Level 1's `Subworld`.
  - A `load_next_level` system could be running concurrently on Level 2's `Subworld`. 
  - Then when Level 2 begins, the primary systems operate the on later `Subworld` while the former one gets saved/dropped.

### Things `Subworlds` would allow 🙂
- Running systems on subworld `A` on thread `A`, while running the same systems on subworld `B` on thread `B`.
- Running systems on specific disjoint groups of entities based on some criteria.
- Running systems on subworld `A` every frame and on subworld `B` every 10 seconds.
- Helps systems stay clean and not require overuse of _marker-components_ (and possibly _relations_) which can lead to excessive archetypal fracturing which will end up hindering performance. 
- Give developers the choice to use either a _single_, _global_, world that contains every entity (_note_: for some games, that might makes the most sense and be the most performing), or multiple `Subworlds` if they so desire.

## Reference-level explanation 🧑‍🏫

TODO: expand upon this

```rust
struct Worlds {
    subworlds: HashMap<WorldId, World>,
}

struct World { // aka `Subworld`
    entities: Entities,
    components: Components,
    // more stuff
}
```

### Example API (This is not final!) 💻
```rust
fn setup(mut commands: Commands) {
    commands
        .spawn() // this will spawn the entity into the default world
        .create_world("World2")
        .create_world("World3")
        .set_world("World2") // sets the current world being operated on
        .spawn(); // will spawn this entity into "World2"
}

fn on_level_change(mut commands: Commands) {
    commands
        .remove_world("World2") // removes the world and all of it's components
        .add_world("AnotherWorld")
        .with_system(some_system.system())
        .unwrap();
}

fn main() {
    App::build()
        // This system will operate on *all* worlds by default
        .add_system(general_system.system()) 
        // add this system which only operates on world 2
        .add_world_system(["World2"], world2_system.system())
        // add this system which runs on the Default world and World2
        .add_world_system(["Default", "World2"], two_world_system.system())
        // provide a RunCriteria
        .add_world_set(
            WorldSet::new()
                .with_run_criteria(some_criteria.system())
                .with_system(another_system.system())
                .with_worlds(["World2"])
        )
        .run();
}
```

- By default, there is a single `World` contained within the `Worlds` (this mirrors the previous behavior of `bevy`).
- Queries and systems still only operate entities and components from a single `World`.
  - However, systems can be applied (concurrently) to multiple `Worlds` at a time.

## Drawbacks 🙁

In theory adding a dynamic number of `Subworlds` would add overhead since they must be kept in (something like) a `HashMap<Id, Subworld>`, where as having a single _global_ `World` does not incur this overhead.  

## Rationale and alternatives 📜

- Using _marker-components_ to distinguish entities that reside in disjoint sets. 
  - This works, however, if `EntitiesA` {E1, E2, E3} are completely disjoint from `EntitiesB` {E4, E5, E6}, you shouldn't have to keep them grouped into a single set (`World`).
  - Also, when constructing queries, this adds the overhead of need to check all entities instead of the entities of a specific `Subworld` that you are interested in. 

- _Relations_ could also be a potential solution to this problem. 
  - However the same problems described above apply. 
  - The overhead of needing to check entity relationships exceeds that of running a query on a specific `Subworld`.  
  - Overuse of _Relations_ and _marker-components_ could also lead to extreme archetypal fractaling, and worse case a single archetype-per-entities scenario. 

## Prior art 🎨

- See [here](https://discord.com/channels/691052431525675048/749335865876021248/834310334264115210) for the initial discord discussion that sparked this RFC.

- The popular C++ ECS library [EnTT](https://github.com/skypjack/entt) supports this behavior, and even encourages it, via giving you direct access their `World` structure (`EnTT` calls these _registries_).
```cpp
auto level1 = entt::registry{}; // registry is analogous to bevy's World.
auto level2 = entt::registry{};

// depending on the current level, we can run the same system on a different sub-<worlds/registries>.
some_system(level1);
some_system(level2);
```
  - Currently supporting this is `bevy` is not possible, as systems and the `World` they operate on are closely coupled.
  - `EnTT` is able to support this since systems are nothing more than free functions.

## Unresolved questions ❓

- How do resources behave in respect to each `Subworld`? Are they shared between them or made exclusive to each. (or both?? E.g. `Res<T>`/`ResMut<T>` vs `Local<T>`).
- How do we support moving/copying entities and their component from one `Subworld` to another?
- Should/Would it be possible for `Subworlds` to communicate? Or more specifically, the entities inside them to communicate?
- What would the API look like for creating/modifying/removing `Subworld`s and how would you prescribe systems to run on specific sets of `Subworld`. E.g. a `SubworldRunCriteria`?
- We should aim to **not** actually change the name `World` to `Subworld`. And instead introduce some new type `Worlds` (which is a collection of the sub-worlds). 
- How does this interact with `Stages` and `RunCriteria`?

## Future possibilities 🚀

- Entity move
- World-specific resources

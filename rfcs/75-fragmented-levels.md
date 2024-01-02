# Feature Name: `fragmented-levels`

## Summary

Enable better collaboration in the future bevy editor among artists, designers, and large teams by creating an API for managing multiple scenes in a level, and a file format for loading levels composed of multiple scenes.



## Motivation

After working in Godot, Unreal, and Unity; and working in Unreal Engine 5 with larger teams as technical support and repository manager, it has been a constant pain point where levels can only be edited by one person at a time. If you are using something like Git instead of Perforce, then it's even more annoying because you can have merge conflicts which aren't resolvable and you're forced to destroy someones work.

I (@codercommand) want to invest in Bevy instead of trying to add level fragmenting to Godot or Unreal, because of this I came up with a solution to fix the problem to multiple people working in a single level - that the future bevy editor would use, and the API has additional use-cases.

The level file format would be a list of pointers to scenes that makeup the level. The level API would provide functionality for: loading, unloading, and changing levels; moving entities between scenes; adding and removing scenes from levels (not creating or deleting scenes); changing scene visibility or transparency; controling which scenes are loaded or not loaded.

This API would mostly be used by the future bevy editor to add support for layers (layers refer to scenes) in levels, the same way in which you can have layers in Photoshop or paint.net.

Levels composed of scenes would enable stuff like:
1. Game designer works in scene1 making a blockout, continually pushing their work. Mutiple artists work in other layers/scenes and set the transparency of the blockout to something like 0.3 so it appears as a hologram they can work ontop of. Now everyone can work at the same time, and the layers can be merged at a later date.
2. Someone is working in a level with a single layer/scene, and someone else needs to add new stuff but not change existing work. They could create a new layer for the level and add their work in, then at a later point have it merged into 1 scene without breaking anything.
3. Fragmenting a 1:1 level/scene into a level with multiple scenes 1:many so people can work on certain parts without requiring access to the entire level.

Outside the future bevy editor levels api has applications in:
1. Chunk loading based on distance.
2. Loading and unloading parts of a level based on segment player is in.

Normally people have to write chunk loaders themselves or write systems for loading parts of a level because those systems don't exist in Bevy. This would provide a simple premade solution that handles loading and unloading of those scenes in a level. All someone would need to do is write some systems to call those functions to handle when they want those scenes loaded.



## User-facing explanation

### Level Concepts
- **Merging** - Combining multiple scenes into one scene.
- **Fragmenting** - Spliting a scene into multiple scenes.
- **Composing** - One or more scenes that belong to a single level file asset.

### What Are Levels
Levels are files with a list of file directories that point to scenes. Scenes in levels can also be referend to as layers, but layers and scenes are the same thing. The scenes format is still the same.

This feature does not replace scenes or ECS world, instead it provides an API management layer via a Bevy resource to provide fragmenting and merging of scenes, or composing of levels.

### Purpose of Levels
Levels are an API and abstraction for managing multiple scenes in an editor. This API is mostly intended for the Bevy editor, tools/scripts, and their party editors.

Like stated before it would greatly enhance the ability for large teams or multiple people to work in the same game level without merge conflicts or file locks.

### When Would I use this in Code?
Asside from development tools, scripts, and editors it has one additional application. That's controling which parts of a level are loaded based on player progression, location, and other metrics. For large open worlds, or detailed environments with lots of things this can be very important feature.

#### Example of Level API
```rust
// Example of loading and unloading parts of a large open world level.
fn load_region(
    events: EventReader<Region>,
    mut level_manager: ResMut<MyLevel>,
) {
    for region in events.iter() {
        match region {
            Region::EnterCave => level_manager.load_scene("mysterious cave"),
            Region::LeaveCave => level_manager.unload_scene("mysterious cave"),
            Region::EnterMansion => level_manager.unload_all_scenes().load_scene("mansion interior"),
            Region::LeaveMansion => level_manager.unload_all_scenes().load_scene("large forest north"),
        }
    }
}

// Example of changing zones/levels in an MMO like Albion Online or Lost Ark.
fn change_zone(
    events: EventReader<Zone>,
    mut level_manager: ResMut<MyLevel>,
) {
    for zone in events.iter() {
        match zone {
            // .change_level would unload all the scenes belonging to the current level,
            // then load in all or only specificed scenes from the new level.

            // Loads specific scenes from level
            Zone::LazzyForest => level_manager.change_level("levels/lazzy_forest.level",
                "north-outer-road",
                "north-outer-town",
                "north-outer-forest"
            ),
            // Loads all scenes from level
            Zone::LakeCastleMount => level_manager.change_level("levels/lake_castle_mount.level");
        }
    }
}
```

### Does This Make Bevy Scene's Complicated?
No, scenes are separate to levels and don't know what levels are. Levels are an opt in feature of Bevy. If you are making an application that doesn't need levels, don't use them. If you want levels, but also want to load some scenes manually without using levels that's also fine.

If a scene is loaded through the level bevy resource, then it's managed by the level resource. If you load a scene without using the level resource, then it's external and the levels API won't control it.



## Implementation strategy

`Fragmented-Levels` would be implemented in a crate called `bevy_level`. (I reserved the name for this encase the RFC is accepted and it's needed [https://crates.io/crates/bevy_level](https://crates.io/crates/bevy_level), if cart/bevy wants it let me know and I can transfer the name.)

Levels are loaded via the bevy asset loader and stored inside a resource for easy management.
```rust
// LevelAPI adds methods for managing the scene so you don't have to do: handle.0.load_scene(...);
// SceneComposition is the resource type made from the `.cos`/`.sc` file.
#[derive(Resource, LevelAPI)]
struct GameplayLevel(Handle<SceneComposition>);

// Startup system
fn setup_gameplaylevel(
    mut commands: Commands,
    server: Res<AssetServer>
) {
    let handle: Handle<SceneComposition> = server.load("level/maze.cos");
    commands.insert_resource(GameplayLevel(handle));
}
```

The level format and API can be implemented independantly of scenes and ECS world. ECS world and scenes don't understand or have any dependancies on level. Because of this levels are an opt in feature that are not be part of the default plugin.

Because of this ECS world and scenes can be worked on indipendantly from levels without breaking anything.

### Level File Format
todo



## Drawbacks

- It's an extra crate to maintain.


## Rationale and alternatives

To my knowledge there is no existing implementation of layers in other existing game engines or Bevy. Because of this implementing this new feature correctly would be an innovation on how level collaboration currently works in the games industry.

Because Bevy is ECS it's in a very good position to support this new concept unlike other engines since implementation would be very simple and not brake any existing systems or require migration.

If Bevy doesn't do this, then what it gives up is not attempting to push a new innovative way to create levels, which it's very capable of doing. If this feature exists when the bevy editor makes its first stable release, then it would encourage companies with large game projects to undertake them in bevy because of how much easier it makes the creation and iteration process.

Getting larger companies or dev teams working in bevy would increase funding and development support of Bevy.

This feature could be an external ecosystem crate, but it's very simple, has an API many games would use for loading levels or parts of levels, and would be used by the bevy editor. It will also have dependencies to bevy so it wouldn't have any as use an external crate for the wider Rust ecosystem.

You could make it a third party plugin of bevy editor instead of tightly intergrating it, or make another editor that supports the crate. This would be an problem because Bevy would be giving up a strong selling point of using its own editor. Don't want third party editors to all have better features than the bevy editor, otherwise we'll have a fragmented ecosystem which would be undesirable.



## Prior Art

### Unity
Unity supports additive scenes and has APIs for managing those scenes and loading more than one scene at once https://github.com/bevyengine/rfcs/pull/75#issuecomment-1873990935.

`Fragmented-Levels` is very similar to Unity additive scenes. This was accidental, but proves that this RFC is a desired feature in a game engine, and we should look into how to best support this feature in bevy.



## Unresolved questions

1. At present I haven't made a prototype, and a prototype would need to be made to verify the claims the RFC is making.
2. When this RFC is complete it should specify how the level API is exposed (resources, entities, other, etc...), 
3. A first draft of the features/functions the API should have must be specified - not final and during implementation can be changed, first draft is mostly a guide on what's expected of the API to be capable of.
4. A file format for level files woud need to be decided.
5. Would need to decided if levels API can do stuff like create and delete scenes, or is that something that shouldn't be part of the API. - Current belief is it shouldn't, and if additional features are missing should be implemented on those types directly.

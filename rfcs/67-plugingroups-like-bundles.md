# Feature Name: Plugingroups like Bundles

## Summary

Make PluginGroups like Bundles, so they are just structs. Then the ordering will be made by a gen_graph function, maybe with the new planned `bevy_graph` (https://github.com/bevyengine/bevy/discussions/6719) ?
This will make PluginGroups from the design consistent to the other bevy types.

## Motivation

First, a new possibilty: adding plugingroups inside of pluginsgroup (cool). (a guy on discord asked this https://discord.com/channels/691052431525675048/1044672077451513906) and what he wanted (adding plugingroups inside of plugingroups) just wans't possible :c.
Every time when I see PluginGroups it makes my brain think about **Design consitency**, because most of the other bevy types are just structs with default rust constructors.
It also bothers me when i set plugins that for example then implement `WindowPlugin::default()` only to let me use `.set(WindowPlugin { .. })` later.

## Example Design Impl

```rust
#[derive(Default)]
pub struct MyPlugins {
  main_plugin: MainPlugin,
  #[plugingroup]
  other_plugin_group: OtherPlugins,
  #[cfg(feature = "optional")]
  optional: OptionalPlugin,
}

impl PluginGroup for MyPlugins {
  fn gen_graph(mut self) -> PluginGraph<Self> {
    let mut graph_builder = PluginGraph::new();
    builder.append::<MainPlugin>();
    builder.before::<OtherPlugins, MainPlugin>();
    #[cfg(feature = "optional")]
    {
      builder.append::<OptionalPlugin>();
    }
    graph_builder.build(self)
  }
}

app.add_plugins(MyPlugins {
  main_plugin: MainPlugin { .. },
  ..Default::default()
}).plugin_graph().after::<OtherPlugins, MainPlugin>(); // make plugin graph editable in app, make it also change order later
```

## Final Note

I'm not sure about the naming, about the RFC in general. But thats a RFC about (i think). But with your help i think this can be a well done feature :).

And a thing i just saw: this will remove a functionality (which i never saw used and i am even not sure if it is even possible):
```rust
app.add_plugins(MyPlugins::custom_setting(CustomSetting::Minimum)); // this would for example just add Plugin x and y, with CustomSetting::Everything, it would also add Plugin z.
// something like optional plugins via constructor isn't possible with the new design, i think.
```

So yeah, I'm looking forward to make this feature good and cool so everybody likes it (hopefully) and it can be adopted to also make it easier for the above mentioned `bevy_graph` and bevy in general.
Maybe this could be also an addition to the new stageless design? idk, we'll see :)

# Feature Name: `editor-foundation`

## Summary

This RFC outlines the foundation for Bevy's editor. It describes its basic
components and how they will interact with Bevy's engine. Following this RFC
will result in a minimum viable product for the Bevy Editor.

This RFC does NOT cover the particular user interface (UI) design for the
editor. Instead, it provides the building blocks for a proper UI design, with a
minimal usable interface to place these components in context.

## Motivation

Unlike most major game engines, Bevy lacks an interactive editor. This has
three major downsides:

1. it increases the barrier to entry for potential new developers;
2. this forces scene designers to learn to create scenes with code; and
3. developers must rebuild their project before seeing any changes they make.

While teams can mitigate the last two points by creating project specific
tools, this does require them to divert resources building those tools that they
could have spent building the game.

Bevy's plugin-heavy design poses several challenges to making a traditional
editor. Primarily, unlike other game engines, Bevy does limit developers access
to the core game loop, similar instead to many game frameworks. While an
interactive editor could be built on top of Bevy that takes away this freedom;
doing so violates the spirit of Bevy's plugin-heavy design, thus robbing it of
Bevy's expressive nature.

**Bevy's editor must acknowledge the duel nature of Bevy as both a game engine
and a game framework.**

A secondary, smaller challenge, is that Bevy relies on the Cargo build system
for modularity. An interactive-editor that is core to Bevy must both respect
this design, and work with the limitations and abilities of Cargo.

## User-facing explanation

The easiest way to get started with Bevy's editor by downloading it with Cargo:

```
$ cargo install bevy
```

(Note the use of `install`, not `add`!)

This will add `bevy` to our path:

```
$ bevy
Bevy's Command Line Tool

Usage: bevy [OPTIONS] [COMMAND]

...
```

This tool allows us to create new projects, manage assets, edit projects, and
even publish games. Note that `bevy` is not meant to replace `cargo`, but to
work alongside it.

Also, just like Cargo, the Bevy version used in a project does not need to match
the version of `bevy` you have installed. You can upgrade and downgrade them
independently. Therefore, we recommend you always use the latest version of the
`bevy` tool.

Bevy's editor can be used graphically, or as a command line tool. This document
covers the command line usage, but running `bevy edit` in a project will open
the graphical view.

**The `bevy` command line interface is not stable. Do not use it in build
scripts!**

### Default Project Structure

The first entry point for the `bevy` command is `new` and `init`, for creating
new projects, meant to mimic the commands in cargo:

```
bevy new [options] <project-name>
```

As an example, let's create a new project, `flappers`:

```
$ bevy new flappers
```

This creates a new `flappers` folder in your current directory, using the
default template, similar to using `cargo new`. Unlike Cargo, however, a Bevy
project can start from any number of templates you can get from the Bevy asset
repository.

Next, let's take a look at the project structure:

```
flappers/
    README.md
    Cargo.toml
    LICENSE-Apache2.txt
    assets/
        ...
    src/
        main.rs
    tests/
        ...
    target/
        ...
    editor/
        Cargo.toml
        src/
            main.rs
    .git/
        ...
    .github/
        workflows/
            ...
```

The default template provides a lot more than a standard new cargo project. Much
of this is not strictly required, but will be wanted for the average project.
Because this is mostly convention, feel free to change the layout as needed. The
first three files: `RREADME.md`, `Cargo.toml`, and `LICENSE-Apache2.txt` are
standard meta-information that are in most crates. The next folder: `assets` is
the default location to place project non-code assets. The bulk of the source
code goes in the `src` directory, likewise tests go in `tests`. Project builds
go into the `target` directory.

The `editor` directory provides a sub-crate for the Bevy editor. Associating
Bevy's editor with the project allows you to have multiple projects using
different versions of Bevy in the same environment. The `bevy` command line tool
delegates to this editor for any project specific actions. It also has the side
benefit of allowing you to more heavily tune the editor for your specific game
while still having access to the full suite of tools Bevy provides.

The final directories are for version control management and game publishing.

### Bevy Assets

The main repository for Bevy assets can be found on [the bevy webpage][asset-page]

Assets can be added to your project through the `bevy` tool:

```
bevy assets add <asset-name>
```

Likewise, you can remove the asset with:

```
bevy assets remove <asset-name>
```

In general, assets come in three varieties: *code*, *data*, and *template*. Code
assets are little more than Cargo crates, many can even be installed with the
`cargo` command.

Data assets are overlaid onto the existing project, and usually go into the
`assets` directory. Frequently these assets are large, should not be added to
the repository directly, or should be loaded at build time from an external
source. Therefore, in addition to copying the assets into the project, data
assets can also be references to an asset at an external location. **Note that
while referenced asses can be removed from the project, copied data assets can
not!**

Finally, template assets are not added to a project, but are instead used by the
`bevy` tool to start new projects. The `-t` flag is used to pick a project
template:

```
bevy new -t <template> <project>
```

For example, we can instead create our Flapper project using the `foxtrot`
template:

```
$ bevy new -t foxtrot flapper
```

Sometimes it makes sense to install an asset onto a system rather then adding it
to a project. The most common case for this is template assets, which are used
to create the initial project. To install an asset to the system you can use the `install` command:

```
bevy install <asset-name>
```

This will install the asset to your user's data directory. If you want to
install an asset to a different location, you can set the `BEVY_ASSETS_DIR`
environment variable.

### Asset Repositories

In addition to the main repository for assets, projects can use additional
repositories for storing and retrieving assets. You can add a new repository with:

```
bevy repo add [repository-name] <repository-url>
```

This adds a repository to the current project. Optionally, a repository name can
be provided, which is used by `bevy assets`:

```
bevy assets add repository-name/asset-name
```

Despite their similarities asset repos are different from git remotes in one
crucial way: they are generally checked into the repository. In this sense, they
are more akin to git submodules or git large file storage than pure remotes.

Sometimes it makes sense to have a remote stored system wide, rather than with
any particular project. The `-g` flag allows you to add repositories globally.

```
bevy repo add -g [repository-name] <repository-url>
```

Alternatively, you can set the `BEVY_ASSET_REPO` environment variable to one or
more asset repositories.

### Publishing Tools

Bevy also helps you publish your games, either by building distributables, or
even having CI distribute builds each time its pushed.

Use `bevy build` to build your game:

```
bevy build <target>
```

This will place your game, bundled with all needed assets, into your projects
`target` folder.

Alternatively, Bevy can publish a new build every time you push to a repository.
Right now this is only possible using GitHub and Itch.io, sign in with `bevy
login`:

```
$ bevy login github
Enter GitHub access token: ***
$ bevy login itch
Enter Itch.io access token: ***
```

**Note that these commands store the tokens in you system vault, not within the
repository.**

### Scene and Entity Manipulator

Bevy comes with an entity and scene manipulator. The easiest way to use these
tools is with the graphical editor, run `bevy editor` in the project directory.

The entity manipulator lists all of the entities in a given world, as well as
the components that are associated with those entities. You can add new entities
from this view, and attach new components to them. You can also edit the
entities that were added using the manipulator.

You can view all entities in your world, not just ones created with the
manipulator. However, you are only guaranteed to be able to edit static
entities, and entities created with the manipulator.

In addition to the entity manipulator, Bevy also provides developers with a
scene manipulator. This allows developers to place entities in a larger level,
and get a real-time view of how they interact with each other.

### Systems Manager

In addition to the manipulator, Bevy provides developers with a view of all the
systems their games uses. In addition to creating new systems, developers can
use this manager to find and open the code for existing systems. The systems
manager can also be viewed in the graphical editor, run `bevy editor` in the
project directory.

### Localization, Accessibility, Porting, and Plugins

Just like the rest of the Bevy engine, the editor can be extended using plugins.
Some plugins are installed to the system, while others are added only to a
project. Some possible plugin features are:

* Adding new sub-commands to the `bevy` CLI tool.
* Adding new views to the graphical editor.
* Adding context-specific renderings of component data.
* Localize both the editor and underlying game.
* Alter game behavior when porting to new platforms.

## Implementation strategy

The following is a list of features that are needed for the minimum viable
editor:

* Project/System separation
* Version dispatching
* CLI Interface
* Window Interface
* New project construction
* Asset management
* Distributable packaging
* Credential/secrets management
* Scene/Entity manipulator (with possible component editing dispatch).
* Systems list
* Plugin architecture

While it is possible to make a prototype with a subset of these features, each
feature implementation must (1) makes space for every other feature, and can (2)
assumes the other features exist.

The rest of this section will be a sketch out the rough implementation for each
of these features.

### Project/System separation

1. Create two crates: `bevy_editor` and `bevy_editor_command`, the first will be
   added to projects (as a `bevy` crate feature), while the second will be installed on the user's system.
2. The `bevy_editor_command` crate will be a library crate, and provide a
   function named `cli`,  this will be the main entry point for the CLI and
   graphical editor.
3. Add a `main.rs` file to the `src/` folder associated with the `bevy` crate.
   Call `cli` from that crate.
    1. This makes two `bevy` crates, a library one, which already exists, and an
       executable one, which will be what the user installs.
4. Add `bevy_editor_command` as a dependency for `bevy`, this can be done
   through the `bevy_internal` crate, or as an optional feature.
5. The `cli` function must do basic argument parsing, project templating, and
   system/credential management. It will dispatch everything else to the `bevy_editor` crate.


### Version dispatching

1. Any project using the `bevy` tool is expected to have an `editor` binary
   crate in their workspace, the template will fill this in automatically.
2. The dispatch can use `cargo`, as a simple example, `bevy editor` can become
   an alias for:
```
cargo run -p editor
```
3. For this reason, the `editor` crate must provide the main function, so the
   dispatch should ideally have graceful failing if it doesn't exist.
4. We are using a separate crate so that the end distributable does not need to
   rely on the editor, developers that _do_ want the deliverable to rely on the editor can call it themselves.

### CLI Interface

1. Simple command line argument parsing can be done with clap.
2. Simple error handling can be provided with anyhow.
3. There does not seem to be a standard pretty-output library for rust,
   prettytable seems to be a start.
   1. Options like crossterm might also fit the bill, but also seem like overkill.
4. The manipulator/live viewing components can delegate to the window interface.

### Window Interface

1. Extend from the base crate's world.
1. Ideally, everything should be do-able through the graphical interface, but
   some things can be pushed to later releases.
2. The `bevy_editor_command` window only needs to handle: (1)Credential Storage,
   (2) Template Downloads, and (3) Project Selection, everything else is handled
   by the a separate `bevy_editor` window.
3. The `bevy_editor_command` window must seamlessly transition to the
   `bevy_editor` window, as if opening a project.
4. The ui should be made using bevy components.

### New project construction

1. Handled by `bevy_editor_command`.
2. Retrieve template from asset repo or local template storage.
3. Build template object mapping template languages to their result.
    1. For example foo.rs.tera should become foo.rs.
4. Fill in appropriate template fields from the user's environment.
5. Add CI workflows.
6. Add licenses.
7. Initialize as Git repo.

### Asset management

1. Template management can be done using terra.
2. Downloading static assets can be managed by reqwest.
3. Downloading code assets can be done with cargo.
4. Downloading template assets can be done with libgit2.
5. Template assets should use `directories` to store system wide.

### Distributable Packaging

1. Bundle `cargo build` with built asset files.
2. Let developers create per-target custom build scripts.
3. Should ideally support WASM out of the box.

### Credential/secrets management

1. Should use the operating system's keychain/credential storage for storing
   secrets.
2. For now, only for uploading to itch.io in CI builds.

### Scene/Entity manipulator (with possible component editing dispatch).

1. Can use rfc 62 as its data model.
2. Should provide an editor for adding and removing components to entities.
3. Should also contain a basic editor for setting the initial values of an
   entity's components.
   1. As a bonus, plugins can allow for domain-specific editor for given
      component types.
4. The scene viewer should place all objects in a scene at their initial
   position.
5. A play button lets users run any given scene with the world's systems.

### Systems list

1. Can use rfc 62 as its data model.
2. Can add an optional feature for Bevy's system to store metadata about the
   source location for systems, this will allow us to even find dynamic systems
   created at run-time.

### Plugin architecture

1. Most of the features listed above should be implemented as plugins.
2. This plugin-heavy approach allows future additions of i18n, l10n, and a11y.
3. a11y should use bevy_a11y
4. Internationalization can be done with either fluent or
   rust-i18n.
    1. If fluent, we should bring bevy_fluent or a similar project into the
       core project.

## Drawbacks

This is a lot of things for a single RFC. It would be possible to make a minimum
editor with fewer features, and would lead to an editor quicker.

However, the size of this RFC is merited for these reasons:

1. Most of these features are already required for making the editor usable.
2. Implementing most of these features can go faster when it assumes the other
   features are build along side it. For example, the metadata used for the
   entity manipulator can also be used for systems browser.
3. Making these features as plug-ins helps us make sure the plugin system is
   appropriately powerful.

## Rationale and alternatives

1. Split crate approach: This allows the tool to work in the users environment
   while having projects with different versions of Bevy. Putting everything
   into a single crate is simpler, but would either require more work for new
   users, or assume users only use the latest version of Bevy.
2. Providing a central editor base allows people to more easily extend the
   editor and/or make their own editor plugins.
3. We could make a more traditional editor, but that would 'lock-in' developers
   into using it, forcing them to choose between it or their own project scaffolding.
4. Similarly we could not make an editor, but then developers would be require
   to make their own scaffolding anytime they want to make a new Bevy project.

## Prior art

The following game engines have interactive editors:

1. Unity - The editor is primarily a world manipulator, editing the behavior of
   objects is (mostly) done in an external text editor.
2. Unreal - Like unity, but also provides a visual programming language called
   blueprints.
3. Godot - The default setup is also a world manipulator, but also has a built
   in textual editor, uses a custom programming language though.
4. Game Maker - Similar to godot.

There are also several systems for bidirectional and live programming:

1. [Lighttable](http://lighttable.com) - Provides a live editing environment for
   programs involving graphics.
2. [Hybrid Syntax](https://visr.pl) (disclaimer, I am the author of this work) -
   Focuses on mixing visual and textual elements in a single context.
3. [Sketch-n-Sketch][sketch-n-sketch] - A bidirectional editor for svg-like
   image creation.
4. [Lively](https://lively-next.org/) - A smalltalk inspired live programming
   environment for the web.
5. [Pharo](https://pharo.org/) - A modern smalltalk inspired system.
6. [Livelits][livelits] - Mixes some aspects of hybrid syntax with live
   programming, relies heavily on their type system.

The examples of the split `bevy_editor` and `bevy_editor_command` is common in the JavaScript ecosystem, but a few rust packages have adopted it too:

1. [Webpack](https://webpack.js.org/) - A common build tool in the JS ecosystem
   with a split `-cli` package.
2. [Prisma Client Rust][prism-rust] - A rust package that also uses the split
   `cli` package, but achieves it by requiring the developers to create a crate
   that calls the command line function itself.

## Unresolved questions and future possibilities

1. Asset storage in asset repos - Right now we are using a git repo, but will
   that scale to allowing users to submit their own assets?
2. Sandboxing - Having the `editor` crate extend the base crates world leaves
   the editor crate susceptible to bugs (or possibly malicious behavior) in the
   base crate.
3. UI Language - Bevy's UI system is extremely verbose. While this is fine for
   some simple tools, we will ultimately need a better UI system, ideally one
   that people can use to make their own editor plugins.
4. Presentation - This RFC only lists the features, it doesn't say anything
   about how they will be presented to the user.
5. Text Editor - While making a text editor is outside the scope of the Bevy
   editor, there are advantages that can come from having a text editor or even dom renderer built into Bevy's editor.

[asset-page]: https://bevyengine.org/assets/
[foxtrot]: https://github.com/janhohenheim/foxtrot
[sketch-n-sketch]: https://ravichugh.github.io/sketch-n-sketch/
[livelits]: https://hazel.org/papers/livelits-paper.pdf
[prisma-rust]: https://prisma.brendonovich.dev/getting-started/installation
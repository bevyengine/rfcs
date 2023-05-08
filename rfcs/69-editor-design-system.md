# Feature Name: `editor-design-system`

## Summary

This document proposes and describes a design system for the Bevy editor. From the [Wikipedia article](https://en.wikipedia.org/wiki/Design_system):

> A Design system is a collection of reusable components, guided by clear standards, that can be assembled together to build any number of applications.

## Motivation

A high-quality editor tool is crucial for enabling broad industry adoption of a game engine, as it can streamline workflows that are laborious purely via code, and make the engine more approachable to developers and creatives from various backgrounds.

We would like to significantly enhance the appeal of Bevy and help foster an even more vibrant and inclusive community around the project by shipping a state-of-the-art editor tool. Beyond just raw features, user experience will be vital to the Bevy editor's success. (As an example, consider the momentum Blender gained after the 2.80 UI redesign, despite previous versions already being quite capable.)

Most importantly, a game engine editor is a sizeable piece of software with hundreds (or thousands!) of individual UI pieces. While we _could_ build such a UI in an _ad-hoc_ fashion, this would likely yield less-than-optimal results. We would therefore like to come up with a set of foundational principles and guidelines ahead-of-time to help ensure we stick the landing, delivering a polished, consistent and user-friendly editor from the get-go.

**Note:** There are still several technical challenges to be addressed in order to enable the underlying functionality of the editor, such as IPC, Data Model and Hot Reloading, to name a few. This RFC focuses solely on the UI/UX concerns, as we believe now is the good time to start exploring what the editor will look like and how it will behave, and we expect the findings of this exploration will help inform new Bevy UI features and help us uncover any papercuts and limitations.

## Principles

The following are the key principles the Bevy editor's design should abide to:

- **Professional** — The editor is a professional tool, and as such, is intended to be used (potentially) on a daily basis, for hours at time, to achieve specific goals. We should be mindful of the user's time, screen real estate, attention span, and patience. While the UI can (and should) be appealing, it shouldn't outshine the content the user is working on, and shouldn't interrupt or disrupt their workflow unless strictly necessary.
- **Readable** — The entire (relevant) UI state should be readable by the user, at a glance, at any time. This means that we shouldn't have (non-contextual) toggleable modal states that are invisible, hard to find and get into, and most importantly, _get out of_. However, it's perfectly acceptable (and desireable) to display things only contextually. Whenever possible, multiple redundant signs (text, color, position, visual hierarchy) should be used redundantly when communicating information to the user to ensure maximum readability.
- **Accessible** — We want the Bevy editor to be accessible to users with various abilities and needs. This includes designing the UI with attention to color contrast, font size, screen reader compatibility for users with vision impairments, but also suitable accomodations for motor, sensorial and cognitive differences. The UI should be fully internationalizable, not resource-intensive and compatible with the widest range of commodity hardware, to ensure users of all regions and economic circumstances can make the most of it.
- **Economical** — We acknowledge that Bevy is a project run by volunteers on their free time. We should therefore set up the design system in a way that using and extending it is not onerous. This means that, for example, we shouldn't expect editor developers to submit new features with highly detailed, animated 3D icons that take tens of hours and extensive design experience to author.
- **Familiar** — Whenever possible, the UI should feel familiar to the user. This means acknowledging the baggage of 30+ years of 2D, 3D graphics and developer tools, and leveraging established patterns, such as terminology, shortcuts, symbols, modifiers, layouts, information hierarchy and behaviors. There's of course a wide range of variability across different tools, so _some_ decisions will have to be made to establish a reasonable common denominator for the default settings. While we won't be using native UI frameworks to build the editor, we should honor native OS conventions, whenever possible.
- **Recognizable** — The Bevy editor should be instantly recognizeable as the Bevy editor, when seen in screenshots, and ideally even at a distance. For example, when watching a developer interview with a computer screen in the background, you should be able to point and say “Ha! That's Bevy!” This is of course, achieved tastefully by reusing subtle brand elements from the website, _not_ adding a giant watermark. The goal is to build a stronger sense of community, identity and trust with the users, (And as a bonus also making us feel like we're leaving a mark! 😉)

## Intended Audience

### This Document

This document is geared towards people who will (eventually) be building the Bevy Editor UI, and also to those interested in making mockups and explorations of new editor features.

### The Editor

The Editor is designed with three main audiences in mind:

- **3D Artists** — People who are already familiar with and proficient in existing 3D authoring tools, such as Blender, Maya, 3dsmax, Cinema 4D, or with 3D game engine editors, such as Godot, Unity or Unreal Engine.
- **2D Artists** — People who are already familiar with and proficient in existing 2D authoring tools, such as Krita, Photoshop, Illustrator, Figma, Affinity Designer, Aseprite, with specialized 2D game engine editors, such as GameMaker, Clickteam Fusion, and also with 3D game engine editors that are capable of 2D workflows, such as Godot, Unity or Unreal Engine.
- **1D Artists (Developers!)** — People who are already familiar with and are using Bevy as a game engine today, or are coming from other game engines and trying Bevy out for the first time.

Additionally, whenever possible without compromising the usability and feature-richness for the three audiences above, and without requiring the creation of custom, separate UIs, we'd also like to cater to a fourth audience:

- **Complete Beginners** — People who just got started with game development or design, are just curious to try a game engine for the first time, or have a game or hobby project idea but don't know how to get started.

## Color

Colors are a _very powerful signifier_. The human visual system is typically capable of segmenting, identifying and grouping visual elements based on color much more quickly than by shape, texture and other factors. This means that **color-coding** is a very good strategy for helping users read and navigate potentially crowded UIs.

Unfortunately, it also means that color can be _overpowering_ in terms of  attention, distracting users from the content they're creating. Since human color perception is contextual, the presence of nearby colors can even distort the appearance of other colors, leading to a suboptimal, frustrating experience for workflows such as lighting and color-grading.

We work around this by being _very disciplined_ with color usage, and establishing the **90:9:1 rule** for colors. Most of the UI (90%) should be comprised of unsaturated grays, occasional primary/destructive controls use pastel shades (9%), and highly saturated colors are reserved for their specific consistent semantic meaning, and typically only applied in thin, 1px strokes. (1%)

Furthermore, we use the human tendency to perceive darker shades as being “receded” and lighter shades as being “protuding” from a surface to our favor, to help build a perceived depth-based visual hierarchy.

![Bevy Editor Design System Color Palette](assets/69-editor-design-system/color-palette.png)

Here's a breakdown of the color palette:

- **Background Color** — This is the color used to paint the portion of the UI that goes behind every single major UI element. It's primarily visible in the _gaps_ between UI landmarks such as panels, toolbars, views, acting as some sort of “grout” between the “tiles” of the UI. It should _not_ be used directly in controls, to preserve the sensation of depth.
- **Foreground Color** — This is the color used for text in controls and labels.
- **Surface Colors** — These are 5 gray shades, centered around a “base” color, ranging from -2 to +2. Positive shades are used for protuded controls (such as buttons), and negative shades are used for receded controls (such as text fields). The surface shades can be adjusted on a sliding scale (along with inverting the foreground color) to achieve Dark Mode / Light Mode UI, as well as many intermediate schemes suitable to color-sensitive workflows.
- **Semantic Colors** — These are used sparingly to denote their specific meanings. The accent color is used both to denote selection, as well as a general landmark color for things like folder icons.
- **Pure Red, Green and Blue** — These colors are used to denote _themselves_ (e.g. in material/shader inputs) as well as to denote the X, Y and Z 3D axes. The strong, well established association of these colors from other tools is leveraged to help users orient themselves in the 3D space. Importantly, their intensity values are tweaked independently so that they're all _perceptually_ at the same brightness level.

## Iconography

Our icon style is heavily influenced by Blender's, but further simplified, with the primary goal being to make icon creation as economical as possible. Specifically, a single icon should _not_ take more than **10 minutes** to be created and fully fleshed out by someone comfortable with a vector tool.

Icons are:

- Vector-based;
- Authored in three separate sizes, (16x16, 24x24, 32x32) with small tweaks and simplifications made where appropriate to accomodate for the smaller sizes;
- Hinted to the pixel grid;
- Drawn with a 1px outline style, with a few exceptions (such as folders) meant to be clear visual landmarks;
- Either single color, or at most two colors, with one being the foreground color;
- Whenever “shading” is needed, dotted lines or some other dithering/meshing pattern should be used;
- Exported at 1x, 2x scale factors, and for the 32x32 variant, also exported at a constant 512x512 size, for “very large” display (e.g. on zoomed in icon grids) or as a texture in the editor world.

![Sample Bevy Editor Design System Icons](assets/69-editor-design-system/icons.png)

![Same Icon at Different Sizes](assets/69-editor-design-system/icon-sizes.png)

## 9-Slices

![Sample 9-Slice Images](assets/69-editor-design-system/nine-slices.png)

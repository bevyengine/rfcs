# Feature Name: Visual Scripting

## Summary

In the forthcoming Bevy editor, there should be the ability to do visual scripting, a la Unreal's blueprints system.

## Motivation

Rust, while being fast and modern, is a pretty hard language for first timers to use. Additionally, when you're working on a game, there will be artists and sound designers who will need to use the engine as well. Having a visual scripting solution allows them to work on the game without having to know much programming.

## Guide-level explanation

Here's how I'd think it would work. (For the sake of using a term everyone probably knows, I'll call the visual scripting file a "blueprint")

- Each blueprint that gets created corresponds to an entity/Component Bundle
- The blueprint itself would display all of the components and systems. You could select systems and see what components it uses in their queries.
- Looking at this from an artists perspective, say that they want to add a material to the entity/component bundle. They could open up a menu, search up the material component, and then click add. 
- Then, if they want to have a system to, say, change the color of the material over time, they could right click, spawn in a new system, and then make the system do a loop where one of the variables of the colors could increase by one.

## Reference-level explanation

- I think the best way to handle a visual scripting system would be to just compile it to Rust. The code would be computer generated, but that shouldn't be a problem since its an non-programmer facing tool.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- This design isn't the best way of doing this, but its the best way I can think of right now. I look forward to talking with people about the better way of doing this.
- Again, this feature is meant to make the engine more accessible to non-programmers and non-rust users while still being a rust centric engine.

## \[Optional\] Prior art

* The biggest success for a visual scripting system, and most definitely the biggest influence, would be Unreal Engine 4's Blueprint system.
  * There are a lot of problems with it, but it makes developing templates and sharing assets very easy from a non-technical point of view. In fact, most Unreal projects use Blueprints in some fashion due to the ease of use.

## Unresolved questions

- I'd like to have an actual design of the visual scripting system. I'd like the bevy community to think about this feature seriously because I think the benefits it could offer outweigh the negatives.
- There's also a lot of questions about how exactly to implement this. All I know is that you'd need to generate Rust code with it, and an exploration of how to generate rust code would need to be in order.

## Future possibilities

* Somewhat unrelated, but a conversation system a la Yarnspinner using visual scripting could be nice
* Also, visual scripting is basically a built in state machine, which means we wouldn't need to have an add on for that
* Visual scripting could also be used to take care of animations, and animation transitions
* It's also could be dogfed to support shader coding.
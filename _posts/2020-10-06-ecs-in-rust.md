---
title: ECS in Rust
published: true
---

I've been recently playing around with a number of game development libraries in [Rust](https://www.rust-lang.org/). One game engine in particular that I've been looking at is the [Bevy Engine](https://bevyengine.org/). The overall [ECS](https://en.wikipedia.org/wiki/Entity_component_system) is very easy to work with and it makes common tasks like spawning entities and [loading meshes](https://github.com/bevyengine/bevy/blob/master/examples/3d/load_model.rs) very straightforward. 

In any case, I was in the process of attempting to document what precisely bevy is doing with all of its [default plugins](https://github.com/bevyengine/bevy/blob/master/src/add_default_plugins.rs#L7). At some point soon I will follow-up with that blog post documenting what I have found out. Some of the areas that still need work is how precisely the rendering plugin works. It uses a concept known as a [render graph](https://andrewcjp.wordpress.com/2019/09/28/the-render-graph-architecture/) in combination with its internal typed system for injecting resources, entities, components, and queries. The complexity of the implementation and the difficulty of understanding what's going on tends to cause me to look for simpler more straightforward implementations that I can actually understand. The techniques might not be the best, but it helps give me a sense of "here are the variables and constraints" surrounding an implementation.

## Game Engines

My hope is that at some point someone will come up with a graphics library to do basic tasks rather than embed the entire thing into an engine, or at least provide a high level way of doing things and have swappable backends. Some great examples of engines that make use of other libraries (rather than DIY for everything) include:

* [coffee](https://github.com/hecrj/coffee)

    Makes use of:
    * [winit](https://github.com/rust-windowing/winit) for windowing and input events
    * [stretch](https://github.com/vislyhq/stretch) for responsive layouts
    * [gilrs](https://gitlab.com/gilrs-project/gilrs) for gamepad support
    * [nalgebra](https://github.com/rustsim/nalgebra) for Point, Vector, Transformations
    * [image](https://github.com/image-rs/image) for image loading and texture array buliding
    * [glyph_brush](https://github.com/alexheretic/glyph-brush/tree/master/glyph-brush) for TrueType font rendering
    * [wgpu](https://github.com/gfx-rs/wgpu) for Vulkan, Metal, D3DD11/D3D12

* [ggez](https://github.com/ggez/ggez)

    Makes use of:
    * [winit](https://github.com/rust-windowing/winit) for windowing/input
    * [gfx-rs](https://github.com/gfx-rs) graphics
    * [rodio](https://github.com/RustAudio/rodio) rust audio playback (mp3, wav, vorbis, flac)
    * [rusttype](https://gitlab.redox-os.org/redox-os/rusttype) opentype ttf/otf font support for rasterizing
    * [glyph_brush](https://github.com/alexheretic/glyph-brush/tree/master/glyph-brush) for TrueType font rendering
    * [mint](https://github.com/kvark/mint) Math interoperability types (point, Matrices, Vectors, Quaternions, Euler Angles)

Since these are focused on 2D game engines, 3D game engines get a lot more complicated and the result is something like [bevy](https://bevyengine.org/), [amethyst](https://github.com/amethyst/amethyst), or [piston](https://www.piston.rs/). It's worth taking a look at how these game engines work under the hood, what libraries they use, and what I might be able to do to perform basic rendering tasks in Rust.

## Entity Component Systems

With that, I am now looking at [entity component systems](https://en.wikipedia.org/wiki/Entity_component_system) in Rust. I've found a really great library called [hecs](https://github.com/Ralith/hecs) that provides a minimalist library approach rather than a full on framework that happens to include ECS with it.

```
Entity Component Systems is an architectural pattern that follows composition over inheritance where every object in a game scene is an entity and every entity consists of one or more components for data. Behaviors of entities can be changed at runtime by systems that add, remove or mutate components. ECS is often used with data-oriented design techniques.

* Entity: general purpose object, consists of a unique id
* Component: raw data for one aspect of an object, and how it interacts with the world (implementations typically use structs, classes, associative arrays)
* System: runs continuously and performs global actions on every entity that possesses one or more components of the same aspect of that system.
```

If you're interested in a good talk about ECS in Rust for Game Development, take a look at [RustConf 2018 Closing Keynote by Catherine West](https://www.youtube.com/watch?v=aKLntZcp27M). She does a good job at describing the kind of pitfalls you run into in game development and how ECS helps to solve many of these problems.

In any case, here is a relatively straightforward example of what an ECS looks like when using [hecs](https://github.com/Ralith/hecs/blob/master/examples/ffa_simulation.rs).

## HECS Example

In HECS, as is somewhat common to ECS libraries, to describe components you can use just the simple structs like below. So let's say we wanted to have entities that have a position and velocity.

```rust
use hecs::*;
use std:io;

#[derive(Debug)]
struct Position {
    x: i32,
    y: i32
}

#[derive(Debug)]
struct Velocity(i32);

fn main() {
}
```

To spawn our entities with the above components we simply use the **hecs::World**.

```rust
fn spawn_player(world: &mut World) {
    let pos = Position {
        x: x,
        y: y
    };
    let velocity = Velocity(0);
    world.spawn((pos, velocity))
}

fn main() {
    let mut world = World::new();
    spawn_player(&mut world);
}
```

When we setup a system, we will want to look up entities with a set of components that we are interested in. Let's say we wanted to add a system that responds to input that will move entities with a position and velocity.

```rust
fn movement_system(world: &mut World) {
    for (id, (pos, velocity)) in &mut world.query::<(&mut Position, &Speed)>() {
        pos.x += s.0;
        println!("Unit {:?} moved to {:?}", id, pos);
    }
}
```

The ability to query for entities that have both the position and speed components is really quite handy. hecs also has built in functions for doing more complex logic such as **with** / **without**. [bevy](https://github.com/bevyengine/bevy/blob/master/examples/ecs/ecs_guide.rs), by contrast has a bit more (until hecs adds support) boolean logic available such as **or** and **and** type queries. From what I can tell, however, it looks like Bevy's ECS is actually a fork of hecs and upstream changes may be planned on being merged back into hecs itself (see [#issue 71](https://github.com/Ralith/hecs/issues/71))

## Conclusion

Combined with the hecs example, and several [winit](https://github.com/rust-windowing/winit/blob/master/examples/handling_close.rs) examples, as well as playing around more with [bevy](https://github.com/bevyengine/bevy/blob/master/examples/ecs/ecs_guide.rs) I'm hoping to make a bit more progress in game development. With these things, progress is a matter of consistency because there is simply **a lot** of material to cover.



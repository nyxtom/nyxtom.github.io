---
title: Breakout in Rust with Bevy
published: false
---

In this tutorial we're going to work with the [Bevy Game Engine](https://github.com/bevyengine/bevy) in order to understand just what is going on underneath all those plugins/crates and systems.

```
Bevy is a refreshingly simple data-driven game engine built in Rust. Design goals center around:

  * Capable: Offer a complete 2D and 3D feature set
  * Simple: Easy for newbies to pick up, but infinitely flexible for power users
  * Data Focused: Data-oriented architecture using the Entity Component System paradigm
  * Modular: Use only what you need. Replace what you don't like
  * Fast: App logic should run quickly, and when possible, in parallel
  * Productive: Changes should compile quickly ... waiting isn't fun
```

Keep in mind that the game engine is quite broad so we aren't going to be able to cover most topics, however we will cover enough to get started with the game engine itself. Here is a list of things we're going to cover:

* Window / Render Context
* Camera2dComponents: orthographic 2d camera
* SpriteComponents, SpriteSheetComponents
* UI, Text, Buttons
* Time: frame delta time and overall running time
* Timer: detect when an interval of time has elapsed
* Input (KeyCode, MouseButton), Input Events
* Font: font for rendering text
* Asset Server: loading assets from disk

## Setting up a window

If you haven't already gone through [the book](https://bevyengine.org/learn/book/introduction/) for bevy, I suggest taking a look at that. It covers some of the basics like setup, ecs, plugins, resources and queries. What makes bevy particularly pleasant to work with is that the ECS (entity component system) implementation uses normal Rust datatypes for all concepts. This means that for the most part you are just going to write normal everyday functions and structs for your entire codebase without having to worry about complex traits, lifetimes, patterns or macros.

To get started, make sure you have the latest version of rust with `rustup` and then create a new project for `breakout`.

```
cargo init breakout && cd ./breakout
```

I use a `cargo` utility **[cargo-edit](https://github.com/killercup/cargo-edit)** to be able to quickly manage dependencies on my `Cargo.toml`. 

```
cargo add bevy
```

In `src/main.rs` you can add the following to get started:

```
use bevy::prelude::*;

fn main() {
    App::build()
        .add_default_plugins()
        .run();
}
```

You should be able to go ahead and run this with `cargo run`. The default plugins includes:

* TypeRegistryPlugin
* CorePlugin
* TransformPlugin
* DiagnosticsPlugin
* InputPlugin
* WindowPlugin
* AssetPlugin
* RenderPlugin
* SpritePlugin
* PbrPlugin
* TextPlugin
* GilrsPlugin
* GltfPlugin
* WinitPlugin
* WgpuPlugin

### TypeRegistryPlugin

The type registry is effectively a system of hashmaps of type identifiers and various forms of names to both components and properties (useful for dynamic interaction with struct fields using names, similar to reflection). Within Bevy, we can register components that can be serialized, deserialized for use in scene loading for example. In future versions of Bevy, registration of components will also be made available in the Bevy editor. Additionally, the TypeRegistryPlugin helps manage these properties and components in a thread safe way.

### CorePlugin

The core plugin provides the core property vectors, matrices, quaternions, time resource, and timer components as well as the initial time systems and entity labeling.

### TransformPlugin

The TransformPlugin is effectively a port of **[amethyst/legion_transform](https://github.com/amethyst/legion_transform)** that implements a hiearchical space transform system to support all the usual combinations of **rotations**, **translations**, **local to world transformations**, **scaling**, **parent-child hierarchies**, **affine**, **similarity**, **isometry**, all within the context of *bevy ecs* and [glam](https://crates.io/crates/glam) (for simple and fast 3D math).

### DiagnosticsPlugin

Useful for tracking performance diagnostics, iterations, frame render time, framerate information. Combine with **PrintDiagnosticsPlugin** to render out the diagnostics to the terminal. Also, you can take a look at **WgpuResourceDiagnosticsPlugin** to track things like: buffers, render pipelines, swap chains, samplers, shader modules, window surfaces, textures. Here's an example plugin that displays FPS on the screen.

```rust
use bevy::prelude::*;
use bevy::diagnostic::{Diagnostics, FrameTimeDiagnosticsPlugin};
use bevy::ecs::Mut;

fn main() {
    App::build()
        .add_default_plugins()
        .add_plugin(FrameTimeDiagnosticsPlugin::default())
        .add_plugin(ScreenFpsPlugin)
        .run();
}

pub struct ScreenFpsPlugin;

impl Plugin for ScreenFpsPlugin {
    fn build(&self, app: &mut AppBuilder) {
        app
            .add_plugin(FrameTimeDiagnosticsPlugin)
            .add_startup_system(Self::setup_system.system())
            .add_system(Self::fps_update.system());
    }
}

impl ScreenFpsPlugin {
    fn fps_update(diagnostics: Res<Diagnostics>, mut text: Mut<Text>) {
        if let Some((Some(fps), Some(average))) = diagnostics.get(FrameTimeDiagnosticsPlugin::FPS).map(|x| (x.value(), x.average())) {
            text.value = format!("{:<3.3} ({:<3.3})", fps, average);
        }
    }

    fn setup_system(mut commands: Commands, asset_server: Res<AssetServer>) {
        commands
            .spawn(UiCameraComponents::default())
            .spawn(TextComponents {
                text: Text {
                    value: "FPS".to_string(),
                    font: asset_server.load("assets/fonts/Roboto-Light.ttf").unwrap(),
                    style: TextStyle {
                        font_size: 25.0,
                        color: Color::WHITE,
                    },
                },
                transform: Transform::new(Mat4::from_translation(Vec3::new(0.0, 0.0, 2.0))),
                ..Default::default()
            });
    }
}
```

### InputPlugin

Adds keyboard, mouse and gamepad events, as well as keycode resources, mouse button resource, gamepad button/axis resources, and adds the keyboard and mouse button input systems. Additionally, this adds an AppExit event with the exit_on_esc_system system which just listens for the escape key to be pressed. To check the current state of specific keys or mouse buttons you can use the Input<T> resource on a system.

```rust
fn my_system(keys: Res<Input<KeyCode>>, buttons: Res<Input<MouseButton>>) {
    if keys.pressed(KeyCode::Space) {
        // space is being held down
    }
    if buttons.just_pressed(MouseButton::Left) {
        // a left click just happened
    }
}
```

Detecting all activity can be done by listening to input/key/gamepad events.

```rust
struct GameInputState {
    keys: EventReader<KeyboardInput>,
    cursor: EventReader<CursorMoved>
}

fn game_input_system(
    mut state: ResMut<GameInputState>,
    ev_keys: Res<Events<KeyboardInput>>, 
    ev_cursor: Res<Events<CursorMoved>>) {
    for ev in state.keys.iter(&ev_keys) {
        if ev.state.is_pressed() {
            // just pressed key {:?} ev.key_code
        } else {
            // just released key {:?} ev.key_code
        }
    }

    // absolute cursor position
    for ev in state.cursor.iter(&ev_cursor) {
        // cursor at ev.position
    }
}
```

### WindowPlugin

Support for **WindowResize**, **CreateWindow**, **WindowCreated**, **WindowCloseRequested**, **CloseWindow**, **CursorMoved** events. Adds default *WindowDescriptor* that includes title, width, height, vsync, resizable flags, decorations flags, mode for full screen, and window identifiers. This plugin is mostly utilized as a settings for other plugins and can be used to configure window settings via the *WindowDescriptor* along with support for multiple windows. Used by the **RenderPlugin** when performing pass operations.

### AssetPlugin

Support for typed collections of assets including **textures**, **sounds**, **3d models**, **maps**, **scenes** via an **AssetServer**. By default, all systems run in parallel, this makes things efficient but in the case we want things to run in a specified order Bevy has several approaches to this. Any system that writes a component or resource (ComMut, ResMut) will force synchronization. Any systems that access the data type and were registered before the system will finish first. Any systems registered after the system will wait for it to finish. An alternative to these rules to force sychronization at the right time is to use **stages**. 

*Stage* is a group of systems that execute (in parallel). Stages are executed in order, and the next stage won't start until all systems in the current stage have finished. When you call `add_system(system)` this adds systems to the **UPDATE** stage by default. Default stages include **FIRST**, **EVENT_UPDATE**, **PRE_UPDATE**, **UPDATE**, **POST_UPDATE**, **LAST**. With custom stages created like so:

* `add_stage_before(stage::UPDATE, "before_round")` (before round might be a new player system, new round system)
* `add_stage_after(stage::UPDATE, "after_round")` (after round might be a score check system, game over system)

Examples such as:

* `add_system_to_stage(stage::UPDATE, score_system.system()` (update might be a print message system, score system)
* `add_system_to_stage("before_round", new_player_system.system())`
* `add_system_to_stage("after_round", game_over_system.system())`

In the case of the **AssetPlugin**, there are 2 stages including: **PRE_UPDATE stage::LOAD_ASSETS** and **POST_UPDATE stage::ASSET_EVENTS**.

Take a look at [examples/asset](https://github.com/bevyengine/bevy/tree/master/examples/asset) for a list of examples with the **AssetPlugin** including custom asset loading, hot reloading (detecting changes via file watcher), and basic loading.

### RenderPlugin

Provides rendering functionality for the bevy engine. Most of the functionality for rendering cameras, meshes, colors, textures, materials, asset rendering, perspective/projection calculations, vertex buffers, visible entity calculations, as well as shader pipelines. The way bevy is implemented however, the actual render backend is done through the `WgpuPlugin` crate in [crates/bevy_wgpu](https://github.com/bevyengine/bevy/tree/master/crates/bevy_wgpu/src). [WGPU](https://github.com/gfx-rs/wgpu) provides the backing implementation based on the [WebGPU](https://www.w3.org/community/gpu/) spec with cross platform support from DX11, DX12, Vulkan, Metal and in progress support for OpenGL. Since this plugin covers quite a lot, we'll only be able to go into a brief overview of the various stages of the pipeline.

1. *PRE_UPDATE* **clear_draw_system**

```
Simply clears the render commands Vec<RenderCommand> to be empty where a RenderCommand can be:

* SetPipeline (pipeline Handle<PipelineDescriptor>)
* SetVertexBuffer (slot: u32, buffer: BufferId, offset: u64)
* SetIndexBuffer (buffer: BufferId, offset: u64)
* SetBindGroup (index: u32, bind_group: BindGroupId, dynamic_uniform_indices: Option<Arc<[u32]>>)
* DrawIndexed (indices: Range<u32>, base_vertex: i32, instances: Range<u32>)
* Draw (vertices: Range<u32>, instances: Range<u32>)
```

2. *POST_UPDATE* **active_camera_system**

```
Ensures that an active camera is set by iterating over the ActiveCameras resource.
```

3. *POST_UPDATE* **camera_system::OrthographicProjection**

```
Camera system that responds to window resize, window creation, and window events by calculating the orthographic projection, projection matrix and depth calculations based on the provided changed window parameters.
```

4. *POST_UPDATE* **camera_system::PerspectiveProjection**

```
Camera system that responds to window resize, window creation, and window events by calculating the perspective projection, projection matrix and depth calculations based on the provided changed window parameters.
```

5. *POST_UPDATE* **visible_entities_system**

```
Using the global transform, active camera query, and a query on entities with a draw component to determine transparent, visibility, and entity ordering using the camera depth calculations and positioning.
```

It's important to understand how [queries](https://github.com/jamadazi/bevy-cheatsheet/blob/master/bevy-cheatsheet.md#queries) work in bevy in that they allow you operate on any set of entities with/without and filtering based on component properties. The **visible_entities_system** takes advantage of queries in order to filter entities with a draw component and check for the visibility. This area of the code however, is under active performance improvements (see [issue 190](https://github.com/bevyengine/bevy/issues/190)). Additional discussion around approaches such as despawning entities, adding/removing draw components, as well as just setting *is_visible* to false with a camera view check system (checking to see if entities are within the camera view) are discussed in [issue 518](https://github.com/bevyengine/bevy/issues/518) and [issue 190](https://github.com/bevyengine/bevy/issues/190). 

Custom stages are added after the *ASSET_EVENTS* stage (a **POST_UPDATE**). These stages are in the following order:

* ASSET_EVENTS
* RENDER_RESOURCE
* RENDER_GRAPH_SYSTEMS
* DRAW
* RENDER
* POST_RENDER

6. *RENDER_RESOURCE* **mesh_resource_provider_system**

```
Handles vertex buffer, index buffer conversions from mesh assets and changes via AssetEvents that occur for meshes that are either inserted, removed, or updated. Uses the render pipelines to set mesh primitives and set vertex buffers. Mesh plugin adds additional default primitive shapes for Icosphere, Planes, Quads, and Cubes. Conversion between mesh primitive and vertex positions/normals/uvs/indices is converted into expected vertex buffer bytes corresponding to position, normal, and uv.
```

7. *RENDER_RESOURCE* **texture_resource_system**

```
HdrTexture loading, Image texture loading, with system support for creating texture and samplers from texture assets. Handles changes via AssetEvents that occur for textures that are either inserted, removed, or modified. Uses the resource render context to set asset resources, create texture, create sampler.
```

8. *RENDER_GRAPH_SYSTEMS* **render_graph_schedule_executor_system**

A render graph is a way of defining the rendering pipeline using self contained nodes in an acylic directed graph. Each node has a set of inputs and outputs, which link other nodes or resources. When executed, the graph is traversed executing each node in turn. On major advantage is the ability to traverse the graph before run time to collect useful data such as resource state transitions, temporary resource use... etc. 

9. *DRAW* **draw_render_pipelines_system**
10. *POST_RENDER* **clear_shader_defs_system**

### SpritePlugin

* SpritePlugin
* PbrPlugin
* TextPlugin
* GilrsPlugin
* GltfPlugin
* WinitPlugin
* WgpuPlugin

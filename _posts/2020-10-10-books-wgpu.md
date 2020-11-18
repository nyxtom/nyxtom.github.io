---
title: Rust Graphics Programming with WGPU
published: false
book: true
---

{% raw %}
<style type="text/css">
.hack h1 {
    font-size: 2rem;
}
.hack h2 {
    font-size: 1.4rem;
    border: dashed 10px #f9f9f9;
    margin-top: 18rem;
    margin-bottom: 9rem;
    padding-top: 4rem;
    padding-left: 3rem;
    padding-bottom: 4rem;
    margin-left: -1rem;
    width: calc(100% + 2rem);
}
.hack h3 {
    font-size: 1.3rem;
    padding: 3rem 1rem;
    padding-top: 4rem;
    padding-left: 3rem;
    margin-top: 12rem;
    border-top: dashed 4px #f4f4f4;
    box-shadow: 2px -5px 10px rgba(0, 0, 0, 0.05);
    margin-left: -0.5rem;
    width: calc(100% + 1rem);
}
</style>
{% endraw %}

After spending significant time going through the documentation and source of [gfx-rs/wgpu](https://github.com/gfx-rs/wgpu) and various [gfx-rs/gfx](https://github.com/gfx-rs/gfx) backends, Vulkan portability layers, the book of shaders, SPIR-V, and all else - I decided I wanted to compile all my notes together into a comprehensive guide to **Rust Graphics Programming with WGPU**. 

This guide is a deep dive into WGPU with a high level overview of shader programming in relation to that. It covers a range of topics fundamental to graphics programming including: rendering pipelines, shaders, vertex buffers, instancing, textures, sampling, bind layouts, compute pipelines, debugging, cross compilation, target platform builds and more. Along the way, we will use that knowledge to build simple render pipelines and demos to reinforce what we've learned. By the end of the book, we put it all together and build a simple cross platform game with WGPU.

**Table of Contents**:
* TOC
{:toc}

## Execution Order

Graphics libraries closely follow hardware abstraction layers provided by modern graphics cards. Although, this is sometimes the other way around, where graphics libraries like Vulkan provide a spec and hardware vendors follow this spec to connect with underlying hardware graphics layers. Nevertheless, higher level graphics apis describe an execution order that tends to be very similar no matter what library you end up using.

1. **Initialize device/instance/surfaces**: data structures needed to access GPU and surface handles
2. **Load assets**: compile and load shader modules, descriptors for rendering pipelines, create and populate command buffers for the GPU to execute, send resources to the GPU exclusive memory
3. **Update assets**: update resources and uniforms to shaders, perform application logic
4. **Render Operations**: send list of command buffers to the queue, present swapchain, wait until GPU signals that the frame was rendered to the current back buffer
5. **Repeat 2-4** until close
6. **Exit/Cleanup** finish remaining work on GPU, de-reference and cleanup resources before exiting

## Dependencies

Throughout this book we will end up using a variety of dependencies to perform various tasks in [rust](https://rust-lang.org). The dependencies are meant to be as minimal as possible and are mentioned below:

* [winit](https://github.com/rust-windowing/winit) window handle, event loop, events
* [winit_input_helper](https://github.com/rukai/winit_input_helper) update the winit events into a input state
* [wgpu](https://github.com/gfx-rs/wgpu) cross platform rust library for working with Vulkan, Metal, DirectX 11/12, WebGPU
* [futures](https://github.com/rust-lang/futures-rs) adds additional combinator utilities for async rust
* [bytemuck](https://github.com/Lokathor/bytemuck) working with bytes from structs and types for buffers later
* [image](https://github.com/image-rs/image) image processing functions and methods for converting image formats
* [shaderc](https://github.com/google/shaderc-rs) rust bindings for the collection of tools/libs for shader compilation
* [nanorand](https://github.com/aspenluxxxy/nanorand-rs) zero-dependency library for random number generation
* [rusttype](https://github.com/redox-os/rusttype) font retrieval and glyph calculations with font texture caching

```toml
[dependencies]
winit = "0.23.0"
winit_input_helper = "0.8.0"
wgpu = "0.6.0"
futures = "0.3.6"
bytemuck = { version = "1.4.1", features = ["derive"] }
image = "0.23.10"
shaderc = "0.6.2"
nanorand = "0.4.4"
rusttype = { version = "0.9.2", features = ["gpu_cache"] }
```

## Event Loop

The event loop is essential for any process that needs to interact with messages related to user interactions such as keyboard/mouse events, network traffic, system processing, timer activity, ipc communication, device control and general I/O communication. To do any of this, we need to go through the operating system and each type of operation is dependent on resources that the OS then abstracts over (such as hardware resources and peripheral implementations). 

In fact, you could consider every piece of hardware from the monitor, the GPU, the network card, to storage devices as all independent computers communicating over a network as scheduled by the operating system and CPU. Each operating system has its own implementation of performing blocking and non-blocking I/O. Consider that when we ask the OS to perform any kind of blocking operation, such as waiting for a particular file resource, the OS must suspend the thread that makes this call. Suspending the thread will stop executing code and store the CPU state in order to go on to process other operations.

When data arrives for us through the network, the OS will wake up our thread again and resume operations. Non-blocking I/O by contrast will not suspend the thread that made the initial request and instead give it a handle which the thread can use to ask the OS if the event is ready or not. Asking the OS in this way is considered **polling**.

Non-blocking I/O gives us more freedom at the cost of more frequent polling (such as in a loop), and this can take up CPU time. Each operating system has their own implementations for hooking into these polling and wait systems for events. In some cases, you can hook into the OS to wait for many events rather than being limited to waiting on one event per thread. The most common implementations make use of **epoll**, **kqueue**, and **IOCP**.

In Rust, the most common crate for working with non-blocking I/O is through the [mio](https://crates.io/crates/mio) crate. mio provides the underlying platform specific extensions and is backed by epoll, kqueue, and IOCP. On top of **mio**, another common crate that is used for **window management** and multi-threaded event management is the [winit](https://github.com/rust-windowing/winit) crate.

Winit provides all the core functionality needed to get started for a number of cross platform features:

* Window Initialization
* Window decorations
* Resizing, resize snaps/increments
* Transparency
* Maximization, max toggling, minimization
* Fullscreen, fullscreen toggling
* MiDPI support
* Popup / modal windows
* Mouse events
* Cursor icons, locking cursor
* Touch events, touch pressure
* Multitouch
* Keyboard events
* Drag/drop
* Raw device events
* Gamepad/joystick events
* Device movement events

**Winit** builds on top of **mio** to provide all these features and more while solving a number of multi-threaded issues in a cross-platform compatible way. The simplest example of building a window with an event loop in rust can be done in only a few lines of code.

```rust
use winit::event::{Event, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::WindowBuilder;

fn main() {
    let mut input = WinitInputHelper::new();

    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
        .with_title("Rust by Example: WGPU!")
        .build(&event_loop)
        .unwrap();

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Wait;
        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } => *control_flow = ControlFlow::Exit,
            _ => {}
        }
    });
}
```

Throughout the book we will make use of **winit** to build various examples. Additionally, we will use another crate [winit_input_helper](https://github.com/rukai/winit_input_helper) to update the current input state whenever keys, mouse events or anything else is processed within the event loop. Using the winit_input_helper will make sure we can perform simple state operation checks such as **key_pressed** or **key_released** and others without having to write our own input state management system.

```rust
use winit::event::{Event, VirtualKeyCode, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::{Window, WindowBuilder};
use winit_input_helper::WinitInputHelper;

fn main() {
    let mut input = WinitInputHelper::new();

    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
        .with_title("Rust by Example: WGPU!")
        .build(&event_loop)
        .unwrap();

    event_loop.run(move |event, _, control_flow| {
        if input.update(&event) {
            if input.key_released(VirtualKeyCode::Escape) || input.quit() {
                *control_flow = ControlFlow::Exit;
                return;
            }
        }

        match event {
            Event::RedrawRequested(_) => {
                // render call code
            },
            Event::WindowEvent { event, .. } => match event {
                WindowEvent::Resized(physical_size) => {
                    // resize code
                },
                WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {
                    // resize code
                },
                _ => {}
            },
            Event::MainEventsCleared => {
                window.request_redraw();
            },
            _ => {}
        }
    });
}
```

Now that we have a bit of a better idea of the event loop and managing the input state as well as the window itself, we can move along to some of the other definitions that we are working with.

## Initialization

### Instance

> Context for all other wgpu objects, the primary use of an Instance is to create **[wgpu::Adapter](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html)** and **[wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html)**.

Considered the entry point to the entire code for working with any graphics library. This includes vulkan's **Instance**, DirectX **IDXGIFactory**, **CAMetalLayer** in Metal, **GPU** in WebGPU.

```rust
let instance = wgpu::Instance::new(wgpu::BackendBit::PRIMARY);
```

Note that [wgpu::BackendBit::Primary](https://docs.rs/wgpu/0.6.0/wgpu/struct.BackendBit.html) is referring to the backends that wgpu will use. Primary refers to first tier api support (Vulkan + Metal + DX12 + WebGPU) with secondary support for OpenGL + DX11. Since OpenGL/DX11 is still experimental, targeting through these backends may be unsupported in some areas. Use **PRIMARY** as the default but you can always just target a specific backend such as **wgpu::BackendBit::VULKAN** for instance.

### Surface

> Platform specific surface (e.g. window, vk::Surface, texture back buffer, gpu canvas context) onto which a swap chain may receive a window handle and return back the internal surface for working with the next frame.

In `wgpu` this is referred to by creating a [wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html) *unsafe* function **[create_surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Instance.html#method.create_surface)** from a raw window handle.

```rust
// window: winit::window::Window, instance: wgpu::Instance
let surface = unsafe { instance.create_surface(window) };
```

### Adapter

> Handle to physical graphics and/or compute devices in order to open up a connection to the [wgpu::Device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html) on the host system by using [wgpu::Adapter](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html) [request_device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html#method.request_device) function.

Using the adapter will help enumerate the physical devices available from the system and match against power preferences (namely discrete vs integrated) as well as surface compatibility. With [wgpu::PowerPreference::Default](https://docs.rs/wgpu/0.6.0/wgpu/enum.PowerPreference.html) selecting **integrated, discrete, virtual** with preference for integrated on battery power (or otherwise whatever device is available).

```rust
let adapter = instance.request_adapter(
    &wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::Default,
        compatible_surface: Some(&surface)
    }
).await.unwrap();
```

### Logical Device

> Open connection to a graphics and/or compute [wgpu::Device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html). Responsible for the creation of most rendering and compute resources. Rendering and compute resources are used within commands and submitted to a queue. Provides all the necessary core functions of the API - creating graphics data structures like textures, buffers, queues, pipelines, etc. Same across modern graphics APIs, with Vulkan providing additional control over memory, and creating memory structures through a device.

A [wgpu::Device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html) provides all the functions to [create render pipelines](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html), [compute pipelines](https://docs.rs/wgpu/0.6.0/wgpu/struct.ComputePipeline.html), [shader modules](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderModule.html), [buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html), [textures](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html), [samplers](https://docs.rs/wgpu/0.6.0/wgpu/struct.Sampler.html), and [swap chains](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChain.html) to target a surface. Typically, we will use the adapter [request_device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html#method.request_device) to create a connection to a logical device. At the same time we are doing this, we must pass along a [wgpu::DeviceDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.DeviceDescriptor.html) in order to specify the features and limits that the logical device connection should be compatible with. 

Note, that when we request a device at this time from the adapter, we will receive **both** the `wgpu::Device` and the [wgpu::Queue](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html) in the result.

```rust
let (device, queue) = adapter.request_device(
    &wgpu::DeviceDescriptor {
        features: wgpu::Features::empty(),
        limits: wgpu::Limits::default(),
        shader_validation: true
    },
    None
).await.unwrap();
```

### Swap Chain

> Object that effectively flips between different back buffers for a given surface window. Controls aspects of rendering such as refresh rate, back buffer swapping behavior. The swap chain can get the current frame by returning the **next texture** to be presented for drawing. [wgpu::SwapChain](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChain.html) will return a [wgpu::SwapChainFrame](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChainFrame.html) used in render pipeline attachments. When the frame is dropped the swapchain will present the texture to the associated [wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html).

Creating a [wgpu::SwapChain](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChain.html) must be done via [Device::create_swap_chain](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_swap_chain) like any of the other available creation functions on the device. Doing so, the swap chain is described by how it is presented ([wgpu:PresentMode](https://docs.rs/wgpu/0.6.0/wgpu/enum.PresentMode.html)), the width and height (to match the surface), the texture format ([wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/enum.TextureFormat.html)), and texture usage ([wgpu::TextureUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureUsage.html)). 

```rust
// window: winit:window::Window
// let size = window.inner_size();
let sc_desc = wgpu::SwapChainDescriptor {
    usage: wgpu::TextureUsage::OUTPUT_ATTACHMENT,
    format: wgpu::TextureFormat::Bgra8UnormSrgb,
    width: size.width,
    height: size.height,
    present_mode: wgpu::PresentMode::Fifo
};

// device: wgpu::Device, surface: wgpu::Surface, sc_desc: wgpu::SwapChainDescriptor
let swap_chain = device.create_swap_chain(&surface, &sc_desc);
```

> Note: you can take advantage of the winit:event::WindowEvent::Resized/ScaleFactorChanged events 
> to update the current swap chain being used by resizing it according to the new surface size

```rust
// ...
match event {
    Event::RedrawRequested(_) => {
        state.render();
    },
    Event::WindowEvent { event, .. } => match event {
        WindowEvent::Resized(physical_size) => {
            state.resize(physical_size);
        },
        WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {
            state.resize(*new_inner_size);
        },
        _ => {}
    },
    Event::MainEventsCleared => {
        window.request_redraw();
    },
    _ => {}
}
// ...
fn resize(&mut self, new_size: winit::dpi::PhysicalSize<u32>) {
    self.size = new_size;
    self.sc_desc.width = new_size.width;
    self.sc_desc.height = new_size.height;
    self.swap_chain = self.device.create_swap_chain(&self.surface, &self.sc_desc);
}
```

### Command Encoder

> [wgpu::CommandEncoder](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html) records [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html)es, [wgpu::ComputePass](https://docs.rs/wgpu/0.6.0/wgpu/struct.ComputePass.html)es, and transfer operations between driver managed resources like [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html)s and [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html)s. When finished recording, call [CommandEncoder::finish](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.finish) to obtain a [wgpu::CommandBuffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandBuffer.html) which may be submitted for execution to the GPU.

```rust
let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: Some("RENDER ENCODER") // label in graphics debugger
});
```

### Render Pass

> [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html) contains all the drawing methods used to begin recording commands via the [wgpu::CommandEncoder::begin_render_pass](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.begin_render_pass). The [wgpu::RenderPassDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPassDescriptor.html) describes what color and depth attachments to attach to the given render pass. The render pass operates on a texture **attachment** and can provide a load/clear operation to it. This can additionally perform MSAA (multi-sample anti-aliasing) operations through the resolve_target for instance.

Beginning a render pass starts the process of recording operations in the command encoder. When you first create a render pass with a render pass descriptor, you attach the working texture (**attachment**) and a depth stencil attachment with a load operation to inform what the first command of the render pass needs to do with these attachments. 

```rust
let frame = self.swap_chain.get_current_frame()
    .expect("Timeout getting texture")
    .output;

let render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
    color_attachments: &[
        wgpu::RenderPassColorAttachmentDescriptor {
            attachment: &frame.view,
            resolve_target: None,
            ops: wgpu::Operations {
                load: wgpu::LoadOp::Clear(wgpu::Color {
                    r: 0.9, g: 0.2, b: 0.3, a: 1.0
                }),
                store: true
            }
        }
    ],
    depth_stencil_attachment: None
});
```

Many render passes will be used by starting with various color attachments for diffuse, normals, lighting, depth stencils to be used by the entire render pass operations. The API technically can be performed against any texture view (not just a frame texture) and the color attachments technically can be ignored if subsequent render operations are performed against other resources, however since you have to specify a load operation this means the first operation (load) and last operation (store) of the render pass are both set.

The simplest render pass attachment is the above example of just clearing the current frame ([wgpu::SwapChainTexture::view](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChainTexture.html#structfield.view)) texture with a clear color.

Once all operations to the render pass have been performed (whether that's [setting vertex buffers](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_vertex_buffer), [index buffers](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_index_buffer), [bind groups](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_bind_group), [render pipelines](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_pipeline)... etc), execute the [wgpu::CommandEncoder::finish](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.finish) function to submit [wgpu::CommandBuffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandBuffer.html) operations to the [wgpu::Queue](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.submit).

```rust
self.queue.submit(std::iter::once(encoder.finish()));
```

### Example: Clearing the Screen

With just the above concepts alone, we have enough to:

* **Initialize the api** with an instance of wgpu
* **Setup a surface to draw to** with winit window handle
* **Query for a device** with features, limits, power preference and surface compatibility
* **Setup a swap chain** to work with the current frame buffer
* **Begin encoding commands** to start work on render passes
* **Submit our first operation** to clear the screen

This book is designed to help you build off of previous examples and create some sense of reusability. In order to do this, we will need to setup a graphics library that will include the functionality necessary to perform all the event loop handling, render code, and standard setup code.

To initialize a new library project:

```bash
cargo init wgpu-book --lib
```

This will create a directory with a single `src/lib.rs` file, a `Cargo.toml`, and a `Cargo.lock` file. Create a directory to store the examples with the following:

```bash
cd wgpu-book
mkdir -p examples/clear
touch examples/clear/main.rs
```

The examples directory will store each of the examples throughout this book. You can add an example to the project by updating the `Cargo.toml`.

```toml
[package]
name = "wgpu-book"
version = "0.1.0"
authors = ["Thomas Holloway"]
edition = "2018"

[dependencies]
...

[[example]]
name = "clear"
path = "examples/clear/main.rs"
```

> Note: Refer to the previous section on dependencies for the list of versions used within this book. 

Create a new file named `src/app.rs` to store the application builder. This code will include the setup code for running the event loop, creating a window, updating input state, and calls to the setup and rendering operations. We will build on this functionality to be able to pass anonymous functions that can be used to execute render code and setup resources.

In order to separate the examples from the library code, we need a way to pass along functions that the event loop can call when render or update operations need to occur. These "system" functions will be stored on the **App** and passed along and executed at the various stages of the event loop.

```rust
use super::{RenderState, FrameContext};

pub struct App {
    update_systems: Vec<fn(&mut RenderState)>,
    resource_systems: Vec<fn(&mut RenderState)>,
    render_systems: Vec<fn(&mut RenderState, &mut FrameContext)>
}

impl App {
    pub fn build() -> App {
        let update_systems: Vec<fn(&mut RenderState)> = Vec::new();
        let resource_systems: Vec<fn(&mut RenderState)> = Vec::new();
        let render_systems: Vec<fn(&mut RenderState, &mut FrameContext)> = Vec::new();
        App {
            update_systems,
            resource_systems,
            render_systems
        }
    }
}
```

For now, the **RenderState** and the **FrameContext** have not been defined, we will get to those in a moment. Next, to append functions to each of these vectors we need a few builder functions to add systems.

```rust
impl App {
    // pub fn build() -> App {...}

    pub fn add_resource(mut self, f: fn(&mut RenderState)) -> App {
        self.resource_systems.push(f);
        self
    }

    pub fn update_system(mut self, f: fn(&mut RenderState)) -> App {
        self.update_systems.push(f);
        self
    }

    pub fn system(mut self, f: fn(&mut RenderState, &mut FrameContext)) -> App {
        self.render_systems.push(f);
        self
    }
}
```

Now that each of these system vectors have been defined, we can iterate over them in the proper stage of the event loop or run initialization code.

```rust
use futures::executor::block_on;
use winit::event::{Event, VirtualKeyCode, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::{WindowBuilder};

impl App {
    // ...
    pub fn run(self) {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .with_title("Rust by Example: WGPU!")
            .build(&event_loop)
            .unwrap();
```

The first part of the run code will capture the **App** as **self** so that the event loop can run on the main thread. After initializig the event loop, initialize the window with the **WindowBuilder** and pass along any relevant settings such as the title and the event loop.

```rust
        let mut state = block_on(RenderState::new(&window));
```

When we create the **RenderState** we will execute it as an async function. Because we are inside a non-async function, this will need to make use of the **futures** **block_on** to ensure this async function is completed before moving on. Next, we will need to iterate over the resource systems before starting the event loop. The event loop will need to **update input events**, **execute update systems**, **perform any rendering** or **resizing**, and finally **signal a redraw** or **exit**.

```rust
        // setup resources
        for system in &self.resource_systems {
            (system)(&mut state);
        }
```

> Note: the event loop must be executed on the main thread, this is noted by the use of the **move** qualifier to transfer all scope on the main thread to this callback function.

```rust
        event_loop.run(move |event, _, control_flow| {
            if state.input.update(&event) {
                if state.input.key_released(VirtualKeyCode::Escape) || state.input.quit() {
                    *control_flow = ControlFlow::Exit;
                    return;
                }
            }

            // update systems
            for system in &self.update_systems {
                (system)(&mut state);
            }

            match event {
                Event::RedrawRequested(_) => {
                    // render systems
                    state.render(&self.render_systems);
                },
                Event::WindowEvent { event, .. } => match event {
                    WindowEvent::Resized(physical_size) => {
                        state.resize(physical_size);
                    },
                    WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {
                        state.resize(*new_inner_size);
                    },
                    _ => {}
                },
                Event::MainEventsCleared => {
                    window.request_redraw();
                },
                _ => {}
            }
        });
    }
}
```

Note that the app builder code refers to a `RenderState` struct. The render state is used to store all objects that are of concern to the render loop. This includes the device, queue, surface, swap chain, swap chain descriptor, the size of the window, and the current input state. Create a new file `src/render_state.rs`.

```rust
use winit::window::Window;
use winit_input_helper::WinitInputHelper;

pub struct RenderState {
    pub surface: wgpu::Surface,
    pub device: wgpu::Device,
    pub queue: wgpu::Queue,
    pub sc_desc: wgpu::SwapChainDescriptor,
    pub swap_chain: wgpu::SwapChain,
    pub size: winit::dpi::PhysicalSize<u32>
    pub input: WinitInputHelper
}
```

Within the **RenderState** implementation, we need to initialize each of these objects: **wgpu::Instance**, **wgpu::Surface**, **wgpu:Device**, **wgpu::Queue**, **wgpu::SwapChainDescriptor**, **wgpu::SwapChain** and the **WinitInputHelper**.

```rust
impl RenderState {
    pub async fn new(window: &Window) -> Self {
        let input = WinitInputHelper::new();
        let size = window.inner_size();
```

The initial **RenderState::new** function will build an input handler, store the size, and build all the necessary device initialization code mentioned in previous sections. We can later update this to accept varying settings and preferences but for now we will setup a few defaults.

```rust
        let instance = wgpu::Instance::new(wgpu::BackendBit::PRIMARY);
        let surface = unsafe { instance.create_surface(window) };
        let adapter = instance.request_adapter(
            &wgpu::RequestAdapterOptions {
                power_preference: wgpu::PowerPreference::Default,
                compatible_surface: Some(&surface)
            }
        ).await.unwrap();

        let (device, queue) = adapter.request_device(
            &wgpu::DeviceDescriptor {
                features: wgpu::Features::empty(),
                limits: wgpu::Limits::default(),
                shader_validation: true
            },
            None
        ).await.unwrap();

        let sc_desc = wgpu::SwapChainDescriptor {
            usage: wgpu::TextureUsage::OUTPUT_ATTACHMENT,
            format: wgpu::TextureFormat::Bgra8UnormSrgb,
            width: size.width,
            height: size.height,
            present_mode: wgpu::PresentMode::Fifo
        };

        let swap_chain = device.create_swap_chain(&surface, &sc_desc);
```

Once all initialization code has been done to retrieve a device, queue, swap chain, surface, and an instance of the api - we can return all this information into an initialized **RenderState**.

```rust
        Self {
            surface,
            device,
            queue,
            sc_desc,
            swap_chain,
            size,
            input
        }
    }
}
```

Let's add the resize function mentioned previously to handle updating the swap chain descriptor and generate a new swap chain whenever the window is resized.

```rust
impl RenderState {
    // .. new

    pub fn resize(&mut self, new_size: winit::dpi::PhysicalSize<u32>) {
        self.size = new_size;
        self.sc_desc.width = new_size.width;
        self.sc_desc.height = new_size.height;
        self.swap_chain = self.device.create_swap_chain(&self.surface, &self.sc_desc);
    }
}
```

Note that the **App::run** render system functions refer to a **FrameContext**. On each render cycle, we need to use a swap chain frame texture, and a command encoder to encode draw operations. The **FrameContext** will be passed along to each of the render system functions within the **RenderState::render** function.

```rust
pub struct FrameContext {
    pub frame: wgpu::SwapChainTexture,
    pub encoder: wgpu::CommandEncoder
}

impl RenderState {
    // ...
    pub fn render(&mut self, render_systems: &Vec<fn(&mut RenderState, &mut FrameContext)>) {
        let frame = self.swap_chain.get_current_frame()
            .expect("Timeout getting texture")
            .output;

        let encoder = self.device.create_command_encoder(
            &wgpu::CommandEncoderDescriptor {
                label: None,
            }
        );

        let mut context = FrameContext {
            frame,
            encoder
        };

        for system in render_systems {
            (system)(self, &mut context);
        }

        // submit the commands to the queue!
        self.queue.submit(std::iter::once(encoder.finish()));
    }
}
```

Excellent, now we update the `src/lib.rs` to simply export all of these **pub** functions and structs for the library. Open up the `src/lib.rs` and replace the code with a few exports of the library code. This will need to export the **FrameContext**, **App**, **RenderState**.

```rust
mod render_state;
mod app;

pub use render_state::*;
pub use app::*;
```

Now we can update the `examples/clear/main.rs` to build a simple clear example. Many of the examples will create a separate **[example]_system.rs** to demonstrate that you can create multiple systems for the same application run sequence. To initiate a clear, we will need to construct a render pass and load with a clear operation as described in the previous sections. Add a new file `examples/clear/clear_system.rs`.

```rust
use wgpu_book::*;

pub fn clear(_state: &mut RenderState, context: &mut FrameContext) {
    let encoder = &mut context.encoder;
    {
        let _render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            color_attachments: &[
                wgpu::RenderPassColorAttachmentDescriptor {
                    attachment: &context.frame.view,
                    resolve_target: None,
                    ops: wgpu::Operations {
                        load: wgpu::LoadOp::Clear(wgpu::Color {
                            r: 0.1,
                            g: 0.4,
                            b: 0.5,
                            a: 1.0
                        }),
                        store: true
                    }
                }
            ],
            depth_stencil_attachment: None
        });
    }
}
```

Note that the render pass takes advantage of the **FrameContext** to use the current frame and encoder. Now all that is left is to update the **examples/clear/main.rs** to load the **clear_system::clear** function when building the app and call **App::run**.

```rust
mod clear_system;

use wgpu_book::*;

fn main() {
    App::build()
        .system(clear_system::clear)
        .run();
}
```

To run an example (once it has been defined in the **Cargo.toml**) you can execute with:

```bash
cargo run --example clear
```

![Clear Screen](/assets/rust-by-example-wgpu-clear.png)

## Shaders

### Shader Modules

> Shader modules are represented typically by fragment, vertex, and compute shaders and are run against GPU Hardware Abstraction Layers (HAL) through an intermediary representation [SPIR-V](https://www.khronos.org/registry/spir-v/). [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) reflects SPIR-V in order to disassemble back into high level shading languages such as HLSL or Metal Shading Language (MSL).
> 
> GPU instruction sets are typically highly specialized and specific to each GPU version (and in NVIDIA's case under nda). As such, graphics drivers will work closely with system defined hardware abstraction layers, CPU instructions sets and kernel functionality to translate the right amount of work between both the CPU and GPU hardware specific machine code. SPIR-V is the closest thing anyone will come to "GPU assembly" as an intermediary language for the entire graphics stack.
>
> GPU Vendors that support the Vulkan API will load SPIR-V and compile it into GPU specific machine code as it relates to the system being targeted. Within the Microsoft ecosystem, Windows graphics drivers run with D3D11, D3D12, or Vulkan support as vendors implement the HAL apis exposed by the OS and as such SPIR-V can run here as well as loading HLSL or DXIL directly.
>
> Within the Apple ecosystem, SPIR-V must be cross-compiled into Metal Shading Language (MSL) in order to be be run on the Metal platform. SPIRV-Cross is the source of many bugs and much work is being done to reflect any shader language (MSL, HLSL, GLSL versions) back and forth and additionally compile output into rust to abstract away pipeline descriptors and bind groups through the [gfx-rs/naga](https://github.com/gfx-rs/naga) project. 
>
> GLSL comes in many variants, versions, and so it helps that SPIRV-Cross is available to actually cross compile these versions into something of an intermediary representation. Within the Vulkan ecosystem, shaders are written GLSL with [Vulkan Extension for GLSL](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_vulkan_glsl.txt). The extension itself supports OpenGL GLSL versions 1.4 and higher, OpenGL ES ESSL versions 3.1 and higher. It also appears that Vulkan supports HLSL through the integration of a [SPIR-V backend into DXC](https://www.khronos.org/blog/hlsl-first-class-vulkan-shading-language) (Microsoft's open source HLSL compiler) which appears to perform front-end parsing and validation against HLSL and handles according to SPIR-V or DXIL to get HLSL into the right target.

While you can leverage the SPIR-V toolset in the terminal from various shader languages, you can also compile directly from GLSL in rust with [google/shaderc](https://github.com/google/shaderc) using the [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html). The type of shader being loaded must be specified with a [shaderc::ShaderKind](https://docs.rs/shaderc/0.6.2/shaderc/enum.ShaderKind.html). Loading shader modules from SPIR-V bytes can then be done in wgpu with [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module). 

```glsl
// shader.vert
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

Within every shader you will notice that there is an **entry** point function used as a single point of execution. The name of this function, typically **main**, corresponds to the parameter passed to [shaderc::compile_into_spirv](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html#method.compile_into_spirv) as the **entry_point_name**. 

```rust
let vs_src = include_str!("shader.vert");
let mut compiler = shaderc::Compiler::new().unwrap();
let vs_spirv = compiler.compile_into_spirv(
    vs_src, 
    shaderc::ShaderKind::Vertex, 
    "shader.vert", 
    "main", 
    None
).unwrap();
let vs_module = self.device.create_shader_module(
    wgpu::util::make_spirv(&vs_spirv.as_binary_u8())
);
```

You can also skip the compilation step if it's already been compiled into spirv and load the raw bytes to directly create the shader module. In either scenario, calling [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module) will return a [wgpu::ShaderModule](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderModule.html) which references a [ShaderModuleId](https://docs.rs/wgpu/0.6.0/src/wgpu/lib.rs.html#634) where the shader module will be referenced when executing it on a render pipeline.

> Note: wgpu provides a macro for including spir-v directly with [wgpu::include_spirv!](https://docs.rs/wgpu/0.6.0/wgpu/macro.include_spirv.html) as in `wgpu::include_spirv("shader.vert.spv")`. To do this, you will need to add the build step to ensure that all required shader dependencies in the form of **.vert**, **.frag** or **.comp** are compiled with the [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html).

### Render Pipeline

Executing the shader module requires the use of a [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html) as created with [wgpu::Device::create_render_pipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_render_pipeline) and passing along a [wgpu::RenderPipelineDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipelineDescriptor.html) to describe all the different programmable stages, rasterization process, primitive topology, color states, depth stencil states, vertex states, and sampling settings.

```rust
let render_pipeline_layout = self.device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
    label: Some("pipeline layout"), // debug label
    bind_group_layouts: &[],
    push_constant_ranges: &[]
});

let render_pipeline = self.device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
    label: Some("pipeline"),
    layout: Some(&render_pipeline_layout),
    vertex_stage: wgpu::ProgrammableStageDescriptor {
        module: &vs_module,
        entry_point: "main"
    },
    fragment_stage: wgpu::ProgrammableStageDescriptor {
        module: &fs_module,
        entry_point: "main"
    }),
    // ...
});
```

In the above pipeline descriptor, the first two things you will notice are the **vertex_stage** and **fragment_stage** designated as [wgpu::ProgrammableStageDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.ProgrammableStageDescriptor.html). There is nothing particularly out of the ordinary here, as these properties are currently the only programmable stages that can provide a shader module.

![Rendering Pipeline](/assets/wgpu-rendering-pipeline.png)

Within the rendering pipeline inputs are assembled and prepared from memory to be executed against a vertex shader function. Each vertex shader receives a single vertex composed of a series of vertex attributes. Following this stage is a fixed function unit that converts vector primitives (points, lines, triangles) into grid-aligned fragments. Finally, a fragment shader will receive these individual fragment samples and produce a single fragment as an output with color, depth and possible stencil values.

> Note: traditionally, this pipeline included tesselation and geometry shading as well. However, both of these additional stages are either not supported on mobile platforms and vary in support across desktops. Additionally, these type of shaders are typically now done through the use of compute shaders. Compute kernels that generate the effects of tesselation and geometry have a much better throughput performance and provide additional flexibility given that they can compute anything that can be later used through a simple passthrough rendering pipeline.

Extending the existing code above (in addition to the specified **vertex_stage** and **fragment_stage**) we can set a few more properties such as:

* **[wgpu::PrimitiveTopology](https://docs.rs/wgpu/0.6.0/wgpu/enum.PrimitiveTopology.html)** specifying the behavior of each vertex as

    * **PointList**: vertex data is a list of points, each vertex is a new point
    * **LineList**: vertex data is a list of lines. Each pair of vertices composes a new line
        Vertices `0 1 2 3` creates two lines `0 1` and `2 3`
    * **LineStrip**: vertex data is a strip of lines, each set of two adjacent vertices form a line
        Vertices `0 1 2 3` create three lines `0 1`, `1 2`, and `2 3`
    * **TriangleList**: vertex data is a list of triangles, each set of 3 vertices composes a new triangle
        Vertices `0 1 2 3 4 5` creates two triangles `0 1 2` and `3 4 5`
    * **TriangleStrip**: vertex data is a triangle strip, each set of three adjacent vertices form a triangle
        Vertices `0 1 2 3 4 5` creates four triangles `0 1 2`, `2 1 3`, `3 2 4`, and `4 3 5`

    ```rust
        // ...
        primitive_topology: wgpu::PrimitiveTopology::TriangleList,
        // ...
    ```

* **[wgpu::ColorStateDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.ColorStateDescriptor.html)** controlling the color aspect of the output target.

    This includes properties such as [alpha_blend](https://docs.rs/wgpu/0.6.0/wgpu/struct.ColorStateDescriptor.html#structfield.alpha_blend) and [color blend](https://docs.rs/wgpu/0.6.0/wgpu/struct.ColorStateDescriptor.html#structfield.color_blend) used for the pipeline. Note that the color state descriptor includes a [wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/struct.ColorStateDescriptor.html#structfield.format) that must also match the format of the corresponding color attachment in the command encoder's initial **begin_render_pass**.

    ```rust
        // ...
        color_states: &[
            wgpu::ColorStateDescriptor {
                format: sc_desc.format,
                color_blend: wgpu::BlendDescriptor::REPLACE,
                alpha_blend: wgpu::BlendDescriptor::REPLACE,
                write_mask: wgpu::ColorWrite::ALL
            }
        ],
        // ...
    ```

* **[wgpu::DepthStencilStateDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.DepthStencilStateDescriptor.html)** describes the depth and stencil state

    Similar to the **color_states**, the **depth_stencil_state** on the render pipeline descriptor has a [wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/struct.DepthStencilStateDescriptor.html#structfield.format) - in this case matching the format of the depth/stencil attachment in the command encoder's **begin_render_pass**. Additionally, this descriptor includes a comparison function used to compare depth values in a depth test through [wgpu::CompareFunction](https://docs.rs/wgpu/0.6.0/wgpu/struct.DepthStencilStateDescriptor.html#structfield.depth_compare).

    For our purposes, since we have no depth stencil attachment in our code at the moment, this will be set to **None**.

    ```rust
        // ...
        depth_stencil_state: None
        // ...
    ```

* **[wgpu::VertexStateDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexStateDescriptor.html)** describes the vertex input state which may include any [wgpu::VertexBufferDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexBufferDescriptor.html) formats and index formats associated with pipeline.

    Since we don't have any associated vertex buffers in our code yet, this state descriptor will be mostly empty.
    ```rust
        // ...
        vertex_state: wgpu::VertexStateDescriptor {
            index_format: wgpu::IndexFormat::Uint16,
            vertex_buffers: &[]
        }
        // ...
    ```

* **sample_mask** / **sample_count** describes the sampling settings in the pipeline

    The last several properties of the render pipeline describe how sampling works such as in **sample_count** (number of samples calculated per pixel for MSAA, 1 in non-multisampled textures), **sample_mask** (bitmask to restrict samples of a pixel, !0 for all samples to be enabled), and **alpha_to_coverage_enabled** (producing a sample mask per pixel based on alpha)

    ```rust
        // ...
        sample_count: 1,
        sample_mask: !0,
        alpha_to_coverage_enabled: false
        // ...
    ```

Once a render pipeline has been setup, we can revisit the [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html) to set the currently active render pipeline (possibly even one among many pipeline layouts that we are describing within our system) This is done with the [wgpu::RenderPass::set_pipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_pipeline). All draw operations such as used by [wgpu::RenderPass::draw](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.draw) (passing along vertices, and instances) will follow the behavior defined by the active [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html). 

```rust
render_pass.set_pipeline(&self.render_pipeline);
```

### Draw Operations

In order to execute a render pipeline, according to its pipeline layout, we must call draw operations on that render pass. Draw operations vary with the simplest being [wgpu::RenderPass::draw](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.draw) which takes a range of vertices and range of instances. Other operations like [wgpu::RenderPass::draw_indexed](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.draw_indexed) take a range of indices, base vertex, and a range of instances that will draw indexed primitives using the active index buffer and active vertex buffers. It's important to note that **any** of the draw operations work according to what is currently active on the render pass. This includes:

* **set_pipeline**: [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.set_pipeline) where subsequent draw operations use the behavior defined by the pipeline
* **set_index_buffer**: [wgpu::BufferSlice](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.set_index_buffer) which sets the active index buffer where *draw_indexed* operations use the source index buffer
* **set_vertex_buffer**: [wgpu::BufferSlice](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.set_vertex_buffer) which sets the active vertex buffer where *draw*, *draw_indexed*, use vertex buffers set by this
* **set_bind_group**: [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.set_bind_group) which sets the active bind group according to the layout in an active pipeline (used by daw operations)

For example, calling the **[wgpu::RenderPass::draw](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.draw)** requires a range of vertices and range of instances. In the simplest example of a pass through vertex shader, we can define the vertices manually in the shader itself.

```glsl
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

In the above vertex shader example, we are making use of two reserved variables **gl_Position** and **gl_VertexIndex**. There are other global variables such as **gl_InstanceIndex**. In a later section, when we discuss [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) we will get into how attributes are defined and how a [wgpu::VertexAttributeDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html) instructs the pipeline how to get buffer data into the vertex stage. In the above example, however, we are defining the vertices as a constant triangle constructed from 3 vec2 positions. We can call the **draw** function to indicate we would like all 3 of these vertices and 1 instance to be passed through the vertex stage. The vertex shader will be executed for each vertex index and for each instance of those vertices.

```rust
render_pass.draw(0..3, 0..1);
```

### Shader Layout

With the [Vulkan Extension](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_vulkan_glsl.txt) to GLSL, and subsequently compiling GLSL to SPIR-V intermediate language, Vulkan adds a number of concepts to help with ways to describe the input/output/binding layouts of a shader. This comes in the form of [Descriptor Sets](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_vulkan_glsl.txt#L221) and provides qualifiers according to how the API will bind in data to mimic the pipeline layout described through the Vulkan API.

Recall the implementation for a simple static triangle vertex shader below:

```glsl
// shader.vert
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

Let's add to this example pipeline by including a fragment shader that outputs a single color for all pixels in the generated geometry.

```glsl
// shader.frag
#version 450

layout(location=0) out vec4 f_color;

void main() {
    f_color = vec4(0.1, 0.2, 0.4, 1.0);
}
```

Note the **layout(location=0) out vec4 f_color** line of code. Without the Vulkan Extension to GLSL, an OpenGL GLSL shader may simply set the variable **gl_FragColor**. This implies an [implicit broadcast](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_vulkan_glsl.txt#L537) to all outputs. According to the docs, this broadcast is not available in SPIR-V and as such it is usually better to write it in the form of a variable being assigned that has the *same type* as **gl_FragColor** and set to the **location=0**. This leads us to describing what exactly an *input* and *output* variable is in Vulkan's GLSL.

In order to compile GLSL into SPIR-V a layout must be specified for each input and output. This takes the format of:

```glsl
layout (location = INDEX) DIRECTION TYPE NAME;
```

* **INDEX** is an integer value that must match up with the locations for binding data into GLSL. Within **wgpu** or **gfx-rs** this depends on which part of the pipeline is being described. 

    > **Vertex** data is described with a [wgpu::VertexBufferDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexBufferDescriptor.html) with the [wgpu::VertexAttributeDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html) as the attributes for vertex inputs. For each attribute, you have a **[shader_location](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html#structfield.shader_location)** that must match the location **INDEX** shown above.

    > **Fragment** shaders with `layout(location = 0)` for instance specifies that the values of an assigned variable would be saved to the buffer assigned at location 0 in the application. Usually this is the current texture from the swapchain.

    Between the shaders in a pipeline the locations and variable outputs from one stage to another need to match the locations and names of the next stage. This would mean that `layout(location=0) out vec3 v_color` in the **shader.vert** would need to match `layout(location=0) in vec3 v_color` in a **shader.frag**.

* **DIRECTION** specifies either `in` or `out`, or `uniform`. The `in` variable specifies that the value comes from the previous stage (such as a single vertex attribute value in the vertex shader or the `v_color` in the fragment shader. The `out` variable goes onto the next stage of the pipeline. `uniform` is a read-only keyword typically used in combination with more complex bindings.

* **TYPE** is a variable type (such as `float`, `int`, `uint`, `vec3`, ..etc)

* **NAME** is the name of the variable

In later sections, we will discuss the use of **set**, **binding** and **push_constants** as they relate to more complex pipeline layout descriptors. For now, we will worry about the most basic set descriptor for passing variables between stages of the pipeline.

### Example: Draw a Triangle

Before we move onto to more complex binding layouts, vertex buffers, textures, and samplers - with the above concepts alone we have enough to execute the most basic shader - **drawing a triangle**. 

* **Write a vertex shader** with a *const* array of vertices describing a static triangle
* **Write a fragment shader** to output a color for the geometry
* **Compile/load shader** into spir-v and create a shader module
* **Setup render pipeline** to load the shaders as programmable stages
* **Assign the render pipeline** by setting the render pipeline on the first render pass
* **Call draw on the render pass** by passing along the vertices and instance(s)

The vertex shader will be the same one we've seen in the previous sections. Namely, it will be a constant set of vertices that describe our triangle. Shaders will need to be compiled and loaded as a resource, so we will take advantage of the **App:resource_systems** defined in the previous library example code. To start off, let's create a new example for triangle with:

```bash
mkdir -p examples/triangle
touch examples/triangle/main.rs
touch examples/triangle/triangle_system.rs
touch examples/triangle/triangle.vert
touch examples/triangle/triangle.frag
```

Then update the `Cargo.toml` to include this new example so that it can be executed.

```toml
[[example]]
name = "triangle"
path = "examples/triangle/main.rs"
```

In a file called `examples/triangle/triangle.vert` add the following code:

```glsl
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

We make use of the **gl_VertexIndex** to select the right **vec2** in the **positions** array, then output the result to **gl_Position**.

Next, the fragment shader will be the same as in the previous sections. It simply outputs the same color to the `location=0` output variable. As mentioned previously, this usually refers to the passed in current swap chain texture attached at the beginning of our render pass. Add this code to `examples/triangle/triangle.frag` file.

```glsl
#version 450

layout(location=0) out vec4 f_color;

void main() {
    f_color = vec4(0.1, 0.2, 0.4, 1.0);
}
```

Recall that our previous code for setting up the state included creating everything from the instance, adapter, device, swap chain descriptor, and all the code from the winit event handling. In order to load resources that will be accessible from the render system functions, we need a way to store all these resources into a hash map that can later be refered to by key. Let's expand the **RenderState** to include these **std::collections::HashMap** of resources to store both the pipeline and pipeline layouts. 

```rust
use std::collections::HashMap;

use winit::window::Window;
use winit_input_helper::WinitInputHelper;

pub struct RenderState {
    pub surface: wgpu::Surface,
    pub device: wgpu::Device,
    pub queue: wgpu::Queue,
    pub sc_desc: wgpu::SwapChainDescriptor,
    pub swap_chain: wgpu::SwapChain,
    pub size: winit::dpi::PhysicalSize<u32>,
    pub input: WinitInputHelper,
    pub pipelines: HashMap<String, wgpu::RenderPipeline>,
    pub pipeline_layouts: HashMap<String, wgpu::PipelineLayout>,
    pub compiler: shaderc::Compiler,
}
```

In addition to the render pipelines and pipeline layouts, we will also store a **shaderrc::Compiler** so we can use that to compile shader source code into a **wgpu::ShaderModule**. Let's update the **new** function to initialize these hashmaps and the shader compiler now.

> Note: we can optionally compile the shader source ahead of time as **.spirv** files and simply include them as shader modules instead of compiling them at runtime. For now, let's just include this here for the sake of convenience.

```rust
impl RenderState {
    pub async fn new(window: &Window) -> Self {
        // .. input, instance, size, device, queue...etc

        let pipelines: HashMap<String, wgpu::RenderPipeline> = HashMap::new();
        let pipeline_layouts: HashMap<String, wgpu::PipelineLayout> = HashMap::new();
        let compiler = shaderc::Compiler::new().unwrap();

        Self {
            surface,
            device,
            queue,
            sc_desc,
            swap_chain,
            size,
            input,
            pipelines,
            pipeline_layouts,
            compiler
        }
    }
    // ..
```

The render state will store an instance of the [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html). We can use this to create a small utility function to create a [wgpu::ShaderModule](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderModule.html) with [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module).

```rust
impl RenderState {
    // ...
    pub fn compile_shader(&mut self, kind: shaderc::ShaderKind, src: &str, name: &str) -> wgpu::ShaderModule {
        let spirv = self.compiler.compile_into_spirv(
            src,
            kind,
            name,
            "main",
            None
        ).unwrap();
        self.device.create_shader_module(
            wgpu::util::make_spirv(&spirv.as_binary_u8())
        )
    }
}
```

We have updated the library to handle compiling shaders as well as storing [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html) and [wgpu::PipelineLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.PipelineLayout.html) within resource associated hashmaps. The triangle example needs to create these resources during the initialization setup code as a **resource_system**. Within the **examples/triangle/triangle_system.rs** add a function to load and compile these shader sources.

```rust
use wgpu_book::*;

pub fn load_shaders(state: &mut RenderState) {
    let vs_module = state.compile_shader(
        shaderc::ShaderKind::Vertex,
        include_str!("triangle.vert"),
        "triangle.vert"
    );

    let fs_module = state.compile_shader(
        shaderc::ShaderKind::Fragment,
        include_str!("triangle.frag"),
        "triangle.frag"
    );
```

After creating the shader modules, a [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html) must be created as described in the previous section using [wgpu::Device::create_render_pipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_render_pipeline).

```rust
    let layout = state.device.create_pipeline_layout(
        &wgpu::PipelineLayoutDescriptor {
            label: None,
            bind_group_layouts: &[],
            push_constant_ranges: &[]
        }
    );
```

The render pipeline layout does not contain anything notable at this point. We are not using any bind group layouts and we are not yet passing along any push constant ranges with it. We will simply pass along this simple layout to the render pipeline creation.

```rust
    let pipeline = state.device.create_render_pipeline(
        &wgpu::RenderPipelineDescriptor {
            label: None,
            layout: Some(&render_pipeline_layout),
            vertex_stage: wgpu::ProgrammableStageDescriptor {
                module: &vs_module,
                entry_point: "main"
            },
            fragment_stage: Some(wgpu::ProgrammableStageDescriptor {
                module: &fs_module,
                entry_point: "main"
            }),
            rasterization_state: Some(
                wgpu::RasterizationStateDescriptor {
                    front_face: wgpu::FrontFace::Ccw,
                    cull_mode: wgpu::CullMode::Back,
                    depth_bias: 0,
                    depth_bias_slope_scale: 0.0,
                    depth_bias_clamp: 0.0,
                    clamp_depth: false
                } 
            ),
            color_states: &[
                wgpu::ColorStateDescriptor {
                    format: sc_desc.format,
                    color_blend: wgpu::BlendDescriptor::REPLACE,
                    alpha_blend: wgpu::BlendDescriptor::REPLACE,
                    write_mask: wgpu::ColorWrite::ALL
                }
            ],
            primitive_topology: wgpu::PrimitiveTopology::TriangleList,
            depth_stencil_state: None,
            vertex_state: wgpu::VertexStateDescriptor {
                index_format: wgpu::IndexFormat::Uint16,
                vertex_buffers: &[]
            },
            sample_count: 1,
            sample_mask: !0,
            alpha_to_coverage_enabled: false
        }
    );
```

> Note: the render pipeline parameters here largely are all defaults. Specifically, we would like the rasterization to use a **Ccw** (counter clock-wise) rendering of primitives and culling based on the back fact of primitives. The color states will primarily replace colors and alpha blending. Finally, the primitive topology will be a triangle list by default.

Additionally, since the **load_shaders** function has access to the current **RenderState**, it can insert the pipeline, and pipeline layouts to the state. 

```rust
    state.pipelines.insert(String::from("triangle"), pipeline);
    state.pipeline_layouts.insert(String::from("triangle"), layout);
}
```

Another **render_system** function can then refer to the `String::from("triangle")` resources inserted when executing a render pass to **draw** the vertices of the triangle.

```rust
use wgpu_book::*;

pub fn load_shaders(state: &mut RenderState) {...}

pub fn triangle(state: &mut RenderState, context: &mut FrameContext) {
    let resource = String::from("triangle");
    let encoder = &mut context.encoder;
    {
        let mut render_pass = encoder.begin_render_pass(
            &wgpu::RenderPassDescriptor {
                color_attachments: &[
                    wgpu::RenderPassColorAttachmentDescriptor {
                        attachment: &context.frame.view,
                        resolve_target: None,
                        ops: wgpu::Operations {
                            load: wgpu::LoadOp::Clear(wgpu::Color {
                                r: 0.0,
                                g: 0.0,
                                b: 0.0,
                                a: 1.0
                            }),
                            store: true
                        }
                    }
                ],
                depth_stencil_attachment: None
            }
        );

        render_pass.set_pipeline(&state.pipelines.get(&resource).unwrap());
        render_pass.draw(0..3, 0..1);
    }
}
```

The last step needed is to build the **App** with these two system functions.

```rust
mod triangle_system;

use wgpu_book::*;

fn main() {
    App::build()
        .add_resource(triangle_system::load_shaders)
        .system(triangle_system::triangle)
        .run();
}
```

```bash
cargo run --example triangle
```

![Draw Triangle Shader](/assets/wgpu-draw-triangle-shader.png)

## Shader Uniforms

### GLSL Uniforms

> A uniform is a global shader variable declared with the **uniform** qualifier. These are parameters that must be passed into the shader program during the pipeline execution. A uniform value does not change from one shader invocation to the next within a rendering call so they remain *uniform* among all invocations.

Passing data into a shader can be done in one of several ways. You can specify data between shader stages such as assigning output and input bindings from one stage to the next. Shaders, such as vertex shaders, can receive vertex attributes assigned in from vertex buffer objects. Or you can assign in **uniform** buffer data where the uniform value is the same for every execution of that (and in some cases all) stage(s). 

Since a uniform is simply a qualifier, the actual type associated with the value can be of any type, or any aggregation of types such as *bool*, *int*, *uint*, *float*, *double*, *vecN* vectors, *matN* and *matnxm* matrices, samplers, textures and more. 

```glsl
#version 450
layout(set=0, binding=0) uniform texture2D tex;
layout(set=0, binding=1) uniform sampler samp;

layout(location = 1) in vec2 frag_uv;

layout(location = 0) out vec4 color;

void main() {
    color = texture(sample2D(tex, samp), frag_uv);
}
```

Note the use of **set** and **binding** in the above example. The set identifier specifies the **descriptor set** the object belongs to. Within WGPU, descriptor sets are setup using a [wgpu::BindGroupLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayout.html) and are passed along to the [wgpu::PipelineLayoutDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.PipelineLayoutDescriptor.html#structfield.bind_group_layouts). Pipeline layouts, as mentioned in a previous section, are setup at the start of the program right after initialization with the [wgpu::Device::create_pipeline_layout](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_pipeline_layout). 

> We will go over this part in more detail in a later section specifically on bind groups and layouts. Just note that the **set** qualifier is an index we will pass along at the time when a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html) is actually set on a render pass through [wgpu::RenderPass::set_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_bind_group). While the **binding** qualifier must match the [wgpu::BindGroupEntry::binding](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupEntry.html#structfield.binding), an object that is created whenever we are setting up the bind group layout itself from [wgpu::Device::create_bind_group_layout](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group_layout). 

In order to pass long any resources (whether that's a texture, an image, vectors, matrices, buffer data), each resource must be described in a **descriptor set**. The assignment is a **layout** of (set number, binding number, array element) that defines its location. The **set** number and **binding** number are explicitly provided within the **layout** qualifier as shown in the previous example. The array element is implicitly assigned for each uniform variable specified in the shader.

```glsl
#version 450
layout(location=0) out vec4 color;
layout(push_constant) uniform blockName {
    float u_time;
}

void main() {
    color = vec4(abs(sin(u_time)), abs(cos(u_time)), 1.0, 1.0);
}
```

In addition to uniform layout bindings through descriptor sets, you can also specify a uniform block that uses the **push_constant** layout qualifier. Native platforms such as Metal, Vulkan, D3DX, provide a small reserved buffer of memory that can be used to pass along small constants and variables. These variables can be typically updated frequently and can be passed along to to the pipeline without needing to setup an additional binding layout or allocate an intermediary buffer object. Push constants, in Vulkan for instance set this small reserved buffer of memory to have a maximum size (in bytes) has a default limit set to 128 bytes.

> In Metal, this is done through [SetBytes](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder/1443159-setbytes), [SetVertexBytes](https://developer.apple.com/documentation/metal/mtlrendercommandencoder/1515846-setvertexbytes), and [SetFragmentBytes](https://developer.apple.com/documentation/metal/mtlrendercommandencoder/1516192-setfragmentbytes?language=objc). Metal requests the data buffers set by these implementations to be **less than 4KB** and must be one-time-use data buffers in each render.

> In D3D12, this kind of constant is done through the [Root Constants](https://docs.microsoft.com/en-us/windows/desktop/direct3d12/root-signatures-overview) feature and applications can define each as a set of 32-bit values. Typically root constants will be used in scenarios such as [dynamic indexing](https://github.com/Microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12DynamicIndexing). D3D12 has limits such as the maximum size of a root signature at 64 DWORDS, with root constants costing 1 DWORD each.

> WebGPU however, does not appear to support push constants, and the outlook for them doesn't look promising in terms of implementation any time soon. At the time of this writing, a proposal was in place but the WebGPU group at the moment has not decided on including it into the spec ([issue #75](https://github.com/gpuweb/gpuweb/issues/75)).

In addition to the uniform block with a **push_constant** layout qualifier, you can also specify a uniform block with the **set** and **binding** layout qualifiers that were previously used to assign uniform constants.

```glsl
#version 450

layout(location=0) in vec3 pos;

layout(set=0, binding=0) uniform UpdateBlock {
    mat4 cam_view_proj;
    float u_time;
}

void main() {
    gl_Position = cam_view_proj * vec4(pos, 1.0);
}
```

Layout blocks can also include an **INSTANCE_NAME** at the end of the block so that variables can be accessed with the *dot notation*.

```glsl
#version 450

layout(location=0) in vec3 pos;

layout(set=0, binding=0) uniform UpdateBlock {
    mat4 cam_view_proj;
    float u_time;
} updates;

void main() {
    gl_Position = updates.cam_view_proj * vec4(pos, 1.0);
}
```

### Push Constants

As noted in the previous section, one method of assigning **uniform** data into a shader is to use a concept known as **push constants**. While, this method is not supported on WebGPU (for instance), it is a method that is used quite regularly to pass along small variables or indices used to describe dynamic offsets for memory backed resources. WGPU provides this native extension through the use of the [wgpu::Features::PUSH_CONSTANTS](https://docs.rs/wgpu/0.6.0/wgpu/struct.Features.html#associatedconstant.PUSH_CONSTANTS) feature that can setup at the time when device initialization occurs.

```rust
let (device, queue) = adapter.request_device(
    &wgpu::DeviceDescriptor {
        features: wgpu::Features::PUSH_CONSTANTS,
        limits: wgpu::Limits {
            max_push_constant_size: 128,
            ..wgpu::Limits::default()
        },
        shader_validation: true
    },
    None
).await.unwrap();
```

When you specify the additional [wgpu::Features::PUSH_CONSTANTS](https://docs.rs/wgpu/0.6.0/wgpu/struct.Features.html#associatedconstant.PUSH_CONSTANTS) at the time of requesting a logical device, you must also specify the **max_push_constant_size**. The [max_push_constant_size](https://docs.rs/wgpu/0.6.0/wgpu/struct.Limits.html#structfield.max_push_constant_size) available depends on the target platform such as 128-256 bytes for Vulkan, 256 bytes for D3D12, Metal 4KB.

```rust
let render_pipeline_layout = device.create_pipeline_layout(
    &wgpu::PipelineLayoutDescriptor {
        label: Some("pipeline layout"),
        bind_group_layouts: &[],
        push_constant_ranges: &[wgpu::PushConstantRange {
            stages: wgpu::ShaderStage::FRAGMENT,
            range: 0..4
        }]
    }
);
```

After specifying the push constants feature and max push constant size, for each render pipeline layout you must specify the range of constants that the pipeline will use. This is specified through the [wgpu::PushConstantRange](https://docs.rs/wgpu/0.6.0/wgpu/struct.PipelineLayoutDescriptor.html#structfield.push_constant_ranges). Each stage that uses push constants must define the range in push constant memory that corresponds to a single **layout(push_constant) uniform BLOCK {}**.

Finally, when you are ready to pass along these variables to the shader stage, call the [wgpu::RenderPass::set_push_constants](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_push_constants) - passing along the offset, [wgpu::ShaderStage](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderStage.html) and byte data.

```rust
render_pass.set_push_constants(
    wgpu::ShaderStage::FRAGMENT, 
    0, 
    bytemuck::cast_slice(&[self.u_time])
);
```

### Example: Set current timestep

Using the **push_constant uniform** block and the [wgpu::Features::PUSH_CONSTANTS](https://docs.rs/wgpu/0.6.0/wgpu/struct.Features.html#associatedconstant.PUSH_CONSTANTS) feature in wgpu, we can create a simple example that passes along the current timestep and manipulate the fragment shader color in the process. This builds off of the same triangle example in a previous section.

To setup another example to demonstrate how to work with push constants, create a new examples directory and import that into `Cargo.toml`.

```bash
mkdir -p examples/constants
touch examples/constants/main.rs
touch examples/constants/constants_system.rs
touch examples/constants/constants.vert
touch examples/constants/constants.frag
```

Import the **constants** example in the `Cargo.toml` so that it can be run.

```toml
[[example]]
name = "constants"
path = "examples/constants/main.rs"
```

Recall the implementation for the vertex shader in the previous example on drawing a triangle. The constants we are using in this example will not effect the vertex shader so we can copy that code over into **examples/constants/constants.vert**.

```glsl
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

In the fragment shader (**examples/constants/constants.frag**), we will update it to include a new **layout(push_constant) uniform** block to allow us to specify a simple timestep float.

```glsl
#version 450

layout(location=0) out vec4 f_color;
layout(push_constant) uniform Uniforms {
    float u_time;
};

void main() {
    f_color = vec4(abs(sin(u_time)), abs(cos(u_time)), 0.4, 1.0);
}
```

This fragment shader makes use of one of the many built-in functions in GLSL. Namely, [cos()](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/cos.xhtml), [sin()](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/sin.xhtml), and [abs()](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/abs.xhtml) functions. Next, in our render pipeline - after loading these shader modules - we will make sure to specify that we want this **uniform block** to be available to the fragment shader and the bytes covered will be approximately **0..4** (matching a 32-bit float type: **f32**). Calling [wgpu::Device::create_pipeline_layout](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_pipeline_layout) and specifying a [wgpu::PipelineLayoutDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.PipelineLayoutDescriptor.html#structfield.push_constant_ranges) to include our [wgpu::PushConstantRange](https://docs.rs/wgpu/0.6.0/wgpu/struct.PushConstantRange.html) will do this.

> Note: Since the render pipeline will look largely the same as the triangle example, we can create a utility function to create a **wgpu::RenderPipeline** with all the default values we are interested in.

```rust
pub struct Pipeline<'a> {
    pub vs_module: wgpu::ShaderModule,
    pub fs_module: wgpu::ShaderModule,
    pub layout: &'a wgpu::PipelineLayout
}
```

We can create a simple struct to describe a pipeline by including the programmable shader modules and the initial **wgpu::PipelineLayout** reference. This will be helpful later when we need to additionally include other settings such as blend modes, vertex buffers, and rasterization settings other than the defaults.

```rust

impl RenderState {
    // ...
    pub fn create_render_pipeline(&mut self, pipeline: Pipeline) -> wgpu::RenderPipeline {
        self.device.create_render_pipeline(
            &wgpu::RenderPipelineDescriptor {
                label: None,
                layout: Some(&pipeline.layout),
                vertex_stage: wgpu::ProgrammableStageDescriptor {
                    module: &pipeline.vs_module,
                    entry_point: "main"
                },
                fragment_stage: Some(wgpu::ProgrammableStageDescriptor {
                    module: &pipeline.fs_module,
                    entry_point: "main"
                }),
                rasterization_state: None,
                color_states: &[
                    wgpu::ColorStateDescriptor {
                        format: self.sc_desc.format,
                        color_blend: wgpu::BlendDescriptor::REPLACE,
                        alpha_blend: wgpu::BlendDescriptor::REPLACE,
                        write_mask: wgpu::ColorWrite::ALL
                    }
                ],
                primitive_topology: wgpu::PrimitiveTopology::TriangleList,
                depth_stencil_state: None,
                vertex_state: wgpu::VertexStateDescriptor {
                    index_format: wgpu::IndexFormat::Uint16,
                    vertex_buffers: &[]
                },
                sample_count: 1,
                sample_mask: !0,
                alpha_to_coverage_enabled: false
            }
        )
    }
}
```

However, in order to use any push constants within a render pipeline (as specified in a pipeline layout), the device must be initialized to require the [wgpu::Features::PUSH_CONSTANTS](https://docs.rs/wgpu/0.6.0/wgpu/struct.Features.html#associatedconstant.PUSH_CONSTANTS) feature. Within the **new** function during initialization, we need to update the logical device to include this push constants feature as well as specify the maximum push constant size in bytes.

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ... size, instance, surface, adapter
        let (device, queue) = adapter.request_device(
            &wgpu::DeviceDescriptor {
                features: wgpu::Features::PUSH_CONSTANTS,
                limits: wgpu::Limits {
                    max_push_constant_size: 128,
                    ..wgpu::Limits::default()
                },
                shader_validation: true
            },
            None
        ).await.unwrap();
        // ...
    }
    // ..resize, render
}
```


Then in **constants_system.rs**, we can proceed to compile and load the shaders. Note that that the pipeline layout has changed to now include the push constant range for the fragment shader. The number of bytes specified will cover a single timestamp **f32**, or **4 bytes**.

```rust
// examples/constants/constants_system.rs
use wgpu_book::*;

pub fn load_shaders(state: &mut RenderState) {
    let vs_module = state.compile_shader(
        shaderc::ShaderKind::Vertex,
        include_str!("constants.vert"),
        "constants.vert"
    );

    let fs_module = state.compile_shader(
        shaderc::ShaderKind::Fragment,
        include_str!("constants.frag"),
        "constants.frag"
    );

    let layout = state.device.create_pipeline_layout(
        &wgpu::PipelineLayoutDescriptor {
            label: None,
            bind_group_layouts: &[],
            push_constant_ranges: &[
                wgpu::PushConstantRange {
                    stages: wgpu::ShaderStage::FRAGMENT,
                    range: 0..4
                }
            ]
        }
    );

    let pipeline = state.create_render_pipeline(Pipeline {
        vs_module,
        fs_module,
        layout: &layout
    });

    state.pipelines.insert(String::from("constants"), pipeline);
    state.pipeline_layouts.insert(String::from("constants"), layout);
}
```

Unfortunately, we do not yet have a way to store additional state resources in the **RenderState** or the application. Additional state objects may include entities, the current timestamp, or any number of variables that we are planning on updating with each iteration of the render loop. We can add this functionality to the **RenderState** by including a hash map of types to values.

```rust
use std::collections::HashMap;
use std::any::{Any, TypeId};

pub struct RenderState {
    pub surface: wgpu::Surface,
    pub device: wgpu::Device,
    pub queue: wgpu::Queue,
    pub sc_desc: wgpu::SwapChainDescriptor,
    pub swap_chain: wgpu::SwapChain,
    pub size: winit::dpi::PhysicalSize<u32>,
    pub input: WinitInputHelper,
    pub pipelines: HashMap<String, wgpu::RenderPipeline>,
    pub pipeline_layouts: HashMap<String, wgpu::PipelineLayout>,
    pub compiler: shaderc::Compiler,
    pub u_time: f32,
    pub resources: HashMap<TypeId, Box<dyn Any>>
}
```

The hash map stores a map of types to values where the values can be of **any** type. In this most basic implementation we can simple use a few generic functions to set and retrieve values mutably or by reference. There are a few edge cases here where the value may not necessarily exist, but for the time being we will simply unwrap to simplify things.

```rust
impl RenderState {
    // ...
    pub fn add_resource<T>(&mut self, value: T)
        where T : 'static {
        self.resources.insert(TypeId::of::<T>(), Box::new(value));
    }

    pub fn resource<T>(&self) -> &T
        where T : 'static {
        let val = self.resources.get(&TypeId::of::<T>());
        val.unwrap().downcast_ref::<T>().unwrap()
    }

    pub fn resource_mut<T>(&mut self) -> &mut T
        where T : 'static {
        let val = self.resources.get_mut(&TypeId::of::<T>());
        val.unwrap().downcast_mut::<T>().unwrap()
    }
    // ...
}
```

> Note: if any of this looks unfamiliar, the **Any** trait in the **std::any** crate specifies a number of different operations that can be used to "downcast" between the **Any** type and the intended type **T**. `downcast_ref` downcasts the unwrapped value into the specified type as a reference, while `downcast_mut` returns a mutable reference. We make use of the **std::collections::HashMap**'s built-in `get` and `get_mut` functions to correspond to these reference and mutable reference return values.

Next, let's go ahead and revisit the **App** to be able to actually load these resources just before the event loop runs. Since the resources are only loaded just before the event loop runs, the functions that are executed (such as **load_shaders**) is only actually executed once. We can modify the **App** vectors to store **FnOnce** so that rust can guarantee that a move occurs and these functions can only be executed once.

```rust
pub struct App {
    update_systems: Vec<Box<dyn Fn(&mut RenderState)>>,
    load_systems: Vec<Box<dyn FnOnce(&mut RenderState)>>,
    render_systems: Vec<Box<dyn Fn(&mut RenderState, &mut FrameContext)>>,
}

impl App {
    pub fn build() -> App {
        let update_systems: Vec<Box<dyn Fn(&mut RenderState)>> = Vec::new();
        let load_systems: Vec<Box<dyn FnOnce(&mut RenderState)>> = Vec::new();
        let render_systems: Vec<Box<dyn Fn(&mut RenderState, &mut FrameContext)>> = Vec::new();
        App {
            update_systems,
            load_systems,
            render_systems
        }
    }
    //..
}
```

We can then update the functions associated with the build operations to push these boxed values into the vectors. A **resource** function will allow us to add any **struct** object or value without having to wrap it in a system function.

```rust
impl App {
    //..
    pub fn add_resource<F>(mut self, f: F) -> App
        where F : FnOnce(&mut RenderState) + 'static {
        self.load_systems.push(Box::new(f));
        self
    }

    pub fn resource<T>(self, t: T) -> App
        where T : 'static {
        self.add_resource(move |state: &mut RenderState| {
            state.add_resource(t);
        })
    }

    pub fn update_system<F>(mut self, f: F) -> App 
        where F : Fn(&mut RenderState) + 'static {
        self.update_systems.push(Box::new(f));
        self
    }
    
    pub fn system<F>(mut self, f: F) -> App
        where F : Fn(&mut RenderState, &mut FrameContext) + 'static {
        self.render_systems.push(Box::new(f));
        self
    }
    //..
}
```

> Note: notice that we are using a closure function in the **resource** function in order to modify the render state and add an additional resource object. Because we are using closures and not just function pointers, the vectors must be defined as function traits rather than function pointers. Defining the system vectors in this way allows us to pass along **either** function pointers or anonymous closures.

Now that we have a few builder functions, the only thing left to do in **src/app.rs** is to iterate over the load systems and execute the appropriate functions once we have a built **RenderState**. Now that we are using **FnOnce**, the rust compiler makes sure that these functions will only be executed once. This means instead of iterating of the **load_systems** in a simple loop, we actually need to pop values off of the vector and execute them that way.

```rust
impl App {
    // ..
    pub fn run(mut self) {
        let event_loop = EventLoop::new();
        let window = WindowBuilder::new()
            .build(&event_loop)
            .unwrap();

        let mut state = block_on(RenderState::new(&window));
        while self.load_systems.len() > 0 {
            let system = self.load_systems.pop().unwrap();
            (system)(&mut state);
        }

        // event_loop.run...
    }
}
```

Unfortunately, since we have updated the vector of systems defined in the **App** struct, we also need to update the **RenderState::render** to be able to accept these boxed functions.

```rust
impl RenderState {
    // ...
    pub fn render(&mut self, render_systems: &Vec<Box<dyn Fn(&mut RenderState, &mut FrameContext)>>) {
        // ...
    }
}
```

Now our graphics library fully supports storing any type of data and the **App** will handle loading those resources during the build and setup code. We can now proceed to actually setting up the state we would like to pass along and update in the **examples/constants/constants_system.rs**.

```rust
use wgpu_book::*;

pub struct State {
    pub u_time: f32
}

pub fn load_shaders(state: &mut RenderState) { ... }

pub fn update(state: &mut RenderState) {
    let s = state.resource_mut::<State>();
    s.u_time += 0.01;
}
```

Great! Now all we have to do is create a function the **constants_system.rs** to setup a render pass with a simple render pipeline (defined in the **load_shaders** function) as well assign the push constants to it using the [wgpu::RenderPass::set_push_constants](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_push_constants) function.

```rust
pub fn push_constants(state: &mut RenderState, context: &mut FrameContext) {
    let resource = String::from("constants");
    let encoder = &mut context.encoder;
    {
        let mut render_pass = encoder.begin_render_pass(
            &wgpu::RenderPassDescriptor {
                color_attachments: &[
                    wgpu::RenderPassColorAttachmentDescriptor {
                        attachment: &context.frame.view,
                        resolve_target: None,
                        ops: wgpu::Operations {
                            load: wgpu::LoadOp::Load,
                            store: true
                        }
                    }
                ],
                depth_stencil_attachment: None
            }
        );

        render_pass.set_pipeline(&state.pipelines.get(&resource).unwrap());

        let s = state.resource::<State>();
        render_pass.set_push_constants(
            wgpu::ShaderStage::FRAGMENT,
            0,
            bytemuck::cast_slice(&[s.u_time])
        );

        render_pass.draw(0..3, 0..1);
    }
}
```

Last thing for us to do is actually build the **App** itself and pass along these system functions and the initial state.

```rust
mod constants_system;

use wgpu_book::*;
use constants_system::State;

fn main() {
    App::build()
        .resource(State {
            u_time: 0.0
        })
        .add_resource(constants_system::load_shaders)
        .update_system(constants_system::update)
        .system(constants_system::push_constants)
        .run();
}
```

![WGPU Push Constants](/assets/wgpu-uniform-pushconstant.gif)

## Buffers

### Buffers

> [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) is a blob of data created from the [Device::create_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_buffer) specifying a size, and [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html) stored on the gpu to store anything from graph structures, structs, arrays, vertex data, index data. BufferUsage will limit operations performed on the buffer data and causes a *panic* to throw when used in any way not specified by **BufferInitDescriptor::usage**.

Using the [wgpu::Device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html) it is possible to create a buffer with data initialized onto it using the **create_buffer_init** function. This utility method will use a [wgpu::BufferInitDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/util/struct.BufferInitDescriptor.html) to create a **wgpu::BufferDescriptor** and use the device context to effectively allocate a contiguous slice of memory and then copy over the range of data from the contents of the init descriptor. It seems handy to have this kind of utility in the main api but for now it appears to be via a device extension via `use wgpu::DeviceExt` trait.

```rust
use wgpu::DeviceExt;

#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct Vertex {
    position: [f32; 3],
    color: [f32; 3]
}

const VERTICES: &[Vertex] = &[
    Vertex { position: [0.0, 0.5, 0.0], color: [1.0, 0.0, 0.0] },
    Vertex { position: [-0.5, -0.5, 0.0], color: [0.0, 1.0, 0.0] },
    Vertex { position: [0.5, -0.5, 0.0], color: [0.0, 0.0, 1.0] }
];

let vertex_buffer = self.device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
    label: Some('Vertices'), // debug label in graphics debugging
    contents: bytemuck::cast_slice(VERTICES), // convert to &[u8]
    usage: wgpu::BufferUsage::VERTEX
});
```

Another common method for creating buffers, especially in the case of dynamic buffers, is to use the [wgpu::Device::create_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_buffer) directly rather than call [wgpu::DeviceExt::create_buffer_init](https://docs.rs/wgpu/0.6.0/wgpu/util/trait.DeviceExt.html#tymethod.create_buffer_init) and pass in the bytes.

```rust
struct Uniforms {
    timestep: f32,
    mouse_pos: [f32; 2]
}
let uniform_buffer = self.device.create_buffer(&wgpu::BufferDescriptor {
    label: Some('Vertices'),
    size: std::mem::size_of::<Uniforms>() as wgpu::BufferAddress,
    usage: wgpu::BufferUsage::UNIFORM | wgpu::BufferUsage::COPY_DST,
    mapped_at_creation: false
});
```

This will create a buffer that will approximately match the size in bytes of the struct Uniforms that can be used later during a render pass. The use of [wgpu::BufferUsage::COPY_DST](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_DST) allows the buffer to be written directly to the queue via [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_buffer). This method of updating an existing uniform buffer, applies to all types of buffers - whether that's a vertex buffer, a index buffer, or a uniform buffer.

```rust
// update vertices when the positions updated
self.queue.write_buffer(&self.vertex_buffer, 0, bytemuck::cast_slice(VERTICES));
// update the uniforms when the camera, or mouse positions changed
self.queue.write_buffer(&self.uniform_buffer, 0, bytemuck::cast_slice(&[uniforms]));
```

### Buffer Usage

> Note: [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html) has a number of available options that can be combined with one another. The most common ones apply to the type of buffer the data represents such as Vertex, Index, Uniform, or Storage. These coorespond to the input processing in the case of vertex shaders and compute shaders, while uniform and storage refer to the shader uniform storage types.

* **[wgpu::BufferUsage::VERTEX](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.VERTEX)** referring to usage as a vertex buffer during a draw operation (in association with the [wgpu::RenderPass::set_vertex_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_vertex_buffer)).

* **[wgpu::BufferUsage::INDEX](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.INDEX)** referring to usage as an index buffer during a draw operation (in association with the [wgpu::RenderPass::set_index_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_index_buffer)).

* **[wgpu::BufferUsage::UNIFORM](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.UNIFORM)** referring to usage as a **UNIFORM** binding within a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html).

* **[wgpu::BufferUsage::STORAGE](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.STORAGE)** referring to usage as a **STORAGE** binding within a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html).

In addition to these common uses, there are the uses that involve the [wgpu::CommandEncoder](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html) and [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html) that require either **[wgpu::BufferUsage::COPY_SRC](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_SRC)** or **[wgpu::BufferUsage::COPY_DST](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_DST)** (in the case of *copy_buffer_to_buffer*, *copy_texture_to_buffer*, or *write_buffer*). 

Other use cases such as [wgpu::BufferUsage::MAP_READ](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.MAP_READ) refer to allowing a buffer to be read from or written to in the case of [wgpu::BufferUsage::MAP_WRITE](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.MAP_WRITE). These scenarios typically involve being able to asynchronously read or write to a buffer that has already been allocated on the device.

### Vertex Buffers

> Vertex buffers are a type of rendering resource where the data consists of vertex attributes that can be configured to include any number of attribute data used in vertex processing. Vertex buffers may also be used with index buffers and primitive indicies in order to assemble rendering primitives such as triangle strips. Typically vertex buffers reside directly in graphics device memory rather than system memory so no additional memory transfers may need to occur (unless otherwise buffers are explicitly written to). 

In wgpu, a vertex buffer is represented as a [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) and have an additional [wgpu::VertexBufferDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexBufferDescriptor.html) used to describe how the vertex buffer will be interpreted by the graphics pipeline. Vertex buffers, or any buffer for that matter, is represented by all the vertex data packed into a single contiguous slice of memory stored on the device.

![Vertex Buffer Data](/assets/wgpu-vertex-buffer-data.png)

Within a vertex shader we may have the following code block to output the vec3 position from the layout at **location=0** and assign the input **vec3 color** to the output **vec4 f_color**.

```glsl
#version 450

layout(location=0) in vec3 pos;
layout(location=1) in vec3 color;

layout(location=0) out vec4 f_color;

void main() {
    gl_Position = vec4(pos, 1.0);
    f_color = vec4(color, 1.0);
}
```

The vertex is then represented in rust as a struct with the position and color set as such:

```rust
struct Vertex {
    pos: [f32; 3],
    color: [f32; 3]
}
```

In order for the render pipeline to understand how to interpret the a given vertex buffer (which simply has a size and a usage), we must specify a few properties. Vertex buffer descriptor properties indicate how large a vertex is in bytes (**stride**), how to iterate over the data (**step mode**), and the actual underlying attributes of that vertex buffer defined through the a [wgpu::VertexAttributeDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html). 

```rust
let render_pipeline = device.create_render_pipeline(
    &wgpu::RenderPipelineDescriptor {
        // ...
        vertex_state: wgpu::VertexStateDescriptor {
            index_format: wgpu::IndexFormat::Uint16,
            vertex_buffers: &[
                wgpu::VertexBufferDescriptor {
                    stride: std::mem::size_of::<Vertex>(),
                    step_mode: wgpu::InputStepMode::Vertex,
                    attributes: &[
                        wgpu::VertexAttributeDescriptor {
                            format: wgpu::VertexFormat::Float3,
                            offset: 0,
                            shader_location: 0
                        },
                        wgpu::VertexAttributeDescriptor {
                            format: wgpu::VertexFormat::Float3,
                            offset: std::mem::size_of::<[f32; 3]>(),
                            shader_location: 1
                        }
                    ]
                }
            ]
        },
        // ...
    }
)
```

The vertex attribute's [wgpu::VertexFormat](https://docs.rs/wgpu/0.6.0/wgpu/enum.VertexFormat.html) should match the format in the shader. An offset is specified to indicate the offset **in bytes** from the previous attribute read in the vertex. Finally, the **shader_location** must match the **layout(location=INDEX)** in the shader.

![Vertex Buffer Descriptor](/assets/wgpu-vertex-buffer-descriptor.png)

> Note: Since much of the vertex attribute data may appear to be redundant if done in a sequenced order (shader location 0, 1 with incrementing offsets based on the attribute types), we can actually use the [wgpu::vertex_attr_array!](https://docs.rs/wgpu/0.6.0/wgpu/macro.vertex_attr_array.html) macro. The macro will map a **shader location** to a **vertex format** and calculate offsets automatically.
> 
> ```rust
> use wgpu::VertexFormat::{Float3};
> wgpu::VertexBufferDescriptor {
>    stride: std::mem::size_of::<Vertex>(),
>    step_mode: wgpu::InputStepMode::Vertex,
>    attributes: &wgpu::vertex_attr_array![0 => Float3, 1 => Float3]
> }
> ```

Once we are ready to execute a render pass during a render operation, you can call the **[wgpu::RenderPass::set_vertex_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_vertex_buffer)** to assign a vertex buffer to a particular **slot**. The slot should match the index of the matching descriptor in **[wgpu::VertexStateDescriptor::vertex_buffers](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexStateDescriptor.html#structfield.vertex_buffers)**. 

```rust
render_pass.set_vertex_buffer(0, &self.vertex_buffer.slice(..))
```

### Index Buffers

> Index buffers are a type of rendering resource where the data consists of indices that point into a vertex buffer. This allows you to effectively reorder the vertex data and reuse existing data for multiple vertices. By combining an index buffer with a vertex buffer you can save on space needed to generate many primitives where the vertex points happen to be the same; this also saves on execution time since the vertex shader will not necessarily need to be executed again for a given vertex.

A good way to visualize an index buffer is to consider how a primitive triangle is normally created when you have a [wgpu::PrimitiveTopology](https://docs.rs/wgpu/0.6.0/wgpu/enum.PrimitiveTopology.html) set to a **TriangleList**. The order of the indices are highly dependent on the primitive topology so it's important to know which one you are using.

![Indexing Vertex Buffers](/assets/wgpu-indexing-buffers.png)

In a standard quad example, when rendering with a triangle list, the quad requires 2 triangles with 6 vertices to create both triangles together. Using an index buffer, we only have to specify 4 vertices and 6 indices to create the quad.

An index buffer can be constructed in the same way a vertex buffer is using the [wgpu::Device::create_buffer_init](https://docs.rs/wgpu/0.6.0/wgpu/util/trait.DeviceExt.html#tymethod.create_buffer_init). The only difference is that the [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.INDEX) is set to **INDEX** instead of **VERTEX**. Additional buffer uses can be made if you need to write or copy the buffer later.

```rust
let indices: Vec<u16> = vec![0, 1, 2];
let index_buffer = self.device.create_buffer_init(
    &wgpu::util::BufferInitDescriptor {
        label: Some("Index Buffer"),
        contents: bytemuck::cast_slice(&indices),
        usage: wgpu::BufferUsage::INDEX
    }
);
```

> Note: The order of the indices not only matters in terms of the vertices that they are pointing to, but the order should also keep in mind the [wgpu::RasterizationStateDescriptor::front_face](https://docs.rs/wgpu/0.6.0/wgpu/struct.RasterizationStateDescriptor.html#structfield.front_face). When specifying a [wgpu::FrontFace::Ccw](https://docs.rs/wgpu/0.6.0/wgpu/enum.FrontFace.html#variant.Ccw) the triangles with vertices in counter clockwise order are considered the front face (default in right-handed coordinate spaces). This is important for how **culling** works during the rasterization step. If you find missing primitives, the indices might be in the wrong order.

It should be noted that the indices themselves need to match the **index_format** specified on the [wgpu::VertexStateDescriptor::index_format](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexStateDescriptor.html#structfield.index_format). Usually this is set to **Uint16** but you can also specify **Uint32** for 32-bit unsigned integers.

```rust
wgpu::VertexStateDescriptor {
    index_format: wgpu::IndexFormat::Uint16,
    vertex_buffers: &[
        Vertex::desc()
    ]
}
```

Once you have an index buffer, it can be assigned onto the render pass through the [wgpu::RenderPass::set_index_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_index_buffer). Subsequent calls to [wgpu::RenderPass::draw_indexed](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.draw_indexed) will draw the indexed primitives using the active index buffer and active vertex buffers.

```rust
render_pass.set_index_buffer(&self.index_buffer.slice(..));
// ...
render_pass.draw_indexed(0..num_indices, 0, 0..num_instances);
```

### Input Step Mode and Instancing

> Instancing is a technique used to render multiple copies of the same mesh/vertex/primitive data. The vertex buffer data will describe what a single instance would be rendered as, whereas additional instance uniform data may be used to add additional transforms, translations, and variations to the vertex data. This makes rendering large numbers of instances easy to highly parallelize as only the initial vertex data for a single mesh is required to be stored on the device.

When working specifically with vertex buffers, it is possible to create additional vertex buffer objects that can be interpreted as instance data rather than per vertex primitives. In a previous section, we created a **Vertex** struct that stored a **float2** for position data. The [wgpu::VertexBufferDescriptor](https://docs.rs/0.6.0/wgpu/struct.VertexBufferDescriptor.html) object defined the stride to be over the size of the **Vertex**, with attributes defining offset, shader location, and vertex format.

The input step mode is normally defined as per vertex, but if you are using [wgpu::InputStepMode::Instance](https://docs.rs/wgpu/0.6.0/wgpu/enum.InputStepMode.html#variant.Instance), you can pass vertex buffer data that will be iterated as instance data rather than per vertex data. The vertex buffer descriptor might look the same in terms of how it defines each of the attributes and the stride of the data, but specifying the input step as **Instance** would indicate to the render pipeline that the next slice of data (after incrementing by **stride**) is another instance, not the next vertex.

Consider the following vertex shader that has 2 layout attributes to work with. The first **vec2 pos** is the initial vertex position, while the **vec2 i_pos** will be a *per instance position*.

```glsl
#version 450

layout(location=0) in vec2 pos;
layout(location=1) in vec2 i_pos;

void main() {
    gl_Position = vec4(pos + i_pos, 0.0, 1.0);
}
```

> Note: you can access the current instance position in a vertex shader with the **gl_InstanceIndex** variable. This is useful if you decide to pass along a large uniform buffer that contains all the mat4 transformations for every instance. This kind of indexing into a large uniform buffer array is a different technique than using vertex buffer objects and iterating over them as instance data.

The rust that complements this would need to define an additional vertex buffer descriptor over a struct that can store per instance data such as the instance position.

```rust
#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct EntityInstance {
    pos: [f32; 2]
}

impl EntityInstance {
    fn desc<'a>() -> wgpu::VertexBufferDescriptor<'a> {
        wgpu::VertexBufferDescriptor {
            stride: std::mem::size_of::<EntityInstance>() as wgpu::BufferAddress,
            step_mode: wgpu::InputStepMode::Instance,
            attributes: &[
                wgpu::VertexAttributeDescriptor {
                    offset: 0,
                    shader_location: 1,
                    format: wgpu::VertexFormat::Float2
                }
            ]
        }
    }
}
```

The vertex buffer descriptor begins its shader location at **1**, since there is another vertex buffer that has already defined a vertex attribute descriptor at **shader_location = 0**. Other than that difference, and the use of [wgpu::InputStepMode::Instance](https://docs.rs/wgpu/0.6.0/wgpu/enum.InputStepMode.html) we can pass this new descriptor into the render pipeline.

```rust
let render_pipeline = device.create_render_pipeline(
    &wgpu::RenderPipelineDescriptor {
        // ...
        vertex_state: wgpu::VertexStateDescriptor {
            index_format: wgpu::IndexFormat::Uint16,
            vertex_buffers: &[
                Vertex::desc()
                EntityInstance::desc()
            ]
        }
    }
)
```

The creation of the buffer itself is the same as in the case of vertices through **create_buffer_init** and passing along the instance data. We simply need to assign the additional **instance** vertex buffer to the render pass before executing draw on a number of instances.

```rust
render_pass.set_vertex_buffer(0, entity.vertex_buffer.slice(..));
render_pass.set_vertex_buffer(1, entity.instances_buffer.slice(..));
render_pass.draw(0..entity.num_vertices, 0..entity.num_instances);
```

### Example: Drawing a Shape

With the first set of resource concepts, namely vertex and uniform buffers, we have enough to build on the previous examples and pass along custom vertex data. The example triangle in the previous demo utilized the static vertices in the vertex shader. Let's go ahead and update the vertex shader below to be able to take in position data.

```glsl
// shader.vert
#version 450

layout(location=0) in vec2 pos;

void main() {
    gl_Position = vec4(pos, 0.0, 1.0);
}
```

The vertex shader now takes a single vec2 for the position of the vertex. We can use this shader to create any shapes we like by passing along a number of vertices. To do this, we need to update the **new** function in the render state to store the additional vertex data.

```rust
#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct Vertex {
    pos: [f32; 2]
}
```

Notice that in this example, we have added trait attributes for **Copy**, **Clone**, **Pod**, and **Zeroable**. These traits are required in order to be used by **bytemuck::Pod** and **bytemuck::cast_slice**. The **repr(C)** also indicates that this will be interpreted as a C-lang style struct as is. All of this will ensure that the **Vertex** struct will be treated as a normal array of bytes when **cast_slice** is executed. Following this, we need to create an actual vertex buffer descriptor that can be applied to **any Vertex**. To do this, we can leverage a **static lifetime** function on the Vertex impl to return the descriptor of the stride, step mode, and attribute layout.

```rust
impl Vertex {
    fn desc<'a>() -> wgpu::VertexBufferDescriptor<'a> {
        wgpu::VertexBufferDescriptor {
            stride: std::mem::size_of::<Vertex>() as wgpu::BufferAddress,
            step_mode: wgpu::InputStepMode::Vertex,
            attributes: &[
                wgpu::VertexAttributeDescriptor {
                    offset: 0,
                    shader_location: 0,
                    format: wgpu::VertexFormat::Float3
                }
            ]
        }
    }
}
```

Next, let's update the render pipeline created in the **RenderState** to define how to interpret the vertex buffer descriptor we prevoiusly defined. This will be assigned to the **vertex_buffers** property in the [wgpu::VertexStateDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexStateDescriptor.html#structfield.vertex_buffers).

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ..initialization
        let render_pipeline = device.create_render_pipeline(
            &wgpu::RenderPipelineDescriptor {
                // ...
                vertex_state: wgpu::VertexStateDescriptor {
                    index_format: wgpu::IndexFormat::Uint16,
                    vertex_buffers: &[
                        Vertex::desc()
                    ]
                }
                // ...
            }
        );
        // ..
    }
    // resize, render..
}
```

Now that the vertex buffer descriptor has been setup in the render pipeline, we can move along to actually creating the vertices that define the shape. Let's define a few utility methods that can generate a few different types of shapes such as a circle, quad, and triangle. 

```rust
struct Shape {
    vertices: Vec<Vertex>
}

impl Shape {
    fn triangle(center: [f32; 2], width: f32) -> Shape {
        let w2 = width / 2.0;
        Shape {
            vertices: (&[
                Vertex { pos: [center[0], center[1] + w2] },
                Vertex { pos: [center[0] - w2, center[1] - w2] },
                Vertex { pos: [center[0] + w2, center[1] - w2] }
            ]).to_vec()
        }
    }

    fn quad(center: [f32; 2], width: f32) -> Shape {
        let w2 = width / 2.0;
        Shape {
            vertices: (&[
                Vertex { pos: [center[0] - w2, center[1] - w2] },
                Vertex { pos: [center[0] + w2, center[1] - w2] },
                Vertex { pos: [center[0] - w2, center[1] + w2] },
                Vertex { pos: [center[0] - w2, center[1] + w2] },
                Vertex { pos: [center[0] + w2, center[1] - w2] },
                Vertex { pos: [center[0] + w2, center[1] + w2] }
            ]).to_vec()
        }
    }
}
```

Implementing a triangle, as you would expect, is as simple as passing along 3 vertices. Since the render pipeline is using the rasterization [wgpu::PrimitiveTopology::TriangleList](https://docs.rs/wgpu/0.6.0/wgpu/enum.PrimitiveTopology.html#variant.TriangleList), 3 vertices will equal a single triangle. The quad function following this will then require 6 vertices to create 2 triangles. 

> Note: If you wish to reduce the number of vertices required to generate a triangle, consider using a different primitive topology such as [wgpu::PrimitiveTopology::TriangleStrip](https://docs.rs/wgpu/0.6.0/wgpu/enum.PrimitiveTopology.html#variant.TriangleStrip) where each set of three adjacent vertices form a triangle. Remember however, that you can only set the rasterization primitive topology once per render pipeline - so all draw operations will follow this topology.

Next, to create a circle, we can consider looping over the degrees of a circle and convert each degree to radians. The **x** and **y** coordinates around the circle is as simple as a **x = radians.cos()** and **y = radians.sin()**. Since we are using a triangle list, we need to create a new triangle through the center position, the coordinate at x and y, plus another vertex at the x and y of the current degree plus a step.

```rust
impl Shape {
    // ...triangle, quad

    fn circle(center: [f32; 2], width: f32) -> Shape {
        let radius = width / 2.0;
        let mut vertices: Vec<Vertex> = Vec::new();
        let step = 5.0;
        let max_steps = (360.0 / step) as i32;
        for i in 0..max_steps {
            let degrees = (i as f32) * step;
            let radians = degrees * std::f32::consts::PI / 180.0;
            let radius2 = (degrees + step) * std::f32::consts::PI / 180.0;
            let x1 = radius * radians.cos() + center[0];
            let x2 = radius * radians2.cos() + center[0];
            let y1 = radius * radians.sin() + center[1];
            let y2 = radius * radians2.sin() + center[1];
            vertices.push(Vertex { pos: center});
            vertices.push(Vertex { pos: [x1, y1] });
            vertices.push(Vertex { pos: [x2, y2] });
        }
        Shape {
            vertices: vertices
        }
    }
}
```

> Tip: try adjusting the step to reduce the number of triangles/smoothing for the circle. 

A few short-hand functions to generate a list of triangles for each primitive type is implemented. Next, to use these shapes they need to be converted into a [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) along with the number of vertices. To do this, let's create an **Entity** struct to store these values, and add a **Vec<Entity>** to the **RenderState** struct. 

```rust
struct Entity {
    vertex_buffer: wgpu::Buffer,
    num_vertices: i32
}

struct RenderState {
    // surface, device, queue, swap chain, size, render pipeline...
    entities: Vec<Entity>
}
```

Then, within a **spawn** function, we can convert the passed in **Shape** into an **Entity** that stores a newly created buffer and number of vertices. Later on, we can use the return value of the buffer and store it as a reference for instancing.

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ...instance, surface, adapter, device, queue
        // ...swap chain, compile and load shaders
        // ...setup render pipeline

        let entities: Vec<Entity> = Vec::new();

        Self {
            // surface, device, queue, sw_desc, swap chain, size
            // render_pipeline, u_time
            entities
        }
    }

    fn spawn(&mut self, shape: Shape) {
        self.entities.push(Entity {
            vertex_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Vertex Buffer"),
                    contents: bytemuck::cast_slice(shape.vertices),
                    usage: wgpu::BufferUsage::VERTEX
                }
            ),
            num_vertices: shape.vertices.len()
        });
    }

    // resize, render...
}
```

> Note: When we approach instancing, we will talk about how we can leverage the same **wgpu::Buffer** for multiple different instances. This is very useful because we may have dozens or even hundreds or thousands of the same mesh or shapes but the only difference between the instances is a simple transform.

Great! We have a place to store entities, utility methods to generate shapes, and a render pipeline that can correctly interpret vertex buffers. All that's left is to actually spawn a number of shapes and set the vertex buffers on the render pass and draw them.

```rust
// ...
fn main() {
    // ...input helper, event loop, window

    let mut state = block_on(RenderState::new(&window));
    state.spawn(Shape::triangle([-0.5, -0.5], 0.1));
    state.spawn(Shape::quad([0.5, -0.25], 0.4));
    state.spawn(Shape::circle([0.25, 0.25], 0.4));

    // event loop.run...
}
// ...
impl RenderState {
    // ...new, spawn, resize

    fn render(&mut self) {
        // ...get current frame
        // ...create command encoder
        {
            let mut render_pass = // encoder.begin_render_pass...

            render_pass.set_pipeline(&self.render_pipeline);
            for entity in &self.entities {
                render_pass.set_vertex_buffer(0, entity.vertex_buffer.slice(..));
                render_pass.draw(0..entity.num_vertices, 0..1);
            }
        }

        // ...queue.submit
    }
}
```

![Shapes](/assets/wgpu-vertex-buffer-polygons.png)

### Example: Particle Instancing

With **instancing** we can now build a very rudimentary particle system that can render thousands of primitives without having to pass along vertex data for each and every instance. Recall that the previous example created several shapes with a few utility methods to pass along position and width. Instead of doing this, we would like to have **EntityInstance** define a translate position that will effectively translate all the vertices associated with a given instance.

To do this, let's start by updating the vertex shader to have an additional variable for storing **per instance** position data, let's additionally add a rotation angle. The rotation angle will be converted into a 2d matrix transform.

```glsl
#version 450

layout(location=0) in vec2 pos;
layout(location=1) in vec2 i_pos;
layout(location=2) in float i_rot;

mat2 rotate2d(float angle) {
    return mat2(cos(angle), -sin(angle),
        sin(angle), cos(angle));
}

void main() {
    gl_Position = vec4(rotate2d(i_rot) * (pos + i_pos), 0.0, 1.0);
}
```

Then, we will need to setup a struct and a vertex buffer descriptor to store instance data.

```rust
#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct EntityInstance {
    pos: [f32; 2],
    rot: f32
}

impl EntityInstance {
    fn desc<'a>() -> wgpu::VertexBufferDescriptor<'a> {
        wgpu::VertexBufferDescriptor {
            stride: std::mem::size_of::<EntityInstance>() as wgpu::BufferAddress,
            step_mode: wgpu::InputStepMode::Instance,
            attributes: &[
                wgpu::VertexAttributeDescriptor {
                    offset: 0,
                    shader_location: 1,
                    format: wgpu::VertexFormat::Float2
                },
                wgpu::VertexAttributeDescriptor {
                    offset: std::mem::size_of<[f32; 2]>(),
                    shader_location: 2,
                    format: wgpu::VertexFormat::Float
                }
            ]
        }
    }
}
```

Pass along this additional vertex buffer descriptor to the render pipeline's **vertex_state**.

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ...initialization
        let render_pipeline = device.create_render_pipeline(
            &wgpu::RenderPipelineDescriptor {
                // ...
                vertex_state: wgpu::VertexStateDescriptor {
                    index_format: wgpu::IndexFormat::Uint16,
                    vertex_buffers: &[
                        Vertex::desc(),
                        EntityInstance::desc()
                    ]
                }
                // ..
            }
        );
        // ...
    }
    // resize, render..
}
```

Since all shape instances are going to be translated according to an instance's position, we can simplify the shape spawning to take a single **width** value and use the default **[0.0, 0.0]** as the center position for all shapes.

```rust
fn triangle(width: f32) -> Shape {
    let h = width / 2.0;
    let nh = -1.0 * h;
    Shape {
        vertices: (&[
            Vertex { pos: [0.0, h] },
            Vertex { pos: [nh, nh] },
            Vertex { pos: [h, nh] }
        ]).to_vec()
    }
}

fn quad(width: f32) -> Shape {
    let h = width / 2.0;
    let nh = -1.0 * h;
    Shape {
        vertices: (&[
            Vertex { pos: [nh, nh] },
            Vertex { pos: [h, nh] },
            Vertex { pos: [nh, h] },
            Vertex { pos: [nh, h] },
            Vertex { pos: [h, nh] },
            Vertex { pos: [h, h] }
        ]).to_vec()
    }
}
```

The previous implementation of the circle function included the center position with the calculations. We can update that function to generate the polar coordinates without those additions and just consider the **radius * radians.cos/sin** calculations.

```rust
fn circle(radius: f32) -> Shape {
    let radius = width / 2.0;
    let mut vertices: Vec<Vertex> = Vec::new();
    let step = 5.0;
    let max_steps = (360.0 / step) as i32;
    for i in 0..max_steps {
        let degrees = (i as f32) * step;
        let radians = degrees * std::f32::consts::PI / 180.0;
        let radians2 = (degrees + step) * std::f32::consts::PI / 180.0;
        let x1 = radius * radians.cos();
        let y1 = radius * radians.sin();
        let x2 = radius * radians2.cos();
        let y2 = radius * radians2.sin();
        vertices.push(Vertex { pos: [0.0, 0.0] });
        vertices.push(Vertex { pos: [x1, y1] });
        vertices.push(Vertex { pos: [x2, y2] });
    }
    Shape {
        vertices: vertices
    }
}
```

To **spawn** a number of instances, we are going to need to update the render state **spawn** function to convert a **Vec<EntityInstance>** into a new buffer that can be stored on the entity. The buffer will be initialized using the **create_buffer_init** function we used before.

```rust
struct Entity {
    vertex_buffer: wgpu::Buffer,
    entity_buffer: wgpu::Buffer,
    num_vertices: u32,
    num_instances: u32
}

impl RenderState {
    // ...
    fn spawn(&mut self, shape: Shape, instances: Vec<EntityInstance>) {
        self.entities.push(Entity {
            vertex_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Vertex Buffer"),
                    contents: bytemuck::cast_slice(&shape.vertices),
                    usage: wgpu::BufferUsage::VERTEX
                }
            ),
            entity_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Instance Buffer"),
                    contents: bytemuck::cast_slice(&instances),
                    usage: wgpu::BufferUsage::VERTEX
                }
            ),
            num_vertices: shape.vertices.len() as u32,
            num_instances: instances.len() as u32
        });
    }
    // ...
}
```

> Note: if we wanted to update the position data for each of the instances on the fly, we would need to store the instance data as well as the buffer. An update function could then make a call to [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_buffer) in order to update the bytes of the entities as data was updated. If the size of buffer has to change, we would need to re-allocate that address space accordingly.

The **render** function can now take advantage of this additional [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) and **num_instances** when setting the additional vertex buffer and calling **draw**.

```rust
impl RenderState {
    // ...
    fn render(&mut self) {
        // get frame, create encoder
        {
            let mut render_pass = // encoder.begin_render_pass...
            render_pass.set_pipeline(&self.render_pipeline);
            for entity in &self.entities {
                render_pass.set_vertex_buffer(0, entity.vertex_buffer.slice(..));
                render_pass.set_vertex_buffer(1, entity.entity_buffer.slice(..));
                render_pass.draw(0..entity.num_vertices, 0..entity.num_instances);
            }
        }

        // self.queue.submit...
    }
}
```

The last thing we need to do is actually spawn the number of instances. Each instance is going to have a random position and random rotation angle that will be used to transform the vertices within the render pipeline. We're going to make use of the [nanorand](https://crates.io/crates/nanorand) crate for the random number generation.

```rust
use nanorand::{RNG, WyRand};

fn rand_instances(num_instances: i32) -> Vec<EntityInstance> {
    let mut instances: Vec<EntityInstance> = Vec::new();
    let mut rng = WyRand::new();
    for i in 0..num_instances {
        let x: f32 = (rng.generate_range::<u32>(1, 200) as f32 - 100.0) / 100.0;
        let y: f32 = (rng.generate_range::<u32>(1, 200) as f32 - 100.0) / 100.0;
        let rot: f32 = rng.generate_range::<u32>(1, 360) as f32;
        instances.push(EntityInstance {
            pos: [x, y],
            rot: rot
        });
    }

    instances
}
```

This can then be used within the **main** function to spawn a number of instances with the associated primitive shapes we have previously defined.

```rust
fn main() {
    // ...
    let mut state = block_on(RenderState::new(&window));
    state.spawn(Shape::triangle(0.04), rand_instances(200));
    state.spawn(Shape::quad(0.004), rand_instances(1000));
    state.spawn(Shape::circle(0.02), rand_instances(100));
    // ...
}
```

![Particle Instancing](/assets/wgpu-particle-instancing.png)

### Example: Drawing with Indices

In each of the previous examples we used the [wgpu::RenderPass::draw](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.draw) to include the range of vertices and range of instances. The shape functions were made up of 3 vertices per triangle primitive. Unfortunately, all of these additional vertices to create basic shapes can add duplication where it is not needed. For instance to create a quad the original implementation was as follows:

```rust
fn quad(width: f32) -> Shape {
    let h = width / 2.0;
    let nh = -1.0 * h;
    Shape {
        vertices: (&[
            Vertex { pos: [nh, nh] },
            Vertex { pos: [h, nh] },
            Vertex { pos: [nh, h] },
            Vertex { pos: [nh, h] },
            Vertex { pos: [h, nh] },
            Vertex { pos: [h, h] }
        ]).to_vec()
    }
}
```

The quad in the above example is rendered using 6 vertices - or 2 triangle primitives. However, to actually create a quad there are only **4 unique vertices**!. The same problem is more noticable when we create a **circle** shape.

```rust
fn circle(radius: f32) -> Shape {
    let radius = width / 2.0;
    let mut vertices: Vec<Vertex> = Vec::new();
    let step = 5.0;
    let max_steps = (360.0 / step) as i32;
    for i in 0..max_steps {
        let degrees = (i as f32) * step;
        let radians = degrees * std::f32::consts::PI / 180.0;
        let radians2 = (degrees + step) * std::f32::consts::PI / 180.0;
        let x1 = radius * radians.cos();
        let y1 = radius * radians.sin();
        let x2 = radius * radians2.cos();
        let y2 = radius * radians2.sin();
        vertices.push(Vertex { pos: [0.0, 0.0] });
        vertices.push(Vertex { pos: [x1, y1] });
        vertices.push(Vertex { pos: [x2, y2] });
    }
    Shape {
        vertices: vertices
    }
}
```

A circle made up of triangle list primitives in our above implementation would duplicate the center and one additional vertex for each of the steps around the degrees of a circle. For a circle made up of 90 triangles (**step = 5.0**) we would end up with 270 vertices for a circle that only actually needs 91 vertices (1 vertex for each position around the circle, and 1 vertex in the center). To solve these problems we can make use of the **index buffer** and the [wgpu::Queue::draw_indexed](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.draw_indexed) function to pass a range of indices instead.

```rust
struct Shape {
    vertices: Vec<Vertex>,
    indices: Vec<u16>
}

struct Entity {
    vertex_buffer: wgpu::Buffer,
    index_buffer: wgpu::Buffer,
    entity_buffer: wgpu::Buffer,
    num_vertices: u32,
    num_indices: u32,
    num_instances: u32
}
```

The first set of changes that need to be made include updating the **Shape** to actually store the indices, and updating the **Entity** struct to store the buffer and number of indices. We can already update the **spawn** function to create the [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) based on this change. Since the buffer will be used as an index buffer we need to make sure to use the [wgpu::BufferUsage::INDEX](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.INDEX) usage format.

```rust
impl RenderState {
    // ...
    fn spawn(&mut self, shape: Shape, instances: Vec<EntityInstance>) {
        self.entities.push(Entity {
            vertex_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Vertex Buffer"),
                    contents: bytemuck::cast_slice(&shape.vertices),
                    usage: wgpu::BufferUsage::VERTEX
                }
            ),
            index_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Index Buffer"),
                    contents: bytemuck::cast_slice(&shape.indices),
                    usage: wgpu::BufferUsage::INDEX
                }
            ),
            entity_buffer: self.device.create_buffer_init(
                &wgpu::util::BufferInitDescriptor {
                    label: Some("Instance Buffer"),
                    contents: bytemuck::cast_slice(&instances),
                    usage: wgpu::BufferUsage::VERTEX
                }
            ),
            num_vertices: shape.vertices.len() as u32,
            num_indices: shape.indices.len() as u32,
            num_instances: instances.len() as u32
        });
    }
    // ...
}
```

Next, we can go ahead and update the render function to set the index buffer using the [wgpu::RenderPass::set_index_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_index_buffer) and pass along the indices to the [wgpu::Queue::draw_indexed](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderBundleEncoder.html#method.draw_indexed).

```rust
impl RenderState {
    // ...
    fn render(&mut self) {
        // ..get frame, create encoder
        {
            let mut render_pass = // encoder.begin_render_pass...

            self.u_time = self.u_time + 0.01;

            render_pass.set_pipeline(&self.render_pipeline);
            render_pass.set_push_constants(
                wgpu::ShaderStage::FRAGMENT, 
                0,
                bytemuck::cast_slice(&[
                    self.size.width as f32, 
                    self.size.height as f32, 
                    self.u_time
                ])
            );

            for entity in &self.entities {
                render_pass.set_vertex_buffer(0, entity.vertex_buffer.slice(..));
                render_pass.set_index_buffer(entity.index_buffer.slice(..));
                render_pass.set_vertex_buffer(1, entity.entity_buffer.slice(..));
                render_pass.draw_indexed(
                    0..entity.num_indices, 
                    0, 
                    0..entity.num_instances
                );
                //render_pass.draw(0..entity.num_vertices, 0..entity.num_instances);
            }
        }

        // queue.submit...
    }
}
```

Great! Since we are using an index buffer, we no longer have to specify the vertex range (only the starting vertex) and we can instead just pass along the index range and instance range. The only thing left is to update the shape creation code to generate indices.

```rust
fn quad(width: f32) -> Shape {
    let h = width / 2.0;
    let nh = -1.0 * h;
    Shape {
        vertices: vec![
            Vertex { pos: [nh, nh] },
            Vertex { pos: [h, nh] },
            Vertex { pos: [nh, h] },
            Vertex { pos: [h, h] }
        ],
        indices: vec![
            0, 1, 2,
            2, 1, 3
        ]
    }
}
```

The circle function is a bit more complex in that we need to use the last 3 indices once the current index > 1 as we step through creating the vertices for the circle. Other than this, the code is actually less than before!

```rust
fn circle(width: f32) -> Shape {
    let radius = width / 2.0;
    let mut vertices: Vec<Vertex> = Vec::new();
    let mut indices: Vec<u16> = Vec::new();
    let step = 5.0;
    let max_steps = 1 + (360.0 / step) as i32;
    vertices.push(Vertex { pos: [0.0, 0.0] });
    for i in 0..max_steps {
        let degrees = (i as f32) * step;
        let radians = degrees * std::f32::consts::PI / 180.0;
        let x1 = radius * radians.cos();
        let y1 = radius * radians.sin();
        vertices.push(Vertex { pos: [x1, y1] });
    }
    for i in 0..vertices.len() {
        if i > 1 {
            indices.push(0 as u16);
            indices.push((i - 1) as u16);
            indices.push(i as u16);
        }
    }
    Shape {
        vertices: vertices,
        indices: indices
    }
}
```

That's it! Let's make one minor adjustment to the fragment shader to perform a smooth step around the edges to give the appearance of a faded border. The **smoothstep** function is within a family of interpolation functions that will simply ensure that value provided is a gradient as it approaches the lower bound of the left edge, and 1 as it approaches the right edge. You can combine multiple smoothsteps, like in the fragment shader, to get a union/difference/intersection of the values to create some fun effects.

```glsl
#version 450

layout(location=0) out vec4 f_color;
layout(push_constant) uniform Locals {
    vec2 u_resolution;
    float u_time;
};

void main() {
    vec2 st = gl_FragCoord.xy / u_resolution;
    float x = smoothstep(0.01, 0.5, st.x) - smoothstep(0.5, 0.95, st.x);
    float y = smoothstep(0.01, 0.5, st.y) - smoothstep(0.5, 0.95, st.y);
    vec3 color = vec3(abs(sin(u_time)), abs(cos(u_time)), 0.4) * (x * y);
    f_color = vec4(color, 1.0);
}
```

![Indexed Drawing](/assets/wgpu-instance-draw-indexed.png)


### Example: Update Particle Positions

The rudimentary particle system we built in the previous example uses instancing to generate thousands of primitives on the screen but none of the particles are actually moving or responding to any kind of parameters. We can imagine a player primitive that would respond to input changes, or have built in flow fields so that particles can follow gravitational paths along the scene. 

No matter the use case, the data associated with those changes needs to be updated to wherever that memory is allocated on the device. There are a few approaches to simulations, one of which involves creating a compute shader that can update uniform buffers in a feedback loop. For now, we're going to create an example that performs the calculations within the main code and passes along write updates to the appropriate instance buffer.

```glsl
#version 450

layout(location=0) in vec2 pos;
layout(location=1) in vec2 i_pos;
layout(location=2) in float i_rot;

mat2 rotate2d(float angle) {
    return mat2(cos(angle), -sin(angle),
        sin(angle), cos(angle));
}

void main() {
    gl_Position = vec4(rotate2d(i_rot) * (pos + i_pos), 0.0, 1.0);
}
```

For the most part, the shader code will remain unchanged from the previous example. Both the vertex shader and fragment shader are simply going to respond to the input passed to it and execute the appropriate stage by design. The vertex shader will receive vertex and instance data and transform the position appropriately.

```glsl
#version 450

layout(location=0) out vec4 f_color;
layout(push_constant) uniform Locals {
    vec2 u_resolution;
    float u_time;
};

void main() {
    vec2 st = gl_FragCoord.xy / u_resolution;
    float x = smoothstep(0.01, 0.5, st.x) - smoothstep(0.5, 0.95, st.x);
    float y = smoothstep(0.01, 0.5, st.y) - smoothstep(0.5, 0.95, st.y);
    vec3 color = vec3(abs(sin(u_time)), abs(cos(u_time)), 0.4) * (x * y);
    f_color = vec4(color, 1.0);
}
```

The fragment shader will simply perform the smooth step between the edges and use the time to animate the color for all the fragments for now. Next, in order to get the position data updated for each of the instances, we need to actually store all the instance data so that it can be updated!

```rust
struct Entity {
    vertex_buffer: wgpu::Buffer,
    index_buffer: wgpu::Buffer,
    entity_buffer: wgpu::Buffer,
    num_vertices: u32,
    num_indices: u32,
    num_instances: u32,
    instances: Vec<EntityInstance>
}
```

In order to write and store the **Vec** of **EntityInstance** we need to modify the way the buffer is created so that it can be written to later. To do this, we need to change the **spawn** function to include a [wgpu::BufferUsage::COPY_DST](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_DST) so that the buffer can be written to.

```rust
fn spawn(&mut self, shape: Shape, instances: Vec<EntityInstance>) {
    self.entities.push(Entity {
        vertex_buffer: self.device.create_buffer_init(
            &wgpu::util::BufferInitDescriptor {
                label: Some("Vertex Buffer"),
                contents: bytemuck::cast_slice(&shape.vertices),
                usage: wgpu::BufferUsage::VERTEX
            }
        ),
        index_buffer: self.device.create_buffer_init(
            &wgpu::util::BufferInitDescriptor {
                label: Some("Index Buffer"),
                contents: bytemuck::cast_slice(&shape.indices),
                usage: wgpu::BufferUsage::INDEX
            }
        ),
        entity_buffer: self.device.create_buffer_init(
            &wgpu::util::BufferInitDescriptor {
                label: Some("Instance Buffer"),
                contents: bytemuck::cast_slice(&instances),
                usage: wgpu::BufferUsage::VERTEX | wgpu::BufferUsage::COPY_DST
            }
        ),
        num_vertices: shape.vertices.len() as u32,
        num_indices: shape.indices.len() as u32,
        num_instances: instances.len() as u32,
        instances: instances
    });
}
```

Then we can create a simple **update** function to iterate over the instances and make a change to the position data. Once we are done making those changes, we simply need to execute the [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_buffer) to update the corresponding entity instance buffer.

```rust
fn update(&mut self) {
    self.u_time = self.u_time + 0.001;
    for entity in self.entities.iter_mut() {
        for instance in entity.instances.iter_mut() {
            let mut pos = instance.pos;
            pos[0] = pos[0] + 0.0001;
            pos[1] = pos[1] + 0.0001;
            if pos[0] > 1.0 {
                pos[0] = -1.0;
            }
            if pos[0] < -1.0 {
                pos[0] = 1.0;
            }
            if pos[1] > 1.0 {
                pos[1] = -1.0;
            }
            if pos[1] < -1.0 {
                pos[1] = 1.0;
            }
            instance.pos = pos;
        }
        self.queue.write_buffer(
            &entity.entity_buffer,
            0,
            bytemuck::cast_slice(&entity.instances)
        );
    }
}
```

> Note: In this example we are writing to the entire buffer for all of the instances - since every instance has changed. In some cases, you may want to only write to a portion of a given **wgpu::Buffer** (such as for only instances that have actually changed, or instances that are now invisible, visible..etc). The second parameter to [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_buffer) is the offset buffer address, so if you only wanted to update a single entity in a large memory space - that is an option.

The update function will effectively iterate over the instances and entities, then make a small change to the position statically. We could store the velocity information on the instance as well, but for now this change will be done hard-coded. Additionally, if the position is outside the normalized bounds the position will be wrapped to the other side. This keeps the simulation running continuously without particles permanently leaving the screen. The only thing left that needs to be done is to make a call to the **update** function from within the event loop in **main**.

```rust
fn main() {
    // ...setup input handling, event loop, window...etc
    let mut state = block_on(RenderState::new(&window));

    state.spawn(Shape::triangle(0.04), rand_instances(200));
    state.spawn(Shape::quad(0.004), rand_instances(1000));
    state.spawn(Shape::circle(0.02), rand_instances(100));

    event_loop.run(move |event, _, control_flow| {
        if input.update(&event) {
            if input.key_released(VirtualKeyCode::Escape) || input.quit() {
                *control_flow = ControlFlow::Exit;
                return;
            }
        }

        state.update();

        match event {
            Event::RedrawRequested(_) => {
                state.render();
            },
            Event::WindowEvent { event, .. } => match event {
                WindowEvent::Resized(physical_size) => {
                    state.resize(physical_size);
                },
                WindowEvent::ScaleFactorChanged { new_inner_size, .. } => {
                    state.resize(*new_inner_size);
                },
                _ => {}
            },
            Event::MainEventsCleared => {
                window.request_redraw();
            },
            _ => {}
        }
    });
}
```

![Write Buffer Updates](/assets/wgpu-particles-moving-write-buffer.gif)

## Textures

### Texture

> Originally referred to as diffuse mapping, textures are simply image representation of pixels to be mapped for color (diffuse), normals, bump mapping, height maps, displacement, reflections, specular, occlusion, and various other techniques used in a materials system. [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html) is created by the [wgpu::Device::create_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_texture) described by a [wgpu::TextureDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDescriptor.html) including size, mip counts, sample counts, dimensions, [wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/enum.TextureFormat.html) and [wgpu::TextureUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureUsage.html). 

Creating a texture in wgpu can be done with the [wgpu::Device::create_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_texture) passing along a [wgpu::TextureDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDescriptor.html).

```rust
let img = image::open("color-diffuse.png");
let dimensions = img.dimensions();
let diffuse_rgba = img.as_rgba8().unwrap();
let diffuse_texture = device.create_texture(&wgpu::TextureDescriptor {
    size: wgpu::Extend3d {
        width: dimensions.0,
        height: dimensions.1,
        depth: 1
    },
    mip_level_count: 1,
    sample_count: 1,
    dimension: wgpu::TextureDimension:D2,
    format: wgpu::TextureFormat::Rgba8UnormSrgb,
    usage: wgpu::TextureUsage::SAMPLED | wgpu::TextureUsage::COPY_DST,
    label: Some("diffuse_texture")
});
```

You will notice a number of properties on the texture descriptor that may sound unfamiliar. A texture typically describes a resource where the basic unit within the texture is a **texel**. A **texel** represents typically 1-4 components depending on the format provided. The [wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/enum.TextureFormat.html) can be any wide variety of color channel variants with the most commonly used format as **Rgba8UnormSrgb**. 

> Note: The most commonly used texture format **Rgba8UnormSrgb** represents red, green, blue and alpha channels with 8 bit integer per channel and Srgb-color 0-255 converted to/from a float 0-1 in shaders. You will find that all the available formats tend to follow (CHANNELS) (Bits Per Channel) (Signed/Unsigned/Normalized/Float/Int in Shader Representation).

The [wgpu::TextureDimension](https://docs.rs/wgpu/0.6.0/wgpu/enum.TextureDimension.html) is represented by either a **1D**, **2D**, or **3D** texture. The dimension typically corresponds to the [wgpu::Extent3d](https://docs.rs/wgpu/0.6.0/wgpu/struct.Extent3d.html) with the added exception that you can use the **depth** value for **2D Texture Arrays** in addition to **3D Textures**.

![Textures](/assets/wgpu-textures-2d-3d.png)

The address space on a texture is most often in a vector coordinate **(u, v)** representing rows and columns. Whereas, the **w** is represented as either a particular slice of a texture array or the slice on a 3d texture. Additionally, textures typically have what is known as a **Level of Detail** or **LOD** **mipmap**. Each level of detail is represented by a smaller pixel space to be used for sampling in a graphics pipeline according to which filter properties have been set. Sampling a texture in a graphics pipepline will usually correspond to a [wgpu::Sampler](https://docs.rs/wgpu/0.6.0/wgpu/struct.Sampler.html) and can only be used if the [wgpu::TextureUsage::SAMPLED](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureUsage.html#associatedconstant.SAMPLED) has been set on the texture descriptor.

In a fragment shader, we can setup a texture (and sampler) according to how we plan on binding it in. The standard 2d texture corresponds to a **texture2D** uniform while a 2d texture array (where depth > 1 and the texture dimension is 2D) would correspond to **texture2DArray**.

```glsl
layout(location=0) in vec2 v_tex_coords;
layout(location=0) out vec4 f_color;

layout(set=0, binding=0) uniform texture2D t_diffuse; // 2d texture
// layout(set=0, binding=0) uniform texture2D t_diffuse[]; // multiple textures
layout(set=0, binding=1) uniform sampler s_diffuse;

void main() {
    f_color = texture(sampler2D(t_diffuse, s_diffuse), v_tex_coords);
}
```

Once the description of the texture has been setup, you can write to the texture using the [wgpu::Queue::write_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_texture) function. It is important to know that, similar to a [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html), the usage must also include [wgpu::TextureUsage::COPY_DST](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureUsage.html#associatedconstant.COPY_DST) in order to allow the texture address space to be written to via the **Queue**, or via the **CommandEncoder**.

```rust
queue.write_texture(
    wgpu::TextureCopyView {
        texture: &diffuse_texture,
        mip_level: 0,
        origin: wgpu::Origin3d::ZERO
    },
    diffuse_rgba,
    wgpu::TextureDataLayout {
        offset: 0,
        bytes_per_row: 4 * dimensions.0,
        rows_per_image: dimensions.1
    }
);
```

When calling [wgpu::Queue::write_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_texture) you need to pass a [wgpu::TextureCopyView](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureCopyViewBase.html) to prepare a buffer image copy based on the [wgpu::TextureDataLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDataLayout.html) with offset (offset into buffer as the start of the texture), bytes per row (1 row of pixels in x direction, must be multiple of 256 unless using **copy_texture_to_buffer**), and rows per image.

> Note: the **TextureCopyView** has a **mip_level**, this corresponds to the target mip level in the texture memory. Level of detail mip levels correspond to how the graphics pipeline will sample a **texel** from a texture and have that either be **nearest** or **linearly** 1-1 with the actual pixels on the screen. In a later section, we will go over level of detail and mip-mapping in greater detail.

After creating a texture, and later writing data to it, a corresponding [wgpu::TextureView](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureView.html) is needed in order to be passed along to a render pipeline. The texture view can be created using the [wgpu::Texture::create_view](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html#method.create_view) by passing along a [wgpu::TextureViewDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureViewDescriptor.html).

```rust
texture.create_view(&wgpu::TextureViewDescriptor::default());
```

### Sampler

> [wgpu::Sampler](https://docs.rs/wgpu/0.6.0/wgpu/struct.Sampler.html) defines how a pipeline will sample (i.e. given a pixel coordinate on or outside the texture boundaries - what color should be returned) from [wgpu::TextureView](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureView.html) by defining image filters (e.g. anisotropy) and address (wrapping) modes (such as in the x, y, z directions), magnified filter, minimized filter - described by [wgpu::SamplerDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.SamplerDescriptor.html).

Sampling occurs when the program provides a coordinate on the texture (texture coordinate) and needs to return a color back based on these configured parameters. When texture coordinates happen to outside of the texture (in the case of address wrapping) the following address wrapping modes are used:

![AddressMode](/assets/wgpu-sampler-addressmode.png)

* **[wgpu::AddressMode::ClampToEdge](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.ClampToEdge)** use the nearest pixel on the edges of the texture.
* **[wgpu::AddressMode::Repeat](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.Repeat)** repeats the texture in a tiling fashion.
* **[wgpu::AddressMode::MirrorRepeat](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.MirrorRepeat)** repeates a texture by mirroring it on every repeat (as in mirror when outside boundaries).

```rust
let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
    address_mode_u: wgpu::AddressMode::ClampToEdge,
    address_mode_v: wgpu::AddressMode::ClampToEdge,
    address_mode_w: wgpu::AddressMode::ClampToEdge,
    mag_filter: wgpu::FilterMode::Linear,
    min_filter: wgpu::FilterMode::Linear,
    mipmap_filter: wgpu::FilterMode::Nearest,
    compare: Some(wgpu::CompareFunction::LessEqual),
    lod_min_clamp: -100.0,
    lod_max_clamp: 100.0,
    ..Default::default()
});
```

> Note: [mipmap](https://en.wikipedia.org/wiki/Mipmap) refers to a technique of pre-calculated, optimized sequence of images, each of which are progressively lower resolution representation of the previous. Height/width of each image, or **level**, in the mipmap is a power of two smaller than the previous level. [wgpu::TextureDescriptor::mip_level_count](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDescriptor.html#structfield.mip_level_count) describes the mip count of a texture and the [wgpu::SamplerDescriptor::mipmap_filter](https://docs.rs/wgpu/0.6.0/wgpu/struct.SamplerDescriptor.html#structfield.mipmap_filter) in a sampler can determine how to filter between mip map levels (with **lod_min_clamp** and **lod_max_clamp** setting min and max mip levels to use).

Additional properties such as the **mag_filter** and **min_filter** are handled by the [wgpu::FilterMode](https://docs.rs/wgpu/0.6.0/wgpu/enum.FilterMode.html) and provide the following options:

* **[wgpu::FilterMode::Linear](https://docs.rs/wgpu/0.6.0/wgpu/enum.FilterMode.html#variant.Linear)** when magnified the textures are smooth but blury as it scales.
* **[wgpu::FilterMode::Nearest](https://docs.rs/wgpu/0.6.0/wgpu/enum.FilterMode.html#variant.Nearest)** when magnified pixels are sampled using nearest neighbor

### Bind Groups and Layouts

> In order to make use of other resources from within shaders we need a way to reference them. BindGroups and PipelineLayouts provide us with a mechanism for plugging in these resources. [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html) represents a set of resources bound to bindings described by a [wgpu::BindGroupLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayout.html). Use the [wgpu::Device::create_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group) to create a bind group.

Recall from a previous section on shaders that there were multiple ways to pass around data within a shader. We can use a vertex buffer to bind in vertex or instance data into a vertex shader through the **layout(location=INDEX)** type layouts. Data can also be passed around between the shader stages with the **in** and **out** qualifiers. We also discussed the use of a **push_constant** uniform block that allowed us to pass in dynamic variables to a small reserved space of memory for simple indexes and small variables.

One other way to get memory addressable data into a shader is through a **binding** layout. In a fragment shader, we may want to sample colors from a texture resource using a particular sampling function. Both the **texture** and the **sampler** would need to be bound into the shader in order to use them.

```glsl
#version 450

layout(location=0) in vec2 tex_coords;
layout(location=0) out vec4 f_color;

layout(set=0, binding=0) uniform texture2D tex;
layout(set=0, binding=1) uniform sampler sam;

void main() {
    f_color = texture(sampler2D(tex, sam), tex_coords);
}
```

A binding layout in a shader will correspond to a bind group layout in wgpu. The **set** in `layout(set=INDEX, binding=BINDING)` refers to the index passed along to the [wgpu::RenderPass:set_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_bind_group) while the **binding** refers to the [wgpu::BindGroupLayoutEntry::binding](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayoutEntry.html#structfield.binding). We can create a bind group layout with the [wgpu::Device::create_bind_group_layout](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group_layout) function. 

```rust
let texture_bind_group_layout = device.create_bind_group_layout(
    &wgpu:BindGroupLayoutDescriptor {
        entries: &[
            wgpu::BindGroupLayoutEntry {
                binding: 0,
                visibility: wgpu::ShaderUsage::FRAGMENT,
                ty: wgpu::BindingType::SampledTexture {
                    multisampled: false,
                    dimension: wgpu::TextureViewDimension::D2,
                    component_type: wgpu::TextureComponentType::Uint
                },
                count: None
            },
            wgpu::BindGroupLayoutEntry {
                binding: 1,
                visibility: wgpu::ShaderUsage::FRAGMENT,
                ty: wgpu::BindingType::Sampler {
                    comparison: false,
                },
                count: None
            }
        ],
        label: Some("texture_bind_group_layout")
    }
);
```

The [wgpu::BindGroupLayoutDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayoutDescriptor.html) includes a number of [wgpu::BindGroupLayoutEntry](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayoutEntry.html) entries. Because a bind group layout can be assigned to a particular render pipeline, each bind group layout entry must assign which [wgpu::ShaderStage](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderStage.html) the binding is visible to. Finally, a bind group layout entry must specify the [wgpu::BindingType](https://docs.rs/wgpu/0.6.0/wgpu/enum.BindingType.html). Binding types can be of one of the following forms:

* **[wgpu::BindingType::UniformBuffer](https://docs.rs/wgpu/0.6.0/wgpu/enum.BindingType.html#variant.UniformBuffer)** a [wgpu:Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) for uniform values.


    ```glsl
    layout(std140, binding = 0) uniform Globals {
        mat4 view_proj;
    }
    ```

* **[wgpu::BindingType::StorageBuffer](https://docs.rs/wgpu/0.6.0/wgpu/enum.BindingType.html#variant.StorageBuffer)** similar to the uniform buffer, however exposes additional properties that allow it to be quite a bit larger and can even be variable length in memory as opposed to fixed uniform buffers.

    ```glsl
    layout(set=0, binding=0) buffer storageBuffer {
        vec4 elements[];
    }
    ```

* **[wgpu::BindingType::Sampler](https://docs.rs/wgpu/0.6.0/wgpu/enum.BindingType.html#variant.Sampler)** refers to a [wgpu::Sampler](https://docs.rs/wgpu/0.6.0/wgpu/struct.Sampler.html) and is used to sample a texture that is typically also bound in

    ```glsl
    layout(set=0, binding=1) uniform sampler s;
    ```

* **[wgpu::BindingType::SampledTexture](https://docs.rs/wgpu/0.6.0/wgpu/enum.BindingType.html#variant.SampledTexture)** refers to a [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html) where the **dimension** must match the texture view dimension of the texture.

    ```glsl
    layout(set=0, binding=0) uniform texture2D t;
    // layout(set=0, binding=0) uniform texture2DArray tArray;
    // layout(set=0, binding=0) uniform texture3D t3d;
    ```

> Note: when the bind group layout entry includes a **count** the corresponding binding type must be a sampled texture. This indicates that the binding is an array of textures. Typically you will see texture arrays include a count and the shader would include the array as such:
>
> ```glsl
> layout(set=0, binding=0) uniform texture2D textures[2];
> ```
> 
> The count can be specified explicitly in the shader, or it can be set without a constant as in:
>
> ```glsl
> layout(set=0, binding=0) uniform texture2D textures[];
> ```

In any case, the bind group layout entries describe how each binding is defined according to the type, binding location, along with which shader stage it is visible to. Of course, since the layout is just a description, in order to actually get a resource onto the GPU we need to create a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html). We can use the [wgpu::Device::create_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group) to generate a bind group that will be later used to assign onto a render pass with [wgpu::RenderPass::set_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_bind_group).

```rust
let texture_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    layout: &texture_bind_group_layout,
    entries: &[
        wgpu::BindGroupEntry {
            binding: 0,
            resource: wgpu::BindingResource::TextureView(&view)
        },
        wgpu::BindGroupEntry {
            binding: 1,
            resource: wgpu::BindingResource::Sampler(&sampler)
        }
    ]
});
```

When you are ready to execute a render pipeline on a render pass, you can set your resources, vertex buffers, index buffers, pipeline, and finally the bind group. Setting the bind group needs to happen after you have set the render pipeline and typically may come before setting other resources. All draw operations **must** be queued after you've set the bind group and resources for a render pipeline for the ordering to be correct on the queue.

```rust
render_pass.set_bind_group(0, &self.bind_group, &[]);
```

The set bind group function has 3 parameters including the **index**, the **bind group** itself, and a **dynamic offsets** array. The **index** in the first parameter refers to the same **set** index in the `layout(set=INDEX...`. The bind group is the same one that was created earlier with the **create_bind_group**. The dynamic offsets parameter will correspond to an offset into either a **UniformBuffer** or **StorageBuffer** that the render pipeline will use to read from.

> Note: Dynamic offsets can be especially useful if you have a large buffer that has been allocated (fixed size in the case of a uniform buffer, variable size in a storage buffer) and you need to read specific sets of data at a particular offset in that buffer. An example of this would be in the case of a large number of entities and all the transform matrix data was in that buffer for all of the entities.

### Font Rendering

Early days of font rendering typically involved extracting a particular character out of a texture known as a **bitmap font**. The bitmap font represents a collection of already rasterized images for each of the characters or glyphs of a particular font character set. Each glyph is dedicated to a specific region of the texture along with the cooresponding coordinates for that glyph. Bitmap fonts are relatively easy to implement since all the font glyphs are pre-rasterized, making the process efficient. The downside to this approach is that the glyphs do not scale very well and any time you want to use more characters or a different font you will need to recompile the bitmap. 

These days, you can use a more modern library approach to load font families, render each of the glyph functions to a texture and use any number of adjusted font operations. One such library is known as [freetype](https://www.freetype.org/). 

> Freetype is used on a number of platforms as the font renderer including: Android, iOS, macOS (next to Apple Advanced Typography / CoreText), Playstation, Linux to name a few. Freetype has support for a number of font formats including TrueType and OpenType, PFR, PCF, PostScript, and BDF. Truetype and OpenType contain glyphs represented by a combination of functions and spline definitions. This allows fonts to be defined procedurally based on a preferred font height.

In rust, we have a number of choices to implement font rendering. One alternative to freetype is a pure rust implementation known as [rusttype](https://gitlab.redox-os.org/redox-os/rusttype). Rusttype supports loading truetype font collections including opentype (**.otf** and **.ttf**). Glyphs are laid out horizontally using vertical and horizontal matrices with glyph-pair-specific kerning. The implementation appears to be based on an [analytic rasterization of curves with polynomial filters](http://josiahmanson.com/research/scanline_rasterization/) research paper to improve on vector graphics.

RustType also has a gpu cache implementation that allows you to maintain glyph renderings in a dynamic cache in GPU memory in order to minimize texture uploads per-frame. This allows draw call counts for text to remain especially low as all recently used glyphs are kept in a single GPU texture.

```rust
let font_data: &[u8] = include_bytes!("Hack.ttf");
let font: Font<'static> = rusttype::Font::try_from_bytes(font_data).unwrap();
```

Loading in the font itself is as simple as creating a [rusttype::Font](https://docs.rs/rusttype/0.9.2/rusttype/enum.Font.html) and loading from the **try_from_bytes** function. In order to actually draw each individual glyph, it's important to understand how metrics are generally calculated based on the font height scale. This is done using the [rusttype::Scale](https://docs.rs/rusttype/0.9.2/rusttype/struct.Scale.html). The **Scale** defines the size of the rendered face, in pixels, horizontally and vertically. The vertical scale of **y** pixels is the distance between **ascent** and **descent**. 

![Glyph Metrics](/assets/wgpu-glyph-metrics.png)

Each of these metrics can be found in the api of rusttype. [rusttype::Font::v_metrics](https://docs.rs/rusttype/0.9.2/rusttype/enum.Font.html#method.v_metrics) provides the vertical metrics shared by all glyphs of the font. The vertical metrics will include **descent**, **ascent**, **line_gap** of which can be used to calculate the **"advance height"**.

```rust
let scale = rusttype::Scale::uniform(16.0);
let v_metrics = self.font.v_metrics(scale);
let advance_height = v_metrics.ascent - v_metrics.descent + v_metrics.line_gap;
```

Meanwhile, we can iterate over a string of characters and use the [rusttype::Font::glyph](https://docs.rs/rusttype/0.9.2/rusttype/enum.Font.html#method.glyph) to return a [rusttype::Glyph](https://docs.rs/rusttype/0.9.2/rusttype/struct.Glyph.html) object. Each glyph does not have an inherit scale associated with it, so additional operations must provide a size with [rusttype::Glyph::scaled](https://docs.rs/rusttype/0.9.2/rusttype/struct.Glyph.html#method.scaled) and [rusttype::ScaledGlyph::positioned](https://docs.rs/rusttype/0.9.2/rusttype/struct.ScaledGlyph.html#method.positioned) to offset with a position.

```rust
struct FontLayout {
    font: rusttype::Font<'static>,
    scale: rusttype::Scale,
    max_width: i32
}

fn layout_glyphs(config: FontLayout, content: String) -> Vec<rusttype::PositionedGlyph<'static>> {
    let v_metrics = config.font.v_metrics(config.scale);
    let advance_height = v_metrics.ascent - v_metrics.descent + v_metrics.line_gap;

    let mut glyphs = vec![];
    let mut caret = rusttype::point(0.0, v_metrics.ascent);
    let mut last_glyph: Option<rusttype::GlyphId> = None;
    for c in content.chars() {
        let g = config.font.glyph(c).scaled(config.scale);
        if let Some(last) = last_glyph {
            // calculate kerning between two glyphs
            caret.x += config.font.pair_kerning(config.scale, last, g.id());
        }
        let mut g = g.positioned(caret);
        last_glyph = Some(g.id());

        caret.x += g.unpositioned().h_metrics().advance_width;
        glyphs.push(g);
    }

    glyphs
}
```

In order to calculate the layout of a string of characters, each glyph needs to be positioned by the horizontal and vertical metrics. Horizontal metrics help us determine the **advance_width** to increment horizontally based on the uniform scaling provided by the font. A caret represented by a [rusttype::point](https://docs.rs/rusttype/0.9.2/rusttype/fn.point.html) is helpful for maintaining an incrementing position as we iterate over the characters. A pair of glyphs can also have **kerning** - an adjusted space between two character forms. Consider that mono-space fonts will have uniform spacing between any pair of characters - in this case the pair_kerning would be the same. 

We can of course, modify this function to include the calculations to ensure that the current **caret** position is reset when the bounding box of a glyph is outside of the max_width using the [rusttype::PositionedGlyph::pixel_bounding_box](https://docs.rs/rusttype/0.9.2/rusttype/struct.PositionedGlyph.html#method.pixel_bounding_box). 

```rust
fn layout_glyphs(config: FontLayout, content: String) -> Vec<rusttype::PositionedGlyph<'static>> {
    // ...
    for c in content.chars() {
        // ...
        let mut g = g.positioned(caret);
        last_glyph = Some(g.id());

        if let Some(bb) = g.pixel_bounding_box() {
            if config.max_width > 0 && bb.max.x > config.max_width {
                caret = rusttype::point(0.0, caret.y + advance_height);
                g.set_position(caret);
                last_glyph = None;
            }
        }

        caret.x += g.unpositioned().h_metrics().advance_width;
        glyphs.push(g);
    }

    glyphs
}
```

> Note that in this particular layout the characters will be positioned one after the other in a left-aligned paragraph style. Other layout types will need to consider how to align a string of characters that have other alignments such as centering, or right-alignment.

Finally, let's add one additional change to ensure that control characters like new lines change the caret to the next line.

```rust
fn layout_glyphs(config: FontLayout, content: String) -> Vec<rusttype::PositionedGlyph<'static>> {
    // ...
    for c in content.chars() {
        if c.is_control() {
            match c {
                '\n' => {
                    caret = rusttype::point(0.0, caret.y + advance_height);
                },
                _ => {}
            }
            continue;
        }

        // ...
    }

   glyphs
}
```

So far, we have loaded a font and generated a list of glyphs from a string, but we haven't yet rendered anything. The glyphs are merely representations of underlying functions and combinations of spline formulas based on the chosen font representation. In order to actually render anything we need to rasterize these chosen glyphs onto a cached texture. Rusttype provides a [rusttype::Cache](https://docs.rs/rusttype/0.9.2/rusttype/gpu_cache/struct.Cache.html) as a GPU memory available cache where we can take the data returned from a queued operation and write it to a texture.

```rust
let config = FontLayout {
    font: font,
    scale: rusttype::Scale::uniform(16.0),
    max_width: 400
};

let content = String::from("Hello, world!");
let glyphs = layout_glyphs(config, content);

let mut cache = rusttype::gpu_cache::Cache::builder()
    .dimensions(500, 500)
    .multithread(true)
    .build();

for glyph in &glyphs {
    cache.queue_glyph(0, glyph.clone());
}
```

Beneath the layers of rusttype are several libraries that parse fonts into different outline builders and translate into path operations including line, curves, and move operations. Queuing a glyph to the GPU cache in this case will setup a queue of packed layout of glyphs in a LRU (least recently used) sort order. Actual rasterization of these glyphs doesn't yet occur until the cache is ready to be processed through the **cache_queued** callback function.

```rust
cache.cache_queued(|rect, data| {
    queue.write_texture(
        wgpu::TextureCopyView {
            texture: texture,
            mip_level: 0,
            origin: wgpu::Origin3d {
                x: rect.min.x,
                y: rect.min.y,
                z: 0
            }
        },
        data,
        wgpu::TextureDataLayout {
            offset: 0,
            bytes_per_row: rect.width(),
            rows_per_image: 0
        },
        wgpu::Extend3d {
            width: rect.width(),
            height: rect.height(),
            depth: 1
        }
    );
});
```

The **rect** passed along to the callback function here will be provided with the boundaries while the **data** consists of the pre-rasterized glyph data calculated from various curve, line, and move operations. Note that the dimensions of the cache should be enough to correspond to the varying glyphs that will be used **with font scaling**. This data can be passed along to a resource on the GPU such as a [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html) using the [wgpu::Queue::write_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_texture) operation.

### [Refactor] Rust Modules

Up until this point all the examples have been written in a single `main.rs` file. In order to make things a bit easier to work with, we're going to refactor our project into modules and systems so that we can swap out any render pipeline or stage multiple render passes wherever we want.

### Example: Fragment Shader Editor

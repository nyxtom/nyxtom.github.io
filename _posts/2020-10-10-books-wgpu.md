---
title: Rust Graphics Programming with WGPU
published: true
book: true
---

{% raw %}
<style type="text/css">
.hack h2 {
    font-size: 1.4rem;
    padding-top: 12rem;
    padding-bottom: 4rem;
}
.hack h3 {
    font-size: 1.2rem;
    padding: 2rem 1rem;
    padding-top: 8rem;
}
pre {
    margin-top: 3rem;
    margin-bottom: 3rem;
    margin-left: 1rem;
    margin-right: 1rem;
    background-color: #fbfbfb;
    border: none;
    border-radius: 8px;
}
.hack blockquote {
    margin-top: 3rem;
    margin-bottom: 3rem;
}
</style>
{% endraw %}

After spending significant time going through the documentation and source of [gfx-rs/wgpu](https://github.com/gfx-rs/wgpu) and various [gfx-rs/gfx](https://github.com/gfx-rs/gfx) backends, Vulkan portability layers, the book of shaders, SPIR-V, and all else - I decided I wanted to compile all my notes together into a comprehensive guide to **Rust Graphics Programming with WGPU**. 

This guide is a deep dive into WGPU with a high level overview of shader programming in relation to that. It covers a range of topics fundamental to graphics programming including: rendering pipelines, shaders, vertex buffers, instancing, textures, sampling, bind layouts, compute pipelines, debugging, cross compilation, target platform builds and more. Along the way, we will use that knowledge to build simple render pipelines and demos to reinforce what we've learned. By the end of the book, we put it all together and build a simple cross platform game with WGPU.

**Table of Contents**:
* TOC
{:toc}

## 0.1 Execution Order

Graphics libraries closely follow hardware abstraction layers provided by modern graphics cards. Although, this is sometimes the other way around, where graphics libraries like Vulkan provide a spec and hardware vendors follow this spec to connect with underlying hardware graphics layers. Nevertheless, higher level graphics apis describe an execution order that tends to be very similar no matter what library you end up using.

1. **Initialize device/instance/surfaces**: data structures needed to access GPU and surface handles
2. **Load assets**: compile and load shader modules, descriptors for rendering pipelines, create and populate command buffers for the GPU to execute, send resources to the GPU exclusive memory
3. **Update assets**: update resources and uniforms to shaders, perform application logic
4. **Render Operations**: send list of command buffers to the queue, present swapchain, wait until GPU signals that the frame was rendered to the current back buffer
5. **Repeat 2-4** until close
6. **Exit/Cleanup** finish remaining work on GPU, de-reference and cleanup resources before exiting

## 0.2 Dependencies

Throughout this book we will end up using a variety of dependencies to perform various tasks in [rust](https://rust-lang.org). The dependencies are meant to be as minimal as possible and are mentioned below:

* [winit](https://github.com/rust-windowing/winit) window handle, event loop, events
* [winit_input_helper](https://github.com/rukai/winit_input_helper) update the winit events into a input state
* [wgpu](https://github.com/gfx-rs/wgpu) cross platform rust library for working with Vulkan, Metal, DirectX 11/12, WebGPU
* [futures](https://github.com/rust-lang/futures-rs) adds additional combinator utilities for async rust
* [bytemuck](https://github.com/Lokathor/bytemuck) working with bytes from structs and types for buffers later
* [image](https://github.com/image-rs/image) image processing functions and methods for converting image formats
* [shaderc](https://github.com/google/shaderc-rs) rust bindings for the collection of tools/libs for shader compilation

```toml
[dependencies]
winit = "0.23.0"
winit_input_helper = "0.8.0"
wgpu = "0.6.0"
futures = "0.3.6"
bytemuck = "1.4.1"
image = "0.23.10"
shaderc = "0.6.2"
```

## 0.3 Event Loop

These dependencies follow into the sample code below for setting up a window with an event loop and handling events:

> Note: If you need a primer on what the event loop is, take a look at my post on [winit and rust](/2020/10/07/winit-rust/), you can also check out the in depth look at [epoll, kqueue, iocp explained with rust](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/).

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

Now let's get into some of the other definitions that we are working with.

## 1 Initialization

### 1.1 Instance

> Context for all other wgpu objects, the primary use of an Instance is to create **[wgpu::Adapter](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html)** and **[wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html)**.

Considered the entry point to the entire code for working with any graphics library. This includes vulkan's **Instance**, DirectX **IDXGIFactory**, **CAMetalLayer** in Metal, **GPU** in WebGPU.

```rust
let instance = wgpu::Instance::new(wgpu::BackendBit::PRIMARY);
```

Note that [wgpu::BackendBit::Primary](https://docs.rs/wgpu/0.6.0/wgpu/struct.BackendBit.html) is referring to the backends that wgpu will use. Primary refers to first tier api support (Vulkan + Metal + DX12 + WebGPU) with secondary support for OpenGL + DX11. Since OpenGL/DX11 is still experimental, targeting through these backends may be unsupported in some areas. Use **PRIMARY** as the default but you can always just target a specific backend such as **wgpu::BackendBit::VULKAN** for instance.

### 1.2 Surface

> Platform specific surface (e.g. window, vk::Surface, texture back buffer, gpu canvas context) onto which a swap chain may receive a window handle and return back the internal surface for working with the next frame.

In `wgpu` this is referred to by creating a [wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html) *unsafe* function **[create_surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Instance.html#method.create_surface)** from a raw window handle.

```rust
// window: winit::window::Window, instance: wgpu::Instance
let surface = unsafe { instance.create_surface(window) };
```

### 1.3 Adapter

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

### 1.4 Logical Device

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

### 1.5 Swap Chain

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

### 1.6 Command Encoder

> [wgpu::CommandEncoder](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html) records [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html)es, [wgpu::ComputePass](https://docs.rs/wgpu/0.6.0/wgpu/struct.ComputePass.html)es, and transfer operations between driver managed resources like [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html)s and [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html)s. When finished recording, call [CommandEncoder::finish](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.finish) to obtain a [wgpu::CommandBuffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandBuffer.html) which may be submitted for execution to the GPU.

```rust
let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: Some("RENDER ENCODER") // label in graphics debugger
});
```

### 1.7 Render Pass

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

### 1.8: Example: Clearing the Screen

With just the above concepts alone, we have enough to:

* **Initialize the api** with an instance of wgpu
* **Setup a surface to draw to** with winit window handle
* **Query for a device** with features, limits, power preference and surface compatibility
* **Setup a swap chain** to work with the current frame buffer
* **Begin encoding commands** to start work on render passes
* **Submit our first operation** to clear the screen

```rust
use futures::executor::block_on;

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

    let mut state = block_on(RenderState::new(&window));

    event_loop.run(move |event, _, control_flow| {
        if input.update(&event) {
            if input.key_released(VirtualKeyCode::Escape) || input.quit() {
                *control_flow = ControlFlow::Exit;
                return;
            }
        }

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

Recall the code above that sets up the event loop, creates the window, updates the input state, and calls to our render state to either render or resize based on the correct window eventing. Next, we will setup the actual render state to encapsulate all of our initalization objects mentioned in the previous sections.


```rust
struct RenderState {
    surface: wgpu::Surface,
    device: wgpu::Device,
    queue: wgpu::Queue,
    sc_desc: wgpu::SwapChainDescriptor,
    swap_chain: wgpu::SwapChain,
    size: winit::dpi::PhysicalSize<u32>
}
```

Within the **RenderState** implementation, we need to initialize each of these objects: **wgpu::Instance**, surface, **wgpu:Device**, **wgpu::Queue**, **wgpu::SwapChainDescriptor**, and **wgpu::SwapChain**.

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        let size = window.inner_size();
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

        Self {
            surface,
            device,
            queue,
            sc_desc,
            swap_chain,
            size
        }
    }
    
}
```

Let's add the resize function mentioned previously to handle updating the swap chain descriptor and generate a new swap chain whenever the window is resized.

```rust
impl RenderState {
    // .. new

    fn resize(&mut self, new_size: winit::dpi::PhysicalSize<u32>) {
        self.size = new_size;
        self.sc_desc.width = new_size.width;
        self.sc_desc.height = new_size.height;
        self.swap_chain = self.device.create_swap_chain(&self.surface, &self.sc_desc);
    }

}
```

Finally, the **render** function can be implemented to grab the current frame, create a command encoder, begin a render pass and submit the command buffer to the queue.

```rust
impl RenderState {
    // ..new...

    // ..resize...
    
    fn render(&mut self) {
        let frame = self.swap_chain.get_current_frame()
            .expect("Timeout getting texture")
            .output;

        let mut encoder = self.device.create_command_encoder(
            &wgpu::CommandEncoderDescriptor {
                label: Some("RENDER ENCODER") // label in graphics debugger
            }
        );

        {
            let _render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
                color_attachments: &[
                    wgpu::RenderPassColorAttachmentDescriptor {
                        attachment: &frame.view,
                        resolve_target: None,
                        ops: wgpu::Operations {
                            load: wgpu::LoadOp::Clear(wgpu::Color {
                                r: 0.9,
                                g: 0.2,
                                b: 0.3,
                                a: 1.0
                            }),
                            store: true
                        }
                    }
                ],
                depth_stencil_attachment: None
            });
        }


        // submit the commands to the queue!
        self.queue.submit(std::iter::once(encoder.finish()));
    }
}
```

![Clear Screen](/assets/rust-by-example-wgpu-clear.png)

## 2 Shaders

### 2.1 Shader Modules

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

### 2.2 Render Pipeline

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

### 2.3 Draw Operations

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

### 2.4 Shader Layout

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

### 2.5 Example: Draw a Triangle

Before we move onto to more complex binding layouts, vertex buffers, textures, and samplers - with the above concepts alone we have enough to execute the most basic shader - **drawing a triangle**. 

* **Write a vertex shader** with a *const* array of vertices describing a static triangle
* **Write a fragment shader** to output a color for the geometry
* **Compile/load shader** into spir-v and create a shader module
* **Setup render pipeline** to load the shaders as programmable stages
* **Assign the render pipeline** by setting the render pipeline on the first render pass
* **Call draw on the render pass** by passing along the vertices and instance(s)

The vertex shader will be the same one we've seen in the previous sections. Namely, it will be a constant set of vertices that describe our triangle. In a file called `shader.vert` add the following code:

```glsl
#version 450

const vec2 positions[3] = vec2[3](
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

We make use of the **gl_VertexIndex** to select the right **vec2** in the **positions** array, then output the result to **gl_Position**.

Next, the fragment shader will be the same as in the previous sections. It simply outputs the same color to the `location=0` output variable. As mentioned previously, this usually refers to the passed in current swap chain texture attached at the beginning of our render pass.

```glsl
#version 450

layout(location=0) out vec4 f_color;

void main() {
    f_color = vec4(0.1, 0.2, 0.4, 1.0);
}
```

Recall that our previous code for setting up the state included creating everything from the instance, adapter, device, swap chain descriptor, and all the code from the winit event handling. Let's extend the **new** function to also setup the render pipeline we described earlier as well as loading our shader code. Using the same code from [demo - clearing the screen](#demo-18-clearing-the-screen), the following code will extend the **new** function to get the pipeline setup and load the shader code written previously.

First, since we are going to use the render pipeline when we queue a render pass (within the **render** function). We need to store our render pipeline definition in the **RenderState**.

```rust
struct RenderState {
    // ..surface, device, queue, swap chain, size
    render_pipeline: wgpu::RenderPipeline
}
```

Next, we are going to import the **shader.vert** and **shader.frag** shaders, compile them with [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html) and create a [wgpu::ShaderModule](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderModule.html) with [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module).

```rust
impl RenderState {
    async fn new(window: &Window) -> Self {
        // .. instance, device, queue, swap chain code
        let mut compiler = shaderc::Compiler::new().unwrap();
        let vs_src = include_str!("shader.vert");
        let fs_src = include_str!("shader.frag");
        let vs_spirv = compiler.compile_into_spirv(
            vs_src, 
            shaderc::ShaderKind::Vertex, 
            "shader.vert", 
            "main", 
            None
        ).unwrap();
        let fs_spirv = compiler.compile_into_spirv(
            fs_src, 
            shaderc::ShaderKind::Fragment, 
            "shader.frag", 
            "main", 
            None
        ).unwrap();
        let vs_module = device.create_shader_module(
            wgpu::util::make_spirv(&vs_spirv.as_binary_u8())
        );
        let fs_module = device.create_shader_module(
            wgpu::util::make_spirv(&fs_spirv.as_binary_u8())
        );
        // ..
    }
}
```

After creating the shader modules, a [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html) must be created as described in the previous section using [wgpu::Device::create_render_pipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_render_pipeline).

```rust
// ..
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ..instance, device, queue, swap chain code
        // ..load and compile shaders
        let vs_module = // device.create_shader_module..
        let fs_module = // device.create_shader_module..
        let render_pipeline_layout = device.create_pipeline_layout(
            &wgpu::PipelineLayoutDescriptor {
                label: Some("pipeline layout"),
                bind_group_layouts: &[],
                push_constant_ranges: &[]
            }
        );

        let render_pipeline = device.create_render_pipeline(
            &wgpu::RenderPipelineDescriptor {
                label: Some("pipeline"),
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

        Self {
            // ...surface, device, queue, sc_desc, swap_chain, size
            render_pipeline
        }
    }
    // ..resize, render
}
```

The last step, is to update the **render** function to set the active render pipeline on the current render pass and call **draw** to draw the vertices of the triangle.

```rust
// ..
impl RenderState {
    // ..new, resize
    fn render(&mut self) {
        // .. get frame, create command encoder
        {
            let mut render_pass = // encoder.begin_render_pass...

            render_pass.set_pipeline(&self.render_pipeline);
            render_pass.draw(0..3, 0..1);
        }
        // ..queue.submit
    }
}
```

![Draw Triangle Shader](/assets/wgpu-draw-triangle-shader.png)

## 3 Shader Uniforms

### 3.1 GLSL Uniforms

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

### 3.2 Push Constants

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

### 3.3 Example: Set current timestep

Using the **push_constant uniform** block and the [wgpu::Features::PUSH_CONSTANTS](https://docs.rs/wgpu/0.6.0/wgpu/struct.Features.html#associatedconstant.PUSH_CONSTANTS) feature in wgpu, we can create a simple example that passes along the current timestep and manipulate the fragment shader color in the process. This builds off of the same triangle example in a previous section.

Within the **new** function during initialization, we need to update the logical device to include the push constants feature mentioned in the previous section, as well as specify the maximum push constant size in bytes.

```rust
// ...
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
// ...
```

Next, during initialization the code setup a simple render pipeline layout that loaded the two shaders **shader.vert** and **shader.frag**. Recall the implementation for the **shader.vert** vertex shader.

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

In the fragment shader, we will update it to include a new **layout(push_constant) uniform** block to allow us to specify a simple timestep float.

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

```rust
// ..
impl RenderState {
    async fn new(window: &Window) -> Self {
        // ... size, instance, surface, adapter, 
        let (device, queue) = // adapter.request_device..
        // .. swap chain, compile and load shaders
        let render_pipeline_layout = device.create_pipeline_layout(
            &wgpu::PipelineLayoutDescriptor {
                label: Some("pipeline layout"),
                bind_group_layouts: &[],
                push_constant_ranges: &[
                    &wgpu::PushConstantRange {
                        stages: wgpu::ShaderStage::FRAGMENT,
                        range: 0..4
                    }
                ]
            }
        )
        let render_pipeline  = // device.create_render_pipeline..

        let u_time: f32 = 0.0;

        Self {
            // ..surface, device, queue, sc desc, swap chain, size
            render_pipeline,
            u_time
        }
    }
    // ..resize, render
}
```

Great! Now all we have to do is update our render pass code to actually update and push along our time variable using the [wgpu::RenderPass::set_push_constants](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_push_constants).

```rust
// ...
impl RenderState {
    // ...new, resize
    fn render(&mut self) {
        // ..get frame, create command encoder
        {
            let mut render_pass = // encoder.begin_render_pass ...

            self.u_time = self.u_time + 0.01;

            render_pass.set_pipeline(&self.render_pipeline);
            render_pass.set_push_constants(
                wgpu::ShaderStage::FRAGMENT,
                0,
                bytemuck::cast_slice(&[self.u_time])
            );
            render_pass.draw(0..3, 0..1);
        }

        // ..queue.submit
    }
}
```

![WGPU Push Constants](/assets/wgpu-uniform-pushconstant.gif)

## 4 Resources

### 4.1 Buffers

> [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) is a blob of data created from the [Device::create_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_buffer) specifying a size, and [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html) stored on the gpu to store anything from graph structures, structs, arrays, vertex data, index data. BufferUsage will limit operations performed on the buffer data and causes a *panic* to throw when used in any way not specified by **BufferInitDescriptor::usage**.

Using the [wgpu::Device](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html) it is possible to create a buffer with data initialized onto it using the **create_buffer_init** function. This utility method will use a [wgpu::BufferInitDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/util/struct.BufferInitDescriptor.html) to create a **wgpu::BufferDescriptor** and use the device context to effectively allocate a contiguous slice of memory and then copy over the range of data from the contents of the init descriptor. It seems handy to have this kind of utility in the main api but for now it appears to be via a device extension via `use wgpu::DeviceExt` trait.

```rust
use wgpu::DeviceExt;

struct Vertex {
    position: [f32; 3],
    color: [f32; 3]
}

// in order for bytemuck::cast_slice to work 
// on a struct, we need two traits to be specified
// bytemuck::Pod and bytemuck::Zeroable
unsafe impl bytemuck::Pod for Vertex {}
unsafe impl bytemuck::Zeroable for Vertex {}

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

### 4.2 Buffer Usage

> Note: [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html) has a number of available options that can be combined with one another. The most common ones apply to the type of buffer the data represents such as Vertex, Index, Uniform, or Storage. These coorespond to the input processing in the case of vertex shaders and compute shaders, while uniform and storage refer to the shader uniform storage types.

* **[wgpu::BufferUsage::VERTEX](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.VERTEX)** referring to usage as a vertex buffer during a draw operation (in association with the [wgpu::RenderPass::set_vertex_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_vertex_buffer)).

* **[wgpu::BufferUsage::INDEX](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.INDEX)** referring to usage as an index buffer during a draw operation (in association with the [wgpu::RenderPass::set_index_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_index_buffer)).

* **[wgpu::BufferUsage::UNIFORM](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.UNIFORM)** referring to usage as a **UNIFORM** binding within a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html).

* **[wgpu::BufferUsage::STORAGE](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.STORAGE)** referring to usage as a **STORAGE** binding within a [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html).

In addition to these common uses, there are the uses that involve the [wgpu::CommandEncoder](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html) and [wgpu::Queue::write_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html) that require either **[wgpu::BufferUsage::COPY_SRC](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_SRC)** or **[wgpu::BufferUsage::COPY_DST](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.COPY_DST)** (in the case of *copy_buffer_to_buffer*, *copy_texture_to_buffer*, or *write_buffer*). 

Other use cases such as [wgpu::BufferUsage::MAP_READ](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.MAP_READ) refer to allowing a buffer to be read from or written to in the case of [wgpu::BufferUsage::MAP_WRITE](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html#associatedconstant.MAP_WRITE). These scenarios typically involve being able to asynchronously read or write to a buffer that has already been allocated on the device.

### 4.3 Vertex Buffers

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

### 4.4 Example: Drawing a Polygon

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
#[derive(Copy, Clone)]
struct Vertex {
    pos: [f32; 2]
}

unsafe impl bytemuck::Pod for Vertex {}
unsafe impl bytemuck::Zeroable for Vertex {}
```

Notice that in this example, we have added two trait attributes for **Copy**, and **Clone**. Both of these traits are required in order to be used by **bytemuck::Pod** and **bytemuck::cast_slice**. The **repr(C)** also indicates that this will be interpreted as a C-lang style struct as is. All of this will ensure that the **Vertex** struct will be treated as a normal array of bytes when **cast_slice** is executed. Following this, we need to create an actual vertex buffer descriptor that can be applied to **any Vertex**. To do this, we can leverage a **static lifetime** function on the Vertex impl to return the descriptor of the stride, step mode, and attribute layout.

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
        // ... instance, surface, adapter, device, queue
        // ... swap chain
        // .. compile shaders load shader modules
        // ... create render pipeline layout
        let render_pipeline_layout = // device.create_pipeline_layout..
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
        )
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

### 4.4 Input Step Mode and Instancing

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
#[derive(Copy, Clone)]
struct EntityInstance {
    pos: [f32; 2]
}

unsafe impl bytemuck::Pod for EntityInstance {}
unsafe impl bytemuck::Zeroable for EntityInstance {}

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
                    offset: std::mem::size_of::<[f32; 2]>() as wgpu::BufferAddress,
                    shader_location: 2,
                    format: wgpu::VertexFormat::Float
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

### 4.5 Example: Drawing Particle Instances

### 4.6 Textures

> Originally referred to as diffuse mapping, textures are simply image representation of pixels to be mapped for color (diffuse), normals, bump mapping, height maps, displacement, reflections, specular, occlusion, and various other techniques used in a materials system. [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html) is created by the [wgpu::Device::create_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_texture) described by a [wgpu::TextureDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDescriptor.html) including size, mip counts, sample counts, dimensions, [wgpu::TextureFormat](https://docs.rs/wgpu/0.6.0/wgpu/enum.TextureFormat.html) and [wgpu::TextureUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureUsage.html). 

WGPU provides the method to create the texture from the device, but in order to set the actual bytes of the texture you must pass that along to the queue via [wgpu::Queue::write_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_texture).

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

When calling [wgpu::Queue::write_texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.write_texture) you need to pass a wgpu::TextureCopyView](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureCopyViewBase.html) to prepare a buffer image copy based on the [wgpu::TextureDataLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureDataLayout.html) with offset (offset into buffer as the start of the texture), bytes per row (1 row of pixels in x direction, must be multiple of 256 unless using **copy_texture_to_buffer**), and rows per image.

> Note: Many examples will use the [image](https://crates.io/crates/image) crate to process images for a wide variety of formats. A texture/image is loaded, and converted into a `Vec` of rgba bytes used when we pass it into the queue.write_texture.

Writing a texture to the queue effectively creates a single buffer (similar to `create_buffer_init` from before) that downstream operations will have access to when referring to the texture descriptors. A descriptor's usage (like a buffer's usage) can ensure that the texture can be used within a shader (or other various uses) and in this case be able to copy data to it. Additional setup is needed later one to use it with a **wgpu::TextureView** like we've seen in other operations.

### 4.3 Sampler

> [wgpu::Sampler](https://docs.rs/wgpu/0.6.0/wgpu/struct.Sampler.html) defines how a pipeline will sample (i.e. given a pixel coordinate on or outside the texture boundaries - what color should be returned) from [wgpu::TextureView](https://docs.rs/wgpu/0.6.0/wgpu/struct.TextureView.html) by defining image filters (e.g. anisotropy) and address (wrapping) modes (such as in the x, y, z directions), magnified filter, minimized filter - described by [wgpu::SamplerDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.SamplerDescriptor.html).

Sampling occurs when the program provides a coordinate on the texture (texture coordinate) and needs to return a color back based on these configured parameters. When texture coordinates happen to outside of the texture (in the case of address wrapping) the following address wrapping modes are used:

* **[wgpu::AddressMode::ClampToEdge](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.ClampToEdge)** use the nearest pixel on the edges of the texture.
* **[wgpu::AddressMode::Repeat](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.Repeat)** repeats the texture in a tiling fashion.
* **[wgpu::AddressMode::MirrorRepeat](https://docs.rs/wgpu/0.6.0/wgpu/enum.AddressMode.html#variant.MirrorRepeat)** repeates a texture by mirroring it on every repeat (as in mirror when outside boundaries).

```rust
let diffuse_texture_view = diffuse_texture.create_view(&wgpu::TextureViewDescriptor::default());
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

### 4.4 Bind Groups and Layouts

> In order to make use of resources (e.g. Textures, Samplers, Buffers) from within shaders we need a way to reference them. BindGroups and PipelineLayouts provide us with a mechanism for plugging in these resources. [wgpu::BindGroup](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroup.html) represents a set of resources bound to bindings described by a [wgpu::BindGroupLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayout.html). Use the [wgpu::Device::create_bind_group](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group) to create a bind group.

```rust
let texture_bind_group_layout = device.create_bind_group_layout(&wgpu:BindGroupLayoutDescriptor {
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
});

let diffuse_bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
    layout: &texture_bind_group_layout,
    entries: &[
        wgpu::BindGroupEntry {
            binding: 0,
            resource: wgpu::BindingResource::TextureView(&diffuse_texture.view)
        },
        wgpu::BindGroupEntry {
            binding: 1,
            resource: wgpu::BindingResource::Sampler(&diffuse_texture.sampler)
        }
    ]
});
```

## Rust Sections

### ranges

### drop traits

### mut

### &

### ranges

### some, option, none

### structs, types

### match

### functions -> Self

### unwrap

### macros

### move |

### u_time: f32 = 0.0

### *control_flow = 


---
title: Cross-Platform Graphics Programming with WGPU
published: false
---

Following up on a post on [framebuffers, wgpu and rust](/2020/10/08/framebuffers/), and after spending significant time dissecting [gfx-rs/wgpu](https://github.com/gfx-rs/wgpu) and various [gfx-rs/gfx](https://github.com/gfx-rs/gfx) backends, Vulkan portability layers, SPIR-V, and all else - I've been working hard to put together a comprehensive guide to **End-to-End Cross-Platform Graphics Programming with WGPU**. The following book describes the general execution order to working with GPUs and all the typical terminology involved. This guide is a deep dive into WGPU with a high level overview of shader programming in relation to that. Along the way, we will use that knowledge to build simple render pipelines and demos to reinforce what we've learned. By the end of the book, we put it all together and build a simple cross platform game with WGPU.

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
* [wgpu](https://github.com/gfx-rs/wgpu) cross platform rust library for working with Vulkan, Metal, DirectX 11/12, WebGPU, OpenGL
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

## 1.1 Instance

> Context for all other wgpu objects, the primary use of an Instance is to create **[wgpu::Adapter](https://docs.rs/wgpu/0.6.0/wgpu/struct.Adapter.html)** and **[wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html)**.

Considered the entry point to the entire code for working with any graphics library. This includes vulkan's **Instance**, DirectX **IDXGIFactory**, **CAMetalLayer** in Metal, **GPU** in WebGPU.

```rust
let instance = wgpu::Instance::new(wgpu::BackendBit::PRIMARY);
```

Note that [wgpu::BackendBit::Primary](https://docs.rs/wgpu/0.6.0/wgpu/struct.BackendBit.html) is referring to the backends that wgpu will use. Primary refers to first tier api support (Vulkan + Metal + DX12 + WebGPU) with secondary support for OpenGL + DX11. Since OpenGL/DX11 is still experimental, targeting through these backends may be unsupported in some areas. Use **PRIMARY** as the default but you can always just target a specific backend such as **wgpu::BackendBit::VULKAN** for instance.

## 1.2 Surface

> Platform specific surface (e.g. window, vk::Surface, texture back buffer, gpu canvas context) onto which a swap chain may receive a window handle and return back the internal surface for working with the next frame.

In `wgpu` this is referred to by creating a [wgpu::Surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Surface.html) *unsafe* function **[create_surface](https://docs.rs/wgpu/0.6.0/wgpu/struct.Instance.html#method.create_surface)** from a raw window handle.

```rust
// window: winit::window::Window, instance: wgpu::Instance
let surface = unsafe { instance.create_surface(window) };
```

## 1.3 Adapter

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

## 1.4 Logical Device

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

## 1.5 Swap Chain

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

## 1.6 Command Encoder

> [wgpu::CommandEncoder](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html) records [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html)es, [wgpu::ComputePass](https://docs.rs/wgpu/0.6.0/wgpu/struct.ComputePass.html)es, and transfer operations between driver managed resources like [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html)s and [wgpu::Texture](https://docs.rs/wgpu/0.6.0/wgpu/struct.Texture.html)s. When finsihed recording, call [CommandEncoder::finish](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.finish) to obtain a [wgpu::CommandBuffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandBuffer.html) which may be submitted for execution to the GPU.

```rust
let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
    label: Some("RENDER ENCODER") // label in graphics debugger
});
```

## 1.7 Render Pass

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

## (demo) 1.8: Clearing the Screen

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

    let mut state = block_on(State::new(&window));

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

struct State {
    surface: wgpu::Surface,
    device: wgpu::Device,
    queue: wgpu::Queue,
    sc_desc: wgpu::SwapChainDescriptor,
    swap_chain: wgpu::SwapChain,
    size: winit::dpi::PhysicalSize<u32>
}

impl State {
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

    fn resize(&mut self, new_size: winit::dpi::PhysicalSize<u32>) {
        self.size = new_size;
        self.sc_desc.width = new_size.width;
        self.sc_desc.height = new_size.height;
        self.swap_chain = self.device.create_swap_chain(&self.surface, &self.sc_desc);
    }

    fn render(&mut self) {
        let frame = self.swap_chain.get_current_frame()
            .expect("Timeout getting texture")
            .output;

        let mut encoder = self.device.create_command_encoder(&wgpu::CommandEncoderDescriptor {
            label: Some("RENDER ENCODER") // label in graphics debugger
        });

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

## 2.1 Shader Modules

> Shader modules are represented typically by fragment, vertex, and compute shaders (tesselation, geometry, and many others as well) and are run against GPU Hardware Abstraction Layers (HAL) through an intermediary representation [SPIR-V](https://www.khronos.org/registry/spir-v/). [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) reflects SPIR-V in order to disassemble back into high level shading languages such as HLSL or Metal Shading Language (MSL).
> 
> GPU instruction sets are typically highly specialized and specific to each GPU version (and in NVIDIA's case under nda). As such, graphics drivers will work closely with system defined hardware abstraction layers, CPU instructions sets and kernel functionality to translate the right amount of work between both the CPU and GPU hardware specific machine code. SPIR-V is the closest thing anyone will come to "GPU assembly" as an intermediary language for the entire graphics stack.
>
> GPU Vendors that support the Vulkan API (for instance) will load SPIR-V and compile it into GPU specific machine code as it relates to the system being targeted. Within the Microsoft ecosystem, Windows graphics drivers run with D3D11, D3D12, or Vulkan support as vendors implement the HAL apis exposed by the OS and as such SPIR-V can run here as well as loading HLSL or DXIL directly.
>
> Within the Apple ecosystem, SPIR-V must be cross-compiled into Metal Shading Language (MSL) in order to be be run on the Metal platform. SPIRV-Cross is the source of many bugs and much work is being done to reflect any shader language (MSL, HLSL, GLSL versions) back and forth and additionally compile output into rust to abstract away pipeline descriptors and bind groups through the [gfx-rs/naga](https://github.com/gfx-rs/naga) project. 
>
> GLSL comes in many variants, versions, and so it helps that SPIR-V is available to actually cross compile these versions into something of an intermediary representation. Within the Vulkan ecosystem, shaders are written in a Vulkan specific shader language subset [GLSL](https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_vulkan_glsl.txt).

While you can leverage the SPIR-V toolset in the terminal from various shader languages, you can also compile directly from GLSL in rust with [google/shaderc](https://github.com/google/shaderc) using the [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html). The type of shader being loaded must be specified with a [shaderc::ShaderKind](https://docs.rs/shaderc/0.6.2/shaderc/enum.ShaderKind.html). Loading shader modules from SPIR-V bytes can then be done in wgpu with [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module). 

```rust
let vs_src = include_str!("shader.vert");
let fs_src = include_str!("shader.frag");
let mut compiler = shaderc::Compiler::new().unwrap();
let vs_spirv = compiler.compile_into_spirv(vs_src, shaderc::ShaderKind::Vertex, "shader.vert", "main", None).unwrap();
let fs_spirv = compiler.compile_into_spirv(fs_src, shaderc::ShaderKind::Fragment, "shader.frag", "main", None).unwrap();
let vs_module = self.device.create_shader_module(wgpu::util::make_spirv(&vs_spirv.as_binary_u8()));
let fs_module = self.device.create_shader_module(wgpu::util::make_spirv(&fs_spirv.as_binary_u8()));
```

You can also skip the compilation step if it's already been compiled into spirv and load the raw bytes to directly create the shader module. In either scenario, calling [wgpu::Device::create_shader_module](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_shader_module) will return a [wgpu::ShaderModule](https://docs.rs/wgpu/0.6.0/wgpu/struct.ShaderModule.html) which references a [ShaderModuleId](https://docs.rs/wgpu/0.6.0/src/wgpu/lib.rs.html#634) where the shader module will be referenced when executing it on a render pipeline.

> Note: wgpu provides a macro for including spir-v directly with [wgpu::include_spirv!](https://docs.rs/wgpu/0.6.0/wgpu/macro.include_spirv.html) as in `wgpu::include_spirv("shader.vert.spv")`. To do this, you will need to add the build step to ensure that all required shader dependencies in the form of **.vert**, **.frag** or **.comp** are compiled with the [shaderc::Compiler](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html).

## 2.2 Shader Stages


## 2.3 Shader Layouts

In the below example, many GLSL shader examples include a version header to deliniate which level of features we are interested in within GLSL (given how many versions exist). This interpretation is currently what **shaderc** understands.

```glsl
// shader.vert
#version 450

layout(location=0) in vec3 a_position;
layout(location=1) in vec3 a_color;

layout(location=0) out vec3 v_color;

void main() {
    v_color = a_color;
    gl_Position = vec4(a_position, 1.0);
}
```

In order to compile GLSL into SPIR-V a layout must be specified for each input and output. This takes the format of:

```glsl
layout (location = INDEX) DIRECTION TYPE NAME;
```

* **INDEX** is an integer value that must match up with the locations for binding data into GLSL. Within **wgpu** or **gfx-rs** this depends on which part of the pipeline is being described. 

    * **Vertex** data is described with a [wgpu::VertexBufferDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexBufferDescriptor.html) with the [wgpu::VertexAttributeDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html) as the attributes for vertex inputs. For each attribute, you have a **[shader_location](https://docs.rs/wgpu/0.6.0/wgpu/struct.VertexAttributeDescriptor.html#structfield.shader_location)** that must match the location **INDEX** shown above.

    * **Fragment** shaders with `layout(location = 0)` for instance specifies that the values of an assigned variable would be saved to the buffer assigned at location 0 in the application. Usually this is the current texture from the swapchain.

    * Between the shaders in a pipeline the locations and variable outputs from one stage to another need to match the locations and names of the next stage. This would mean that `layout(location=0) out vec3 v_color` in the **shader.vert** would need to match `layout(location=0) in vec3 v_color` in a **shader.frag**.

* **DIRECTION** specifies either `in` or `out`, or `uniform`. The `in` variable specifies that the value comes from the previous stage (such as a single vertex attribute value in the vertex shader or the `v_color` in the fragment shader. The `out` variable goes onto the next stage of the pipeline. `uniform` is a read-only keyword typically used in combination with more complex bindings.

* **TYPE** is a variable type (such as `float`, `int`, `uint`, `vec3`, ..etc)

* **NAME** is the name of the variable

More complex bindings will take advantage of an alternative layout syntax such as:

```glsl
layout(set = 0, binding = 0) uniform texture2D t_diffuse;
layout(set = 0, binding = 1) uniform sampler s_diffuse;
```

Where in the above syntax, **set** and **binding** correspond to [wgpu::BindGroupLayout](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayout.html) as created from [wgpu::Device::create_bind_group_layout](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_bind_group_layout). A bind group (as described in more detail later on) is a descriptor of the resources that will be bound when executing a pipeline. The layout corresponds to the bind group and for each [wgpu::BindGroupLayoutEntry](https://docs.rs/wgpu/0.6.0/wgpu/struct.BindGroupLayoutEntry.html#structfield.binding) there is a binding that must match the **shader index** as shown in the above example with **layout(set = 0, binding = 0)** for example.

```glsl
void main() {
    v_color = a_color;
    gl_Position = vec4(a_position, 1.0);
}
```

Within every shader you will notice that there is an **entry** point function used as a single point of execution. The name of this function, typically **main**, corresponds to the parameter passed to [shaderc::compile_into_spirv](https://docs.rs/shaderc/0.6.2/shaderc/struct.Compiler.html#method.compile_into_spirv) as the **entry_point_name**. 

```rust
let vs_spirv = compiler.compile_into_spirv(vs_src, shaderc::ShaderKind::Vertex, "shader.vert", "main", None).unwrap();
```

## 2.4 Render Pipeline

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

Once a render pipeline has been setup, we can revisit the [wgpu::RenderPass](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html) to set the currently active render pipeline (possibly even one among many pipeline layouts that we are describing within our system) This is done with the [wgpu::RenderPass::set_pipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.set_pipeline). All draw operations such as used by [wgpu::RenderPass::draw](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPass.html#method.draw) (passing along vertices, and instances) will follow the behavior defined by the active [wgpu::RenderPipeline](https://docs.rs/wgpu/0.6.0/wgpu/struct.RenderPipeline.html). 

```rust
render_pass.set_pipeline(&self.render_pipeline);
```

## (demo) 2.5 Rendering a Fragment Shader

Before we move onto to more complex binding layouts, vertex buffers, textures, and samplers - with the above concepts alone we have enough to execute the most basic fragment shader that effectively mimics our clear the screen example. We will take advantage of the existing code from the previous example to setup a render pass, but we will add the following:

* **Write a fragment shader** to output a color
* **Compile/load shader** into spir-v and create a shader module
* **Setup render pipeline** to load the fragment shader as a programmable stage
* **Assign the render pipeline** by setting the render pipeline on the first render pass

```rust

```

Recall that our last call to **[wgpu::CommandEncoder::finish](https://docs.rs/wgpu/0.6.0/wgpu/struct.CommandEncoder.html#method.finish)** finishes recording and returns a command buffer for execution. Calling [wgpu::Queue::submit](https://docs.rs/wgpu/0.6.0/wgpu/struct.Queue.html#method.submit) executes the recorded CommandBuffer on the queue. Once the [wgpu::SwapChainFrame](https://docs.rs/wgpu/0.6.0/wgpu/struct.SwapChainFrame.html) in the scope is [dropped](https://doc.rust-lang.org/std/ops/trait.Drop.html) then the swap chain will present the frame to the back buffer. The result is demonstrated below.


## 3.1 Buffer

> [wgpu::Buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Buffer.html) is a blob of data created from the [Device::create_buffer](https://docs.rs/wgpu/0.6.0/wgpu/struct.Device.html#method.create_buffer) specifying a size, and [wgpu::BufferUsage](https://docs.rs/wgpu/0.6.0/wgpu/struct.BufferUsage.html) stored on the gpu to store anything from graph structures, structs, arrays, vertex data, index data. BufferUsage will limit operations performed on the buffer data and causes a *panic* to throw when used in any way not specified by **BufferInitDescriptor::usage**.

**wgpu** provides a somewhat undocumented api called **create_buffer_init** that is used to create the buffer with data assigned to it. The reason is that the utility method will use a [wgpu::BufferInitDescriptor](https://docs.rs/wgpu/0.6.0/wgpu/util/struct.BufferInitDescriptor.html) to create a **wgpu::BufferDescriptor** and use the device context to effectively allocate a contiguous slice of memory and then copy over the range of data from the contents of the init descriptor. It seems handy to have this kind of utility in the main api but for now it appears to be via a device extension via `use wgpu::DeviceExt` trait.

```rust
use wgpu::DeviceExt;
use zerocopy::AsBytes;

#[derive(AsBytes)]
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
    contents: VERTICES.as_bytes(), // convert to &[u8]
    usage: wgpu::BufferUsage::VERTEX // if buffer is used in any way that isn't specified here, operation panics
});
```

In the above example, I'm using the [zerocopy](https://crates.io/crates/zerocopy) crate to convert the struct into bytes, but I've seen examples that also use [bytemuck](https://crates.io/crates/bytemuck) if you need to serialize into [u8].

## 2.5 Texture

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

## 2.6 Sampler

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

## 2.7 Bind Groups and Layouts

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


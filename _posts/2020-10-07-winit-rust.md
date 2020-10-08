---
title: Winit Rust and Pixels
published: true
---

In every game engine built in [Rust](https://rust-lang.org), the package to use for window management starts off with [winit](https://github.com/rust-windowing/winit). This is because winit provides a cross-platform library for managing windows, handling events (such as window resizing, key press events, mouse movements). The library is meant to be a one of the main building blocks when it comes to building on with other libraries like [wgpu](https://github.com/gfx-rs/wgpu). Here are a few of the features fully supported and within the scope of winit.

### Core Windowing

- **Window Initialization**
- **Pointers for OpenGL, Vulkan**
- **Window decorations**
- **Resizing, resize snaps/increments**
- **Transparency**
- **Maximization, max toggling, minimization**
- **Fullscreen, fullscreen toggling**
- **MiDPI support**
- **Popup / modal windows**
- **Monitor listing**
- **Video mode query**

### Input Handling

- **Mouse events**
- **Mouse set location**
- **Cursor icons, locking cursor**
- **Touch events, touch pressure**
- **Multitouch**
- **Keyboard events**
- **Drag/drop**
- **Raw device events**
- **Gamepad/joystick events**
- **Device movement events**

For an actively updating list of these features and more, see [features.md](https://github.com/rust-windowing/winit/blob/master/FEATURES.md).

## Setting up winit settings

In this article, we're going to take advantage of these features and set up our window in order to prepare it for rendering something with [wgpu](https://github.com/gfx-rs/wgpu). First, setup a new package for our project with `cargo init winit-rust-example`. Then, assuming you have [cargo-edit](https://github.com/killercup/cargo-edit) (if not use, `cargo install cargo-edit`) you can go ahead and add the `winit` package with `cargo add winit`.


```markdown
# TIP

If you are having some trouble with cargo target build directory sizes, you 
can use `export CARGO_TARGET_DIR="$HOME/.cache/cargo"` to set the 
cargo target cache to help free up some compilation space. Note, running 
`cargo clean` will clear out builds for all the projects, but it does 
help a little bit when you need to work on a few of these kind of examples.
```

Now, let's get the window setup with a few things like a title, transparency, and sizing. To do that we need to first setup the `winit::event_loop::EventLoop`.

## EventLoop

The `EventLoop` is essential for any process that needs to interact with messages related to *user interactions like keyboard/mouse events*, *network traffic*, *system processing*, *timer activity*, *ipc communication*, *device control* and in general I/O communication. To do any of this, we need to go through the operating system and each type of operation is dependent on resources that the OS abstracts over (such as hardware resources and peripheral implementations). 

When we ask the OS to perform a blocking operation it will suspend the thread that makes the call (stop executing code and store the CPU state and go onto do other things). When data arrives for us through the network it will wake up our thread again and let us resume. Non-blocking I/O by constrast will not suspend the thread that made the I/O request, and instead give it a handle which the thread can use to ask the OS if the event is ready or not. When we ask the OS using our handle we call that **polling**. Non-blocking I/O gives us more freedom at the cost of more frequent polling (such as in a loop) which can take up CPU time. The methods for hooking into the OS to wait for many events, instead of being limited to waiting on one event per thread, is through [epoll](https://en.wikipedia.org/wiki/Epoll), [kqueue](https://en.wikipedia.org/wiki/Kqueue) and [IOCP](https://en.wikipedia.org/wiki/Input/output_completion_port).

> For an in depth look on these approaches explained with Rust
> [Epoll, Kqueue and IOCP Explained with Rust](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/)

The approach taking in the [gitbook](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/) above, results in the [mio](https://crates.io/crates/mio) crate used by winit. In summary, we get:

* Non-blocking TCP, UDP
* I/O event queue backed by epoll, kqueue, and IOCP
* Platform specific extensions
* Support for (Android, BSD Variants, Solaris, Windows, iOS, macOS, Linux)

The bulk of the work on top of mio in **winit** is getting the dreaded multi-threaded event loop working in such a way that things like [window resizing and blocked threads don't hang](https://github.com/rust-windowing/winit/issues/459) among other issues. In any case, this brief overview should help you understand a bit more about what precisely you need the **EventLoop** for.


```rust
use winit::event::{Event, WindowEvent};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::WindowBuilder;

fn main() {
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
            .with_title("Rust by Example: Winit!")
            .build(&event_loop)
            .unwrap();

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Wait;
        match event {
            Event::WindowEvent {
                event: WindowEvent::CloseRequested,
                ..
            } => *control_flow = ControlFlow::Exit,
            _ => ()
                
        }
    });
}
```

## Input State

Rather than attempt to match every event, it's nicer to look at these events and pass it along to an input state that has processed each and every event for us. This would allow us to determine whether a particular key is pressed, query for mouse changes, current positions...etc. To do this, we're going to use the `winit_input_helper` crate: [winit_input_helper](https://github.com/rukai/winit_input_helper). Add this by `cargo add winit_input_helper`.

```rust
use winit::event::{Event, VirtualKeyCode};
use winit::event_loop::{ControlFlow, EventLoop};
use winit::window::WindowBuilder;
use winit_input_helper::WinitInputHelper;

fn main() {
    let mut input = WinitInputHelper::new();

    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
            .with_title("Rust by Example: Winit!")
            .build(&event_loop)
            .unwrap();

    event_loop.run(move |event, _, control_flow| {
        if input.update(&event) {
            if input.key_released(VirtualKeyCode::Escape) || input.quit() {
                *control_flow = ControlFlow::Exit;
                return;
            }

            let mouse_diff = input.mouse_diff();
            if mouse_diff != (0.0, 0.0) {
                println!("Mouse diff is: {:?}", mouse_diff);
                println!("Mouse position is: {:?}", input.mouse());
            }
        }
    });
}
```

Note the mouse position state in the console when you move around!

## Pixels

Before I decide to move onto a more complicated library like [wgpu](https://github.com/gfx-rs/wgpu) I'm taking a look at this neat library called [pixels](https://github.com/parasyte/pixels).

> Pixels is a way to rapidly prototype a simple 2D game, pixel-based animations, software renderers, or an emulator for your favorite platform. Then add shaders to simulate a CRT or to spice it up with some nice VFX. It's more of a library than a framework which allows you to have full control of the event loop, window environment, and input handling.

Out of anything I've come across, this library is very promising and fun to use! Best of all, it is built on top all the incredible work from [wgpu](https://github.com/gfx-rs/wgpu). We can modify our example to get pixels working by first making sure we have pixels with `cargo add pixels`.

Then go ahead and import `pixels`.

```rust
use pixels::{Error, Pixels, SurfaceTexture};
```

We're going to use the `SurfaceTexture` in order to draw on as our main texture (in terms of an actual texture in wgpu). Then we simply create a Pixels instance based on this texture and add our draw code while passing along the raw frame bytes we want to manipulate.

```rust
let window_size = window.inner_size();
let surface_texture = SurfaceTexture::new(window_size.width, window_size.height, &window);
let mut pixels = Pixels::new(320, 240, surface_texture)?;
```

Inside the event loop, we will use the `Event::RedrawRequested` to see whether we can actually perform this draw operation on this frame. 

```rust
if let Event::RedrawRequested(_) = event {
    draw(pixels.get_frame());
    if pixels.render().is_err() {
        *control_flow = ControlFlow::Exit;
        return;
    }
}
```

While at the end of our event loop to actually request the redraw itself.

```rust
window.request_redraw();
```


Finally, our draw code is simply going to use the raw bytes to manipulate the pixels.

```rust
fn draw(frame: &mut [u8]) {
    for (i, pixel) in frame.chunks_exact_mut(4).enumerate() {
        let x = (i % 320 as usize) as i16;
        let y = (i / 320 as usize) as i16;

        let inside = x >= 10 && x < 110
            && y > 20 && y < 120;

        let rgba = if inside {
            [0x5e, 0x99, 0x39, 0xff]
        } else {
            [0x48, 0xb2, 0xe8, 0xff]
        };

        pixel.copy_from_slice(&rgba);
    }
}
```

![WINIT Pixels](/assets/rust-by-example-winit-pixels.png)

Very exciting stuff! The [examples on pixels](https://github.com/parasyte/pixels) make it really easy to get started including an [invaders clone](https://github.com/parasyte/pixels/tree/master/examples/invaders). I will definitely be using this library to create some quick games out of just manipulating the pixel frame. Great stuff!

---
title: "Writing a Software Rasteriser: 1.Acquiring the Framebuffer"
---

Now that the maths library is done a way to draw to the screen is necessary. Preferably our method of drawing to the screen would be something similar to that available before graphics cards were commonplace, CPU bound.

Linux provides an abstraction over the graphics device called `fbdev`, short for framebuffer device, which allows you to draw to the screen by simply writing to a raw file-backed framebuffer, a big dump of bytes which map directly to pixel brightness values. It is available on nearly any Linux device with a screen and is drawn to by the CPU, pixel by pixel. If you want to follow along on another platform you'll need to come up with your own method of plotting pixels to the screen, SDL for example.

You have probably already seen `fbdev` in action if you use Linux, it is what the console (often referred to as *"the tty"*) uses these days. We will make a new file in our repo for interacting with it; create `src/fb.rs`. Again, import it in `lib.rs` so we get doc tests and Rust Language Server support.

~~~rust
mod fb;
pub use fb::*;
~~~

Now in `fb.rs` we create the structure which we will use to safely manage the framebuffer.

~~~rust
pub struct Framebuffer {

}
impl Framebuffer {

}
~~~

To access the framebuffer we open its file, `/dev/fb0` then memory map the file using the `mmap` system call. In C there is a stdlib function for this, but not in Rust so we can do one of the following:
- Call the C functions directly in Rust.
- Write our logic to open the framebuffer in C.
- Write our own function which performs the system call.
- Use a crate which exposes `mmap` to Rust.

Using someone elses crate is no fun so we can just dive into unsafe! Making a system call requires some assembly which can only be done with nightly Rust. Calling the C function in Rust sounds simple, but it would require rewriting all the headers in Rust before we can actually call them. Our best bet is to write the logic in C then call a few simpler functions. [The nomicon explains](https://doc.rust-lang.org/nomicon/ffi.html) the process of calling an FFI function.

We will declare the functions in Rust before moving over to define them in C. We add `#[repr(C)]` to the structure to ensure it has the same memory layout as the one we will define in C. The creation of the framebuffer could also fail for a number of reasons so we must be able to indicate when file creation fails, the easiest way of which is to set buffer to null in C and return early with a partially filled struct. We must ensure buffer is not null before looking at any other fields or else we would encounter undefined behaviour. Our struct in both languages must match with equivalent types at all times to ensure we do not experience undefined behaviour, this applies to the `extern "C"` functions too. To move the manual memory management of C into the [RAII world of Rust](https://doc.rust-lang.org/rust-by-example/scope/raii.html) we use the `Drop` operator to cleanup when Rust is ready for us to.

~~~rust
struct Framebuffer;

#[link(name = "fb", kind = "static")]
extern "C" {
    fn fb_create() -> Framebuffer;
    fn fb_destroy(this: *mut Framebuffer);
}

#[repr(C)]
pub struct Framebuffer {
    buffer: *mut u32,
    buffer_len: usize,
    bytes_per_pixel: u32,
    red_offset: u32,
    green_offset: u32,
    blue_offset: u32,
    x_offset: u32,
    y_offset: u32,
    line_length: u32
}
impl Framebuffer {
    pub fn new() -> Option<Self> {
        let this = unsafe { fb_create() };
        if this.buffer.is_null() {
            None
        } else {
            Some(this)
        }
    }
}
impl Drop for Framebuffer {
    fn drop(&mut self) {
        unsafe { fb_destroy(self) }
    }
}
~~~

Cargo allows you to create a [nifty little build script](https://doc.rust-lang.org/cargo/reference/build-scripts.html) that it will run before compiling the Rust program, perfect for building our C component. Create `build.rs` in the project root, **not** in `src/`.

~~~rust
fn main() {
    println!("cargo:rerun-if-changed=src/fb.c");
    cc::Build::new()
        .warnings(true)
        .file("src/fb.c")
        .compile("fb");
}
~~~

[`cc`](https://crates.io/crates/cc) is a crate for compiling and linking C parts to a Rust project exactly like in our usecase. We only need it during compilation, not as part of our program, so we include it as a `build-dependency` rather than a normal `dependency` in `Cargo.toml`.

~~~toml
[build-dependencies]
cc = "1.0"
~~~

We should also go back and add a doc-comment and test.

~~~rust
/// Provides access to the linux framebuffer device.
/// Requires that the user is in the `video` group.
/// ```rust
/// use tendon::*;
/// let fb = Framebuffer::new();
/// ```
pub struct Framebuffer {...}
~~~

Try running cargo test now and you will see most tests somehow pass even though we haven't defined the functions yet! This is possible because Rust will optimise out all of the `Framebuffer` structure code **before** linking to the C code for all of the other unrelated tests. As expected, the `Framebuffer` test fails with a linking error.

To fix this we define the functions in `src/fb.c`.

~~~c
#include <fcntl.h>
#include <linux/fb.h>
#include <sys/mman.h>

struct fb {
    unsigned* buffer;
    size_t buffer_len;
    unsigned bytes_per_pixel;
    unsigned red_offset;
    unsigned green_offset;
    unsigned blue_offset;
    unsigned x_offset;
    unsigned y_offset;
    unsigned line_length;
};

struct fb fb_create() {
    struct fb fb = {};
    return fb;
}
void fb_destroy(struct fb* fb) {

}
~~~

Running the test now you will see it pass.

To actually map the framebuffer to memory we need to `open` the file, get the size of the buffer and other information using an `ioctl` then `mmap` the buffer into our programs virtual memory. [This post](http://betteros.org/tut/graphics1.php) also covers the process in great depth.

I'll be staying inside the `fb_create()` function for now. We open the file, checking for an error and returning with an error flag as previously discussed.

~~~c
struct fb fb_create() {
    int fb_file = open("/dev/fb0", O_RDWR);
    if (fb_file < 0) {
        fb.buffer = 0;
        return fb;
    }
~~~

Then we get information about the framebuffer so that we know its size and format. Pixels can be represented in many different ways. The order of the colour channels, the number of colour channels and the size of each channel in bits can all change, if the framebuffer were backed by a printer or other unique device we might not even use the additive RGBA colours we are used to on computers, we could very well end up with a CMY framebuffer. We will only focus on RGB colours since that is what we are most likely to encounter. We also try changing to a format with 32 bit colour for 1 byte per channel.

~~~c
    struct fb_fix_screeninfo fix_info;
    struct fb_var_screeninfo var_info;
    ioctl(fb_file, FBIOGET_FSCREENINFO, &fix_info);
    ioctl(fb_file, FBIOGET_VSCREENINFO, &var_info);

    var_info.bits_per_pixel = 32;
    var_info.grayscale = 0;
    if (ioctl(fb_file, FBIOPUT_VSCREENINFO, &var_info) < 0)
        // Failed to change to the desired value so reload
        ioctl(fb_file, FBIOGET_VSCREENINFO, &var_info);
    
    fb.bytes_per_pixel = var_info.bits_per_pixel / 8;
    fb.red_offset = var_info.red.offset;
    fb.green_offset = var_info.green.offset;
    fb.blue_offset = var_info.blue.offset;
    fb.x_offset = var_info.xoffset;
    fb.y_offset = var_info.yoffset;
    fb.line_length = fix_info.line_length / sizeof(unsigned);
    fb.buffer_len = fix_info.smem_len;
~~~

With the framebuffer info saved we can map the file then close the file descriptor. We then end the `fb_create()` function then clean up the buffer in `fb_destroy()`.

~~~c
    void* error;
    fb.buffer = error = mmap(NULL, fb.buffer_len, PROT_WRITE, MAP_SHARED, fb_file, 0);
    if (error == MAP_FAILED)
        fb.buffer = 0;

    return fb;
}

void fb_destroy(struct fb* fb) {
    munmap(fb->buffer, fb->buffer_len);
}
~~~

Back in the world of Rust we can now implement a function to allow us to correctly index the framebuffer. We can't just directly reinterpret it as a 2d array as there are areas in the framebuffer that are offscreen, be that for [blanking](https://en.wikipedia.org/wiki/Vertical_blanking_interval) or otherwise. I would've liked to get a little bit creative with overloading the `Index` operator however due to restrictions with generics it you can only return a reference which means that without doing some seriously unsafe and dodgy stuff you can only return fields of the structure. We set the pixel by indexing the `u32` at the offset y position multiplied by the horizontal length of a line plus the offset x position.

~~~rust
    pub fn set(&mut self, x: usize, y: usize, colour: Colour) {
        let pos = (x + self.x_offset as usize) + (y + self.y_offset as usize) * self.line_length as usize;
        unsafe {*self.buffer.add(pos) = *colour }
    }
~~~

Everything looks ok, but now there is a new `Colour` type we haven't yet defined. Without this type we would be writing a raw `u32` whose byte order depends on the system and does not necessarily align with the layout of the colour channels. We can instead construct a `Colour` which we know to be formatted correctly using the `{red,green,blue}_offset` fields.
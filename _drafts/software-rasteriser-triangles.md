---
title: "Writing a Software Rasteriser: 2.Triangle Rasterisation"
---

# Memory Safety is Important
Before we get started I let us talk about memory safety! In the last post we used a little bit of `unsafe` to interface with C and to index the framebuffer. Unsafe code is well, unsafe, so as a good Rust programmer you must be very sure that when you use `unsafe`, you guarantee program correctness. Were you looking really carefully in the last post? When we indexed the buffer last time, we used the `ptr::add()` function which is marked unsafe. Whenever you use an unsafe function you should [double check the documentation](https://doc.rust-lang.org/std/primitive.pointer.html#method.add) to see **why** it is unsafe and ensure that you can guarantee you are not breaking any of the rules assumed for safe usage.

I recommend that when you use `unsafe` you leave a comment to say why your usage is still considered sound. Let us do that now for `Framebuffer::set()`, addressing the three dot points the documentation states.

~~~rust
    pub fn set(&mut self, x: usize, y: usize, colour: Colour) {
        let pos = (x + self.x_offset as usize) + (y + self.y_offset as usize) * self.line_length as usize;
        // Safety: ???
        // Safety: ???
        // Safety: We do not rely on wrapping
        unsafe {*self.buffer.add(pos) = colour.convert(self) }
    }
~~~

Uhhhh... We broke two of the rules specified by the function! If you draw to `8000x8000` you will probably set some memory outside of the framebuffer which is very much undefined behaviour, a buffer overflow! We should start by ensuring that we do not write out of bounds.

~~~rust
    pub fn set(&mut self, x: usize, y: usize, colour: Colour) {
        let pos = (x + self.x_offset as usize) + (y + self.y_offset as usize) * self.line_length as usize;
        if pos >= self.buffer_len {
            panic!("Pixel ({}, {}) is out of framebuffer bounds", x, y)
        }
        // Safety: We have checked that the buffer access is in-bounds
        // Safety: ???
        // Safety: We do not rely on wrapping
        unsafe {*self.buffer.add(pos) = colour.convert(self) }
    }
~~~

But what about that last guarantee? Well on x86_64 we are fine since `pos` is now limited by `buffer_len` which originiated from a `u32` and is therefore never going to overflow an `isize` since x86_64 has a virtual address space of 48 or 52 bits, leaving us quite a bit of extra room. There is still the (slight) possibilty of breaking this guarantee on other platforms so we should check it just in case on other platforms. To ensure we don't waste on time on the unecessary check on x86_64 we use [conditional compilation](https://doc.rust-lang.org/reference/conditional-compilation.html#target_arch).

~~~rust
    pub fn set(&mut self, x: usize, y: usize, colour: Colour) {
        let pos = (x + self.x_offset as usize) + (y + self.y_offset as usize) * self.line_length as usize;
        if pos >= self.buffer_len {
            panic!("Pixel ({}, {}) is out of framebuffer bounds", x, y)
        }
        #[cfg(not(target_arch="x86_64"))]
        if pos + self.buffer as usize >= isize::MAX {
            panic!("Framebuffer cannot be indexed due to platform addressing constraints")
        }
        // Safety: We have checked that the buffer access is in-bounds
        // Safety: Result does not overflow an isize
        // Safety: We do not rely on wrapping
        unsafe {*self.buffer.add(pos) = colour.convert(self) }
    }
~~~

Alright, now it will crash the program when we can't uphold the guarantees rather than just causing undefined behaviour which could cause something much worse than a crash.

# Drawing Triangles

With that out of the way it is time to draw our first triangle. First, let us add the notion of a triangle to our maths library since we might want to do higher-level operations on them in the future. For now it will be a simple wrapper class over an array of `Vector2`, nothing new.

~~~rust
pub struct Tri(pub [Vector2; 3]);
impl Deref for Tri {
    type Target = [Vector2; 3];
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
impl DerefMut for Tri {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}
~~~

Now we can add our drawing method to `Framebuffer`.

~~~rust
pub fn draw_tri(&mut self, tri: Tri) { 
    // What now?
}
~~~

There are a few ways to draw a triangle, lets way up the **pro**s and **con**s of a some.

The simplest one the has probably come to your mind is to just to fill the box around the triangle from `(min(x), min(y))` to `(max(x), max(y))` and test each pixel for being inside the triangle.
![Drawing a triangle by drawing a box and discarding or keeping regions that fall inside the triangle](/blog/assets/triangle_discard_keep.png)

This approach is simple and only draws over each pixel once, however, it wastes a lot of time checking unused pixels and the check itself is not very cheap either.


We may also follow the line down by stepping down by the gradient each time, a simple addition, and then draw that line horrizontally across the screen.
![Drawing a triangle by going through each scanline](/blog/assets/triangle_scanlines.png)

This approach is slightly more complicated and requires maybe segmenting the triangle, but it does draw each pixel exactly once, and only if it is actually inside the triangle.

Another approach would be to draw lines from point `a` to progressively accross the line `BC`. The line drawing algorithm used is generally [Bresenham's algorithm](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm).
![Drawing a triangle by drawing lines progressively](/blog/assets/triangle_bresenhams.png)

As is obvious there is a lot of overdraw near the convergence point. Drawing a pixel more than once is not much of an issue now, sure, but in the future if we want to draw slightly transparent objects and do alpha blending then the overdraw would mean that at best we need a second buffer to allow the tracking of draws so we can tell if it is one of the extra draws or not.

We will take the second option since the only compromise is to our laziness. If you want more information on these three approaches [this page does a decent job explaining things](http://www.sunshine2k.de/coding/java/TriangleRasterization/TriangleRasterization.html). My approach to their *"standard algorithm"* is slightly different as I don't completely split the triangle into two but the idea is the same.

The first step is to sort the points by height so that we can make certain assumptions, such as which edge is the highest and which point is in the middle. All of the code for now is inside the `Framebuffer::Draw_tri()` method.

~~~rust
let mut a = 0;
let mut b = 1;
let mut c = 2;
if tri[a].y > tri[b].y {
    std::mem::swap(&mut a, &mut b)
}
if tri[a].y > tri[c].y {
    std::mem::swap(&mut a, &mut c)
}
if tri[b].y > tri[c].y {
    std::mem::swap(&mut b, &mut c)
}
~~~

Because of borrowing rules we can't swap elements of an array directly, since that would be a mutable access to the same object more than once. Indexing by `a`, `b` and `c` is more clear anyway. Next we calculate the vectors for each edge as well as the location of `b` on the highest edge by the same height. We get the proportion of the height of the top edge to the hight of high edge, `top_edge.y / high_edge.y`, next we travel along the x position of the high edge by that proportion, `* high_edge.x` before translating it back to a point, rather than a line, `+ tri[a].x`. The way in which we travel down the lines also depends on the handedness of the triangle, so we determine if its high edge is on the left or right of the other edges and midpoint.

~~~rust
let high_edge = tri[c] - tri[a];
let top_edge = tri[b] - tri[a];
let bottom_edge = tri[c] - tri[b];
let midpoint_x = tri[a].x + top_edge.y / high_edge.y * high_edge.x;
let left_triangle = midpoint_x < tri[b].x;
~~~

Next we actually need a way to travel down the line. The easiest and most efficient way is to just calculate the amount to go across by for each step down, or the inverse gradient. This does however introduce an additive floating point error and a bit of innacuracy which may make our triangle look a little funny in the extremes, but it is worth it since the alternative includes heavy computations for every pixel. The direction we step depends on if it is a left or right triangle so we switch to match. We also only want to draw portion above the midpoint if there actually is a top portion.

~~~rust
if tri[a].y as usize != tri[b].y as usize {
    let (l_grad, r_grad) = if left_triangle {
        (high_edge.inverse_gradient(), top_edge.inverse_gradient())
    } else {
        (top_edge.inverse_gradient(), high_edge.inverse_gradient())
    };
~~~

We didn't think to add an inverse gradient function back when we created our maths library so we will do so once we are done here. Luckily it is super simple.
To draw down to the mid point we then iterate through the y coordinates from point `a` to `b` and draw each line by plotting pixels from the `x_start` to the `x_end` then moving them by the `l_grad` and `r_grad` gradients respectively.

~~~rust
let mut x_start = tri[a].x;
let mut x_end = x_start;
for y in tri[a].y as usize .. tri[b].y as usize {
    for x in x_start as usize .. x_end as usize {
        self.set(x, y, Colour(0xFF0000FF))
    }
    x_start += l_grad;
    x_end += r_grad;
}
~~~

To now draw the lower half of the triangle we reset the start and end points, for both aleviating error and incase the top half didn't have any height, calculate the new gradient then do the nested loop with gradient updates again. Once again, we only do so if the lower half of the triangle exists.

~~~rust
if tri[b].y as usize != tri[c].y as usize {
    let mut x_start = tri[b].x;
    let mut x_end = midpoint_x;
    if left_triangle {
        std::mem::swap(&mut x_start, &mut x_end)
    }
    let (l_grad, r_grad) = if left_triangle {
        (high_edge.inverse_gradient(), bottom_edge.inverse_gradient())
    } else {
        (bottom_edge.inverse_gradient(), high_edge.inverse_gradient())
    };
    for y in tri[b].y as usize ..= tri[c].y as usize {
        for x in x_start as usize .. x_end as usize {
            self.set(x, y, Colour(0xFF0000FF))
        }
        x_start += l_grad;
        x_end += r_grad;
    }
}
~~~

And that should be enough to draw a triangle. Let's add the `Vector2::inverse_gradient()` method then create a quick test scene.

~~~rust
    /// The inverse of the gradient of the line given by the run divided by the rise of the line
    /// ```rust
    /// use tendon::*;
    /// let v = Vector2 { x: 8.0, y: 4.0 };
    /// let dif = v.inverse_gradient() - 2.0;
    /// assert!(dif.abs() < 1e-10);
    /// ```
    pub fn inverse_gradient(self) -> f64 {
        self.x / self.y
    }
~~~

I hope you have been doing your documentation and testing!

Now lets actually show it to the screen. In `src/main.rs` we can open the framebuffer then draw a triangle.

~~~rust
mod fb;
mod maths;

use maths::*;

fn main() {
    let mut fb = fb::Framebuffer::new().unwrap();
    fb.draw_tri(Tri([
        Vector2 { x: 150.0, y: 50.0 },
        Vector2 { x: 75.0, y: 100.0 },
        Vector2 { x: 200.0, y: 130.0 },
    ]));
}
~~~

To see it we need to actually be in the tty so that our window manager doesn't draw over it. If we execute `cargo run` now we will see our little triangle. If you have errors make sure you are in the video group so that you have permission to open `/dev/fb0`.
![a not-so-pretty triangle](/blog/assets/not_so_pretty_triangle.png)

What a perfectly fine triangle. It isn't too pretty though, we haven't considered colours or textures yet. For our game we will only be wanting textured surfaces, which saves us having to implement different draw_tri functions, our equivalent to fragment shaders. For a textured surface you need two things, texture coordinates, normally named `UV`s and a way to get a pixel colour from a texture, normally called a texture sampler. But before that you were probably wondering how to take a screenshot of your triangle, so let's do that first.

First we need a way to `Framebuffer::get` a pixel value from the framebuffer, this is more or less the same as `Framebuffer::set`.

~~~rust
pub fn get(&self, x: usize, y: usize) -> u32 {
    let pos = (x + self.x_offset as usize) + (y + self.y_offset as usize) * self.line_length as usize;
    if pos >= self.buffer_len {
        panic!("Pixel ({}, {}) is out of framebuffer bounds", x, y)
    }
    #[cfg(not(target_arch="x86_64"))]
    if pos + self.buffer as usize >= isize::MAX {
        panic!("Framebuffer cannot be indexed due to platform addressing constraints")
    }
    unsafe { *self.buffer.add(pos) }
}
~~~

We also need to convert the colour to useable RGB components, again this is just the opposite of our previous experience in `Colour`. I'm using an associated function so that we don't conflate a `Colour` with a raw `u32` value.

~~~rust
pub fn u32_to_rgb(colour: u32, fb: &Framebuffer) -> (u8, u8, u8) {
    (
        (colour >> fb.red_offset & 0xFF) as u8,
        (colour >> fb.green_offset & 0xFF) as u8,
        (colour >> fb.blue_offset & 0xFF) as u8
    )
}
~~~

You should add a doc comment and test to that. Next we need a way to save to a common file format. Writing an image encoder is very much out of scope for us, and there is a crate that can do it much better than us anyway so let's just add `image` to our `Cargo.toml`

~~~toml
[dependencies]
image = "0.23"
~~~

Make sure to use the latest version available to you and make any necessary changes when using the library. Finally we dump the image to disk by creating an image buffer then converting each pixel in the framebuffer to the matching RGB values and setting them in the image. `image` can save the image buffer to disk for us, we just need to propogate the potential errors to our future selves.

~~~rust
pub fn dump<P: AsRef<Path>>(&self, path: P) -> image::ImageResult<()> {
    let mut img = image::ImageBuffer::new(1366, 768);
    for (x, y, pixel) in img.enumerate_pixels_mut() {
        let (r, g, b) = Colour::u32_to_rgb(self.get(x as _, y as _), self);
        *pixel = image::Rgb([r, g, b])
    }
    img.save(path)
}
~~~

Now we already have `image` which will be useful when we want to load textures for texture mapping in a moment. Texture mapping is done by interpolating between texture coordinates at each vertex of the triangle. This means we need another 3 `Vector2`'s to store all that information. Aditionally we need that afformentioned sampler, so let's edit the definition of `Framebuffer::draw_tri()`

~~~rust
pub fn draw_tri(&mut self, tri: Tri, uvs: Tri, sampler: &Sampler) {
    // ...
}
~~~
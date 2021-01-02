---
layout: post
title: "Writing a Software Rasteriser: 0.Vectors"
---

# Introduction

The single most expensive component of your PC is *probably* your graphics card. Despite this, you mainly use it for one thing: computer graphics. So what makes it so special?
What your GPU is mostly doing is drawing things to the screen, in batches to make it much faster, doing so by drawing triangles.
The rasterisation, converting to a flat image of pretty pixels, of triangles is what the GPU excels at and is what allows it to display 2D games and interfaces as well as complex 3D scenes.
Our goal is to replace the GPU with our own replacement which may not be nearly as fast, but a good learning experience as well as a challenge, though it shouldn't be hard as id Software have already done it with much, much slower hardware and without all of the problems solved for them. Nontheless, we will manage to struggle along the way, as that is how we learn.

By the way, I'll be using Rust. It shouldn't be too hard to follow even if it is your first time with it but language doesn't matter much for concepts anyway so don't worry if it looks like gibberish to you.

The repo is available at [https://github.com/AidoP/tendon](AidoP/tendon).

# Vectors, Matrices and Maths

To start our journey we need a solid maths library behind our beneath our feet to prevent us falling too far. Computer graphics is fairly maths heavy, but for the basics we only need to understand Vectors and Matrices.

Start a new project by creating a directory then using `$ cargo init`{:.language-shell} inside. Now we should add a `src/lib.rs` file on the offchance we want to reuse components of our program, such as the maths library.
The real reason you should do this will come about later. `lib.rs` needs to know where to find our future maths module so we add, and re-export it as so:

~~~rust
mod maths;
// Re-export for ease of use later
// You probably shouldn't do this in a real library
pub use maths::*;
~~~

We can get into it now by creating `src/maths.rs` and adding our first structure, a 3 dimensional vector. 

~~~rust
#[derive(Copy, Clone, Debug)]
pub struct Vector3 {
    pub x: f64,
    pub y: f64,
    pub z: f64
}
~~~

You could also use f32 instead as we shouldn't be needing the extra precision, but the cost is inconsequential on my 64 bit machine so why not use a double precision float instead. Alternatively, you may use generics, or simpler yet, just use a `type` alias like you would a `const` or `#define`. Our lovely vector structure isn't very useful though so let's add some basic functions we might need.

~~~rust
impl Vector3 {
    pub fn magnitude(self) -> f64 {
        f64::sqrt(self.x.powi(2) + self.y.powi(2) + self.z.powi(2))
    }
    pub fn normal(self) {
        let f = 1.0 / self.magnitude();
        Self {
            x: self.x * f,
            y: self.y * f,
            z: self.z * f
        }
    }
}
~~~

Since the struct is small and `Copy`, which means it is a fast bit copy like a simple primitive, it is probably best to keep it pass by value. For our Matrix structs this will not be the case however.

Because we are **good** little programmers we are now going to go back and **document** our structure and its functions. Rust lets us go a step further by letting us write tests directly in our doc-comments. That means we get a neat little example **and** a functional test in one neat package. Tests are also important as they ensure our functions work as intended and that no one (especially not us!) comes along and introduces a regression which could be very difficult to find indeed. These tests only run for libraries which is why we added a `lib.rs` earlier.

~~~rust
/// A 3-dimensional Vector with f64 components
/// ```rust
/// use tendon::*;
/// let v = Vector3 { x: 0.0, y: 0.0, z: 0.0 };
/// ```
pub struct Vector3 {

    /// The length of a vector
    /// ```rust
    /// use tendon::*;
    /// let v = Vector3 { x: 3.0, y: 4.0, z: 5.0 };
    /// let dif = v.magnitude() - 50.0f64.sqrt();
    /// assert!(dif < 1e-10);
    /// ```
    pub fn magnitude(self) -> f64 {
~~~

And so on... Please keep in mind that [https://bitbashing.io/comparing-floats.html](floats cannot be compared) using `==` without the sky falling down which is why I've taken the same approach as the Rust stdlib team in ensuring they are close enough, or in other words, the difference between the result and the expected result are very close indeed. If you use `$ cargo test` now it should display something like ![cargo test results](/assets/cargo_test_success.png).

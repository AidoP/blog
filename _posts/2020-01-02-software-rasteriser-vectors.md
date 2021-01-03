---
title: "Writing a Software Rasteriser: 0.Vectors"
---

# Introduction

The single most expensive component of your PC is *probably* your graphics card. Despite this, you mainly use it for one thing: computer graphics. So what makes it so special?
What your GPU is mostly doing is drawing things to the screen, in batches to make it much faster, doing so by drawing triangles.
The rasterisation, converting to a flat image of pretty pixels, of triangles is what the GPU excels at and is what allows it to display 2D games and interfaces as well as complex 3D scenes.
Our goal is to replace the GPU with our own replacement which may not be nearly as fast, but a good learning experience as well as a challenge, though it shouldn't be hard as id Software have already done it with much, much slower hardware and without all of the problems solved for them. Nontheless, we will manage to struggle along the way, as that is how we learn.

By the way, I'll be using Rust. It shouldn't be too hard to follow even if it is your first time with it but language doesn't matter much for concepts anyway so don't worry if it looks like gibberish to you.

The repo is available at [AidoP/tendon](https://github.com/AidoP/tendon).

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

We can get into it now by creating `src/maths.rs` and adding our first structure, a 3 dimensional vector. Rather than storing the length and direction of the vector directly we will store them together by representing the vector in terms of its dimensional `x`, `y` and `z` components. Unsurprisingly this sort of vector is named a component vector.

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

Since the struct is small and `Copy`, which means it is a fast bit copy like a simple primitive, it is probably best to keep it pass by value. For our Matrix structs this will not be the case however. Normalisation of a vector means scaling it so that its length is 1.0 while keeping its direction. Such *direction vectors* will be necessary for a variety of operations and will be used whenever a simple direction needs to be stored. A vector with a length of one is named a unit vector.

Because we are **good** little programmers we are now going to go back and **document** our structure and its functions. Rust lets us go a step further by letting us write tests directly in our doc-comments. That means we get a neat little example **and** a functional test in one neat package. Tests are also important as they ensure our functions work as intended and that no one (especially not us!) comes along and introduces a regression which could be very difficult to find indeed. These tests only run for libraries which is why we added a `lib.rs` earlier.

~~~rust
/// A 3-dimensional Vector with f64 components
/// ```rust
/// use tendon::*;
/// let v = Vector3 { x: 0.0, y: 0.0, z: 0.0 };
/// ```
pub struct Vector3 { ... }

impl Vector3 {
    /// The length of a vector
    /// ```rust
    /// use tendon::*;
    /// let v = Vector3 { x: 3.0, y: 4.0, z: 5.0 };
    /// let dif = v.magnitude() - 50.0f64.sqrt();
    /// assert!(dif < 1e-10);
    /// ```
    pub fn magnitude(self) -> f64 {
~~~

And so on... Please keep in mind that [floats cannot be compared](https://bitbashing.io/comparing-floats.html) using `==` without the sky falling down which is why I've taken the same approach as the Rust stdlib team in ensuring they are close enough, or in other words, the difference between the result and the expected value is minimal. If you use `$ cargo test` now it should display something like
![cargo test results](/blog/assets/cargo_test_success.png)

The vector structure is still not very useful without the basic math operators implemented. They can be imported from `std::ops`. Useful operations include addition, subtraction, multiplication and division by a scalar value (a single float). Only addition and subtraction should be defined for the other operand being a vector which we will discuss in a moment. Doc-comments will not apply for implementing traits but tests can still be written in the same way.

At the top of `maths.rs` add

~~~rust
use std::ops::{Add, AddAssign, Sub, SubAssign, Mul, MulAssign, Div, DivAssign};
~~~

~~~rust
impl Add<f64> for Vector3 {
    type Output = Self;
    fn add(self, scalar: f64) -> Self {
        Self {
            x: self.x + scalar,
            y: self.y + scalar,
            z: self.z + scalar
        }
    }
}
impl Add for Vector3 {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        Self {
            x: self.x + rhs.x,
            y: self.y + rhs.y
            z: self.z + rhs.z
        }
    }
}
~~~

You get the idea. The reason why we only implement `Mul<f64>` and **not** `Mul<Vector3>` is because there are two types of multiplication for vectors, dot product and cross product. You could implement it in the same way we did for addition and subtraction but that would give us component-wise scaling, a less useful operation, but we would be better off making that a function rather than overloading multiplication and giving us a 3rd potentially confusing, multiplication definition.

In the original `impl Vector3` we should add dot and cross product. Dot product returns a scalar value rather than a vector is useful as among other tricks, it can be used to derive the angle between two vectors where `a.dot(b)` is 0 when `a` and `b` are perpendicular one another and 1 when they are parallel, for unit vectors. Dot product for a component vector is simply `a.x * b.x + a.y * b.y + a.z * b.z`.

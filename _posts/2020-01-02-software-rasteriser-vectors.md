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

To start our journey we need a solid maths library beneath our feet to prevent us falling too far. Computer graphics is fairly maths heavy, but for the basics we only need to understand Vectors and Matrices.

Start a new project by creating a directory then using `$ cargo init`{:.language-shell} inside. Now we should add a `src/lib.rs` file on the offchance we want to reuse components of our program, such as the maths library.
The real reason you should do this will come about later. `lib.rs` needs to know where to find our future maths module so we add, and re-export it as so:

~~~rust
mod maths;
// Re-export for ease of use later
// You probably shouldn't do this in a real library
pub use maths::*;
~~~

## Vectors

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

The vector structure is still not very useful without the basic math operators implemented. They can be imported from `std::ops`. Useful operations include addition, subtraction, multiplication and division by a scalar value (a single float). Only addition and subtraction should be defined for the other operand being a vector which we will discuss in a moment. Doc-comments will not apply for implementing traits but tests can still be written in the same way if you want to be very thorough.

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
    fn add(self, vector: Self) -> Self {
        Self {
            x: self.x + vector.x,
            y: self.y + vector.y
            z: self.z + vector.z
        }
    }
}
~~~

You get the idea. The reason why we only implement `Mul<f64>` and **not** `Mul<Vector3>` is because there are two types of multiplication for vectors, dot product and cross product. Although you could implement it in the same way we did for addition and subtraction but that would give us component-wise scaling, a less useful operation, but we would be better off making that a function rather than overloading multiplication and giving us a 3rd potentially confusing, multiplication definition.

In the original `impl Vector3` we should add dot and cross product. Dot product returns a scalar value rather than a vector is useful as among other tricks, it can be used to derive the angle between two vectors where `a.dot(b)` is 0 when `a` and `b` are perpendicular one another and 1 when they are parallel, for unit vectors. Dot product for a component vector is simply `a.x * b.x + a.y * b.y + a.z * b.z`.

~~~rust
/// The dot (scalar) product of self and other.
/// ```rust
/// use tendon::*;
/// let a = Vector3 { x: 3.0, y: 4.0, z: 5.0 };
/// let b = Vector3 { x: -1.0, y: 1.5, z: 0.5 };
/// let dif = a.dot(b) - 5.5;
/// assert!(dif.abs() < 1e-10);
/// ```
pub fn dot(self, other: Self) -> f64 {
    self.x * other.x + self.y * other.y + self.z * other.z
}
~~~

Cross product is not quite as simple, returning a vector. `a x b` (`a` *cross* `b`) is only defined for 3-dimensional vectors and returns a vector that is perpendicular both of the other vectors, for example, if I had a vector parallel the x-axis and *crossed* it with a vector parallel the y-axis we would get a vector parallel the z-axis. Again there are various mathematical tricks that we can use with cross products as well as its utility in providing us with the normal of a plane, defined by two direction vectors.

~~~rust
/// The cross (vector) product of self and other.
/// ```rust
/// use tendon::*;
/// let x = Vector3 { x: 1.0, y: 0.0, z: 0.0 };
/// let y = Vector3 { x: 0.0, y: 1.0, z: 0.0 };
/// let dif = x.cross(y) - Vector3 { x: 0.0, y: 0.0, z: 1.0 };
/// const TINY: f64 = 1e-10;
/// assert!(dif.x.abs() < TINY && dif.y.abs() < TINY && dif.z.abs() < TINY);
/// ```
pub fn cross(self, other: Self) -> Self {
    Self {
        x: self.y * other.z - self.z * other.y,
        y: self.z * other.x - self.x * other.z,
        z: self.x * other.y - self.y * other.x
    }
}
~~~

The Vector3 structure is looking pretty slick now with everything we could want for now. A Vector2 is next on the list but I won't cover it here since it is almost identical to its higher-dimension sibling. Simply remove the z-component and the cross product function after a highly sophisticated copy-paste. Looking back at the test for cross product and seeing all those `0.0`'s is a great reminder that we missed some last functions we will proably want in the future. First we will want to implement `Default` on our vectors to get an easy zero vector, this could be done manually in an `impl`, or since `f64` already implements `Default` it could simply be added into the derive list at the top of the function like so.

~~~rust
#[derive(Copy, Clone, Debug, Default)]
pub struct Vector3 { ... }
~~~

An all `1.0` vector may also be handy so let's add that as a constant value.

~~~rust
impl Vector3 {
    pub const ONE: Self = Self { x: 1.0, y: 1.0, z: 1.0 };
    ...
}
~~~

Ok, *now* it is done!

## An extra dimension & Matrix4

Vectors are pretty powerful but doing certain operations on them is a little difficult and slow. Even highly complex transformations can be done with a matrix multiplication making them pretty good for easily transforming a large collection of vectors. This is going to be especially important once we try turning 3-dimensional points into 2-dimensional points on a screen. There is another catch however. For reasons we will discuss when writing the renderer an extra dimension is needed! That means we must once again duplicate our vector structure to create a Vector4. Again, it will use all the same operations, but this time including a new component conventionally named `w`. I suggest you also name it `w` to avoid confusion in the future.

With Vector4 finished (last vector, I promise) we can move onto our matrix type. Matrix4 will have four rows and four columns of floats stored in an array of array of floats. To make accessing them as row by column instead of column by row we use the outer array as the rows array and the inner one for columns. Additionally, there is only one field which doesn't really make much sense to name since **it** is the matrix, so we can use fancy tuple struct in Rust.

~~~rust
pub struct Matrix4([[f64; 4]; 4]);
~~~

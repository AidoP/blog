---
title: "Writing a Software Rasteriser: 3.Triangle Clipping and Higher Dimensions"
description: "Clip triangles to fit on the screen and start looking at higher dimensions and the various coordinate spaces needed."
image: ""
---

# Higher Dimensions

So far we have kept in the realm of 2-dimensional screen space coordinates. *"Screen space coordinates"* refers to the fact that the range of valid values in our coordinate system is that of the resolution of the screen. The problem is, however, that our game shouldn't be tied to the screen size since it is to run on a variety of devices with very different screens. What's more is that we are rendering a 3-dimensional world, not a 2-dimensional one. With our goal of screen size-agnostic coordinates we can define a 3-dimensional coordinate system with values between 0 and 1 to be device-agnostic, called normalised device coordinates. The normalised device coordinate cube actually uses values between **-1** and **+1** in [most graphics API's](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#vertexpostproc-clipping) (You may want to read the next few paragraphs first) however we will stick to 0 and 1 to keep true to the *normalised* part of the name. Normalised device coordinates will represent the volume that the camera sees, named the [view frustum](https://en.wikipedia.org/wiki/Viewing_frustum), after it is projected. [Clipping](https://en.wikipedia.org/wiki/Clipping_(computer_graphics)) in 3d space is useful because it allows us to also remove triangles **behind** the camera which would otherwise obscure the view when they shouldn't as well as far away triangles while we also clip triangles that are out of frame on the `x` and `y` axes. If you really wanted to you could clip twice, once in 2-dimensional screen space and once in 3-dimensional clip space which we will talk about a little later. This 3rd dimension would also be very useful to know about when drawing triangles if we needed to keep a depth buffer, but this isn't an issue for us.

We also need to think ahead to the next coordinate system, homogeneous coordinates. [Homogeneous coordinates](https://en.wikipedia.org/wiki/Homogeneous_coordinates) introduce a 4th dimension which encodes translation along with the three other axes. The 4th axis, dubbed the `w` axis, does not differentiate vectors that are the same on the other axes, that is to say, `(x, y, z, 1)` is the same 3D vector as `(2x, 2y, 2z, 2)`. If `w` is 0 that means a vector projects to infinity which is why many people will say that vectors with `w = 0` are called "vectors [encoding direction]" and vectors with `w = 1` are "points". For more discussion on this check out this [stack overflow answer](https://gamedev.stackexchange.com/a/18012). This property means that the vector `(x, y, z)` is equal to `(wx, wy, wz, w)` in homogeneous coordinates, we can therefore divide off the `w` to go back to [Euclidean](https://en.wikipedia.org/wiki/Euclidean_geometry) (*"normal"*) coordinates. This division is known as the *perspective divide* due to the meaning of the `w` component after perspective projection. Euclidean coordinates for us will be normalised device coordinates thanks to our projections that we will be doing eventually, but I refer to them as Euclidean coordinates since this system is not limited to a 3D graphics scenario such as ours. The reason why we care about having multiple points map to the same coordinate is that it allows us to translate and do other complex transformations using a matrix **multiplication**, as shown here.

![Translation of a vector in homogenous coordinate space using matrix multiplication](/blog/assets/homogeneous_translation.jpg)

Although we could just do an **addition** in Euclidean coordinate space for translation, the real importance of this is for other transformations, most importantly, perspective projection. This brings us back to vectors with a `w` component of 0. Again if you look at the above maths, but this time substituting in 0 for the `w` component, you will notice that the transformations do not apply at all! As these vectors project to infinity they can only define a direction, not a location, and as such this property is perfect since translation of a direction doesn't make sense and would change the meaning of our vector in Euclidean space.

I mentioned [clip space](https://gamedev.stackexchange.com/a/162899) previously, so where does that fit in? We [could try clipping in normalised device coordinates](https://community.khronos.org/t/homogenous-normalized-device-coords-and-clipping/61965/3), but we actually lose a little bit of information when convert from homogeneous coordinates to normalised device coordinates. Remember, our normalised device coordinates represent what is inside the view frustum while we want to clip stuff that is outside of this view area. Instead we use stay in homogenous coordinates even though we only care about the three *"real"* (The `w` dimension is just as valid, but you know what I mean here) dimensions. This is called [*clip space*](https://en.wikipedia.org/wiki/Clip_coordinates) and particularly helpful as we can also clip the points which project to infinity since these points are a bit difficult to describe in normalised device coordinates. The key here is that we need to do a projection to get to clip space, in our case we will be doing **perspective projection** which we will gloss over until we get to that point. The projection is the magic which converts our world coordinates of our raw world geometry into our coordinates which correspond to the view frustum.

Finally, we must think about the implication of doing interpolation for our texture mapping in screen space rather than in a 3-dimensional geometry. After we implemented it last time I mentioned that depth is actually important, as shown here with this texture mapped plane using our current screen space implementation versus the *perspective corrected* implementation.

![A plane projected correctly, but with a very odd seam in the texture, next to one with perspective correct interpolation](barycentric_perspective_correction.png)

To understand why, think about the area above and below the midpoint of the square when it is upright. Because it is flat the sides run parallel, but when you lay it down like in the image parallel lines converge at a distant point which means the width of the plane at the top is less when projected to that at the bottom, resulting in the area closer to us to appear larger when we ignore the `z` axis. The solution would be to calculate the area in 3D, however, we would have to do that before perspective projection since that non-linearly squishes the z axis inside the normalised device coordinate cube giving us the exact same scenario. The solution is to do interpolation in *clip space* where we still have homogeneous coordinates where the `w` component actually encodes the original depth of each vertex. [Scratchapixel have an unparalleled walkthrough of the maths behind this mechanism](https://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation/visibility-problem-depth-buffer-depth-interpolation). We will not be doing perspective correction quite yet as it would make more sense once we understand perspective projection itself.

Don't worry if most of that has gone over your head, a high level understanding is all you need to use this stuff. I do think that it is worth explaining though as any sources I could find were very maths dense and full of technical language. And an obligatory take my explanation with a grain of salt, I have neither a formal education on computer graphics nor a second opinion on it (If you know better and have any complaints, or can verify, leave an [issue](https://github.com/AidoP/blog/issues)).

# Triangle Clipping

The number of triangles created after clipping is variable since the triangle could clip with zero or more faces of the viewing frustum. Allocating heap memory and storing the extra triangles there by using a Vec is one possibility, but allocating heap memory is quite expensive and we must keep in mind that we are on the hot path. Since the function is running often we may want to use a preallocated buffer that is passed to the clip function instead, that way we only make a large memory allocation once rather than at every invocation of the function. Rather than having an array of `Option<Tri>`, I opt for returning the number of clipped triangles since `Option<Tri>` will not be free space-wise versus `Tri`. We can declare the clipping function, `Tri::clip()` now.

~~~rust
pub fn clip(tris: &mut [Self]) -> usize {

}
~~~

As discussed, we want to take in triangles with 4-dimensional vertices for clipping, not screen space triangles. What's more is that when we clip a triangle we need to also clip the `UV` coordinates alongside it. That means we need a new triangle structure. Luckily for us, rather than creating one with a different name and introducing a potential world of confusion, we can use Rust's [generics](https://doc.rust-lang.org/book/ch10-01-syntax.html#in-struct-definitions). In Rust you can implement functions for only a particular generic type of a generic structure by specifying the concrete type instead of a generic type parameter. Let's make `Tri` generic now.

~~~rust
pub struct Tri<T>(pub [T; 3]);
impl<T> Deref for Tri<T> {
    type Target = [T; 3];
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
impl<T> DerefMut for Tri<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}
~~~

We can now change the old `impl` block to only apply to a screen space `Tri` of `Vector2`'s, denoted syntactically by `Tri<Vector2>`. and add a new `impl` for `Tri<Vector4>`

~~~rust
impl Tri<Vector2> {
    pub fn interpolate(&self, other: &Self, x: f64, y: f64) -> Vector2 {
        // ...
    }
}
impl Tri<Vector4> {
    pub fn clip(tris: &mut [Self]) -> usize {
        // TODO
    }
}
~~~

We will be clipping a homogeneous triangle into a screen space one. A homogeneous triangle will be a triangle with homogeneous vertices and texture coordinates, which we can call a `Vertex4`. While we are at it we should also change `Tri<Vector2>` to `Tri<Vertex2>` so that we aren't using the same type for both world coordinates and texture coordinates which we could easily conflate. With multiple vertex types we would be better off taking the same approach that we did with `Tri`. Because of the name it would make sense to treat a `Vertex` as just that so we will implement `Deref` on the `Vertex` for convenience.

~~~rust
pub struct Vertex<T> {
    pub vertex: T,
    pub uv: Vector2
}
impl<T> Deref for Vertex<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.vertex
    }
}
impl<T> DerefMut for Vertex<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.vertex
    }
}
~~~

We also need to update our previous usage of `Tri` in `Framebuffer::draw_tri()` to match. Sadly `Deref` means that we only implicitly access `Veretx.vertex` for references so we must go back to all previous usages of the Tri by value, such as with operators, and make changes. This is actually pretty good here since it catches the conflation of the old `Tri`'s vertex and `UV` usage.

~~~rust
pub fn draw_tri<'a>(&mut self, tri: Tri<Vertex<Vector2>>, uvs: Tri<Vertex<Vector2>>, sampler: &Sampler<'a>) {
    // ..
    let high_edge = *tri[c] - *tri[a];
    let top_edge = *tri[b] - *tri[a];
    let bottom_edge = *tri[c] - *tri[b];
    // ..
                let uv = tri.interpolate(x as _, y as _);
    // ..
    
                let uv = tri.interpolate(x as _, y as _);
    // ..
}
~~~

This applies to `Tri<Vertex<Vector2>>::interpolate()` as well.

~~~rust
pub fn interpolate(&self, x: f64, y: f64) -> Vector2 {
    let p = Vector2 { x, y };
    let edge_a = *self[1] - *self[2];
    let edge_b = *self[0] - *self[2];
    let edge_c = *self[0] - *self[1];
    let total_area = edge_a.area_tri(edge_b);
    let weight_a = edge_a.area_tri(p - *self[2]) / total_area;
    let weight_b = edge_b.area_tri(p - *self[2]) / total_area;
    let weight_c = edge_c.area_tri(p - *self[1]) / total_area;
    Vector2 {
        x: weight_a * self[0].uv.x + weight_b * self[1].uv.x + weight_c * self[2].uv.x,
        y: weight_a * self[0].uv.y + weight_b * self[1].uv.y + weight_c * self[2].uv.y,
    }
}
~~~

The extra dereferences should also get optimised out by the compiler since they don't actually do anything at the end of the day. We have yet again used the type system to ensure program correctness at compile time, thereby removing the chance of some harder to diagnose bugs. By the way, if `Vertex<Vector4>` annoys you, you could use a [type alias](https://doc.rust-lang.org/reference/items/type-aliases.html) to rename it a `Vertex4`. Personally I prefer the explicit nature of the generic form, I know what I am getting when I see `Vertex<Vector4>`.

Where were we? Let's get back to `Tri<Vertex<Vector4>>::clip()` which is finally declared as

~~~rust
pub fn clip(tris: &mut [Self]) -> usize
~~~

First we go for the easiest and fastest cases, is the triangle entirely inside or outside the cube? We can easily test for it being outside by seeing if all vertices are on the wrong side of each face. There are cases when all vertices are outside but the triangle intersects the cube, but only when the vertices are on the wrong side of different faces of the cube. I'll test against each pair of parallel faces with the help of a macro. We can order by most to least likely to clip early for a slight but easy speed bonus since we will skip extra checks for the majority of cases and the out of camera planes are most likely behind and to the side of the camera rather than above.

~~~rust
pub fn clip(tris: &mut [Self]) -> usize {
    let tri = &tris[0];
    macro_rules! clip {
        (early => $dimension:ident) => {
            if (tri[0].$dimension > tri[0].w && tri[1].$dimension > tri[1].w && tri[1].$dimension > tri[1].w) ||
                (tri[0].$dimension < -tri[0].w && tri[1].$dimension < -tri[1].w && tri[1].$dimension < -tri[1].w) {
                return 0
            }
        };
    }
    clip!(early => z);
    clip!(early => x);
    clip!(early => y);
~~~

We also ensure every vertex is within bounds.

~~~rust
    if tri.iter().all(|v| v.x <= v.w && v.y <= v.w && v.z <= v.w && v.x >= -v.w && v.y >= -v.w && v.z >= v.w) {
        return 1
    };
~~~
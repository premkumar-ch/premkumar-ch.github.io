---
title: "What a Renderer Actually Does"
date: 2026-10-04
description: "A ground-up introduction to computer graphics, built by writing a small CPU ray tracer in C++ — pixels, rays, intersections, normals, light, shadows, reflection, and the frame loop."
tags:
  - graphics
  - ray-tracing
  - cpp
---

There is a question I wanted to answer properly, and I could not answer it by reading about graphics.

**How does a program turn a mathematical description of a 3D world into the pixels on my screen?**

Not "how do I call the draw API". I mean the whole chain, end to end. There is a sphere at `(0, -1, 3)` with radius `1`. There is a light at `(2, 1, 0)`. And there are 480,000 little squares of memory that my monitor is supposed to show. Between those two facts there is a gap, and the gap is what most of graphics programming actually is.

So I closed it. The result is [`Rays.exe`](https://github.com/premkumar-ch/Rays.exe) — about 300 lines of C++ in a single file, no engine, no framework, SDL3 only so that I can fly a camera around the scene with WASD and the mouse.

This article is the walk-through. I am writing it for the version of me that knew C++ and had never written a line of graphics code, because that is the person I was when I started, and the only way I know how to explain this is the way I had to learn it: slowly, from the pixel outwards.

Everything quoted here is real code from [`RayTracer.cpp`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp).

## The image is the program

Here is the thing that made graphics click for me, and it is almost disappointingly simple.

An image is a grid. Not a compressed, clever, layered, scene-graph-having thing — a grid of numbers. `800 × 600` pixels, each pixel three integers, red, green, blue, somewhere between 0 and 255. That is the entire output. Everything else in this article exists to fill in those numbers.

```cpp
struct Color {
    int r, g, b;
};
```

That is my colour type. No HDR, no spectral rendering, no colour spaces. Three bytes. I promise that the entire visual ambition of this project fits inside those three integers.

If the image is a grid, then rendering a frame is a loop. This is not a simplification, this is what the program does — [`get_curr_frame`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L238-L262), the function that draws one frame:

```cpp
for (int x = -width / 2; x < width / 2; x++) {
    for (int y = -height / 2; y < height / 2; y++) {
        Vec3 D = normalize(canvas_to_viewport(x, y));
        // Apply camera rotation matrix/transformation to ray direction
        D = rotate_vector(D, yaw, pitch);

        Color c = trace_ray(camera, D, 1.0f, inf, spheres, lights, 3);
        put_pixel(res, x, y, c);
    }
}
```

Read it slowly. For every pixel, compute a colour, then store it. That is the whole contract. 480,000 iterations of "decide what this pixel should be".

So the first real idea of graphics programming is not geometry, not matrices, not shaders. It is this:

> **Rendering is the process of deciding what colour each pixel should be.**

Everything else — cameras, intersections, lighting, shadows, reflection — exists only to help you answer that one question for one pixel. When I got comfortable with that framing, the rest stopped feeling like magic.

One small but important detail lives in `put_pixel`, and it is the kind of thing that wastes an evening if nobody warns you:

```cpp
void put_pixel(vector<Color>& res, int x, int y, Color color) {
    int px = x + width / 2;
    int py = height / 2 - y - 1;

    if (px < 0 || px >= width || py < 0 || py >= height)
        return;

    res[py * width + px] = color;
}
```

Two coordinate systems are colliding here. My ray loop counts `y` *upwards*, because that is how I want to think about the world — up is positive. But an image buffer counts `y` *downwards*, because that is how rows are stored: row 0 is the top row. So the pixel row is `height / 2 - y - 1`. The `- 1` is the off-by-one that you only discover empirically, at 2 a.m., staring at a scene that is mirrored and one pixel off.

I have since made this mistake in both directions on different projects. It is not a sign of carelessness. Coordinate conventions are not intuitive; they are conventions.

## From a pixel to a direction

I can loop over pixels, but a pixel is a flat 2D thing and my world is 3D. A pixel `(x, y)` does not mean anything until I decide *where in the world* that pixel is looking from, and in what direction.

This is the job of the camera, and I did not want to build a camera class with matrices and quaternions before I understood what a camera *is* for this program. So I did the simplest possible thing: a floating viewport, one unit tall, sitting one unit in front of the origin.

```cpp
float viewport_size = 1.0f;
float projection_plane_d = 1.0f;

Vec3 canvas_to_viewport(int x, int y) {
    float viewport_width = viewport_size * static_cast<float>(width) / static_cast<float>(height);

    return {
        x * viewport_width / width,
        y * viewport_size / height,
        projection_plane_d
    };
}
```

That is [`canvas_to_viewport`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L228-L236). Read it as a sentence: *take a pixel coordinate, scale it onto a viewport that is `viewport_size` units tall and proportionally wider, and push it one unit away from the camera.*

The camera sits at the origin and looks down `+z`. `forward` is `{0, 0, 1}` in the input code, and that is the same statement said differently.

A few things worth noticing, because they are the whole geometry of the problem:

- **Aspect ratio.** `viewport_width = viewport_size * width / height` = `1.0 * 800/600` ≈ `1.333`. A viewport that is 1.333 units wide and 1 unit tall has the same proportions as an 800×600 image. If I had not done this, circles would render as ellipses, and I would have spent an afternoon wondering whether my intersection math was wrong. It was not; my window was the wrong shape.
- **The `- 1` is gone.** Note the x mapping is `x * viewport_width / width` with no half-width subtraction, because my loop already runs from `-width/2`. So the viewport spans roughly `x ∈ [-0.667, 0.667]`, `y ∈ [-0.5, 0.5]`, at `z = 1`.
- **It returns a point, not a direction.** Yet I immediately call `normalize` on it in the render loop. That is the bridge to the next section.

A pixel on my screen is now a point on an imaginary rectangle floating one unit in front of my eyes. The only thing missing is the line from my eye to that point.

## What is a ray?

So here is the entire idea, and it is the idea the whole project is built on. Instead of asking "which object is at this pixel", I flip the question around: **ask each pixel to shoot a line forwards and see what it runs into.**

A line like that has a name: a ray. And it has the simplest possible description:

```text
P(t) = O + tD
```

- `O` is the origin — where the ray starts. For a camera ray, that is the camera position.
- `D` is the direction — the unit vector pointing where we are looking.
- `t` is distance travelled. `t = 0` is the camera. `t = 1` is one unit away. `t = 7.3` is 7.3 units away.

The useful thing about this formula is not that it is elegant. It is that it turns a visual question into an arithmetic one. Instead of "does this ray hit that sphere?", I can ask a question I know how to answer with numbers:

> **For which values of `t` does `P(t)` satisfy the equation of the sphere?**

That reframing is the whole trick. Intersection becomes substitution, and substitution becomes algebra. If I can find the `t` values, I know exactly where I hit, how far away it is, and therefore what colour that pixel should be.

And in the code, it is literally one line, in [`trace_ray`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L201-L223):

```cpp
Vec3 P = vec_add(O, vec_scale(D, closest_t));
```

Once I have the winning `t`, I have the point on the surface. That is the surface point. Everything from here — normals, lighting, shadows, reflections — is computed from this one `Vec3 P`.

One more note about `D`. In the render loop I call `normalize` on the viewport point before tracing. Normalizing is not fussiness; it is what makes `t` mean *distance in world units*. If `D` were `{0.3, 0.1, 1.0}`, then `t` would be some arbitrary parameter and `t = 3` would not be three units away. After normalizing, `|D| = 1`, and `t` is genuinely a distance. That is why the directional light and the point light both normalize their directions before anything else happens with them.

## "Does this ray hit that sphere?"

Now the fun part. Sphere intersection, from scratch, with the actual function from the repository.

A sphere is the set of all points at distance `r` from its centre `C`. That is a *quadratic* description — it involves a square root — and it is one line of algebra:

```text
|P − C|² = r²
```

A ray is `P(t) = O + tD`. So substitute one into the other and I want to know: for which `t` is the ray's position exactly `r` away from the centre?

```text
|(O + tD) − C|² = r²
```

Let me name the vector from the sphere's centre to the ray's origin, because it appears three times and deserves a name. The code calls it `CO` — the vector from `C` to `O`:

```cpp
Vec3 CO = vec_sub(O, s.center);
```

Now expand. `(CO + tD)·(CO + tD)` is a dot product of a sum with itself, and dot products expand:

```text
(CO + tD)·(CO + tD) = CO·CO + 2t(CO·D) + t²(D·D)
```

So my equation becomes:

```text
t²(D·D) + 2t(CO·D) + (CO·CO) − r² = 0
```

There is the quadratic. And it had to appear: I substituted a linear expression (the ray) into a quadratic one (the sphere), so I got a quadratic in `t`. A line can cut a sphere in two places, which is exactly what "two roots" means geometrically — and, not coincidentally, a ray can also graze the surface and hit it once, which is what a repeated root means.

Now compare that equation with the standard form `at² + bt + c = 0` and read off the coefficients:

- `a = D·D`
- `b = 2(CO·D)`
- `c = CO·CO − r²`

That is the entire function:

```cpp
pair<float, float> intersect_ray(Vec3 O, Vec3 D, const Sphere& s) {
    Vec3 CO = vec_sub(O, s.center);
    float a = dot(D, D);
    float b = 2.0f * dot(CO, D);
    float c = dot(CO, CO) - s.radius * s.radius;
    float disc = b * b - 4.0f * a * c;

    if (disc < 0.0f)
        return { inf, inf };

    float root = sqrt(disc);
    float t1 = (-b - root) / (2.0f * a);
    float t2 = (-b + root) / (2.0f * a);

    return { t1, t2 };
}
```

([source](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L122-L137))

Line by line, and I want to be honest about which lines are the interesting ones:

- `disc = b*b - 4*a*c` is the discriminant, `b² − 4ac`. **If it is negative, the square root is not real, so there is no point on the ray that lies on the sphere. No hit.** This is the single most important line in the function: it is where "does this ray hit this object?" gets its answer. I return `{ inf, inf }` as a sentinel meaning "no hit", which is the tiniest possible lie the type system lets me tell — `inf` is a `float`, so I cannot return "nothing", but any comparison against it will fail, which is all the caller needs.
- The two roots are the two possible intersection distances, and because `root` is non-negative, `t1 ≤ t2` always. `t1` is the **near** hit, `t2` the far one (think: the near and far side of the sphere). If you are inside the sphere, `t1` is negative and `t2` is positive — the code does not care, and that is worth knowing about later.
- `a = dot(D, D)` is *not* hardcoded to 1, even though `D` is normalized everywhere I call this. Keeping the general form costs one multiply and means the function is correct for any direction vector, including the `-D` I hand to reflections.

### Choosing the closest hit

`intersect_ray` answers "does this ray hit *this* sphere". Rendering needs "does this ray hit *anything*, and what is the closest thing it hits?" That is [`closest_intersection`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L139-L158), and its logic is a `for` loop over the scene keeping the best candidate so far:

```cpp
pair<Sphere*, float> closest_intersection(Vec3 O, Vec3 D, float t_min, float t_max, vector<Sphere>& spheres) {
    float closest_t = inf;
    Sphere* closest_sphere = nullptr;

    for (auto& sphere : spheres) {
        auto t = intersect_ray(O, D, sphere);

        if (t.first >= t_min && t.first <= t_max && t.first < closest_t) {
            closest_t = t.first;
            closest_sphere = &sphere;
        }

        if (t.second >= t_min && t.second <= t_max && t.second < closest_t) {
            closest_t = t.second;
            closest_sphere = &sphere;
        }
    }

    return { closest_sphere, closest_t };
}
```

The two `if` blocks are the same test applied to each root, because each sphere has up to two candidate hits and both are worth considering.

`t_min` and `t_max` are the parts I underestimated while writing this. They are not book-keeping; they are what stop the renderer from shooting itself in the foot:

- The camera ray is traced with `t_min = 1.0f`, so the renderer cannot detect a sphere that the camera is sitting inside of, right at `t = 0`.
- A reflection ray starts at `P`, which is *on* a sphere. With `t_min = 0.0f` that sphere would immediately report a hit at `t ≈ 0` and reflect the pixel forever. So every secondary ray in this renderer starts at `0.001f` — an epsilon, a tiny nudge along the ray to escape the surface it was born on.

So the signature of a ray in this renderer is really `(origin, direction, t_min, t_max)`, and I now think of `t_min`/`t_max` as *where this ray is allowed to exist*. The point light uses exactly that trick for shadows, which I will come back to.

## We have a renderer

Here is the part where it clicked for me. Put the pieces together and look at what they form:

```text
pixel
   ↓
camera ray            (canvas_to_viewport + normalize)
   ↓
ray-object test       (closest_intersection over every sphere)
   ↓
closest object        (smallest t that is not the camera's own foot)
   ↓
surface point         (P = O + tD)
   ↓
colour                (surface colour × how much light reaches it)
   ↓
pixel
```

That is a renderer. Not a good one, not a fast one — a *real* one. The whole of ray-based rendering is that pipeline, and every "advanced" feature is an elaboration of one of those arrows.

I want to be precise about why that is such a strange sentence to read. In software, we are used to abstraction hiding work: you call `sort(v)` and you do not think about merges. Here, the opposite is true. There is no hidden machinery at all. The image is *literally* the result of 480,000 independent questions, each one a handful of floating-point operations on a few dozen instructions. Once I accepted that the top of the call stack is the bottom of the abstraction stack, graphics stopped feeling like magic and started feeling like arithmetic.

The pixel that misses everything becomes black:

```cpp
if (closest == nullptr)
    return { 0, 0, 0 };
```

`{0, 0, 0}` is the entire concept of "background". Nothing drew it; nothing is there; the ray simply left and never came back, so the pixel stays black.

## Normals: which way is the surface facing?

Right now I can tell *where* I hit a sphere. To light it, I need to know *which way the surface is facing* at that point. That direction is the **normal**: a unit vector perpendicular to the surface, pointing outward.

Why does orientation matter? Because of how light behaves. A sheet of paper lit from the front is bright; the same sheet lit from behind is dark. Light is not a property of a surface alone, it is an interaction between a surface and a direction. The normal is how the surface tells the lighting code which way it is facing.

For a sphere, the normal is free. The outward direction at a surface point is just "from the centre to the point", so:

```cpp
Vec3 N = normalize(vec_sub(P, closest->center));
```

That is the whole normal computation — no interpolation, no extra data stored per sphere, nothing. A sphere is radially symmetric, so the geometry hands you the normal for free. This is one of the reasons spheres are the first thing everyone renders, and it is worth knowing that they are *unusually* convenient rather than representative.

## Lighting

Now I can make a pixel a colour. `ComputeLighting` accumulates how much light reaches a surface point, and the surface colour is that light times the material colour.

Three kinds of light, one enum:

```cpp
enum class LIGHT_TYPE { ambient, directional, point };
```

and this is the scene I render, straight from the repository:

```cpp
vector<Light> lights = {
    { LIGHT_TYPE::ambient, 0.2f, {0, 0, 0}, {0, 0, 0} },
    { LIGHT_TYPE::point, 0.6f, {2, 1, 0}, {0, 0, 0} },
    { LIGHT_TYPE::directional, 0.2f, {0, 0, 0}, {1, 4, 4} }
};
```

The function is long enough to be worth reading in pieces. First, the accumulation and the ambient case:

```cpp
float intensity = 0.0f;

for (auto& light : lights) {
    if (light.type == LIGHT_TYPE::ambient) {
        intensity += light.intensity;
        continue;
    }
```

Ambient light is the "how much can I see even in the dark" term. It has no direction and no position, so there is nothing to compute against a normal: every surface receives it equally. That is also why it is the reason shadows in this renderer are dark grey rather than pure black — the shadowed side of the red sphere still gets the 0.2 ambient.

Then the two directional lights, distinguished only by how they build the vector `L` towards the light and how far the light is considered to be:

```cpp
Vec3 L;
float t_max;

if (light.type == LIGHT_TYPE::point) {
    L = vec_sub(light.position, P);
    float distance_to_light = length(L);

    if (distance_to_light == 0.0f)
        continue;

    L = normalize(L);
    t_max = distance_to_light;
}
else {
    L = normalize(light.direction);
    t_max = inf;
}
```

- A **point light** lives at a position. The direction to it depends on where the surface point is, so `L` must be recomputed per surface point: from `P` towards `light.position`, normalized. Because it is a real light at a real place, it can be *blocked* by things in between — and `t_max = distance_to_light` says exactly how far the shadow ray needs to travel to reach it.
- A **directional light** is infinitely far away, like the sun. Its direction is the same everywhere, so I can normalize it once and reuse it for every point in the image. Nothing finite can get between the surface and the sun in a way I care to model, so `t_max = inf`.

That is the entire difference between them, and it is three lines. Point lights are directional lights with a position and a finite distance. That kind of collapse — "the complicated case is the simple case plus one detail" — is extremely common in graphics, and it is usually a good sign that you have found the real shape of the problem.

Now the diffuse term, the actual physics-flavoured bit:

```cpp
float n_dot_l = dot(N, L);

if (n_dot_l > 0.0f)
    intensity += light.intensity * n_dot_l;
```

`N` is the surface normal, `L` points towards the light, and `dot(N, L)` is the cosine of the angle between them. That single number is doing all the work:

- Surface facing the light squarely → `N·L` close to `1` → bright.
- Surface at a grazing angle → `N·L` close to `0` → dark. This falloff is what makes a sphere look round instead of like a flat disc.
- Surface facing *away* → `N·L` negative → `if (n_dot_l > 0.0f)` skips it entirely, so a light does not illuminate the back of a surface.

This is the Lambertian model, and it is the oldest trick in lighting: brightness is proportional to the cosine of the angle to the light. No normals, no convincing spheres. Just a dot product.

The last line is a guard rail rather than a feature:

```cpp
return min(intensity, 1.0f);
```

Sum every light's contribution, then clamp to 1. Slightly inelegant — a single very bright light will flatten the others out — but it means I can think in terms of a 0..1 light budget, and the colour multiplication cannot overflow.

One deliberate simplification worth flagging, because it will bite anyone who copies this: **the point light has no distance falloff.** A real point light dims as `1/d²` as you move away from it. Here, intensity `0.6` is `0.6` whether the light is one unit away or ten. That makes the scene read as slightly flat, and it is the first thing I would change. I left it out on purpose: I wanted to be able to say that every visual difference I could see came from a specific piece of maths I understood, and not from attenuation I had skimmed.

So a lit pixel is just multiplication:

```cpp
Vec3 P = vec_add(O, vec_scale(D, closest_t));
Vec3 N = normalize(vec_sub(P, closest->center));
float light = ComputeLighting(P, N, lights, spheres);
Color local_color = closest->color * light;
```

Surface colour times light amount. Multiply `{255, 0, 0}` by `0.45` and you get a dark red — which is exactly the kind of thing that is easy to get wrong and impossible to get wrong by accident, because the multiplication is done per channel with a clamp:

```cpp
Color operator*(Color c, float intensity) {
    return {
        clamp(static_cast<int>(c.r * intensity), 0, 255),
        clamp(static_cast<int>(c.g * intensity), 0, 255),
        clamp(static_cast<int>(c.b * intensity), 0, 255)
    };
}
```

## Shadows are one more ray

This is my favourite part of the whole project, because the cost is one extra ray and the payoff is an entire visual dimension.

Once I have a surface point `P` and I know where the light is, the question "is this point lit?" has an obvious geometric answer: shoot a ray from `P` towards the light and see whether it hits anything on the way. If it does, the light is behind that object, so this point is in shadow.

```cpp
auto shadow = closest_intersection(P, L, 0.001f, t_max, spheres);

if (shadow.first != nullptr)
    continue;
```

Three lines. No shadow volumes, no shadow maps, no second render pass. `continue` means "this light contributes nothing here", and the point keeps whatever ambient it had.

Two details in that call are doing real work:

- **`0.001f` again.** The shadow ray starts exactly on the surface we are shading. Without the epsilon, the sphere we are standing on would block its own light and the entire sphere would be uniformly shadowed. Every hard-coded magic number in a renderer is some version of this problem.
- **`t_max` is the clever one.** For a point light, `t_max` is the distance from `P` to the light. For a directional light it is `inf`. That single parameter is what makes one shadow test correct for both light types: the ray is allowed to travel *only as far as the light is*, so a sphere *behind* the light cannot cast a shadow on `P`. Getting that wrong is a classic bug — shadows that appear in front of objects, or a sphere shadowing itself from a light that is nowhere near it.

Look again at what just happened. I already had a function that casts a ray and reports what it hits. I reused it, unchanged, with different parameters, and got shadows. That is the real lesson of this section, and it generalises far beyond ray tracing:

> **Shadows are not a lighting feature. Shadows are the same visibility query, asked a second time from a different origin.**

Hard shadows, incidentally, are what you get when you ask exactly one question per light per point. Real shadows have soft, grey penumbras because the sun is an extended source, not a point; simulating that means sampling many directions per light, which is a doorway into the next section.

## Reflection is the same ray again

A mirror works because of a specific geometric fact: the angle at which a ray hits a surface equals the angle at which it leaves. Reflect a ray about the surface normal and you have the mirror direction.

The vector formula is short, and it is in the file:

```cpp
Vec3 reflect_ray(Vec3 R, Vec3 N) {
    return vec_sub(vec_scale(N, 2.0f * dot(N, R)), R);
}
```

which is `R' = 2(N·R)N − R`. `2(N·R)N` is twice the projection of the incoming ray onto the normal, and subtracting `R` flips it to the other side of the normal. There is a nice geometric way to see it: if you reflect the *normal* about the ray instead, you get the same vector, and that version looks like a hinge folding across the ray. Either picture works; I drew it on paper about four times before it stuck.

The only subtlety is the sign. `R` must be the direction the ray was *travelling*, so at the call site I pass the negation of the incoming direction:

```cpp
Vec3 reflected_ray = reflect_ray(-D, N);
```

And now the good part. A reflected ray is a ray. I already have a function that casts a ray and returns a colour. So:

```cpp
Vec3 reflected_ray = reflect_ray(-D, N);

Color reflected_color = trace_ray(P, reflected_ray, 0.001f, inf, spheres, lights, recursion_depth - 1);

return local_color * (1.0f - r) + reflected_color * r;
```

([`trace_ray`](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L201-L223))

`trace_ray` calling `trace_ray` is recursion, and it does something a single ray never could:

```text
camera ray
    ↓
object          → surface colour + shading
    ↓
reflected ray
    ↓
another object  → surface colour + shading
    ↓
reflected ray
    ↓
... and so on, until the depth budget runs out
```

The second sphere in that chain is *not in the original scene description of what the pixel should look like*. Nothing told the renderer to draw it. It appears because the maths produced another ray, and that ray happened to hit something. Reflections, and the reason mirrors in ray tracers look "real", fall out of that.

Two details keep this from blowing up:

- `recursion_depth`, passed as `3` from the render loop, decremented on every bounce. Without it, a ray bouncing between two mirrored spheres would recurse until the stack died.
- `r` is a per-sphere reflectivity, clamped to 0..1, and the final colour is a **blend**, not a replacement: `local_color * (1.0f - r) + reflected_color * r`. A sphere with `r = 0.4` shows 40% reflection and 60% its own colour. That single line is the difference between a mirror and a shiny surface, and it is also the difference between a reflection that looks like a light bulb and one that looks like polished metal.

In the current scene, each sphere declares its own reflectivity — `0.2`, `0.4`, `0.3` — and the floor `0.5`. So the red sphere mostly looks red with a faint sheen, and the floor is half mirror, which is why you can see the spheres sitting on it. Change `r` to `1.0f` and the sphere becomes a perfect mirror. That is the whole material model of this renderer, and it is one float per object.

## Where the time goes

I want to talk about performance, because it is the part of graphics that genuinely changed how I think about code.

Ray tracing is conceptually cheap. Every pixel is independent. Every intersection is a few dozen floating-point operations. And yet the moment I added reflections and shadows, the frame time went from "instant" to "visible", and the cause is multiplication.

Let me count honestly, using this renderer:

- 480,000 pixels, one camera ray each.
- Each camera ray tests 4 spheres, each with up to 2 roots: cheap.
- At every hit, `ComputeLighting` casts a shadow ray per non-ambient light — 2 lights here — and each shadow ray scans all 4 spheres again.
- At every hit, a reflective surface spawns a whole new ray, and *that* ray pays the full cost again, including its own lighting and its own shadow rays. The render loop passes a recursion budget of `3`, so a single pixel can trace a chain of four rays: the camera ray plus three bounces.

So the cost is roughly `pixels × objects × bounces × lights`, and every factor is a multiplier. The scene here is 4 spheres and it runs fine. Make it 4,000 spheres — a real scene, a forest, a city — and the linear scan in `closest_intersection` turns into the entire cost of your frame.

That is the real shape of the problem: **graphics programming is not only about making an image correct, it is about making the computation finish before the next frame is due.** Correctness is the easy half. I have never once had a bug where the maths was subtly wrong and the image was mysteriously slow; slowness has a cause you can point at, and it is almost always "you are doing this work more times than you need to".

This is also where the field opens up, and I want to show the path rather than the destination. The obvious next move, staring at that `for (auto& sphere : spheres)` loop, is to stop testing every object and instead ask *which objects could this ray possibly reach?* — a **bounding volume hierarchy**, a tree of nested boxes that says "nothing in here is anywhere near you, skip the whole subtree". Replacing the linear scan with a BVH traversal is usually the single biggest win available in a CPU ray tracer, and it is a data-structure problem wearing a graphics costume.

From there, the paths branch:

- **SIMD.** Every intersection is the same sequence of float operations on independent data. Four or eight spheres at a time is a straightforward restructure of `intersect_ray`.
- **Multithreading.** Pixels share nothing. There is no shared state to protect between pixel 12,000 and pixel 12,001 — this is the textbook *embarrassingly parallel* problem, and a thread pool over rows of the image is almost embarrassingly easy.
- **The GPU.** All of the above is asking "how do I make the CPU faster", when the real answer is "do it somewhere with thousands of cores". The same `trace_ray` translates almost line for line into a compute shader — Vulkan, OpenGL, or DirectX, with the maths unchanged. This project is, almost literally, a compute shader that happens to run on a CPU.
- **Rasterization**, which is the other great approach to the same problem and the reason GPUs are shaped the way they are: instead of asking each pixel what it sees, transform each triangle and figure out which pixels it covers. Enormously faster, and unable to do reflections without extra work.
- **Path tracing**, which is what my renderer is one deterministic sample away from being. Right now I ask one question per pixel and take the answer. Path tracing asks *many* random questions per pixel — randomly chosen light paths, randomly chosen reflection directions — and averages thousands of frames together. Swap the single shadow ray for many, and hard shadows turn into soft ones for free. Swap the single mirror direction for a random one around the normal, and sharp reflections turn into rough metal. Same function, different statistics.

I have not implemented any of that yet, and I do not think I ever will in this repository. But I understand now *why* those techniques exist, which is a much more durable thing than knowing their names.

## From an image to a loop

Until recently, this program produced one image and exited. That is a perfectly good way to learn the maths, and it is worth resisting the urge to add interactivity before the image is interesting — a moving camera over a boring image teaches you nothing.

Then I ported it to SDL3 and the nature of the problem changed completely.

The difference is this: previously the renderer ran once, top to bottom, and the answer appeared. Now there is no "the answer". There is only "the answer for this camera position at this moment", and the moment has already passed by the time you read it. So the program becomes a loop:

```text
input
   ↓
update camera
   ↓
render scene
   ↓
display frame
   ↓
repeat, forever
```

That is the real-time graphics contract, and it is where the "graphics" part stops being geometry and starts being systems programming. Let me show the loop as it exists in the repository, because each piece answers a specific worry I had.

**Timing.** Anything per-frame must be scaled by how much time has passed, or the game will run at different speeds on different machines:

```cpp
auto current_time = chrono::steady_clock::now();
float delta_time = chrono::duration<float>(current_time - previous_time).count();
previous_time = current_time;
```

and then, per frame, once I know how long the frame is:

```cpp
float movement = speed * delta_time;
```

Three world units per second, whatever the frame rate. Note what `chrono` is *not* used for here: I have not instrumented the renderer to measure how long a frame takes to compute. I use it only to make movement frame-rate independent. Profiling is a thing I should do and have not done.

**Input.** Two different APIs, because mouse and keyboard are different kinds of problem:

```cpp
if (event.type == SDL_EVENT_MOUSE_MOTION) {
    float dx = (float)event.motion.xrel;
    float dy = (float)event.motion.yrel;

    yaw += dx * sensitivity;
    pitch += dy * sensitivity;
    if (pitch > 89.0f)  pitch = 89.0f;
    if (pitch < -89.0f) pitch = -89.0f;

    needs_render = true;
}
```

The mouse gives me *relative* motion — how far it moved since last time, not where it is — and I had to ask for that explicitly:

```cpp
SDL_SetWindowRelativeMouseMode(window, true);
```

Relative mode is what makes mouse-look work at all: with absolute positions the cursor would hit the edge of the window and you would stop turning. Accumulating `xrel` into `yaw` and `yrel` into `pitch` means the mouse is not a thing in the world, it is a rate of change of orientation. The `±89°` clamp on pitch exists because at `90°` the yaw axis degenerates and the view flips over — the graphics version of the gimbal lock that wrecks naive Euler-angle camera code.

Keyboard works differently: it is a *state*, not an event, so I poll it every frame instead of waiting to be told something happened:

```cpp
const bool* keyboard = SDL_GetKeyboardState(nullptr);

if (keyboard[SDL_SCANCODE_W]) {
    camera = vec_add(camera, vec_scale(forward, movement));
    needs_render = true;
}
```

(That is `const bool*` rather than SDL2's `const Uint8*` — one of the small API changes in SDL3 that you meet as a compile error.)

The bit I got wrong first, and which is worth stealing: **movement must be relative to where you are looking.** My first version moved along world axes, so pressing W walked you in a straight global line regardless of where you had turned the camera. The fix is to rotate the movement basis by yaw:

```cpp
Vec3 forward = rotate_vector({ 0.0f, 0.0f, 1.0f }, yaw, 0.0f);
Vec3 right = rotate_vector({ 1.0f, 0.0f, 0.0f }, yaw, 0.0f);
```

WASD now means "forward and right *from my point of view*", which is what a player expects. And note the yaw rotation function is the same `rotate_vector` used for rays, reused for input — the same maths, different question.

**Presentation.** The render target is a plain `vector<Color>`, and getting it on screen means pushing those bytes into a texture. SDL3 gives you a raw pointer and a pitch, and you write memory directly:

```cpp
auto* pixel_buffer = static_cast<Uint8*>(pixels);

for (int y = 0; y < height; y++) {
    auto* row = reinterpret_cast<Uint32*>(pixel_buffer + y * pitch);

    for (int x = 0; x < width; x++) {
        const Color& color = image[y * width + x];
        row[x] = (static_cast<Uint32>(color.r) << 24) |
            (static_cast<Uint32>(color.g) << 16) |
            (static_cast<Uint32>(color.b) << 8) |
            255;
    }
}
```

RGBA8888 means four bytes per pixel in memory in R, G, B, A order, so one 32-bit store per pixel packs the colour exactly. `pitch` is the row stride in bytes and may include padding, which is why you must offset by `y * pitch` rather than assuming rows are contiguous. This is the moment where graphics programming stops being abstract and starts being memory: a `Color` struct in a vector is not an image until somebody writes those bytes into a buffer the GPU will read.

And a small SDL3 gotcha that is still in the source as a comment, because I would hit it again:

```cpp
// FIX: In SDL3, SDL_LockTexture returns true on success, false on failure
if (!SDL_LockTexture(texture, nullptr, &pixels, &pitch)) {
```

SDL2 returned an error code; SDL3 returns a bool. Porting code between them is mostly this.

**The dirty flag.** The last piece is the one I like most, because it is a tiny decision with a real lesson in it:

```cpp
if (needs_render) {
    get_curr_frame(image, camera, yaw, pitch);
    fill_texture(texture, image);
    needs_render = false;
}
```

`needs_render` is set by any input, and cleared after a frame is computed. If nothing moved, the renderer does *no work at all* — the loop keeps presenting the same texture. An entire CPU ray tracer costs literally nothing while the mouse is still.

This is not an optimization I would ship in a real engine, and that is the point: it is the smallest possible demonstration that rendering is a function of state, and that you decide when to recompute it. Real engines render continuously because their worlds animate on their own — a flag, a particle, a light that flickers — and this scene has none of those, so nothing would ever change. The moment something animates by itself, "render on change" becomes "render every frame", and that transition is where real-time graphics actually begins.

There is a second reason the dirty flag matters, and it is the one that made the interactive version feel instantly different. When I moved from a single render to a loop, I stopped being able to ignore the frame budget: I could see, with my own eyes, that dragging the mouse re-renders the whole image while holding it still costs nothing. That feedback is what turns performance from an abstraction into something you notice. I do not think I would have cared about the linear scan in `closest_intersection` as much if I had never watched the cost of a frame change under my hands.

## What this renderer is not

I want to be accurate about the edges of this project, because a ray tracer blog post that oversells itself is not useful to anyone.

- **The camera is not a real camera.** `rotate_vector` is applied to ray *directions only*; the spheres never move. So looking around rotates the world around you rather than turning your head in it. It behaves correctly as long as the camera stays near the origin, which is exactly where I leave it.
- **One ray per pixel.** No jitter, no supersampling — so no antialiasing, and the silhouettes are visibly stair-stepped. This is the first thing I would fix, and the fix is conceptually trivial: take several rays per pixel with small offsets and average them.
- **Hard shadows, mirror reflections only.** No roughness, no refraction, no subsurface, no textures or UVs, no tone mapping or gamma correction. Colours are 8-bit integers multiplied in whatever space they happen to be in; a real pipeline would light in linear space and convert to sRGB at the end, and would use floating point HDR values instead of `int`.
- **No spatial acceleration, no threads.** Every ray scans every sphere. Fine at four spheres; hopeless at four thousand.
- **The "floor" is a sphere.** `{{0, -5001, 0}, 5000, ...}` is a sphere of radius 5000 whose top surface sits at `y = -1` — close enough to a plane that you cannot tell, which is a genuinely useful trick (approximate a flat plane with an enormous sphere) and also a confession that I never implemented plane intersection.
- **The scene is rebuilt every frame.** `get_curr_frame` constructs a fresh `vector<Sphere>` and `vector<Light>` on every render. It is the most obvious thing to fix and I have left it, because I would rather have a visible inefficiency I understand than a hidden one I do not.

None of these are limitations I regret. The renderer does exactly one thing — decide a colour per pixel by tracing rays — and it does it with no abstractions in the way.

## A map of the file

If you want to read the code alongside this, the whole project is one file and it is ordered roughly the way the ideas above arrived:

| What it does | Where |
| --- | --- |
| `Vec3`, `Color`, `Sphere`, `Light` types | [L20–40](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L20-L40) |
| Vector helpers, `reflect_ray`, `rotate_vector` | [L42–110](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L42-L110) |
| `put_pixel` — screen ↔ image coordinates | [L112–120](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L112-L120) |
| `intersect_ray` — the quadratic | [L122–137](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L122-L137) |
| `closest_intersection` — pick the nearest hit | [L139–158](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L139-L158) |
| `ComputeLighting` — ambient, point, directional, shadows | [L160–199](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L160-L199) |
| `trace_ray` — normals, shading, recursion | [L201–223](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L201-L223) |
| `canvas_to_viewport` + the scene + the pixel loop | [L228–262](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L228-L262) |
| `fill_texture` — image bytes into an SDL texture | [L264–289](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L264-L289) |
| `main` — window, input loop, WASD, frame presentation | [L291–401](https://github.com/premkumar-ch/Rays.exe/blob/main/RayTracer.cpp#L291-L401) |

Read it top to bottom on a quiet afternoon and you can reconstruct the whole article from the source.

## The bigger picture

A modern graphics engine — the kind that renders a film, or a game that runs at 144 frames per second — looks, from the outside, like an incomprehensible wall of abstractions. Layers upon layers: scenes, materials, pipelines, render graphs, shader compilers, temporal accumulation, dozens of megabytes of code hiding behind a `draw()` call.

And when you strip it away, the bottom is remarkably small. At the lowest level, a renderer is still manipulating:

- numbers, and vectors of numbers
- geometry, and the rules for intersecting it
- memory, laid out so the hardware can read it efficiently
- pixels, one colour at a time
- and algorithms, deciding what to compute and what to skip

Everything above that line is engineering: abstraction, ergonomics, performance, tooling, parallelism — all of it real and all of it necessary, and none of it magic. The abstractions are load-bearing, but they are not *fundamental*. Underneath, someone is deciding a colour for a pixel, the same way they have been since 1968.

That is what `Rays.exe` is for. It is not a production renderer and it was never meant to be one — it has no acceleration structure, no materials to speak of, no antialiasing, and it will not render anything interesting in under a second. But I built every part of it from arithmetic, and I can explain any line in it.

The point was never the image. The point was to find out what was actually happening under the abstractions — and to discover, once the trigonometry stopped being frightening, that graphics programming is not a separate subject at all. It is linear algebra, memory layout, and a very long argument about how much of it you can afford to compute before the frame is due.

That is worth understanding, and it turns out to be a surprisingly short distance from where I started.

Source, build instructions and controls: [`github.com/premkumar-ch/Rays.exe`](https://github.com/premkumar-ch/Rays.exe)

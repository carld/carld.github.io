    Title: Port the Ray Tracer to GLSL
    Date: 2017-06-06T00:43:23
    Tags: Recurse Center, Ray Tracer, GLSL
    Authors: Carl

_Objective: Implement the path tracer in GLSL_

In hope of improved performance and as an exercise in learning GLSL I attempted
to implement the C Ray Tracer in GLSL.

There were two main challenges:

1. GLSL does not support recursion.
2. GLSL does not come with any built-in pseudo random number generator

Apparently due to lack of a stack in the GLSL architecture, recursive function
calls are not permitted by the GLSL compiler.

In my original ray tracer implementation there was only one place where recursion
was being used: when a ray would hit an object
in the scene, the `color` function would be called recursively using the hit point
as the origin for the bounced ray.
It turned out to be a bit tricky to figure out how to translate this recursive
call into an iterative one.

The `color` function takes a ray and returns an RGB value. A ray is composed of
two 3 dimensional vectors, the origin of the ray, and the direction of the ray.
When the ray intersects with an object in the scene, that object contributes
to the RGB value that is returned. That contribution depends on the material of
the object, how much and what color it reflects and refracts.
The color value returned is a product of the color computed at each intersection.
Color can be seen as an accumulator, accumulating with each hit of an object.
Since the color value has only multiplication applied to it, order does not matter.
The first step was to create a local color variable - as previously this value was being
stored on the stack in the recursive implementation.
The next step was to create a loop that would stop either at a maximum number of
bounces of the ray, or on hitting an object where the ray stops.

It's worth a mention here that because the rays start at the camera or eye,
the path tracing approach sends rays in reverse to what occurs in nature where rays
of light begin at the source of light and end at the camera or eye.

Regarding pseudo random number generation a popular approach is to use a
function that returns a number from a largely irregular sequence.
Typically a modulating function like sine is involved.
This PRNG function takes a seed value, whereas in the C implementation I'd
been using `drand48` which does not. This involved some bug fixing on my part
to ensure that where the PRNG function was used that it was seeded with a
different value. As the PRNG returns the same value for the same seed.

With those resolved the remainder of the code was straight forward to write
in GLSL. It's especially nice to have built-in vector types complete with
operator overloads.

    Title: Raytracing at Recurse Center Summer 2017
    Date: 2017-05-31T22:31:55
    Tags: Recurse Center, Ray Tracer
    Authors: Carl

_Objective: Implement a ray tracer_

I bought the Kindle mini book, "Ray Tracing in One Weekend" by Peter Shirley, to use as a guide.
Thanks to some help from people at Recurse Center I got through this pretty
quickly.

To give myself a challenge I decided to use C rather than C++ as used by the book.
Also I had an idea that later on I could try to port the code to a more functional language like Haskell or Scheme. I decided to render
in an X11 window instead of an image file.

Ray tracing is about simulating rays of light as they reflect off
and refract through objects before hitting something that absorbs them.
To produce an image the ray must eventually reach a camera (or eye).

A basic procedure for ray tracing might look like:
1. For each pixel in screen create a ray from the origin (eye/camera) through a projection plane (screen/viewport),
2. If the ray hits something in the viewing area, apply the effect (color, reflection, refraction) of the hit object to the ray,
3. Set the new origin and direction of the ray and go to 2.

Rays of light tend not stop on hitting something, rather they bounce off in a new direction.
This is why ray tracers are be said to be recursive:
the point where a ray hits becomes the new origin for the ray,
and the combination of direction and surface hit provides the new direction for the ray.
This repeats ad infinitum.

A ray travels in a direction that can be represented as a vector in three dimensions, x, y and z.
A ray also has an origin point in space, which can also be represented as a vector in three dimensions, x, y and z.

As the ray bounces off objects, it's direction changes.
The new direction can be calculated by adding/subtracting/multiplying vectors.

E.g. if there are two vectors v1 and v2, to add them the computer must perform the following in pseudo assembly:

```
ADD v1.x v2.x
ADD v1.y v2.y
ADD v1.z v2.z
```

Early on I discovered why C++ is often preferred for implementing Ray Tracers:
operator overloading.

In C code, using a vector structure, VECTOR, the code to add v1 and v2 might look like:

```
VECTOR result, v1, v2;
result.x = v1.x + v2.x;
result.y = v1.y + v2.y;
result.z = v1.z + v2.z;
```

Much of the time three lines of code are needed for an operator involving three dimensional vectors.

In an object oriented language like C++ this repetition can be reduced by creating a function that operates on all of the dimensions in the vector equally.
When the compiler sees an operator for example `+` called with the relevant types the overloaded function is used.

```
Vec3 operator+(Vec3 v1, Vec3 v2) {
  return Vec3(v1.x + v2.x, v1.y + v2.y, v1.z + v2.z);
}
```

In C++ code, using a vector class (or struct), Vec3, the code to add v1 and v2 might look like:

```
Vec3 result = v1 + v2;
```
This is one line of code, yet the compiler is generating instructions to add each dimension thanks to
the overloading of the operator, shown above.

An advantage to operator overloading is the compiler will apply the precedence and associativity that has for the given operator.
Even when the operator is overloaded the Brackets > Exponents > Division > Multiplication > Addition > Subtraction rules remain the same for binary operators like `+`,`-`,`*`.

In my C implementation I implemented functions that could operate on vector structures, for example

```
vec3 vec3_add_vec(vec3 v1, vec3 v2) {
  vec3 v_ = { .x = v1.x + v2.x, .y = v1.y + v2.y, .z = v1.z + v2.z };
  return v_;
}
```

The hope was that such functions would prevent repetition, which it did, however I found early on when chaining for example addition with multiplication, I had to explicitly call the functions according to their matching operator precedence, e.g.

```
vec3_add(v1,
  vec3_multiply(v2, v3)))
```

Having to observe this in all the right places resulted in my code being susceptible to bugs involving operator precedence.

For more information relating to operator precedence and associativity in C and C++ a useful search term is Sequence Points.

With help from a fellow Recurser (thanks Tim!) I successfully implemented the Ray Tracer from the book in C.

The resulting code can be found here [](https://github.com/carld/ray-tracer)

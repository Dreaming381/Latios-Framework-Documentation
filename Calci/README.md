# Calci

Calci is a math and algorithms module for general-purpose game development. It
offers high performance implementations that are architecture-agnostic, which
you can incorporate into your own data structures as needed.

Check out the [Getting Started](Getting%20Started.md) page!

## Features

### Rng and RngToolkit

There are three common strategies for using random numbers in DOTS ECS. The
first is to store the `Random` instance in a singleton, which prevents
multithreading. The second is to store several `Random` instances in an array
and access them using `[NativeThreadIndex]` which breaks determinism. The third
is to store a `Random` instance on every entity which requires an intelligent
seeding strategy and consumes memory bandwidth.

There’s a way better way!

`Rng` is a new type which provides deterministic, parallel, low bandwidth random
numbers to your jobs. Simply call `Shuffle()` before passing it into a job, then
access a unique sequence of random numbers using `GetSequence()` and passing in
a unique integer (`chunkIndex`, `entityInQueryIndex`, ect). The returned
sequence object can be used just like `Random` for the remainder of the job. You
don’t even need to assign the state back to anything.

`Rng` is based on the Noise-Based RNG presented in [this GDC
Talk](https://www.youtube.com/watch?v=LWFzPP8ZbdU) but updated to a more
recently shared version:
[SquirrelNoise5](https://twitter.com/SquirrelTweets/status/1421251894274625536)

However, if you would like to use your own random number generation algorithm,
you can use the `RngToolkit` to help convert your `uint` outputs into more
desirable forms.

See more: [Rng and RngToolkit](Rng%20and%20RngToolkit.md)

### Curves

Need to evaluate a cubic Bezier spline? What about a sequence of parameter
keyframes? Calci has the algorithms to do these things. Not only that, but the
APIs are agnostic to the array storage type you use. So whether your data comes
from a `DynamicBuffer` or a `BlobArray`, Calci can evaluate it for you.

### Search

Calci has a binary search partitioning algorithm. Unlike the binary search in
the collections package, this algorithm finds the partitioning of an inequality.
It is especially useful when you want a range of values within a sorted list,
which is often the reason to have a sorted list over a hashmap anyways.

### QCP

QCP is a lesser-known algorithm that has some very useful properties. Given a
set of point pairs between two objects, it will calculate a rotation and
optionally an additional translation to apply to the first of the two objects,
such that the overall distances between point pairs are minimized.

### Math

What framework would be complete without some simple math helpers? Not this one.
Various miscellaneous math algorithms and some SIMD stuff are here. Help
yourself!

See more: [Math](Math.md)

## Known Issues

-   There is no automatic algorithm to subdivide large curves to improve
    resolution

## Near-Term Roadmap

-   Unity Splines integration APIs
-   More algorithms

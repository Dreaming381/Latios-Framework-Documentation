# Getting Started with Calci

Some aspects of games are crafted by hand with intention to shape the
interactive experience. But for other aspects, calculating the interactive parts
is the way to go. And that’s why Calci is here. Calci provides highly optimized
mathematics and algorithms that can be dropped into any part of your logic where
you need it.

## Overview

Most of Calci is composed of utility structures and static APIs. Most of the
time, the XML documentation in the package will guide you. There’s no bootstrap
installers, no systems, and no baking.

`Rng` requires the most explanation. It provides deterministic random number
generation that supports parallel jobs, but requires that you set up such jobs a
particular way. Therefore, there is a separate page documenting it
[here](Rng%20and%20RngToolkit.md).

The following provides some hints to the remaining APIs.

## Curves

There are currently two types of curves supported by Calci, 3D cubic Beziers and
value/time cubic Bezier curves.

### 3D Cubic Beziers

3D cubic Bezier curves are typically stored as an array of `BezierKnot`
elements. Such an array is often referred to as a *spline*. Unlike the Unity
Splines package, these splines do not have special orientation values, and are
simply the purest Bezier knot representation of position and two tangents.

A `BezierCurve` can be constructed from any two adjacent `BezierKnot` elements.
This curve represents the path between the two knots relative to an invisible
parameter `t` in the range [0, 1], where a value of 0 corresponds to the
position of the first knot, and the value of 1 corresponds to the position of
the second knot. To perform this conversion, call the static method
`BezierCurve.FromKnots()`. To evaluate the `BezierCurve` for a specific `t`, use
the `BezierMath` APIs such as `EvaluatePosition()`, `EvaluateVelocity()`, and
`EvaluateAcceleration()`.

The value of `t` is for practical purposes meaningless. Often you care more
about the distance traveled along a curve or spline. Unfortunately, there’s no
finite formula that can calculate this correctly. Instead, distance is estimated
by subdividing a `BezierCurve` into 32 segments, and then computing the lengths
of each segment. (For reference, the Unity Splines package uses 30 segments.)
`BezierMath` has various APIs for converting between the factor `t` and the
distance along the curve using `BezierCurve.SegmentLengths`, which can be
extracted by calling `BezierMath.SegmentLengthsOf()`.

To evaluate a full spline based on distance, first call
`BezierMath.CurveAndFactorFromApproximateDistanceAlong()`, passing in a
`ReadOnlySpan<BezierKnot>`. This gives you the index of the first knot and a
factor `t`. As long as the span had more than one element, the second knot can
be obtained by adding 1 to the returned index, even if the distance passed in
exceeds the full length of the spline (in such cases, the result is clamped to
the endpoint of the spline). From the two knots, construct the `BezierCurve` and
then evaluate it using the factor `t`.

### Value vs Time Cubic Beziers

Value vs time cubic Beziers are 2D Bezier curves with a key restriction. With
the exception of instantaneous steps, every x-axis value has exactly 1
corresponding y-axis value. For this reason, the x-axis is referred to as
*time*, while the y-axis is referred to as the *value*. Such curves are always
evaluated indexed by time, rather than the Bezier factor `t`. These types of
curves are most often used for animation of individual floating-point
parameters. Thus, a “knot” in this context is called a `Keyframe`, and an array
of `Keyframe`s is referred to as a *curve track*. The curve between `Keyframe`s
is a `KeyedCurve`.

Unlike the 3D Bezier counterpart, there is no need to approximate with segment
subdivisions. However, using tangent weights of exactly 1/3 makes the Bezier
curve fall within a special subset of curves called a Hermite curve. Hermite
curves are significantly faster to evaluate, so prefer using exactly
`Keyframe.kHermite` for tangent weight values.

The Calci keyframe type defines implicit conversion to and from the
`UnityEngine.Keyframe` type.

## Search

Calci defines the static class `BinarySearch` which offers various overloads of
the method `FirstGreaterOrEqual<T>()`. Unlike with the Collections package
binary search, this implementation does not require a matching value, but
instead finds a partition point in the array. This is most often used to zip to
the first element between a range of values.

## QCP

QCP is an algorithm similar to the Kabsch algorithm for computing the rigid
transform that produces the minimum RMSD of one set of points relative to
another set of target points in a pair-wise manner.

Put in simpler terms, this algorithm allows aligning objects based on object
relative points on each object being paired together. Individual point pairs can
be weighted to bias the result. The algorithm always solves for a rotation, but
it can optionally compute a translation. Using a translation will effect the
result of the rotation, as the rotation will no longer attempt to compensate for
distance error accounted for by the translation.

## simdFloat3

Often when working in 3D projects, `float3` instances and Unity’s `math` class
operations are very useful. However, they don’t make full use of 4-lane SIMD
hardware. In addition, using these types automatically disables Burst’s
auto-vectorization of loops.

`simdFloat3` packs four `float3` instances together into SIMD registers so that
operations can make use of full SIMD hardware throughput. You can perform
operator arithmetic on these types just like normal `float3`s, and you can
access some of the useful `math` functions using the SIMD static class instead.
To access any individual `float3`, you can use the letter properties `a`, `b`,
`c`, and `d` to get or set them. There is also swizzling support for shuffling
the order of these `float3` instances. Accessing a component of all `float3`s
can be performed using the `x`, `y`, and `z` properties as usual, although these
use `float4` values as there are four of them.

Operations between `simdFloat3`s and vector types can be confusing. If a
`simdFloat3 p` were to be multiplied by a `float3 v`, the result will be the
equivalent of multiplying each `float3` in `p` by `v`. However, if `p` were
instead multiplied by `float4 s`, the result will be equivalent to performing
`p.a * s.x, p.b * s.y, p.c * s.z, and p.d * s.w`.

## Latios Math

`LatiosMath` contains random useful functions I purposely refactored because of
frequent usage. It is best to look directly at the code to see what they do.

# Changelog

All notable changes to this package will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/en/1.0.0/)
and this project adheres to [Semantic
Versioning](http://semver.org/spec/v2.0.0.html).

## [0.16.2] – 2026-10-17

Officially supports Entities [1.4.8]

### Fixed

-   Fixed computation in `BezierMath.SegmentLengthsOf()`
-   Fixed edge-case handling for `BezierMath` methods
    `FactorFromDistanceApproximately()` and `DistanceAlongApproximately()`
-   Fixed `BezierMath.CurveAndFactorFromApproximateDistanceAlong()` for more
    than two knots
-   Fixed `BezierKnot.FromCurveEndpointB()` using the wrong vertex
-   Fixed implicit conversion from Calci’s `Keyframe` to `UnityEngine.Keyframe`
    mapping weighted tangent modes incorrectly
-   Fixed `LatiosMath.SmoothSlide()` not scaling the tail region correctly
-   Fixed `BinarySearch` methods throwing or accessing invalid memory when there
    are zero elements
-   Fixed incorrect transformation sequence in `Qcp` when solving for a
    translation
-   Fixed incorrect matrix coefficients in the `Qcp` solver that resulted in
    approximately a 3% error in results

## [0.16.0] – 2026-9-20

Officially supports Entities [1.4.8]

### Added

-   Added `BezierMath.Evaluate()` overloads which allow performing multiple time
    sample evaluations against a `KeyedCurve` sequence, offering improved
    performance in common scenarios
-   Added `BezierMath.Integrate()` methods for computing the integrals of
    `KeyedCurve` instances
-   Added `InitTickedSystemRng()`, `GetTickedMainThreadRng()`, and
    `GetTickedJobRng()` extension methods for `SystemState`, offering
    ticking-specialized counterparts to the normal `SystemRng` utility methods
    along with a backing component `TickedSystemRngSeedAndLastVersion`
-   Added `RngEach` and `TickedRngEach` which can be used as parameters inside
    an `IJobEach`

### Improved

-   Added UNITY_DOTS_DEBUG support to safety checks

## [0.15.11] – 2026-7-18

Officially supports Entities [1.4.8]

### Added

-   Added `simd.select()` overload which can select between two `simdFloat3`
    instances at the per `x`, `y`, and `z` level

## [0.15.0] – 2026-4-12

Officially supports Entities [1.4.4]

### Added

-   Added static class `ComputationalGeometry2D` which provides a frontend for
    Burst-compatible Clipper2 operations
-   Added `NativePriorityQueue` which is a quaternary heap priority queue using
    an `IComparer` and backed by a `NativeList`

## [0.14.8] – 2026-1-3

Officially supports Entities [1.4.3]

### Added

-   Added static class `MathShortcuts` and method `MultiplyTransformMatrices()`
    which performs faster matrix multiplication for presentation purposes

## [0.14.5] – 2025-12-13

Officially supports Entities [1.3.14]

### Added

-   Added `RngSequence` extension method `NextAccretionDiskPoint()` which is a
    patterned point distribution method

## [0.14.0] – 2025-10-18

Officially supports Entities [1.3.14]

### This is the first release of *Calci*.

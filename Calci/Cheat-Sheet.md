# Calci Cheat Sheet

## Random Numbers

-   `Rng` – Deterministic random numbers that survive parallel jobs
    -   `Shuffle()` – Call once per job scheduled, before passing the `Rng` in
    -   `GetSequence(int index)` – A unique sequence per chunk or per entity
-   `Rng.RngSequence` – Used just like `Unity.Mathematics.Random`
    -   `NextBool()`, `NextInt()`, `NextUInt()`, and `NextFloat()` – Plus wide
        and ranged variants
    -   `NextFloat3Direction()` and `NextQuaternionRotation()` – Uniform
        directions and orientations
    -   `ShuffleElements()` – Reorders a `NativeArray` or `INativeList` in-place
-   `RngEach` – Random numbers in an `IJobEach` with zero setup
-   `TickedRngEach` – Same as `RngEach`, but reproducible per tick
-   `SystemRng` – For `IJobEntity` wanting a sequence per chunk, not per entity
-   `InitSystemRng()`, `GetJobRng()`, and `GetMainThreadRng()` – `SystemState`
    extensions that manage `SystemRng`, with `Ticked` counterparts
-   `RngToolkit` – Remaps raw `uint` values for your own random generator
-   `PointDistributions.NextAccretionDiskPoint()` – Scattered points along
    spiral arms

## 3D Curves and Splines

-   `BezierKnot` – A position with two tangents, an element in a *spline*
-   `BezierCurve` – The cubic between two adjacent knots, sampled by `t`
    -   `FromKnots()` – Extract from two adjacent knots in a spline
    -   `FromLineSegment()` and `FromQuadratic()` – Construct from simpler
        definitions
-   `BezierMath` – Where all curve evaluation lives
    -   `PositionAt()`, `VelocityAt()`, and `AccelerationAt()` – Sample at a
        factor `t`
    -   `LengthOfApproximately()` – Length of a curve or of a whole spline
    -   `SegmentLengthsOf()` – Compute once, reuse across the distance APIs
    -   `FactorFromDistanceApproximately()` and `DistanceAlongApproximately()` –
        Convert between `t` and distance
    -   `CurveAndFactorFromApproximateDistanceAlong()` – Sample a spline by
        distance traveled
    -   `SplitCurve()` – Divides one curve into two while preserving shape
    -   `TranslateKnot()`, `RotateKnot()`, `ScaleKnot()`, `TransformKnot()` –
        Relocate knots, with matching curve overloads

## Parameter Curves

-   `Keyframe` – Time, value, and tangents, arrayed into a *curve track*
    -   `Keyframe.kHermite` – The tangent weight that unlocks the fast path
-   `KeyedCurve` – The curve between two keyframes, indexed by time
    -   `FromKeyframes()` – Builds the curve from an adjacent pair
-   `BezierMath` – Evaluation and integration of keyed curves
    -   `Evaluate(in KeyedCurve, float)` – One time sample within one segment
    -   `Evaluate(ReadOnlySpan<Keyframe>, float)` – One time sample within a
        full track
    -   `Evaluate(segments, times, results)` – Many time samples at once,
        vectorized for small segment counts
    -   `Integrate()` – Area under the curve, for when it describes a rate

## Search and Containers

-   `BinarySearch.FirstGreaterOrEqual<T>()` – Finds a partition point, not a
    match
-   `NativePriorityQueue<TElement>` – Quaternary heap over a `NativeList`
    -   `Enqueue()`, `TryPeek()`, and `TryDequeue()` – Normal priority queue ops
    -   `EnqueueDequeue()` and `TryDequeueEnqueue()` – Fast compound ops

## Point Alignment

-   `Qcp.Solve()` – The rotation (and optionally translation) that best aligns
    paired point sets

## 2D Computational Geometry

-   `ComputationalGeometry2D.RemoveSelfIntersections()` – Cleans overlapping or
    self-crossing contours

## SIMD

-   `simdFloat3` – Four `float3` values packed into SIMD registers
    -   `a`, `b`, `c`, and `d` – Get or set the individual `float3` values
    -   `x`, `y`, and `z` – One component across all four, as a `float4`
-   `simd` – The `math` equivalent for `simdFloat3`
    -   `dot()`, `cross()`, `length()`, `distance()`, and `normalize()`
    -   `select()` and `shuffle()` – Lane-wise picking and reordering
    -   `cminabcd()`, `cmaxabcd()`, and `csumabcd()` – Reduce across the four
    -   `transform()` and `mul()` – Apply a transform or rotation to all four

## Math Helpers

-   `LatiosMath` – Uncatagorized functions
    -   `SmoothStart()`, `SmoothStop()`, and `SmoothStep()` – Easing curves
    -   `SmoothSlide()` – A `SmoothStep` with a linear middle section
    -   `RotateExtents()` – Conservative AABB extents after a rotation
-   `MathShortcuts.MultiplyTransformMatrices()` – Faster affine `float4x4`
    multiply using hardware intrinsics
-   `xyz1()` and `x0y()` – Global-namespace swizzles `Unity.Mathematics` lacks

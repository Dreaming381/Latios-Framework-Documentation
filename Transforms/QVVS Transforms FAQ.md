# QVVS Transforms FAQ

## Basics for Outsiders

### What does QVVS stand for?

-   Quaternion – rotation
-   Vector – position
-   Vector – stretch
-   Scalar – uniform scale

### Some other framework APIs seem to use TransformQvvs, but I’m using Unity Transforms. Does that mean those features only support QVVS Transforms?

`TransformQvvs` is just a struct that contains transform information. It is not
an ECS component. You can map that to and from Unity Transforms however you
like.

### Isn’t stretch just a simplified PostTransformMatrix?

No. Stretch influences the positions of children.

### What is stretch used for?

Stretch solves big problems in animation. It allows for squashing and stretching
bones without propagating shear and all while maintaining connectivity of the
entities involved.

It can also be useful for gameplay for things like grappling hooks, lasers, and
trigger volumes.

### Is it really worth adding an extra 16 bytes to chunks?

QVVS replaces `LocalToWorld` and `LocalTransform` for root entities, so it uses
considerably less memory in chunks. Static object will often use non-uniform
scaling for decorative effect. And if dynamic objects are characters, they’ll
benefit from the improved animation capabilities.

With all that said, if your game exclusively relies on unscaled or only
uniformly-scaled objects and you want maximum performance, you may wish to use a
custom solution instead.

### If there’s no LocalToWorld, how do you render entities?

The Kinemation module knows how to render the entities. The QVVS is converted to
a matrix in the same compute shader that conputes the matrix inverses. The GPU
performance cost is negligible, while the CPU gains are measurable.

### Is QVVS faster than Unity Transforms

Usually.

### Are there any other features that come with QVVS?

Yes. There are several tools that make it much easier and more performant to set
world-space properties on child entities. Hierarchy handling in general is
better.

## Compatibility

### Does QVVS Transforms support Unity Physics?

No.

### Can you make QVVS Transforms support Unity Physics? I want to use the Character Controller Package.

No.

This is like asking Discord to support old flip phones. You are better off using
SMS or having the flip phone users upgrade. In the same way, you can put the
Latios Framework into Unity Transforms mode or you can modify those other
packages to use QVVS Transforms.

Unity Physics internal algorithms can’t leverage any of the QVVS features
anyways. Psyshock can though, so you could also help implement missing features
in Psyshock and the various add-ons.

### But there’s got to be a way to make QVVS Transforms and Unity Transforms work together, right?

QVVS Transforms are world-space centric, whereas Unity Transforms are
local-space centric. I have yet to find a way to rectify this difference without
ruining all the things that makes QVVS Transforms great in the first place.

If you still aren’t convinced, I’d love to see a design proposal!

### What about 3rd party assets?

Ask them to support QVVS Transforms. They can detect the Latios.Transforms
assembly and define a symbol in their package. Then they can use compiler
directives to check for that symbol and the LATIOS_TRANSFORMS_UNITY symbol to
determine whether to use Latios.Transforms or Unity Transforms.

## Usage

### Do I need to use TransformAspect for writing transforms? Shouldn’t I write to WorldTransform instead?

You should prefer to use the `TransformAspect` type, as writing to
`WorldTransform` directly for an entity with a parent will cause your hierarchy
to get corrupted. Unity’s `IAspect` is going away in the future, but QVVS
Transforms are also getting a redesign, so for now, stick with
`TransformAspect`.

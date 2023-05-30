# Getting Started with QVVS Transforms

The first step when working with QVVS Transforms is to pick a transform system.
There are lots of things to consider when choosing. How big is the simulation?
How much different logic is interacting with transforms? Are there any planned
system ordering conventions when working with transforms? What all needs to read
the world-space transforms? What features are needed?

This page contains a breakdown of the various systems. The default is Cached
QVVS.

Once you have selected your system, scroll down to the corresponding heading to
continue.

## Cached QVVS

Cached QVVS is the default transform system included with the Latios Framework.
You do not need to set any scripting defines to enable it. However, it does
require installation in the Latios Bootstrap.

### Creating a Scene

Create a new Unity Project and install the Latios Framework. Add a *Latios
Bootstrap* using the *Standard – Injection Workflow*.

Next, add the following code to the bootstrap or a custom script. This code will
preserve hierarchies so that we can play with them.

```csharp
class PreserveHierarchyBaker : Baker<UnityEngine.Transform>
{
    public override void Bake(UnityEngine.Transform authoring)
    {
        GetEntity(TransformUsageFlags.Dynamic);
    }
}
```

Create a new subscene. Add a cube inside it at the origin. Then create a child
sphere. Set the sphere’s Y position to 0.5 and switch over to the game view.
Press play. If you did everything right, you should be able to do this:

![](media/1944fa2ea6eb351105aad70e7c3577c4.gif)

Here, we are playing with the `TransformAspect` at runtime. In particular, we
are playing with the `stretch` on the cube. Notice how the sphere moves along
with the top of the cube but does not deform. Play around with the other
`TransformAspect` values. You can switch between local space and world space for
the other three parameters.

## Uncached QVVS

Uncached QVVS is not implemented yet. If you want support for this system,
request it to help boost its priority.

## Unity Transforms Compatibility

Unity Transforms Compatibility Mode is not implemented yet. If you want support
for this system, please get involved in the community, as this system requires
community support to steer its design.

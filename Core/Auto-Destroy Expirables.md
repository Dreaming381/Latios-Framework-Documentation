# Auto-Destroy Expirables

Suppose there is a prefab entity that contains a bunch of effects. Sound
effects, visual effects, passive influence effects, ect. Each of these effects
finish and stop being useful at different points in time. There may be a desire
to destroy this entity once all the effects are finished. Destroying the entity
sooner would result in some effects being cut short. Removing the components
would eventually leave a husk entity behind. And requiring the designer to
specify expiration times is bug-prone.

In addition, requiring a system to know about the statuses of all effect types
creates significant coupling. That is, unless it can be done in a generalized
abstraction. Such an abstraction is what the Expirables feature provides.

To create an expirable feature, simply make an `IComponentData` or
`IBufferElementData` also implement the `IAutoDestroyExpirable` interface, which
inherits the `IEnableableComponent` interface.

The component or buffer will keep the entity **alive** when **enabled**. To
specify the effect is *finished*, you must *disable* the component. These roles
may feel weird at first, but in practice they avoid some Unity ECS pitfalls and
provide better performance.

It is only once all expirable components on an entity are disabled that the
entity will be destroyed. Expirable entities with a `LinkedEntityGroup` will
remove all other expirable entities in the group upon destruction.

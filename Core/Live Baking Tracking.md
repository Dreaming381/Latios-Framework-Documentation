# Live Baking Tracking

Live Baking allows entities to exist at runtime from open subscenes, and allow
for changes in authoring to be immediately reflected in the runtime. However,
this process is not perfect, and comes with alterations in behavior compared to
closed subscenes. In some cases, custom logic is required to detect and correct
for live-baking behavior. The Latios Framework comes with special tools and APIs
to facilitate this.

## LiveBakedTag

In a runtime or editor world, some entities may come from closed subscenes,
while other entities come from open subscenes. If you need to apply additional
handling logic only for entities that come from open subscenes, query for the
`LiveBakedTag`. This tag is added to all entities in a baking system, but
removed in an `EntitySceneOptimizations` system (which only runs when a subscene
is closed).

## Special SuperSystems

Having the tag component helps to focus logic on entities that matter. But
still, sometimes live-baking compensation logic can be expensive, and you only
want to run it when a live bake occurs. For this, you can add your systems to
`BeforeLiveBakingSuperSystem` and `AfterLiveBakingSuperSystem`. These systems
only update when a live bake occurs. The `worldBlackboardEntity` has a
`SystemVersionBeforeLiveBake` component which can be used for change-filtering
specifically for things changed by live baking.

## Incorporating Switches in Existing Systems

If you already have systems that need to update in specific spots in a frame,
but you want to tweak the logic for the rest of the frame after a live-bake, you
can read the `liveBakedThisFrame` property on `LatiosWorldUnmanaged`.

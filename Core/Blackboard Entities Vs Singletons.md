# Blackboard Entities Vs Singletons

Often people wonder why blackboard entities exist when singletons are already
provided by Unity. And while there are plenty of reasons to prefer blackboard
entities, it is important to highlight the primary feature that drove their
creation.

Requiring exactly one instance of a singleton is very frustrating to designers.
If a designer forgets to add the authoring component, error. If the designer has
two instances, error. Nothing works until there is exactly one instance of every
authoring component that creates a singleton in the scene spread across all
subscenes.

Trying to fix this in code is an uphill battle too. Subscenes with the authoring
data don’t load until after `OnCreate` of systems get called, meaning the check
to correct a singleton would have to happen during an update of a system. That
requires a sync point, and also requires that the system update before other
systems that need to read the singleton, or else exceptions occur and the
program could potentially crash. That puts a lot of restrictions on system order
and also incurs a constant runtime cost, because there’s no way to query the
exclusive absence of a singleton. The other option is to have each reader of the
singleton check if the singleton is valid, but that will result in a lot more
error checking at runtime which is never good for performance.

Do we even need singletons? Does there really only have to be one instance? Or
does it only matter that all the systems use the same instance at a given time,
a *designated instance*?

And that’s what blackboard entities are. They are designated instances that
won’t blow up in the designer’s face so easily. Default values can be
initialized in `OnCreate`, and authored instances can merge with or override the
defaults once they load in.

But that’s not the only difference. Here’s some others:

|                 | Singletons                                   | Blackboard Entities                                                                           |
|-----------------|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| Instances       | Requires one component instance              | Multiple component instances allowed                                                          |
| Updates         | Effects RequireMatchingQueriesForUpdate      | Does not effect RequireMatchingQueriesForUpdate by default                                    |
| Usage           | Requires EntityQuery or SystemAPI for access | Access anywhere via BlackboardEntity                                                          |
| Synchronization | Has special sync point rules                 | Uses normal EntityManager sync point rules (collection components solve job use case instead) |
| Memory          | 16 kB per component                          | 16 kB for all components                                                                      |
| Authoring       | No authoring integration                     | Has authoring integration                                                                     |

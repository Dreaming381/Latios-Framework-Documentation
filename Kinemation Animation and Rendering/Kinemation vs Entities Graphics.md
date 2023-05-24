# Kinemation vs Entities Graphics

Entities Graphics is one of the most advanced ECS packages Unity has ever
written. You can learn a lot about high-performance design from deep-diving into
the code. However, with a fresh perspective, there are also many opportunities
for improvement, especially with regards to skinned meshes. The table below
highlights many of the changes Kinemation makes to Entities Graphics in order to
achieve its top-class performance goals.

|                            | Entities Graphics                                         | Kinemation                                                                                               |
|----------------------------|-----------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Dirty material properties  | Uploaded to GPU every frame                               | Only chunks with visible entities have properties uploaded, otherwise properties are remembered as dirty |
| Matrix property uploads    | 4 12-byte copies per matrix                               | Single QVVS array memcpy per chunk; conversion to matrix happens on GPU                                  |
| Triangle winding detection | Matrix determinant                                        | QVVS sign counting                                                                                       |
| Light probes               | Updated on main thread without Burst                      | Updated in job with Burst                                                                                |
| Culling                    | Single callback does everything                           | Callback delegates to extensible ComponentSystemGroup with Burst-compiled systems                        |
| Culling filters            | Applied last                                              | Applied first                                                                                            |
| Rendering toggle           | Tag component                                             | IEnableableComponent                                                                                     |
| Skinning culling           | Culled by mesh                                            | Culled by skeleton                                                                                       |
| Skinning computation       | All meshes of all LODs every frame                        | Only visible meshes of active LODs                                                                       |
| Skinning motion vectors    | Cached from previous frame                                | Recomputed each frame                                                                                    |
| Skin matrices              | Provided by user in Dynamic Buffer                        | Computed on GPU from shared bone transforms                                                              |
| Skinned Mesh Baking        | Requires full transform hierarchy                         | Supports optimized transform hierarchy                                                                   |
| Root bones                 | Requires specified root bone                              | GameObject with Animator is root bone                                                                    |
| Culling bounds             | Estimated from animator                                   | Accurately computed at runtime                                                                           |
| Skinning cache             | Only Compute skinning supported                           | Supports vertex and compute skinning                                                                     |
| Skinning data source       | Dynamically retrieved from mesh at runtime, allocating GC | Retrieved from ref-counted BlobAssets in jobs, no GC                                                     |
| Skinning batching          | Batched by mesh with unique dispatch call per mesh        | Batched by skeleton in single dispatch                                                                   |
| Compute skinning algorithm | Iterate by weight by vertex and load from memory          | Walk batched cached-coherent linked list with matrices cached in groupshared when possible               |
| Blend shape algorithm      | Iterate by weight by vertex                               | Iterate batched vertex groups with memory barrier between batches                                        |

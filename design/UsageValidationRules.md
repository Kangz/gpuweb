## Definition of "resources being used"

Resources in WebGPU can be used in different ways that are declared at resource creation time and interact with various validation rules.

Buffers have the following usages (and commands inducing that usage):
 - `VERTEX` for the `buffers` in calls to `WebGPUCommandEncoder.setVertexBuffers`
 - `INDEX` for `buffer` in calls to `WebGPUCommandEncoder.setIndexBuffer`
 - `INDIRECT` for `indirectBuffer` in calls to `WebGPUCommandEncode.{draw|drawIndexed|dispatch}Indirect`
 - `UBO`, `RO_STORAGE` and `STORAGE` for buffers referenced by bindgroups passed to `setBindGroup`, with the usage corresponding to the binding's type.
 - `TRANSFER_SRC` for buffers used as the copy source of various commands.
 - `TRANSFER_DST` for buffers used as the copy destination of various commands.
 - (Maybe [RO_]STORAGE_TEXEL and UNIFORM_TEXEL?)

Textures have the following usages:
 - `OUTPUT_ATTACHMENT` for the subresources referenced by `WebGPURenderPassDescriptor`
 - `SAMPLED`, `RO_STORAGE` and `STORAGE` for subresources corresponding to the image views referenced by bindgroups passed to `setBindGroup`, with the usage corresponding to the binding's type.

Read only usages are `VERTEX`, `INDEX`, `INDIRECT`, `UBO`, `RO_STORAGE` and `SAMPLED`.

Note that here we split `RO_STORAGE` and `STORAGE` compared to other grphics APIs and that creating a resource with `STORAGE` implies `RO_STORAGE`.

## Render passes

In render passes the only writable resources are textures used as `OUTPUT_ATTACHMENT` and resources used as `STORAGE`.

To avoid data hazards the simplest validation rules would be to check for every subresource that it is used as either:
 - `OUTPUT_ATTACHMENT`
 - A combination of read-only usages
 - A combination of `STORAGE` and `RO_STORAGE`

If there is no usage of `STORAGE` then there are no data-races as everything is read-only, except `OUTPUT_ATTACHMENT` which is well-defined.
`STORAGE` is an escape hatch and the order of reads and writes to the same `[RO_]STORAGE` resources is undefined.

Implementations might need to track usage per mip-level per layer of a resource, if this is deemed too costly, we could only have two sub-resources tracked in textures: the part as `OUTPUT_ATTACHMENT` and the rest.
This would mean for example that a texture couldn't be used as both  `[RO_]STORAGE` and a read-only usage inside a render pass.
The state tracking required in implementation would become significantly simpler at the cost of flexibility for the applications.

## Compute passes

They are similar to render passes, every subresource (alternatively every resource) must be used as either:
 - A combination of read-only usages
 - A combination of `STORAGE` and `RO_STORAGE`

By default implementation ensure there are no data-races between two dispatches.
However WebGPU should provide an opt-out as part of a `WebGPUComputePassDescriptor` such as `WebGPUComputePassDescriptor.manualStorageBarriers`.

## Other operations

Assuming we don't have "copy passes" and that other operations are top-level, then a command buffers looks like a sequence of:
 - Compute passes
 - Render passes
 - Copies and stuff

Implementations ensure that there are no-data races between any of these operations.
There are validation rules to check for example that the source and destinations regions of buffer copies don't have overlap.

## Discussion

In this proposal the only sources of data races come from the `STORAGE` usage:
 - In render passes inside a single drawcall and between drawcalls for accesses to memory wrote to as `STORAGE`.
 - In any compute passes inside a single dispatch for accesses to memory wrote to as `STORAGE`
 - In relaxed compute passes between dispatches for accesses to memory wrote to as `STORAGE`

When using the more constrained version of usage validation for textures, the cost of validation is O(commands).
With the less constrained version, things can get complicated if a texture's subresource usage looks like a checkerboard, but should be O(commands) in the real-world.

The cost of the state tracking necessary to add memory barriers inside compute passes and at command buffer top-level seems manageable (though difficult to describe in the 30 minutes I have before the meeting).

Cost of implicit barriers

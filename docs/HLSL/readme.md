## HLSL

**Shader Semantics**

[Semantics](https://msdn.microsoft.com/en-us/library/bb509647(v=vs.85).aspx#System_Value) are a way to tell the Input Assembler how to link the buffer data supplied by the application to the input parameters expected by the shader. Semantics are also the language syntax used to link output parameters from one shader stage to the input parameters to another shader stage.

In an HLSL shader the semantic name for the variable follows the colon (:) character in a variable declaration.

* SV_PrimitiveID
uint primID : SV_PrimitiveID

When this semantic is specified, it tells the input assembler stage to automatically
generate a primitive ID for each primitive. When a draw call is executed to draw
n primitives, the first primitive is labeled 0; the second primitive is labeled 1; and
so on, until the last primitive in the draw call is labeled n-1. The primitive IDs are
only unique for a single draw call.

If a geometry shader is not present, the primitive ID parameter can be added to
the parameter list of the pixel shader. However, if a geometry shader is present, then the primitive ID parameter must
occur in the geometry shader signature. Then the geometry shader can use the
primitive ID or pass it on to the pixel shader stage (or both).

* SV_VertexID
uint vertID : SV_VertexID

For a Draw call, the vertices in the draw call will be labeled with IDs from 0, 1, …,
n-1, where n is the number of vertices in the draw call. For a DrawIndexed call,
the vertex IDs correspond to the vertex index values.













Shader Model 5.1 introduced the ConstantBuffer template construct in order to enable support for descriptor arrays. See [Resource Binding in HLSL](https://docs.microsoft.com/en-us/windows/desktop/direct3d12/resource-binding-in-hlsl#constant-buffers) for more information.










Each invocation of the vertex shader operates over a single vertex (as opposed to triangles or the entire mesh) and outputs the transformed vertex. Many vertices are processed in parallel and it is not possible to modify variables defined within the scope of the vertex shader and expect that other invocations of the vertex shader will see those changes (even if you declare the variable as static in the global scope of the vertex shader!). For example, you cannot declare a counter variable in the vertex shader and allow each invocation to increment that counter to see how many vertices were processed (you may be able to achieve this using atomic counters, but that is beyond the scope of this lesson).

If a 0.0 was used in the last component of the vector, then the vector would be rotated but not translated.
If the vertex attribute is a position vector, then the 4th component of the vector must be a 1.
If the vertex attribute is a normal vector, then the 4th component of the vector must be a 0.
Positional vectors are often referred to as points (because it represents a point in space). Directional vectors are simply referred to as vectors.

the OUT parameter is returned and the result is passed as input to the Rasterizer stage. The rasterizer will use the Position parameter to determine the pixel’s coordinates (in screen space, by applying the viewport), the depth value (in normalized-device coordinate space) which is written to the depth buffer, and the interpolated Color parameter is passed as input to the pixel shader stage.

### Pixel Shader

The purpose of the pixel shader is to produce the final color that should be written to the currently bound render target(s). The pixel shader can write to a maximum of eight color targets and one depth target.

The pixel shader takes the interpolated color value from the Rasterizer stage and outputs that color to the only bound render target using the [SV_Target](https://msdn.microsoft.com/en-us/library/bb509647(v=vs.85).aspx#System_Value) system value semantic.

``` hlsl
struct PixelShaderInput
{
    float4 Color    : COLOR;
};

float4 main( PixelShaderInput IN ) : SV_Target
{
    return IN.Color;
}
```

### Compute Shaders
A Compute Shader is a programmable shader stage but it cannot be used in a graphics pipeline. Instead, a compute shader must be configured as the only stage of a compute pipeline. Similar to vertex and pixel shaders, a compute shader is defined using HLSL in DirectX but a compute shader does not operate on vertices or pixels. A compute shader is used to create a general purpose program. A compute shader can operate on any data. One of the challenges of writing a compute shader is determining how to organize the data (both input and output) that a compute shader operates on.

### Dispatch
A compute shader is executed as a dispatch. The dispatch can be considered the execution domain of the compute shader. A dispatch is executed using the [ID3D12GraphicsCommandList::Dispatch](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/nf-d3d12-id3d12graphicscommandlist-dispatch) method. The command list type must be either [D3D12_COMMAND_LIST_TYPE_COMPUTE](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_command_list_type) or [D3D12_COMMAND_LIST_TYPE_DIRECT](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_command_list_type). A dispatch cannot be executed on a copy command list.

The ID3D12GraphicsCommandList::Dispatch method accepts three parameters: ThreadGroupCountX, ThreadGroupCountY, and ThreadGroupCountZ. This implies that the dispatch defines a 3D domain. The maximum number of thread groups that can be dispatched is 65,535 in each the X, Y, and Z dimension. The name thread group implies that what is being dispatched is a group of threads. The number of threads in a thread group is determined by the numthreads attribute defined in HLSL. A thread group can have a maximum of 1,024 threads (D3D12_CS_THREAD_GROUP_MAX_THREADS_PER_GROUP) and a maximum of 1,024 threads in the X, and Y dimensions but only 64 threads in the Z dimension.

Thread groups are further divided into waves when executed on the GPU. The threads within a wave are executed in lockstep which means that the instructions in a wave are all executed in parallel on a single streaming multiprocessor. The number of threads in a wave is dependent on the GPU vendor. The number of threads in a wave on a NVidia GPU is typically 32 and 64 on an AMD GPU.

There are several System Value Semantics that can be used to query the index of a thread within a compute shader.
* SV_GroupID: The 3D index of the thread group within the dispatch.
* SV_GroupThreadID: The 3D index of the thread within a thread group.
* SV_DispatchThreadID: The 3D index of the thread within the dispatch.
* SV_GroupIndex: The flattened 1D index of the thread within the thread group.
Unfortunately, it is not possible to query the total number of thread groups in a dispatch or the total number of threads in a thread group. The number of groups in a dispatch must be sent as an argument to the compute shader (using a constant buffer or 32-bit constants).

![image](../images/Compute-Dispatch-1.png)

The above image depicts a Dispatch of (2,2,1) thread groups. Each thread group has (4,4,1) threads. In this example, the SV_DispatchThreadID of the last thread in the dispatch is: =(1,1,0)⋅(4,4,1)+(3,3,0)=(4,4,0)+(3,3,0)=(7,7,0) And the SV_GroupIndex of the last thread in the dispatch is: =(0⋅4⋅4)+(3⋅4)+3=0+12+3=15





If there is no geometry shader (geometry shaders are covered in Chapter 12),
then the vertex shader must output the vertex position in homogenous clip space
with the SV_POSITION semantic because this is the space the hardware expects the
vertices to be in when leaving the vertex shader (if there is no geometry shader).
If there is a geometry shader, the job of outputting the homogenous clip space
position can be deferred to the geometry shader.

A vertex shader (or geometry shader) does not do the perspective divide; it just
does the projection matrix part. The perspective divide will be done later by the
hardware .


As a hardware optimization, it is possible that a pixel fragment is rejected by
the pipeline before making it to the pixel shader (e.g., early-z rejection). This is
where the depth test is done fi rst, and if the pixel fragment is determined to be
occluded by the depth test, then the pixel shader is skipped. However, there are
some cases that can disable the early-z rejection optimization. For example, if
the pixel shader modifi es the depth of the pixel, then the pixel shader has to be
executed because we do not really know what the depth of the pixel is before the
pixel shader if the pixel shader changes it.










### Dynamic Indexing

Dynamic indexing is new to shader model 5.1 and and allows us to dynamically
index an array of texture resources, where the textures in this array can be
different sizes and formats. One application of this is to bind all of our texture
descriptors once per frame, and then index into the texture array in the pixel
shader to use the appropriate texture for a given pixel.

We dynamically index into an array of resources in a shader program.
The index can be specified in various ways:
1. The index can be an element in a constant buffer.
2. The index can be a system ID like SV_PrimitiveID, SV_VertexID, SV_
DispatchThreadID, or SV_InstanceID.
3. The index can be the result of come calculation.
4. The index can come from a texture.
5. The index can come from a component of the vertex structure.

we want to minimize the number of descriptors we set on a per render-item basis. Right now we set the object constant buffer, the material constant buffer, and the diffuse texture map SRV on a per render-item basis. Minimizing the number of descriptors we need to set will make
our root signature smaller, which means less overhead per draw call; moreover,
this technique of dynamic indexing will prove especially useful with instancing.

1. Create a structured buffer that stores all of the material data. That is,
instead of storing our material data in constant buffers, we will store it in a
structured buffer. A structured buffer can be indexed in a shader program.
This structured buffer will be bound to the rendering pipeline once per frame
making all materials visible to the shader programs.
2. Add a MaterialIndex field to our object constant buffer to specify the index of
the material to use for this draw call. In our shader programs, we use this to
index into the material structured buffer.
3. Bind all of the texture SRV descriptors used in the scene once per frame,
instead of binding one texture SRV per render-item.
4. Add a DiffuseMapIndex field to the material data that specifies the texture map
associated with the material. We use this to index into the array of textures we
bound to the pipeline in the previous step.
With this setup, we only need to set a per object constant buffer for each renderitem.
Once we have that, we use the MaterialIndex to fetch the material to use for
the draw call, and from that we use the DiffuseMapIndex to fetch the texture to use
for the draw call.

Recall that a structured buffer is just an array of data of some type that can live
in GPU memory and be accessed by shader programs. Because we still need to be
able to update materials on the fly, we use an upload buffer rather than a default
buffer.

### Instancing

Instancing refers to drawing the same object more than once in a scene, but with
different positions, orientations, scales, materials, and textures.

It would be wasteful to duplicate the vertex and index data for each instance.
Instead, we store a single copy of the geometry (i.e., vertex and index lists) relative
to the object’s local space. Then we draw the object several times, but each time
with a different world matrix and a different material if additional variety is
desired.

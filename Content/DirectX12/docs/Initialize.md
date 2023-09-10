## Initialize DirectX 12

1. Enable the Direct3D 12 Debug Layer   // Always enable the debug layer before doing anything DX12 related
2. Register the Window Class
3. Create Window Instance               // AdjustWindowRect
4. Query DirectX 12 Adapter
5. Create the DirectX 12 Device         // Enable debug messages in debug mode.
6. Create the Command Queue
7. Check for Tearing Support
8. Create the Swap Chain
9. Create a Descriptor Heap
10. Create the Render Target Views
11. Create a Command Allocator
12. Create a Command List               // Before the command list can be reset, it must first be closed.
13. Create a Fence
14. Create an Event
15. Window Message Procedure

**Graphics Pipeline**

![image](../images/DirectX-12-Rendering-Pipeline.png)
The image illustrates the various stages of the DirectX 12 rendering pipeline. The blue rectangular blocks represent the fixed-function stages and cannot be modified programmatically. The green rounded-rectangular blocks represent the programmable stages of the graphics pipeline.

**Command List**
A Command List is used to issue copy, compute (dispatch), or draw commands. In DirectX 12 commands issued to the command list are not executed immediately like they are with the DirectX 11 immediate context. All command lists in DirectX 12 are deferred; that is, the commands in a command list are only run on the GPU after they have been executed on a command queue.

A command list is used for recording commands that are executed on the GPU. Unlike previous versions of DirectX, execution of commands recorded into a command list are always deferred. That is, invoking draw or dispatch commands on a command list are not executed until the command list is sent to the command queue. Unlike the command allocator, the command list can be reused immediately after it has been executed on the command queue. The only restriction is that the command list must be reset first before recording any new commands.

You can use the ID3D12Device::GetNodeCount method to query the number of GPU adapter nodes on the system.

**Command Allocator**
A command allocator must not be reset until we are sure the GPU has finished executing all the commands in the allocator.

A command allocator is the backing memory used by a command list. A command allocator is created using the ID3D12Device::CreateCommandAllocator method and must specify the type of command list the allocator will be used with. The command allocator does not provide any functionality and can only be accessed indirectly through a command list. A command allocator can only be used by a single command list at a time but can be reused after the commands that were recorded into the command list have finished executing on the GPU.

**Command Queue**
The Command Queue in DirectX 12 has a very simple interface. For most common cases only the ID3D12CommandQueue::ExecuteCommandLists method and the ID3D12CommandQueue::Signal method are used.

It is important to understand that each command queue must track it’s own fence and fence value. DirectX 12 defines three different command queue types:
* Copy: Can be used to issue commands to copy resource data (CPU -> GPU, GPU -> GPU, GPU -> CPU).
* Compute: Can do everything a Copy queue can do and issue compute (dispatch) commands.
* Direct: Can do everything a Copy and a Compute queue can do and issue draw commands.

**Fence**
The Fence object is used to synchronize commands issued to the Command Queue. The fence stores a single value that indicates the last value that was used to signal the fence. Although it is possible to use the same fence object with multiple command queues, it is not reliable to ensure the proper synchronization of commands across command queues. Therefore, it is advised to create at least one fence object for each command queue. Multiple command queues can wait on a fence to reach a specific value, but the fence should only be allowed to be signaled from a single command queue. In addition to the fence object, the application must also track a fence value that is used to signal the fence. An example of performing CPU-GPU synchronization using fences will be shown in the following sections.

**GPU-Synchronization**
Whenever the CPU is faster at issuing commands than the command queue is at processing those commands, the CPU will have to stall at some point in order to allow the command queue to catch-up to the CPU.

It gets more complicated if you add an additional queue. In this case, you must be careful not to signal the second queue with a fence value that is larger than, but could be completed before, a fence value that was used on another queue using the same fence object. Doing so could result in the fence reaching the fence value from the other queue before the main queue has reached the earlier fence value.

![image](../images/GPU-Synchronization.png)

The moral of the story is to make sure that every command queue tracks its own fence object and fence value and only signals its own fence object. To be safe, the fence value for a queue should never be allowed to decrease.

**Descriptor Heap**

A descriptor heap can be considered an array of resource views. As of DirectX 12, before resource views can be created (such as Render Target Views (RTV), Shader Resource Views (SRV), Unordered Access Views (UAV), or Constant Buffer Views (CBV)), a descriptor heap needs to be created.

A render target view (RTV) describes a resource that can be attached to a bind slot of the output merger stage. The render target view describes the resource that receives the final color computed by the pixel shader stage.

**Resource Barrier**
In DirectX 12, it is the responsibility of the graphics programmer to ensure that resources are in the correct state before using them. Resources must be transitioned from one state to another using a resource barrier and inserting that resource barrier into the command list. 

There are several types of resource barriers:

* Transition: Transitions a (sub)resource to a particular state before using it. For example, before a texture can be used in a pixel shader, it must be transitioned to the PIXEL_SHADER_RESOURCE state.
* Aliasing: Specifies that a resource is used in a placed or reserved heap when that resource is aliased with another resource in the same heap.
* UAV: Indicates that all UAV accesses to a particular resource have completed before any future UAV access can begin. This is necessary when the UAV is transitioned for:
    * Read > Write: Guarantees that all previous read operations on the UAV have completed before being written to in another shader.
    * Write > Read: Guarantees that all previous write operations on the UAV have completed before being read from in another shader.
    * Write > Write: Avoids race conditions that could be caused by different shaders in a different draw or dispatch trying to write to the same resource (does not avoid race conditions that could be caused in the same draw or dispatch call).
    * A UAV barrier is not needed if the resource is being used as a read-only (Read > Read) resource between draw or dispatches.

---

### Heap and Resources

In DirectX 12 the only interface used to describe a resource is the [ID3D12Resource](https://msdn.microsoft.com/en-us/library/windows/desktop/dn788709(v=vs.85).aspx) interface.

DirectX 12 also provides more control over the way the resource is stored in GPU memory. The [ID3D12Heap](https://msdn.microsoft.com/en-us/library/windows/desktop/dn788687(v=vs.85).aspx) interface allows for various memory mapping techniques to be implemented which may be used to optimize GPU memory utilization.

In the first lesson a swap chain is created which internally contains one or more texture resources that are used to present the final rendered image to the screen. The creation of the texture resource for the swap chain is hidden in the [IDXGIFactory2::CreateSwapChainForHwnd](https://msdn.microsoft.com/en-us/library/hh404557(v=vs.85).aspx) method. In this case, the graphics programmer has no control over how and where the resource is created. In this lesson, several buffer resources are required to render the scene. For these resources, the graphics programmer must decide how those buffer resources are created. There are several different ways that GPU resources are allocated within a memory heap ([ID3D12Heap](https://msdn.microsoft.com/en-us/library/dn788687(v=vs.85).aspx)):

* Committed Resources

A committed resource is created using the [ID3D12Device::CreateCommittedResource](https://msdn.microsoft.com/en-us/library/dn899178(v=vs.85).aspx) method. This method creates both the resource and an implicit heap that is large enough to hold the resource. The resource is also mapped to the heap. Committed resources are easy to manage because the graphics programmer doesn’t need to be concerned with placing the resource within the heap.

![image](../images/Committed-Resources.png)

Committed resources are ideal for allocating large resources like textures or statically sized resources (the size of the resource does not change). Committed resource are also commonly used to create large resource in an upload heap that can be used for uploading dynamic vertex or index buffers (useful for UI rendering or uploading constant buffer data that is changing for each draw call).

* Placed Resources

A placed resource is explicitly placed in a heap at a specific offset within the heap. Before a placed resource can be created, first a heap is created using the [ID3D12Device::CreateHeap](https://msdn.microsoft.com/en-us/library/dn788664(v=vs.85).aspx) method. The placed resource is then created inside the heap using the [ID3D12Device::CreatePlacedResource](https://msdn.microsoft.com/en-us/library/dn899180(v=vs.85).aspx) method.

![image](../images/Placed-Resource.png)

Multiple placed resources can be aliased in a heap as long as they don’t access the same aliased heap space at the same time.

![image](../images/Aliasing-Placed-Resources.png)

Aliasing can help to reduce oversubscribing GPU memory usage since the size of the heap can be limited to the size of the largest resource that will be placed in the heap at any moment in time. Aliasing can be used as long as the same space in the heap is not used by multiple aliasing resources at the same time. Aliased resources can be swapped using a [resource aliasing barrier](https://msdn.microsoft.com/en-us/library/dn986739(v=vs.85).aspx).

* Reserved Resources

Reserved resources are created without specifying a heap to place the resource in. Reserved resources are created using the [ID3D12Device::CreateReservedResource](https://msdn.microsoft.com/en-us/library/dn899181(v=vs.85).aspx) method. Before a reserved resource can be used, it must be mapped to a heap using the [ID3D12CommandQueue::UpdateTileMappings](https://msdn.microsoft.com/en-us/library/dn788641(v=vs.85).aspx) method.

![image](../images/Reserved-Resources.png)

Reserved resources can be created that are larger than can fit in a single heap. Portions of the reserved resource can be mapped (and unmapped) using one or more heaps residing in physical GPU memory.

Using reserved resources, a large volume texture can be created using virtual memory but only the resident spaces of the volume texture needs to be mapped to physical memory. This resource type provides options for implementing rendering techniques that use sparse voxel octrees without exceeding GPU memory budgets.

### Pipeline State Object

In order to provide better state management in DirectX 12, the pipeline state object (PSO) is used to describe the various stages of the rendering (and compute) pipeline. The PSO combines parts of the input assembly (IA), vertex shader (VS), hull shader (HS), domain shader (DS), geometry shader (GS), stream output (SO), rasterizer stage (RS), pixel shader (PS), and output merger (OM) stages of the rendering pipeline. A few properties of the rendering pipeline (such as viewport and scissor rectangle) require additional API calls but most of the configuration of the rendering pipeline is controlled by the PSO.

The Pipeline State Object (PSO) contains most of the state that is required to configure the rendering (or compute) pipeline. The graphics pipeline state object includes the following information:

* Shader bytecode (vertex, pixel, domain, hull, and geometry shaders)
* Vertex format input layout
* Primitive topology type (point, line, triangle, or patch)
* Blend state
* Rasterizer state
* Depth-stencil state
* Number of render targets and render target formats
* Depth-stencil format
* Multisample description
* Stream output buffer description
* Root signature

The pipeline state object structure contains a lot of information and if any of the state needs to change between draw calls (for example, a different pixel shader or blend state is needed) then a new pipeline state object needs to be created.

Although the pipeline state object contains a lot of information, there are a few additional parameters that must be set outside of the pipeline state object.

* Vertex and Index buffers
* Stream output buffer
* Render targets
* Descriptor heaps
* Shader parameters (constant buffers, read-write buffers, and read-write textures)
* Viewports
* Scissor rectangles
* Constant blend factor
* Stencil reference value
* Primitive topology and adjacency information

The pipeline state object can optionally be specified for a graphics command list when the command list is reset using the [ID3D12GraphicsCommandList::Reset](https://msdn.microsoft.com/en-us/library/dn903895(v=vs.85).aspx) method but it can also be changed for the command list at any time using the [ID3D12GraphicsCommandList::SetPipelineState](https://msdn.microsoft.com/en-us/library/dn903918(v=vs.85).aspx) method.

### Root Signatures

Another new concept in DirectX 12 is the root signature. A root signature describes the parameters that are passed to the various programmable shader stages of the rendering pipeline. Examples of parameters that are passed to the programmable shader stages are constant buffers, (read-write) textures, and (read-write) buffers. Texture samplers (an object that determines how the texels in a texture are sampled and filtered) are also defined in the root signature.

A root signature is similar to a C++ function signature, that is, it defines the parameters that are passed to the shader pipeline. The values that are bound to the pipeline are called the root arguments. The arguments that are passed to the shader can change without changing the root signature parameters.

The root parameters in the root signature not only define the type of the parameters that are expected in the shader, they also define the shader registers and register spaces to bind the arguments to in the shader.

### Shader Register & Register Spaces

Shader parameters must be bound to a register. For example, constant buffers are bound to b registers (b0 – bN), shader resource views (textures and non-constant buffer types) are bound to t registers (t0 – tN), unordered access views (writeable textures and buffer types) are bound to u registers (u0 – uN), and texture samplers are bound to s registers (s0 – sN) where N is the maximum number of shader registers.

In previous versions of DirectX, different resources could be bound to the same register slot if they were used in different shader stages of the rendering pipeline. For example, a constant buffer could be bound to register b0 in the vertex shader and a different constant buffer could be bound to register b0 in the pixel shader without causing overlap.

DirectX 12 root signatures require that all shader parameters be defined for all stages of the rendering pipeline and registers used across different shader stages may not overlap. In order to provide a work-around for legacy shaders that may violate this rule, Shader Model 5.1 introduces register spaces. Resources can overlap register slots as long as they don’t also overlap register spaces.

![image](../images/Registers-and-Spaces.png)

### Root Signature Parameters

A root signature can contain any number of parameters. Each parameter can be one of the following types:

[D3D12_ROOT_PARAMETER_TYPE](https://docs.microsoft.com/en-us/windows/desktop/api/d3d12/ne-d3d12-d3d12_root_parameter_type)

* D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS    :    32-bit Root Constants
* D3D12_ROOT_PARAMETER_TYPE_CBV                :    Inline CBV Descriptor
* D3D12_ROOT_PARAMETER_TYPE_SRV                :    Inline SRV Descriptor
* D3D12_ROOT_PARAMETER_TYPE_UAV                :    Inline UAV Descriptor
* D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE   :    Descriptor Table

**32-BIT CONSTANTS**

Constant buffer data can be passed to the shader without the need to create a constant buffer resource by using the 32-bit constants. Dynamic indexing into the constant buffer data is not supported for constant data that is stored in the root signature space. For example, the following constant buffer definition can be mapped to 32-bit constants stored in the root signature:
``` hlsl
cbuffer TransformsCB : register(b0, space0)
{
    matrix Model;
    matrix View;
    matrix Projection;
}
```
But, due to the array, the following constant buffer definition cannot be mapped to 32-bit constants in the root signature and instead requires either an inline descriptor or a descriptor heap to map the resource:
``` hlsl
cbuffer TransformsCB : register(b0, space0)
{
    matrix MVPMatrices[3];
}
```
Each root constant in the root signature costs 1 DWORD (32-bits)

**INLINE DESCRIPTORS**

Descriptors can be placed directly in the root signature without requiring a descriptor heap. Only constant buffers (CBV), and buffer resources (SRV, UAV) resources containing 32-bit (FLOAT, UINT, or SINT) components can be accessed using inline descriptors in the root signature. Inline UAV descriptors for buffer resources cannot contain counters. Texture resources cannot be referenced using inline descriptors in the root signature and must be placed in a descriptor heap and referenced through a descriptor table.

Unlike root constants, constant buffers containing arrays may be accessed using inline descriptors in the root signature. For example, the following constant buffer can be referenced using inline descriptors:
``` hlsl
cbuffer SceneData : register(b0, space0)
{
   uint foo;
   float bar[2];
   int moo;
};
```
Each inline descriptor in the root signature costs 2 DWORDs (64-bits)

**DESCRIPTOR TABLES**

A descriptor table defines several descriptor ranges that are placed consecutively in a GPU visible descriptor heap.

![image](../images/Descriptor-Tables.png)
The above image illustrates a root signature with a single descriptor table parameter (A). The descriptor table contains three descriptor ranges (B): 3 Constant Buffer Views (CBV), 4 Shader Resource Views (SRV), and 2 Unordered Access Views (UAV). CBV’s, SRV’s and UAV’s can be referenced in the same descriptor table since all three types of descriptors can be stored in the same descriptor heap. The GPU visible descriptors (C) must appear consecutively in a GPU visible descriptor heap but the resources that they refer to (D) may appear anywhere in GPU memory, even in different resource heaps.

Each descriptor table in the root signature costs 1 DWORD (32-bits)

### Static Samplers

A texture sampler describes how a texture should be sampled. It describes the filtering, addressing, and Level of Detail (LOD) to use while sampling from a texture. Texture samplers can be defined directly in the root signature without the need of a descriptor heap. The [D3D12_STATIC_SAMPLER_DESC](https://msdn.microsoft.com/en-us/library/dn986748(v=vs.85).aspx) structure is used to define a texture sampler directly in the root signature.

Static samplers do not use any space in the root signature and do not count against the size limit of the root signature.

### Root Signature Constraints

Root signatures are limited to 64 DWORDs (2048-bits) [5]. Each root parameter has a cost that counts towards the root signature limit:
* 32-bit constants each costs 1 DWORD
* Inline descriptors each costs 2 DWORDs
* Descriptor tables each costs 1 DWORD
Static samplers defined in the root signature do not count towards the size of the root signature.

The cost of accessing a root argument in a root signature in terms of levels of indirection is zero for 32-bit constants, 1 for inline descriptors, and 2 for descriptor tables

The graphics programmer should strive to achieve a root signature that is as small as possible but balance the flexibility of using a larger root signature. Parameters in the root signature should be ordered based on the likelihood that the arguments will change. If the root arguments are changing often then they should appear first in the root signature. If the root arguments are not changing often, then they should be the last parameters that appear in the root signature. Since 32-bit constants and inline descriptors have better performance in terms of level of indirection, they should be favored over using descriptor tables as long as the size of the root signature does not become to large.

### Shaders

The Vertex Shader is responsible for transforming the vertices of the object being rendered from object-space to clip-space. The clip-space vertices are required by the (fixed-function) Rasterizer Stage of the rendering pipeline in order to clip the rendering primitives against the view frustum and to compute the per-pixel (or per-fragment) attributes across the face of the rendered primitives. The interpolated vertex attributes are passed to the Pixel Shader in order to compute the final color of the pixel.

The Pixel Shader is responsible for computing the final color of the pixel that is rendered to the screen (or an offscreen render target). It receives the interpolated vertex attributes that are computed by the Rasterizer Stage and usually outputs at least one color value that is written to a render target

#### Vertex Shader
The basic concept of a vertex shader is that it receives the vertices that describe a model (expressed in object-space, or model-space) and performs zero or more transformations on the attributes of the vertices in order to produce the vertex attributes for the next stage of the rendering pipeline (this is usually the rasterizer stage but it could also be the geometry or tessellation stages).

A vertex can contain any number of attributes. For example, the vertex position is a possible attribute of a vertex. Vertex color, normal, or texture coordinate are other examples of vertex attributes.

Vertex attributes can be sent to the GPU in either packed or interleaved format. Attributes that are stored in a packed format are usually stored in separate buffers (one buffer for position, another for color, etc..). Packed attributes are similar in concept to a [Struct of Arrays (SoA)](https://en.wikipedia.org/wiki/AOS_and_SOA).

Attributes that are stored in an interleaved format are usually stored in a single array. Interleaved attributes are similar in concept an [Array of Structs (AoS)](https://en.wikipedia.org/wiki/AOS_and_SOA).

![image](../images/Vertex-Arrays.png)

**SHADER SEMANTICS**

The vertex shader can output one or more attributes to the next stage of the rendering pipeline but it must at least output a position attribute bound to the [SV_Position](https://msdn.microsoft.com/en-us/library/bb509647(v=vs.85).aspx#System_Value) system value semantic. Semantics are a way to tell the Input Assembler how to link the buffer data supplied by the application to the input parameters expected by the shader. Semantics are also the language syntax used to link output parameters from one shader stage to the input parameters to another shader stage.

In an HLSL shader the semantic name for the variable follows the colon (:) character in a variable declaration.
``` hlsl
struct VertexPosColor
{
    float3 Position : POSITION;
    float3 Color    : COLOR;
};
```
The Position attribute is bound to the POSITION semantic and the Color attribute is bound to the COLOR semantic. These semantics will be used later to specify how the application data is bound to the vertex attributes in HLSL.

**TRANSFORMATION AND SPACES**

If a refresher on transformation and spaces is needed, please refer to the articles titled [Coordinate Systems](https://www.3dgep.com/3d-math-primer-for-game-programmers/), [Matrices](https://www.3dgep.com/3d-math-primer-for-game-programmers-matrices/), and [Understanding the View Matrix](https://www.3dgep.com/understanding-the-view-matrix/).

The transformation matrix that rotates the object is called the model matrix and the transformation matrix to place the camera in the scene is called the view matrix. A final transformation called the projection matrix is also applied in order to transform the vertices from view space to clip space. These three matrices can be combined using matrix multiplication to produce a single matrix that can transform the vertices from objects space to clip space. This matrix is appropriately named the model-view-projection matrix or just MVP matrix.

The MVP matrix is required by the vertex shader to perform the transformations on the vertices of the model. This matrix is passed to the vertex shader using a ConstantBuffer object in HLSL.
``` hlsl
struct ModelViewProjection
{
    matrix MVP;
};

ConstantBuffer<ModelViewProjection> ModelViewProjectionCB : register(b0);
```
Shader Model 5.1 introduced the ConstantBuffer template construct in order to enable support for descriptor arrays. See [Resource Binding in HLSL](https://docs.microsoft.com/en-us/windows/desktop/direct3d12/resource-binding-in-hlsl#constant-buffers) for more information.

**VERTEX SHADER OUTPUT**
The vertex shader accepts vertex attributes passed from the application, transforms these inputs in zero or more ways (a vertex shader that performs no transformations on the input data is called a pass-through shader) and outputs the vertex attributes to be consumed by the next stage of the rendering pipeline. Just as the input attributes were bound to semantics, the output attributes are also bound to semantics.
``` hlsl
struct VertexShaderOutput
{
    float4 Color    : COLOR;
    float4 Position : SV_Position;
};
```
The SV_Position system value semantic is a required semantic to be output by the vertex shader and it is the only required parameter that must be passed to the rasterizer stage. The parameter bound to the SV_Position system value semantic is not a required input parameter for the pixel shader and can be omitted from the pixel shader’s input parameters (as will be shown later). However, the byte offsets of the other parameters (in this case the Color parameter has an offset of 0 bytes within the VertexShaderOutput structure) must match between the outputs and inputs of linked shader stages. For this reason, the Position parameter is placed last in the VertexShaderOutput structure.

**THE MAIN ENTRY POINT**
The main entry point for the shader is a single function that takes the vertex attributes as input arguments and outputs the transformed vertex attributes.
``` hlsl
VertexShaderOutput main(VertexPosColor IN)
{
    VertexShaderOutput OUT;

    OUT.Position = mul(ModelViewProjectionCB.MVP, float4(IN.Position, 1.0f));
    OUT.Color = float4(IN.Color, 1.0f);

    return OUT;
}
```
Unlike in GLSL, the name of the entry point function in HLSL is not required to be “main“. In fact, it is possible to define several entry-point functions in a single HLSL source file. The name of the entry point for the shader is specified when compiling the shader.

Each invocation of the vertex shader operates over a single vertex (as opposed to triangles or the entire mesh) and outputs the transformed vertex. Many vertices are processed in parallel and it is not possible to modify variables defined within the scope of the vertex shader and expect that other invocations of the vertex shader will see those changes (even if you declare the variable as static in the global scope of the vertex shader!). For example, you cannot declare a counter variable in the vertex shader and allow each invocation to increment that counter to see how many vertices were processed (you may be able to achieve this using atomic counters, but that is beyond the scope of this lesson).

If a 0.0 was used in the last component of the vector, then the vector would be rotated but not translated.
If the vertex attribute is a position vector, then the 4th component of the vector must be a 1.
If the vertex attribute is a normal vector, then the 4th component of the vector must be a 0.
Positional vectors are often referred to as points (because it represents a point in space). Directional vectors are simply referred to as vectors.

the OUT parameter is returned and the result is passed as input to the Rasterizer stage. The rasterizer will use the Position parameter to determine the pixel’s coordinates (in screen space, by applying the viewport), the depth value (in normalized-device coordinate space) which is written to the depth buffer, and the interpolated Color parameter is passed as input to the pixel shader stage.

#### Pixel Shader

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

### Group Shared Memory

Group shared memory is memory that is shared among all threads in a thread group. According to Microsoft’s documentation, group shared memory is limited to 16KB for D3D10 level hardware and 32KB for D3D11 level hardware. There is no specification for the amount of shared memory that is available for D3D12. The amount of shared memory actually available to the thread group is dependent on the GPU architecture.

In order to maintain maximum thread occupancy, a compute shader should not declare more than 16KB of group shared memory. In order to mitigate latency incurred by fetches to global memory, a streaming multiprocessor will queue multiple thread groups. If a thread group declares more than the maximum amount of shared memory that can be satisfied by all of the queued thread groups, the thread scheduler will reduce the number of thread groups until the utilized shared memory can be satisfied.

Group shared memory is split into (16 or 32 depending on GPU architecture) equally sized memory banks whose addresses are 32-bit interleaved. The interleaved addresses allow each thread in a wave to access a different memory banks without causing bank conflicts. Bank conflicts occur when different threads in wave access the same memory bank but not the same address. Bank conflicts can occur more frequently when the access stride between threads in a wave are not increments of 32-bits. When a bank conflict does occur, the memory controller is required to perform multiple reads from shared memory to satisfy all read requests. When no bank conflict occurs, then the memory controller can satisfy all reads from group shared memory in the same amount of time as a single read (reading data from shared memory can be coalesced).

It is recommended to ensure that threads in a wave access shared memory addresses that are at 32-bit offsets relative to the index of the thread within the thread group (SV_GroupIndex). It is also recommended to split memory declared in group shared memory by 32-bit values. For example, it is more efficient to declare four arrays of floats than a single array of 4-component floats.

### Generate Mipmaps Compute Shader

Generating mipmaps is the process of computing an image that is half the size of the original image. The process is repeated until a 1×1 image is generated. For the remainder of the discussion, the term source mip is used to refer to the mip level that is used as the source of the next mip in the chain and the term destination mip is used to refer to the mip that is being generated.

Undersampling occurs when texels from the source mip are not used during downsampling. In order to avoid undersampling from textures with an odd dimension, multiple samples need to be read from the source mip.

Since sRGB textures are not supported for UAV writes, a UAV compatible copy of the original texture is used to perform the mipmapping. Gamma correction must be applied when the original texture has an sRGB format.

### Non Power of Two

For example, a 5×5 texture reduces to a size of 2.5×2.5. Of course, it is not possible to have a fraction of a pixel in a texture so the 0.5 must be truncated from the texture’s dimension. If only a single sample is taken from the source mip to generate the destination mip, then the resulting mip will be undersampled resulting in a slightly incorrect color in the final mip. If several mip levels are undersampled, noticeable artifacts can appear in the rendered image. In order to avoid undersampling the texture, several samples are taken from the source mip and blended together to produce the destination mip color.

![image](../images/Undersampled-Mipmaps.png)

### sRGB

sRGB (standard red, green, and blue) is an RGB color space developed by HP and Microsoft in 1996 for use on monitors, printers, and the internet[6]. sRGB colors are encoded using a transfer function (gamma curve) that was typical of the gamma response of CRT monitors at the time. The sRGB transfer function is still used in current (low dynamic range) LCD screens which is why image compression formats (such as JPEG, and PNG) will typically still encode images in an sRGB format.

Since sRGB image formats are not compatible with UAV loads and stores[8], a resource with a UAV compatible format is created in order to perform the mipmapping. If the original texture contains a sRGB image format, then the texels sampled from the texture must be linearized (using the sRGB to Linear equation) before blending. When storing the blended texel to the destination mip, the blended texel must be converted back to sRGB (using the Linear to sRGB equation).





**Depth Stencil Buffer**

It is valid to define a graphics pipeline state without a pixel shader if you only need depth information.

The depth buffer stores the depth of the pixel in normalized device coordinate space (NDC). Only pixels that are not occluded based on the current value in the depth buffer are drawn. This ensures that primitives that are further away from the viewer are not drawn on top of primitives that are closer to the viewer.

The depth buffer is not a required resource in order to display the final rendered image on the screen.

Similar to the render target views for the swap chain, the depth buffer requires a depth-stencil view. The depth-stencil view is created in a descriptor heap.





When using runtime compiled HLSL shaders using any of the D3DCompiler functions, do not forget to link against the D3Dcompiler_47.lib library and copy the D3dcompiler_47.dll to the same folder as the binary executable when distributing your project.
A redistributable version of the D3dcompiler_47.dll file can be found in the Windows 10 SDK installation folder at C:\Program Files (x86)\Windows Kits\10\Redist\D3D\.

For more information, refer to the MSDN blog post at: https://blogs.msdn.microsoft.com/chuckw/2012/05/07/hlsl-fxc-and-d3dcompile/






The [D3D_PRIMITIVE_TOPOLOGY](https://learn.microsoft.com/en-us/windows/win32/api/d3dcommon/ne-d3dcommon-d3d_primitive_topology) enumeration specifies how the Input Assembler stage interprets the vertex data that is bound to the rendering pipeline.

![image](../images/Primitive-Topologies.gif)





A front-facing polygon is said to be facing the viewer while a back-facing polygon is said to be facing away from the viewer. Since back-facing polygons are generally not visible to the viewer (assuming the mesh is fully convex and the material is not transparent) then it is not efficient to shade the pixels that will not be visible in the final render. As an optimization, it is possible to tell the Rasterizer stage to cull (remove from the rendering pipeline) back-facing polygons.

![image](../images/Cube-Vertices-and-Faces-1.png)




Initializing the scissor rectangle to {0, 0, LONG_MAX, LONG_MAX} ensures that the scissor rectangle covers the entire screen regardless of the size of the screen. If the screen is resized, the scissor rectangle does not need to be updated.




The viewport can be smaller than the size of the screen (this is common when implementing split-screen rendering) but it should not be larger than the rendertarget that is bound to the output merger stage. The viewport is specified on the rasterizer stage and it is used to tell the rasterizer how to transform the vertices from normalized device coordinate space (NDC) into screen space which is required for the pixel shader. More specifically, the viewport should not be larger than the currently bound render target. When using offscreen render targets, the currently bound render target is not necessarily the same size as the screen.





Vertex Buffer View
A [D3D12_VERTEX_BUFFER_VIEW](https://msdn.microsoft.com/en-us/library/dn903819(v=vs.85).aspx) structure is used to tell the Input Assembler stage where the vertices are stored in GPU memory.

Index Buffer View
Similar to the Vertex Buffer View, the [D3D12_INDEX_BUFFER_VIEW](https://msdn.microsoft.com/en-us/library/dn891445(v=vs.85).aspx) is used to describe the index buffer to the Input Assembler stage of the rendering pipeline.




In the previous lesson, a swap-chain is created to be the target for the final rendered image that is presented to the user’s screen. The swap-chain can contain two or more texture buffers but it does not create a depth buffer. A depth buffer is not strictly required for rendering. When all of the elements in a scene are 2D (common for side-scrolling platform games), then the scene elements can be rendered from back-to-front and all elements will appear correctly. When rendering a 3D scene (even if the scene only contains a single geometric object) then a depth buffer is required for achieving correct rendering. The depth buffer stores the depth of a pixel in normalized device coordinate space (NDC). Each pixel stores a depth value in the range {0⋯1} where 0 is closest to the viewer (near clipping plane) and 1 is farthest away (far clipping plane).

![image](../images/Depth-Buffer-View-Space-Normalized-Device-Coordinate-Space.png)





If the Visual Studio project is configured correctly, then any HLSL shader file (any file with the .hlsl extension) that is added to the project will automatically be compiled into a shader object file that can be loaded by the application at runtime. By default, shaders are compiled into a Compiled Shader Object file (.cso) with the same name as the original .hlsl file. For example, VertexShader.hlsl will be compiled as VertexShader.cso. The CSO file can be loaded directly into a binary object using the D3DReadFileToBlob function.

By default, the shader compiler (FXC.exe) will generate the .cso file in the same directory as the output directory for the project. If the current working directory for the currently running process is configured to be the same folder where the executable is located, then it should be sufficient to load the shader from the .cso file using the path relative to the executable.

There are methods to compile the HLSL shader files at runtime. For example, you can use the [D3DCompileFromFile](https://msdn.microsoft.com/en-us/library/hh446872(v=vs.85).aspx) function to compile your HLSL shaders at runtime instead of using the FXC compiler integrated in Visual Studio.





INPUT LAYOUT
The Input Layout describes the layout of the vertex buffers that are bound to to the Input Assembler stage of the rendering pipeline.

There is no way to specify whether the vertex attributes are passed as packed or interleaved formats in HLSL. The only way to specify the buffer format is by using the Input Layout.

The Input Layout is specified using an array of [D3D12_INPUT_ELEMENT_DESC](https://msdn.microsoft.com/en-us/library/dn770377(v=vs.85).aspx) structures.

It is possible to avoid the use of Input Layouts by storing the vertex buffer data in one or more [StructuredBuffers](https://msdn.microsoft.com/en-us/library/ff471514(v=vs.85).aspx) in HLSL.




Denying access to shader stages that do not require root signature access is a minor optimization on some hardware.




Similar to pre-compiling HLSL shaders, it is also possible to pre-compile root signatures. Pre-compiling root signatures has the benefit of shorter load times when there are many shader combinations each requiring a unique root signature. For this demo, the root signature is serialized at runtime but for production code, it is advisable to serialize the root signatures during compilation and save them to disc and load the pre-serialized root signature at runtime. It is also possible to define the root signatures directly in HLSL code and read the serialized root signature directly from the compiled shader object. Specifying root signatures in HLSL is beyond the scope of this article, but anyone that is interested is encouraged to read [Specifying Root Signatures in HLSL](https://msdn.microsoft.com/en-us/library/dn913202(v=vs.85).aspx).




The PSO is described using a Pipeline State Stream structure. The Pipeline State Stream structure contains a set of stream tokens that describe how to configure the PSO. An example of a stream token for a Pipeline State Stream object is the Vertex Shader (VS), Geometry Shader (GS) or Pixel Shader (PS). You only need to specify stream tokens that will override the default values of that token. For example, if tessellation shaders (Hull and Domain Shaders) are not required for the PSO, then they do not need to be defined in the Pipeline State Stream structure.

The order that the stream tokens appear in the Pipeline State Stream structure are not important. When the Pipeline State Stream is parsed, the type of each token is used to determine how the PSO is created.



Try to reason about why the swap chain requires multiple back buffers (at least 2) but only a single depth-stencil buffer is needed for correct rendering. Does the depth-stencil buffer ever need to be in both a read and write state at the same time?






# New

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

Unlike vertex and index buffers, constant buffers are usually updated once per
frame by the CPU. For example, if the camera is moving every frame, the constant
buffer would need to be updated with the new view matrix every frame. Therefore,
we create constant buffers in an upload heap rather than a default heap so that we
can update the contents from the CPU.

Constant buffers also have the special hardware requirement that their size
must be a multiple of the minimum hardware allocation size (256 bytes).

Even though we allocate constant data in multiples of 256, it is not necessary to
explicitly pad the corresponding constant data in the HLSL structure because it
is done implicitly

The root signature defines what resources the application will bind to the rendering
pipeline before a draw call can be executed and where those resources get mapped
to shader input registers.

If we think of the shader programs as a function, and the input resources the
shaders expect as function parameters, then the root signature can be thought
of as defi ning a function signature (hence the name root signature). By binding
different resources as arguments, the shader output will be different. So, for
example, a vertex shader will depend on the actual vertex being input to the
shader, and also the bound resources.



MaxAnisotropy
Clamping value used if D3D12_FILTER_ANISOTROPIC or D3D12_FILTER_COMPARISON_ANISOTROPIC is specified as the filter. Valid values are between 1 and 16.

D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
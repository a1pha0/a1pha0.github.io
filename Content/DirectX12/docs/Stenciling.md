## Stenciling

The stencil buffer is an off-screen buffer we can use to achieve some special effects. The stencil buffer has the same resolution as the back buffer and depth buffer. When a stencil buffer is specified, it comes attached to the depth buffer. As the name suggests, the stencil buffer works as a stencil and allows us to block the rendering of certain pixel fragments to the back buffer.

**Depth/Stencil Formats**

D->Depth S->Stencil
1. DXGI_FORMAT_D32_FLOAT_S8X24_UINT
   Specifies a 32-bit floating-point depth buffer, with 8-bits (unsigned integer) reserved for the stencil buffer mapped to the [0, 255] range and 24-bits not used for padding.
2. DXGI_FORMAT_D24_UNORM_S8_UINT
   Specifies an unsigned 24-bit depth buffer mapped to the [0, 1] range with 8-bits (unsigned integer) reserved for the stencil buffer mapped to the [0, 255] range.

**Clearing**

[ID3D12GraphicsCommandList::ClearDepthStencilView](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-cleardepthstencilview)

[D3D12_CLEAR_FLAGS](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_clear_flags)

**Stencil Test**

```
if ((StencilRef & StencilReadMask) ? (Value & StencilReadMask))
	accept pixel
else
	reject pixel
```

The stencil reference value is set with the [ID3D12GraphicsCommandList::OMSetStencilRef](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-omsetstencilref) method.

The ? operator is any one of the functions defined in the [D3D12_COMPARISON_FUNC](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_comparison_func) enumerated type

**Depth/Stencil State**

[D3D12_DEPTH_STENCIL_DESC](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_depth_stencil_desc)

The stencil buffer (and also the depth buffer) state is configured by filling out a D3D12_DEPTH_STENCIL_DESC instance and assigning it to the D3D12_GRAPHICS_PIPELINE_STATE_DESC::DepthStencilState field of a pipeline state object (PSO).

Observe that the stenciling behavior for front facing and back facing triangles can be different. The BackFace settings are irrelevant in the case that we do not render back facing polygons due to back face culling.

### IMPLEMENTING PLANAR MIRRORS

1. Render the floor, walls, and skull to the back buffer as normal (but not the
mirror). Note that this step does not modify the stencil buffer.
2. Clear the stencil buffer to 0. Figure 11.3 shows the back buffer and stencil
buffer at this point (where we substitute a box for the skull to make the
drawing simpler).
3. Render the mirror only to the stencil buffer. We can disable color writes to the
back buffer by creating a blend state that sets
D3D12_RENDER_TARGET_BLEND_DESC::RenderTargetWriteMask = 0;
and we can disable writes to the depth buffer by setting
D3D12_DEPTH_STENCIL_DESC::DepthWriteMask = D3D12_DEPTH_WRITE_MASK_
ZERO;
When rendering the mirror to the stencil buffer, we set the stencil test to
always succeed (D3D12_COMPARISON_ALWAYS) and specify that the stencil buffer
entry should be replaced (D3D12_STENCIL_OP_REPLACE) with 1 (StencilRef) if
the test passes. If the depth test fails, we specify D3D12_STENCIL_OP_KEEP so that
the stencil buffer is not changed if the depth test fails (this can happen, for
example, if the skull obscures part of the mirror). Since we are only rendering
the mirror to the stencil buffer, it follows that all the pixels in the stencil
buffer will be 0 except for the pixels that correspond to the visible part of
the mirror—they will have a 1. Figure 11.4 shows the updated stencil buffer.
Essentially, we are marking the visible pixels of the mirror in the stencil buffer.
4. Now we render the reflected skull to the back buffer and stencil buffer. But
recall that we only will render to the back buffer if the stencil test passes. This
time, we set the stencil test to only succeed if the value in the stencil buffer
equals 1; this is done using a StencilRef of 1, and the stencil operator D3D12_
COMPARISON_EQUAL. In this way, the reflected skull will only be rendered to areas
that have a 1 in their corresponding stencil buffer entry. Since the areas in the
stencil buffer that correspond to the visible parts of the mirror are the only
entries that have a 1, it follows that the reflected skull will only be rendered
into the visible parts of the mirror.
5. Finally, we render the mirror to the back buffer as normal. However, in order
for the skull reflection to show through (which lies behind the mirror), we
need to render the mirror with transparency blending. If we did not render
the mirror with transparency, the mirror would simply occlude the reflection
since its depth is less than that of the reflection. To implement this, we simply
need to define a new material instance for the mirror; we set the alpha channel
of the diffuse component to 0.3 to make the mirror 30% opaque, and we
render the mirror with the transparency blend state as described in the last
chapter (§10.5.4).
6. When a triangle is reflected across a plane, its winding order does not reverse, and
thus, its face normal does not reverse. Hence, outward facing normals become
inward facing normals (see Figure 11.5), after reflection. To correct this, we tell
Direct3D to interpret triangles with a counterclockwise winding order as frontfacing
and triangles with a clockwise winding order as back-facing (this is the
opposite of our usual convention—§5.10.2). This effectively reflects the normal
directions so that they are outward facing after reflection. We reverse the winding
order convention by setting the following rasterizer properties in the PSO:
drawReflectionsPsoDesc.RasterizerState.FrontCounterClockwise = true;

### IMPLEMENTING PLANAR SHADOWS

[XMMatrixShadow](https://learn.microsoft.com/en-us/windows/win32/api/directxmath/nf-directxmath-xmmatrixshadow)

When we flatten out the geometry of an object onto the plane to describe its
shadow, it is possible (and in fact likely) that two or more of the flattened triangles
will overlap. When we render the shadow with transparency (using blending),
these areas that have overlapping triangles will get blended multiple times and
thus appear darker. Figure 11.10 shows this.
We can solve this problem using the stencil buffer.
1. Assume the stencil buffer pixels where the shadow will be rendered have been
cleared to 0. This is true in our mirror demo because we are only casting a
shadow onto the ground plane, and we only modified the mirror stencil buffer
pixels.
2. Set the stencil test to only accept pixels if the stencil buffer has an entry of 0. If
the stencil test passes, then we increment the stencil buffer value to 1.
The first time we render a shadow pixel, the stencil test will pass because the
stencil buffer entry is 0. However, when we render this pixel, we also increment the corresponding stencil buffer entry to 1. Thus, if we attempt to overwrite to an
area that has already been rendered to (marked in the stencil buffer with a value of
1), the stencil test will fail. This prevents drawing over the same pixel more than once,
and thus prevents double blending.

### Depth complexity
Depth complexity refers to the number of pixel fragments that compete, via the
depth test, to be written to a particular entry in the back buffer.

Potentially, the graphics card could fill a pixel several times each frame.
This overdraw has performance implications, as the graphics card is wasting
time processing pixels that eventually get overridden and are never seen.
Consequently, it is useful to measure the depth complexity in a scene for
performance analysis.
We can measure the depth complexity as follows: Render the scene and use the
stencil buffer as a counter; that is, each pixel in the stencil buffer is originally
cleared to zero, and every time a pixel fragment is processed, we increment
its count with D3D12_STENCIL_OP_INCR. The corresponding stencil buffer entry
should always be incremented for every pixel fragment no matter what, so
use the stencil comparison function D3D12_COMPARISON_ALWAYS. Then, for
example, after the frame has been drawn, if the ijth pixel has a corresponding
entry of five in the stencil buffer, then we know that that five pixel fragments
were processed for that pixel during that frame (i.e., the pixel has a depth
complexity of five). Note that when counting the depth complexity, technically
you only need to render the scene to the stencil buffer.
To visualize the depth complexity (stored in the stencil buffer), proceed as
follows:
a. Associate a color ck for each level of depth complexity k. For example, blue
for a depth complexity of one, green for a depth complexity of two, red for
a depth complexity of three, and so on. (In very complex scenes where the
depth complexity for a pixel could get very large, you probably do not want
to associate a color for each level. Instead, you could associate a color for a
range of disjoint levels. For example, pixels with depth complexity 1-5 are
colored blue, pixels with depth complexity 6-10 are colored green, and so
on.)
b. Set the stencil buffer operation to D3D12_STENCIL_OP_KEEP so that we do not
modify it anymore. (We modify the stencil buffer with D3D12_STENCIL_OP_
INCR when we are counting the depth complexity as the scene is rendered,
but when writing the code to visualize the stencil buffer, we only need to
read from the stencil buffer and we should not write to it.)
c. For each level of depth complexity k:
(i) Set the stencil comparison function to D3D12_COMPARISON_EQUAL and set
the stencil reference value to k.
(ii) draw a quad of color ck that covers the entire projection window. Note
that this will only color the pixels that have a depth complexity of k
because of the preceding set stencil comparison function and reference
value.

Another way to implement depth complexity visualization is to use additive
blending. First clear the back buffer black and disable the depth test. Next,
set the source and destination blend factors both to D3D12_BLEND_ONE, and the
blend operation to D3D12_BLEND_OP_ADD so that the blending equation looks
like C  Csrc  Cdst. Observe that with this formula, for each pixel, we are
accumulating the colors of all the pixel fragments written to it. Now render
all the objects in the scene with a pixel shader that outputs a low intensity
color like (0.05, 0.05, 0.05). The more overdraw a pixel has, the more of these
low intensity colors will be summed in, thus increasing the brightness of
the pixel. If a pixel was overdrawn ten times, for example, then it will have a
color intensity of (0.5, 0.5, 0.5). Thus by looking at the intensity of each pixel
after rendering the scene, we obtain an idea of the scene depth complexity.

### early z-test
The depth test occurs in the output merger stage of the pipeline, which occurs
after the pixel shader stage. This means that a pixel fragment is processed through
the pixel shader, even if it may ultimately be rejected by the depth test. However,
modern hardware does an “early z-test” where the depth test is performed before
the pixel shader. This way, a rejected pixel fragment will be discarded before
being processed by a potentially expensive pixel shader. To take advantage of
this optimization, you should try to render your non-blended game objects in
front-to-back order with respect to the camera; in this way, the nearest objects
will be drawn fi rst, and objects behind them will fail the early z-test and not be
processed further. This can be a signifi cant performance benefi t if your scene
suffers from lots overdraw due to a high depth complexity. We are not able to
control the early z-test through the Direct3D API; the graphics driver is the one
that decides if it is possible to perform the early z-test. For example, if a pixel
shader modifi es the pixel fragment’s depth value, then the early z-test is not
possible, as the pixel shader must be executed before the depth test since the pixel
shader modifi es depth values.

We mentioned the ability to modify the depth of a pixel in the pixel shader. How
does that work? A pixel shader can actually output a structure, not just a single
color vector as we have been doing thus far:
``` hlsl
struct PixelOut
{
   float4 color : SV_Target;
   float depth : SV_Depth;
};
PixelOut PS(VertexOut pin)
{
   PixelOut pout;
   // ... usual pixel work
   pout.Color = float4(litColor, alpha);
   // set pixel depth in normalized [0, 1] range
   pout.depth = pin.PosH.z - 0.05f;
   return pout;
}
```
The z-coordinate of the SV_Position element (pin.PosH.z) gives the unmodified
pixel depth value. Using the special system value semantic SV_Depth, the pixel
shader can output a modified depth value.

### z-fighting

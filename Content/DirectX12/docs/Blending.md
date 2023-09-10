## Blending

[D3D12_BLEND_DESC](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_blend_desc)

1. AlphaToCoverageEnable: Specify true to enable [alpha-to-coverage](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-blend-state#alpha-to-coverage), which is
a multisampling technique useful when rendering foliage or gate textures.
Specify false to disable alpha-to-coverage. Alpha-to-coverage requires
multisampling to be enabled (i.e., the back and depth buffer were created with
multisampling).

2. IndependentBlendEnable: Direct3D supports rendering to up to eight render
targets simultaneously. When this flag is set to true, it means blending can be
performed for each render target differently (different blend factors, different
blend operations, blending disabled/enabled, etc.). If this flag is set to false, it
means all the render targets will be blended the same way as described by the
first element in the D3D12_BLEND_DESC::RenderTarget array (only the RenderTarget[0] members are used; RenderTarget[1..7] are ignored.). Multiple render targets are used for advanced algorithms.

[D3D12_RENDER_TARGET_BLEND_DESC](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_render_target_blend_desc) 

The runtime implements RGB blending and alpha blending separately. Therefore, blend state requires separate blend operations for RGB data and alpha data.

1. It's not valid for LogicOpEnable and BlendEnable to both be TRUE.

[D3D12_BLEND](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_blend) : Blend Factors
[D3D12_BLEND_OP](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_blend_op)
[D3D12_LOGIC_OP](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_logic_op)

2. RenderTargetWriteMask

[D3D12_COLOR_WRITE_ENABLE](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_color_write_enable) : Identifies which components of each pixel of a render target are writable during blending.

**Examples**

1. No Color Write
   Set the source pixel blend factor to D3D12_BLEND_ZERO, the destination blend factor to 3D12_BLEND_ONE, and the blend operator to D3D12_BLEND_OP_ADD. Another way to implement the same thing would be to set the D3D12_RENDER_TARGET_BLEND_DESC::RenderTargetWriteMask member to 0, so that none of the color channels are written to.

2. Adding/Subtracting
   Set the source blend factor to D3D12_BLEND_ONE, the destination blend factor to D3D12_BLEND_ONE, and the blend operator to D3D12_BLEND_OP_ADD/D3D12_BLEND_OP_SUBTRACT.

3. Multiplying
   Set the source blend factor to D3D12_BLEND_ZERO, the destination blend factor to D3D12_BLEND_SRC_COLOR, and the blend operator to D3D12_BLEND_OP_ADD.

4. Transparency
   Set the source blend factor to D3D12_BLEND_SRC_ALPHA and the destination blend factor to 3D12_BLEND_INV_SRC_ALPHA, and the blend operator to D3D12_BLEND_OP_ADD.

**Clipping Pixels**

Sometimes we want to completely reject a source pixel from being further processed. This can be done with the intrinsic HLSL clip(x) function. This function can only be called in a pixel shader, and it discards the current pixel from further processing if x < 0.

the same result can be obtained using blending, but this is more efficient. For one thing, no blending calculation needs to be done (blending can be disabled). Also, the draw order does not matter. And furthermore, by discarding a pixel early from the pixel shader, the remaining pixel shader instructions can be skipped.

Due to filtering, the alpha channel can get blurred a bit, so you should leave some buffer room when clipping pixels. For example, clip pixels with alpha values close to 0, but not necessarily exactly zero.

**Alpha Test**

``` hlsl
#ifdef ALPHA_TEST
	// Discard pixel if alpha < 0.1
	clip(alpha - 0.1f);
#endif
```

Because we can now see through alpha tested objects, we want to disable back face culling for alpha tested objects.
D3D12_GRAPHICS_PIPELINE_STATE_DESC::RasterizerState::CullMode = D3D12_CULL_MODE_NONE

**Fog**

Fog can mask distant rendering artifacts and prevent popping. Popping refers to when an object that was previously behind the far plane all of a sudden comes in front of the frustum, due to camera movement, and thus becomes visible; so it seems to “pop” into the scene abruptly.

foggedColor = (1 - s) * litColor + s * fogColor
The parameter s ranges from 0 to 1 and is a function of the distance between the camera position and the surface point.

``` hlsl
#ifdef FOG
	float fogAmount = saturate((distToEye - gFogStart) / gFogRange);
	litColor = lerp(litColor, gFogColor, fogAmount);
#endif
```

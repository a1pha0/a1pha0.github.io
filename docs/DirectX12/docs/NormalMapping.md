## Normal Mapping

A normal map is a texture, but instead of storing RGB data at each texel, we store a
compressed x-coordinate, y-coordinate, and z-coordinate in the red component,
green component, and blue component, respectively. These coordinates define a
normal vector; thus a normal map stores a normal vector at each pixel.

For illustration, we will assume a 24-bit image format, which reserves a byte
for each color component, and therefore, each color component can range from
0-255. (A 32-bit format could be employed where the alpha component goes
unused or stores some other scalar value such as a heightmap or specular map.
Also, a fl oating-point format could be used in which no compression is necessary,
but this requires more memory.)

Given a compressed texture coordinate in the range 0-255,
when we sample a normal map in a shader like this:
``` hlsl
float3 normalT = gNormalMap.Sample( gTriLinearSam, pin.Tex );
```
The color vector normalT will have normalized components (r, g, b) such that 0 <=> r, g, b <=> 1. We complete the transformation by shifting and scaling each component in [0, 1] to [-1, 1].

If you want to use a compressed texture format to store normal maps, then use the
BC7 (DXGI_FORMAT_BC7_UNORM) format for the best quality, as it significantly reduces
the errors caused by compressing normal maps.

### Texture/Tangent Space

Normals stored in a normal map relative to a texture space coordinate system defined by the vectors T (x-axis), B (y-axis), and N (z-axis). The T vector runs right horizontally to the texture image; the B vector runs down vertically to the texture image; and N is orthogonal to the texture plane.

### Vertex Tangent Space

In the previous section, we derived a tangent space per triangle. However, if we use
this texture space for normal mapping, we will get a triangulated appearance since
the tangent space is constant over the face of the triangle. Therefore, we specify
tangent vectors per vertex, and we do the same averaging trick that we did with
vertex normals to approximate a smooth surface:
1. The tangent vector T for an arbitrary vertex v in a mesh is found by averaging
the tangent vectors of every triangle in the mesh that shares the vertex v.
2. The bitangent vector B for an arbitrary vertex v in a mesh is found by averaging
the bitangent vectors of every triangle in the mesh that shares the vertex v.

Generally, after averaging, the TBN-bases will generally need to be
orthonormalized, so that the vectors are mutually orthogonal and of unit length.
This is usually done using the Gram-Schmidt procedure.

In our system, we will not store the bitangent vector B directly in memory.
Instead, we will compute B = N × T when we need B.

## Collision

[directxcollision.h](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/)

---

### Bounding Volume

#### [BoundingBox](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/ns-directxcollision-boundingbox)

The axis-aligned bounding box (AABB) of a mesh is a box that tightly surrounds
the mesh and such that its faces are parallel to the major axes.

An AABB can be described by a minimum point vmin and a maximum point vmax. The minimum point vmin is found by searching through all the vertices of the mesh and finding the minimum x-, y-, and z-coordinates, and the maximum point vmax is found by searching through all the vertices of the mesh and finding the maximum x-, y-, and z-coordinates.

Given a bounding box defined by vmin and vmax, the center/extents representation is given by:
center = 0.5 * (vmin + vmax)
extents = 0.5 * (vmax - vmin)

[XMVectorMin](https://learn.microsoft.com/en-us/windows/win32/api/directxmath/nf-directxmath-xmvectormin)

[XMVectorMax](https://learn.microsoft.com/en-us/windows/win32/api/directxmath/nf-directxmath-xmvectormax)

#### [BoundingOrientedBox](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/ns-directxcollision-boundingorientedbox)

If we compute the AABB of a mesh in local space, it gets transformed to an oriented bounding box (OBB) in world space.

#### [BoundingSphere](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/ns-directxcollision-boundingsphere)

#### [BoundingFrustum](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/ns-directxcollision-boundingfrustum)

---

### Frustum Culling

The idea of frustum culling is for the application code to cull groups of triangles
at a higher level than on a per-triangle basis. We build a bounding volume, such as a sphere or box, around each object in the scene. If the bounding volume does not intersect the frustum, then we do not need to submit the object to Direct3D for drawing. This saves the GPU from having to do wasteful computations on invisible geometry, at the cost of an inexpensive CPU test.

---

### Picking

``` C++
typedef struct D3D12_VIEWPORT
{
FLOAT TopLeftX;
FLOAT TopLeftY;
FLOAT Width;
FLOAT Height;
FLOAT MinDepth;
FLOAT MaxDepth;
} D3D12_VIEWPORT;
```

the viewport matrix transforms vertices from normalized device coordinates to screen space; it is given below:
$$
 M = \left[
 \begin{matrix}
   Width/2 & 0 & 0 & 0 \\
   0 & -Height/2 & 0 & 0 \\
   0 & 0 & MaxDepth-MinDepth & 0 \\
   TopLeftX+Width/2 & TopLeftY+Height/2 & MinDepth & 1
  \end{matrix}
  \right]
$$

Generally, for a game, the viewport is the entire backbuffer and the depth buffer range is 0 to 1. Thus, TopLeftX = 0, TopLeftY = 0, MinDepth = 0, MaxDepth = 1, Width = w, and Height = h, where w and h, are the width and height of the backbuffer, respectively. Assuming this is indeed the case, the viewport matrix simplifies to:
$$
 M = \left[
 \begin{matrix}
   w/2 & 0 & 0 & 0 \\
   0 & -h/2 & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   w/2 & h/2 & 0 & 1
  \end{matrix}
  \right]
$$

Now let $ p_{ndc} = [x_{ndc}, y_{ndc}, z_{ndc}, 1] $ be a point in normalized device space (i.e., $ -1 \leq x_{ndc} \leq 1 $, $-1 \leq y_{ndc} \leq 1$, and $0 \leq z_{ndc} \leq 1$). Transforming $ p_{ndc} $ to screen space yields:

$$
 [x_{ndc}, y_{ndc}, z_{ndc}, 1]\left[
 \begin{matrix}
   w/2 & 0 & 0 & 0 \\
   0 & -h/2 & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   w/2 & h/2 & 0 & 1
  \end{matrix}
  \right] = [\frac{x_{ndc}w+w}{2}, \frac{-y_{ndc}h+h}{2}, z_{ndc}, 1]
$$

The coordinate zndc is just used by the depth buffer and we are not concerned with any depth coordinates for picking. The 2D screen point $ p_s(x_s, y_s) $ corresponding to $ p_{ndc} $ is just the transformed x- and y-coordinates:

$ x_s = \frac{x_{ndc}w + w}{2} $, $ y_s = \frac{-y_{ndc}h + h}{2} $

Solving the above equations for $ p_{ndc} $ yields:

$ x_{ndc} = \frac{2x_s}{w} - 1 $, $ y_{ndc} = -\frac{2y_s}{h} + 1 $

Recall that we mapped the projected point from view space to NDC space by dividing the x-coordinate by the aspect ratio r. Thus, to get back to view space, we just need to multiply the x-coordinate in NDC space by the aspect ratio. The clicked point in view space is thus:

$ x_v = r(\frac{2x_s}{w} - 1) $, $ y_v = -\frac{2y_s}{h} + 1 $

the projection window lies at a distance $ d = cot(\frac{\alpha }{2}) $ from the origin, where $ \alpha $ is the vertical field of view angle. So we could shoot the picking ray through the point $ (x_v, y_v, d) $ on the projection window. However, this requires that we compute $ d = cot(\frac{\alpha }{2}) $. Thus, we can shoot our picking ray through the point $ (x_v', y_v', 1) = (\frac{x_v}{d}, \frac{y_v}{d}, 1) $ instead.

$ x_v' = \frac{x_v}{d} = \frac{x_v}{cot(\frac{\alpha}{2})} = x_v\cdot tan(\frac{\alpha}{2}) = (\frac{2x_s}{w} - 1)rtan(\frac{\alpha}{2}) $

$ y_v' = \frac{y_v}{d} = \frac{y_v}{cot(\frac{\alpha}{2})} = y_v\cdot tan(\frac{\alpha}{2}) = (-\frac{2y_s}{w} + 1)tan(\frac{\alpha}{2}) $

Recalling that $ P_{00} = \frac{1}{rtan(\frac{\alpha}{2})} $ and $ P_{11} = \frac{1}{tan(\frac{\alpha}{2})} $ in the projection matrix, we can rewrite this as:

$ x_v' = (\frac{2x_s}{w} - 1)/P_{00} $
$ y_v' = (-\frac{2y_s}{h} + 1)/P_{11} $

The code that computes the picking ray in view space is given below:

``` C++
DirectX::XMFLOAT4X4 P = mCamera.GetProjMatrix4x4f();

// Ray definition in view space.
float vx = (+2.0f * screenX / mClientWidth - 1.0f) / P(0, 0);
float vy = (-2.0f * screenY / mClientHeight + 1.0f) / P(1, 1);
DirectX::XMVECTOR rayOrigin = DirectX::XMVectorSet(0.0f, 0.0f, 0.0f, 1.0f);
DirectX::XMVECTOR rayDirection = DirectX::XMVector3Normalize(DirectX::XMVectorSet(vx, vy, 1.0f, 0.0f));
```

**Ray/Box Intersection**

[BoundingBox::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-boundingbox-intersects(fxmvector_fxmvector_float_))

**Ray/Sphere Intersection**

[BoundingSphere::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-boundingsphere-intersects(fxmvector_fxmvector_float_))

**Ray/Triangle Intersection**

[TriangleTests::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-intersects)

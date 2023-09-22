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

**Ray/Box Intersection**

[BoundingBox::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-boundingbox-intersects(fxmvector_fxmvector_float_))

**Ray/Sphere Intersection**

[BoundingSphere::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-boundingsphere-intersects(fxmvector_fxmvector_float_))

**Ray/Triangle Intersection**

[TriangleTests::Intersects](https://learn.microsoft.com/en-us/windows/win32/api/directxcollision/nf-directxcollision-intersects)


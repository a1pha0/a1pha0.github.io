## Visual Appearance

### Light Source

The light vector **l** is usually defined pointing in a direction opposite to the direction the light is traveling.

The emission of a directional light source can be quantified by measuring power through a unit area surface perpendicular to **l**. This quantity, called *irradiance*, is equivalent to the sum of energies of the photons passing through the surface in one second.

E is used in equations for irradiance, most commonly for irradiance perpendicular to n. We will use EL for irradiance perpendicular to l. Note that a negative value of the cosine corresponds to the case where light
comes from below the surface. In this case the light does not illuminate the surface at all, so we will use the cosine clamped to non-negative values (symbolized by cos):

$$ E = E_L \bar{\cos}θ_i = E_L \max(n · l, 0) $$

### Material

Fundamentally, all light-matter interactions are the result of two phenomena: *scattering* and *absorption*.

Scattering happens when light encounters any kind of optical discontinuity. This may be the interface between two substances with different optical properties, a break in crystal structure, a change in density, etc. Scattering does not change the amount of light—it just causes it to change direction (reflection and refraction).

Absorption happens inside matter and causes some of the light to be converted into another kind of energy and disappear. It reduces the amount of light but does not affect its direction.

The most important optical discontinuity in rendering is the interface between air and object that occurs at a model surface. Surfaces scatter light into two distinct sets of directions: into the surface (refraction or transmission) and out of it (reflection).

It is common to separate surface shading equations into two terms. The *specular term* represents the light that was reflected at the surface, and the *diffuse term* represents the light which has undergone transmission, absorption, and scattering.

Incoming illumination is measured as surface irradiance. We measure outgoing light as *exitance*, which similarly to irradiance is energy per second per unit area. The symbol for exitance is M. Exitance divided by irradiance is a characteristic property of the material. For surfaces
that do not emit light on their own, this ratio must always be between 0 and 1. The ratio between exitance and irradiance can differ for light of different colors, so it is represented as an RGB vector or color, commonly called the *surface color* **c**. To represent the two different terms, shading equations often have separate specular color (**c**spec) and diffuse color (**c**diff) properties, which sum to the overall surface color **c**.

We assume that the diffuse term has no directionality it represents light going uniformly in all directions. The specular term does have significant directionality that needs to be addressed. Unlike the surface colors that were primarily dependent on the composition of the surface, the directional distribution of the specular term depends on the surface smoothness. The beam of reflected light is tighter for the smoother surface, and more spread out for the rougher surface.

### Sensor

After light is emitted and bounced around the scene, some of it is absorbed in the imaging sensor. An imaging sensor is actually composed of many small sensors: rods and cones in the eye, photodiodes in a digital camera, or dye particles in film. Each of these sensors detects the irradiance value over its surface and produces a color signal.

### Shading

*Shading* is the process of using an equation to compute the outgoing radiance Lo along the view ray, **v**, based on material properties and light sources. Here we will present a relatively simple one as an example, and then discuss how to implement it using programmable shaders.

$$ L_o(\textbf{v}) = ( \frac{\textbf{c}_{diff}}{\pi} + \frac{m + 8}{8\pi} (\max(\cos{\theta_h}, 0))^m \textbf{c}_{spec} ) ⊗ E_L\max(\cos\theta_i, 0) $$

where EL is the light’s irradiance, l is the light’s direction vector, θi is the angle between l and n , θh is the angle between h and n, h is halfway between the view vector v and the light vector l

Lighting is additive in nature, so we can sum the contributions of the individual light sources to get the overall shading equation:

$$ L_o(\textbf{v}) = \sum_{k=1}^{n} ( ( \frac{\textbf{c}_{diff}}{\pi} + \frac{m + 8}{8\pi} (\max(\cos\theta_{h_k}, 0))^m \textbf{c}_{spec} ) ⊗ E_{L_k}\max(\cos\theta_{i_k}, 0) ) $$

where θhk is the value of θh for the kth light source.

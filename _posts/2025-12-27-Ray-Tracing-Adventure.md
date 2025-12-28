---
layout: post
title: "Ray Tracing Adventure Part 5 (Homework 5)"
date: 2025-12-27
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 5, focusing on implementing features:

- **Tonemapping**
- **.exr and .hdr Images**
- **Directional Lights**
- **Spot Lights**
- **Environment Lights**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz, at Middle East Technical University.

### Bugs from Previous Parts 
While rendering the Chinese Dragon in previous and this homeworks, I encountered a issue where the model appeared only as scattered points rather than a continuous surface. At first glance, I thought the error's source was backface culling errors, incorrect BVH construction, or insufficient sampling. As a result, I initially focused my debugging efforts on acceleration structures, sampling parameters, and camera or lighting configurations. However, none of these adjustments resolved the issue. The render output remained unchanged even after disabling backface culling and significantly increasing the sample count, which indicated that the problem was occurring at a more fundamental level.

[bugged dragon render]

After further inspection, I realized that the core issue was caused by hard-coded epsilon thresholds in the triangle intersection tests. Since the Chinese Dragon consists of a very large number of extremely small triangles, many valid intersections were incorrectly discarded due to absolute precision checks. As a result, only a small subset of rays successfully intersected the mesh, producing the "scattered points"” appearance in the final image.

Here is the original triangle intersection code snippet with the problematic epsilon values:

```cpp
// If the area is too small, consider it degenerate
if (N_len_sq < 1e-12f) 
    return false; // Degenerate triangle

float NdotD = N.dot(rayDir);

if (fabs(NdotD) < 1e-8f)
  return false; // Plane and ray are parallel
```

For now, I replaced epsilon with smaller values to accommodate the small triangle sizes in the dragon model, but I will get epsilon values directly from the input file and set a smaller value for the default case in future implementations. Her eyou can see the marching_dragons (rendered in 31.2205 seconds) from the Homework 2 also:

[marching_dragons render]

### Tonemapping and .exr/.hdr Images
Until this point, my ray tracer mostly produced standard 8-bit images (PNG), where each color channel is clamped to the 0-255 range. That works fine for normal scenes, but it becomes a limitation as soon as the lighting gets intense. Bright highlights get clipped to pure white, and any detail in those regions is lost permanently.

To solve this, in this part, I added HDR output support by writing renders as .exr file. EXR is a high dynamic range format, meaning pixel values are stored as floating point numbers (not restricted to 0–255). This allows the renderer to preserve real radiance values (even if they exceed 1.0 by a lot) so highlights and bright reflections are still recoverable later.

However, an EXR file is not meant to be viewed directly as a regular image. To display HDR data on a standard monitor, we need tonemapping, which compresses HDR image into the limited range of an LDR image while trying to preserve contrast and details. As stated in homework, each camera can optionally define one or more tonemapping operators in the JSON scene file. This means a single HDR render can be converted into multiple PNG outputs using different tonemapping curves.

For example, for cube_point_hdr scene, ray tracer produces:

- cube_point_hdr.exr (raw HDR output)
- cube_point_phot.png (Photographic tonemap)
- cube_point_aces.png (ACES tonemap)
- cube_point_film.png (Filmic tonemap)

But what are the differences between these tonemapping operators? These different tonemapping operators handle highlights and midtones differently. In Photographic tonemapping, ray tracer produces a more “neutral” result by compressing luminance smoothly. ACES tends to preserve highlight roll-off in a more cinematic way (bright areas fade more naturally instead of clipping harshly). Finally, Filmic tonemapping mimics the response curve of film stock, producing a more contrasty image with deeper shadows and punchier highlights.

Even though all three PNGs originate from the same EXR data, their brightness distribution and contrast should noticeably differ as you can see in the images below:

[cube_point_hdr renders]

The implementation strategy was quite straightforward. First, First, I render the scene once into an HDR framebuffer (a std::vector<Vec3>), where each pixel stores radiance values as floats (not clamped to 0-255). After the rendering is complete, I saved the .exr file using the [TinyEXR library](https://github.com/syoyo/tinyexr), because stb library does not support EXR format even though it supports HDR format. Then, for each tonemapping operator specified in the camera, I applied the corresponding tonemapping function to convert the HDR framebuffer into an LDR framebuffer (clamped to 0-255), and saved that as a PNG using stb_image_write.


```cpp
// Save EXR if the camera output is .exr
if (ImageIO::endsWith(cam.imageName, ".exr")) {
    ImageIO::writeEXR(cam.imageName, width, height, hdr);
}

// Tonemap and save PNGs
for (const auto& tm : cam.tonemaps) {
    auto png = Tonemapper::toPNG(width, height, hdr, tm);
    string outName = ImageIO::replaceExrWithExtension(cam.imageName, tm.extension);
    ImageIO::writePNG(outName, width, height, png);
}
```

This process does not include additional ray tracing passes for each tonemapping operator, since the HDR framebuffer is reused. This keeps render times reasonable even when multiple tonemaps are requested.

I implemented each tonemapping using the formulas discussed in class but I want to highlight the gamma correction step that is applied after tonemapping. It looks minor in code, but makes a huge difference in the final image. In my ray tracer, all lighting computations are done in linear space. This is important because shading equations (diffuse/specular etc.) assume linear intensity values. However, monitors and standard image formats (like PNG) do not display values linearly. Most displays roughly follow an sRGB-like response curve, meaning that if we directly write linear values into an 8-bit image, the result looks too dark, especially in midtones. This happens because the monitor effectively applies its own non-linear curve, compressing low values more than we intended.

That’s why, after tonemapping compresses HDR radiance into an LDR 0-1 range, I apply gamma correction as the very last step before converting to 8-bit:

```cpp
float invGamma = 1.0f / gamma; // gamma is typically around 2.2f

// Apply gamma correction
c.x = pow(clamp01(c.x), invGamma);
c.y = pow(clamp01(c.y), invGamma);
c.z = pow(clamp01(c.z), invGamma);

out[index + 0] = (unsigned char) lround(255.0f * c.x);
out[index + 1] = (unsigned char) lround(255.0f * c.y);
out[index + 2] = (unsigned char) lround(255.0f * c.z);
```

This step converts the image from linear space to display space. If gamma = 2.2, then using pow(x, 1/2.2) lifts mid-range values, making the image look visually correct on a typical monitor. Also another important detail is the order of operations. Gamma correction is applied after tonemapping, because tonemapping curves are defined in linear light. Applying gamma earlier would distort luminance relationships and can cause strange contrast shifts.

### Directional Lights
After supporting point and area lights, the next lighting feature I added was Directional Lights. A directional light represents a light source that is effectively infinitely far away (the classic example is sunlight). Because the source is so far, all incoming rays are assumed to be parallel, meaning the light is defined only by a direction vector and a radiance value.

A key difference from point or spot lights is that directional lights have no distance attenuation. With point lights, intensity falls off as 1 / d², but for directional lights the incoming radiance is constant everywhere in the scene. At each hit point, shading is very similar to a point light except there is no 1 / d^2 term (since d is infinite, no inverse square law) and there is no finite light position, so we treat the shadow ray as going forever.

Because the light direciton is constant, instead of computing wi = normalize(lightPos − hitPoint), we already know the incoming direction, so the direction from the surface to the light is wi = -Direction. For shadow rays, to keep my code structure as it is, I treated directional light shadow checks as infinite distance (very large distance for now, 1e9) and only looked for any occluder. This way, directional lights integrate cleanly into the existing lighting loop without special-casing the intersection logic too much. In future, I may optimize this further by skipping distance checks for directional lights entirely.

```cpp
for (const auto& dl : scene.directionalLights) {
  Vec3 wi = dl.direction.scale(-1.0f).normalize(); // Light direction is opposite to light's direction

  // Shadow ray check for directional light at very large distance
  Vec3 shadowOrigin = hitPoint.add(hitNormal.scale(scene.shadowRayEpsilon));
  if (isInShadow(scene, shadowOrigin, wi, 1e9f, intersector, rayTime)) continue;

  Vec3 eff = dl.radiance;

  // Diffuse and Specular calculations as before with eff added
  // ...
  Vec3 diffuse = kd.multiply(eff).scale(NdotL);
  // ...
  Vec3 specular = ks.multiply(eff).scale(std::pow(NdotH, material.phongExponent));

  color = color.add(diffuse).add(specular);
}
```

### Spot Lights
After directional lights, I implemented Spot Lights, which can be thought of as a point light with a cone. A spot light has a finite position like a point light, but unlike a point light it does not illuminate equally in all directions. Instead, it emits light mainly around a preferred direction, and its contribution depends on the angle between the light direction and the direction from the light to the shaded point. It includes position, direction, intensity, coverage and falloff angle parameters (how wide the cone and how soft its edges).

The usual inverse-square attenuation 1/d^2 a spot light only contributes if the shaded point lies inside the light’s coverage cone. I compute the angle α between the spot axis (light direction) and the direction from the light to the hit point, using a dot product (cosine space).

If α is smaller than the falloff angle, the light behaves exactly like a point light (full intensity). Between the falloff and coverage angles, I apply a smooth falloff term

[pdfteki s formulü]

and scale the irradiance as s x I / d^2. Beyond the coverage angle the contribution becomes zero.

```cpp
for (const auto& sl : scene.spotLights) {
    // ...

    float cosAlpha = max(-1.0f, min(1.0f, spotAxis.dot(lightToHit)));

    float cov = glm::radians(sl.coverageAngle * 0.5f);
    float fall = glm::radians(sl.falloffAngle * 0.5f);

    float cosCov = cos(cov);
    float cosFall = cos(fall);

    float spotFactor = 0.0f;

    if (cosAlpha >= cosFall) {
        spotFactor = 1.0f;
    }
    else if (cosAlpha >= cosCov) {
        float denom = (cosFall - cosCov);
        float t = (cosAlpha - cosFall) / denom;
        t = max(0.0f, min(1.0f, t));
        spotFactor = pow(t, 4.0f);
    }
    else {
        continue;
    }

    Vec3 effIntensity = sl.intensity.scale(spotFactor / d2);

    // Diffuse and Specular calculations as before with effIntensity added
    // ...
}
```

### Environment Lights
The final lighting feature I added was environment lighting. Unlike point/spot/directional lights, an environment light does not come from a single position or direction it provides illumination from all directions, based on an HDR environment map (either latitude-longitude or light-probe format).

The core idea is to treat the environment map as a function that returns radiance for a given direction d. During shading, instead of querying a light position, I randomly sample a direction wi from the hemisphere above the surface. Then I convert that direction into texture coordinates (u, v) using the mapping described in the homework for lat-long as:

[formula for lat-long mapping 5,6,7]

```cpp
static Vec2 dirToUV_LatLong(const Vec3& d) {
    Vec3 dn = d.normalize();
    float u = (1.0f + atan2(dn.x, -dn.z) / (float) M_PI) * 0.5f;
    float v = acos(max(-1.0f, min(1.0f, dn.y))) / (float) M_PI;

    u = u - floor(u); // Wrap u to [0,1]
    v = max(0.0f, min(1.0f, v)); // Clamp v to [0,1]
    return Vec2(u, v);
}
```

and for light-probe mapping as:
[formula for light-probe mapping 8,9,10]

```cpp

static Vec2 dirToUV_Probe(const Vec3& d) {
    Vec3 dn = d.normalize();
    
    float denom = sqrt(dn.x * dn.x + dn.y * dn.y);

    float r = (1.0f / (float) M_PI) * acos(max(-1.0f, min(1.0f, -dn.z))) / denom;
    float u = (r * dn.x + 1.0f) * 0.5f;
    float v = (-r * dn.y + 1.0f) * 0.5f;

    u = u - floor(u); // Wrap u to [0,1]
    v = max(0.0f, min(1.0f, v)); // Clamp v to [0,1]

    return Vec2(u, v);
}
```

and fetch the corresponding HDR radiance from the environment image.

Because we only sample one direction (or a small number of directions), we must compensate for sampling probability. This is done by dividing the fetched radiance by the PDF of the sampling method. I implemented both uniform hemisphere sampling (constant PDF) and cosine-weighted sampling (PDF proportional to cosθ).

Finally, just like directional lights, environment light shadow rays are treated as going to infinity (very large distance, 1e9), if any occluder is hit along wi, that sampled contribution is discarded.

### Outputs and Closing Thoughts
All renders are completed (also the chinese dragon for the first time :), but the 15th .ply file in VeachAjar scene is still causing problems as in previous part (happly gives error for unsigned int), but I still use the not fixed version and get the expected results. Some scenes like teapot_roughness and dragon_new_ply_with_spot took several hours to render. Therefore, I will try to refactor and optimize my ray tracer further in the next parts.

I partially tried to convert my float calculations to double precision in critical sections (like intersection tests) to fix noise in some scenes as Oğuz Hoca suggested, but I could not finish that yet because project is getting bigger and I sometimes take shortcuts and they effect the genericity of the code, so a general change like this became difficult to me. As I said in the previous paragraph, I will try to improve the code structure and optimize performance in future parts to add new features more easily.

As in previous parts, I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance, and Akın Aydemir for contributions to the 3D models. Here are my final renders and their render times:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| cube_directional                 | 1.69274  |
| cube_point                 | 1.75236  |
| cube_point_hdr                 | 2.98016  |
| dragon_spot_light_msaa                 | 79.1459  |
| empty_environment_latlong                 | 1.22012  |
| empty_environment_light_probe                 | 1.22698   |
| glass_sphere_env                 | 2.91417  |
| head_env_light                 | 61.3175  |
| mirror_sphere_env                 | 1.71812  |
| sphere_env_light                 | 1156.8  |
| sphere_point_hdr_texture                 | 2.99435  |
| teapot_roughness                 | 20049.2* (approximately 5.5 hours)  |
| dragon_new_ply_with_spot                 | 15166.2* (approximately 4.2 hours)  |
| audi-tt-glacier                 | 7219.37* (approximately 2 hours)  |
| audi-tt-pisa                 | 7209.42* (approximately 2 hours)  |
| VeachAjar                 | 192.583  |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

**Used CPU: AMD Ryzen 5 7640HS 6-Core Processor (4.30 GHz)*

---
## dragon_new
This render took nearly 4 hours, and when I was losing my faith, my ray tracer finally won its battle with 3.686.400 pixels :). So I wanted to place this render at the top of all other renders.

_Time: 14394.7 s_
<p align="center">
<img alt="dragon_new" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/dragon_new.png" />
</p>

---

### tunnel_of_doom
<video width="100%" controls autoplay muted loop playsinline>
  <source src="{{ site.baseurl }}/videos/tunnelofdoom.mp4" type="video/mp4">
</video>

---
### brickwall_with_normalmap
_Time: 1.60214 s_
<p align="center">
<img alt="brickwall_with_normalmap" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/brickwall_with_normalmap.png" />
</p>

---
### bump_mapping_transformed
_Time: 3.24855 s_
<p align="center">
<img alt="bump_mapping_transformed" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/bump_mapping_transformed.png" />
</p>

---
### cube_cushion
_Time: 1.53794 s_
<p align="center">
<img alt="cube_cushion" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_cushion.png" />
</p>

---
### cube_perlin
_Time: 1.47713 s_
<p align="center">
<img alt="cube_perlin" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_perlin.png" />
</p>

---
### cube_perlin_bump
_Time: 1.56417 s_
<p align="center">
<img alt="cube_perlin_bump" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_perlin_bump.png" />
</p>

---
### cube_wall
_Time: 1.57439 s_
<p align="center">
<img alt="cube_wall" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_wall.png" />
</p>

---
### cube_wall_normal
_Time: 1.566830 s_
<p align="center">
<img alt="cube_wall_normal" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_wall_normal.png" />
</p>

---
### cube_waves
_Time: 1.63458 s_
<p align="center">
<img alt="cube_waves" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/cube_waves.png" />
</p>

---
### ellipsoids_texture
_Time: 2.92964 s_
<p align="center">
<img alt="ellipsoids_texture" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/ellipsoids_texture.png" />
</p>

---
### galactica_dynamic
_Time: 449.681 s_
<p align="center">
<img alt="galactica_dynamic" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/galactica_dynamic.png" />
</p>

---
### galactica_static
_Time: 5.75577 s_
<p align="center">
<img alt="galactica_static" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/galactica_static.png" />
</p>

---
### killeroo_bump_walls
_Time: 26.6139 s_
<p align="center">
<img alt="killeroo_bump_walls" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/killeroo_bump_walls.png" />
</p>

---
### plane_bilinear
_Time: 1.09835 s_
<p align="center">
<img alt="plane_bilinear" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/plane_bilinear.png" />
</p>

---
### plane_nearest
_Time: 1.13796 s_
<p align="center">
<img alt="plane_nearest" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/plane_nearest.png" />
</p>

---
### sphere_nearest_bilinear
_Time: 2.16499 s_
<p align="center">
<img alt="sphere_nearest_bilinear" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nearest_bilinear.png" />
</p>

---
### sphere_nobump_bump
_Time: 1.9995 s_
<p align="center">
<img alt="sphere_nobump_bump" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nobump_bump.png" />
</p>

---
### sphere_nobump_justbump
_Time: 1.94817 s_
<p align="center">
<img alt="sphere_nobump_justbump" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nobump_justbump.png" />
</p>

---
### sphere_normal
_Time: 117.432 s_
<p align="center">
<img alt="sphere_normal" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_normal.png" />
</p>

---
### sphere_perlin
_Time: 2.4421 s_
<p align="center">
<img alt="sphere_perlin" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin.png" />
</p>

---
### sphere_perlin_bump
_Time: 2.55343 s_
<p align="center">
<img alt="sphere_perlin_bump" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin_bump.png" />
</p>

---
### sphere_perlin_scale
_Time: 2.40968 s_
<p align="center">
<img alt="sphere_perlin_scale" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin_scale.png" />
</p>

---
### wood_box
_Time: 1.58755 s_
<p align="center">
<img alt="wood_box" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box.png" />
</p>

---
### wood_box_all
_Time: 1.60994 s_
<p align="center">
<img alt="wood_box_all" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box_all.png" />
</p>

---
### wood_box_no_specular
_Time: 1.5577 s_
<p align="center">
<img alt="wood_box_no_specular" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box_no_specular.png" />
</p>

---
### VeachAjar
_Time: 174.211 s_
<p align="center">
<img alt="VeachAjar" src="https://raw.githubusercontent.com/fsaltunyuva/fsaltunyuva.github.io/refs/heads/main/images/2025-12-15-Ray-Tracing-Adventure/VeachAjar.png" />

</p>

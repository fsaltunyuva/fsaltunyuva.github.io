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

<p align="center">
    <img alt="bugged dragon" src="https://github.com/user-attachments/assets/fcea000b-6c15-44e0-ad01-200a74aa5a3b" />
</p>

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

<p align="center">
    <img alt="marchingdragons" src="https://github.com/user-attachments/assets/0e1ba002-51ea-4c1f-a8d6-6a5cbf76e3b0" />
</p>

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

Even though all three PNGs originate from the same EXR data, their brightness distribution and contrast should noticeably differ as you can see in the images below (Photographic, ACES, and Filmic respectively):

<p align="center">
    <img alt="phot-aces-filmic" src="https://github.com/user-attachments/assets/ced66605-2cbc-42bf-8709-e89b94a16c9a" />
</p>

The implementation strategy was quite straightforward. First, I render the scene once into an HDR framebuffer, where each pixel stores radiance values as floats (not clamped to 0-255). After the rendering is complete, I saved the .exr file using the [TinyEXR library](https://github.com/syoyo/tinyexr) , because stb library does not support EXR format even though it supports HDR format. Then, for each tonemapping operator specified in the camera, I applied the corresponding tonemapping function to convert the HDR framebuffer into an LDR framebuffer (clamped to 0-255), and saved that as a PNG using stb_image_write.


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

This step converts the image from linear space to display space. If gamma = 2.2, then using pow(x, 1/2.2) lifts mid-range values, making the image look visually correct on a typical monitor. Also another important detail is the order of operations. Gamma correction is applied after tonemapping, because tonemapping curves are defined in linear light. Applying gamma earlier would distort luminance relationships and could cause strange contrast shifts. Here you can see gamma corrected, and not gamma corrected example respectively:

<p align="center">
    <img alt="filmic-left-gamma-crtd-right-not" src="https://github.com/user-attachments/assets/013ae604-3fdb-441c-8b58-c343d85b7347" />
</p>


### Directional Lights
After supporting point and area lights, the next lighting feature I added was Directional Lights. A directional light represents a light source that is effectively infinitely far away (the classic example is sunlight). Because the source is so far, all incoming rays are assumed to be parallel, meaning the light is defined only by a direction vector and a radiance value.

A key difference from point or spot lights is that directional lights have no distance attenuation. With point lights, intensity falls off as $\frac{1}{d^2}$, but for directional lights the incoming radiance is constant everywhere in the scene. At each hit point, shading is very similar to a point light except there is no $\frac{1}{d^2}$ term (since d is infinite, no inverse square law) and there is no finite light position, so we treat the shadow ray as going forever.

Because the light direciton is constant, instead of computing ``wi = normalize(lightPos − hitPoint)``, we already know the incoming direction, so the direction from the surface to the light is ``wi = -direction``. For shadow rays, to keep my code structure as it is, I treated directional light shadow checks as infinite distance (very large distance for now, 1e9) and only looked for any occluder. This way, directional lights integrate cleanly into the existing lighting loop without special-casing the intersection logic too much. In future, I may optimize this further by skipping distance checks for directional lights entirely.

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

The usual inverse-square attenuation $\frac{1}{d^2}$ a spot light only contributes if the shaded point lies inside the light’s coverage cone. I compute the angle α between the spot axis (light direction) and the direction from the light to the hit point, using a dot product (cosine space).

If α is smaller than the falloff angle, the light behaves exactly like a point light (full intensity). Between the falloff and coverage angles, I apply a smooth falloff term

$$s = \left( \frac{\cos(\alpha) - \cos(\text{falloff}/2)}{\cos(\text{falloff}/2) - \cos(\text{coverage}/2)} \right)^4$$

and scale the irradiance as: $$\frac{s \times I}{d^2}$$ Beyond the coverage angle the contribution becomes zero.

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

While testing spot lights, I noticed a visible hard ring at the edge of the illumination cone. The light was behaving correctly inside the inner cone, but the transition region produced an unnatural boundary.

<p align="center">
    <img alt="hard-edge" src="https://github.com/user-attachments/assets/98fcdbe3-9fbf-4340-95bb-268aff2c4c3d" />
</p>

The issue was in my falloff interpolation. In the falloff region, the attenuation term should be 1 at the falloff boundary and smoothly decrease to 0 at the coverage boundary.

However, in my first implementation I normalized the parameter incorrectly (effectively making the attenuation become 0 right at the falloff boundary), which caused a sudden intensity drop and therefore a visible ring.

The fix was to parameterize the interpolation so that it maps fallof to 1 and coverage to 0, then clamp it into [0,1] before applying the exponent.

```cpp
else if (cosAlpha >= cosCov) {
  float denom = (cosFall - cosCov);
  float t = (cosAlpha - cosCov) / denom;

  // Clamp t to [0,1]
  if (t < 0.0f) t = 0.0f;
  if (t > 1.0f) t = 1.0f;

  spotFactor = pow(t, 4.0f);
}
```

After this change, the transition became smooth, and the ring artifact disappeared:

<p align="center">
    <img alt="hard-edge-fix" src="https://github.com/user-attachments/assets/80f9588d-7241-4046-a8c2-1738b5068049" />
</p>

### Environment Lights
The final lighting feature I added was environment lighting. Unlike point/spot/directional lights, an environment light does not come from a single position or direction it provides illumination from all directions, based on an HDR environment map (either latitude-longitude or light-probe format).

The core idea is to treat the environment map as a function that returns radiance for a given direction d. During shading, instead of querying a light position, I randomly sample a direction wi from the hemisphere above the surface. Then I convert that direction into texture coordinates (u, v) using the mapping described in the homework for lat-long as:

<p align="center">
    <img width="35%" alt="latlongformulas" src="https://github.com/user-attachments/assets/2bb874c6-23ff-4d2e-994d-c752f1e90d15" />
</p>

```cpp
Vec2 dirToUV_LatLong(const Vec3& d) {
    Vec3 dn = d.normalize();
    float u = (1.0f + atan2(dn.x, -dn.z) / (float) M_PI) * 0.5f;
    float v = acos(max(-1.0f, min(1.0f, dn.y))) / (float) M_PI;

    u = u - floor(u); // Wrap u to [0,1]
    v = max(0.0f, min(1.0f, v)); // Clamp v to [0,1]
    return Vec2(u, v);
}
```

and for light-probe mapping as:

<p align="center">
    <img width="20%" alt="probeformulas" src="https://github.com/user-attachments/assets/4ae3d25c-0393-4577-981e-1e9095de9d17" />
</p>

```cpp
Vec2 dirToUV_Probe(const Vec3& d) {
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

### Degamma
After implementing and testing on several scenes, I tried the VeachAjar scene, and I got this result in ACES tonemapping:

<p align="center">
    <img alt="veachajarbug1" src="https://github.com/user-attachments/assets/546d8946-c9cc-4e11-81fa-96048c3cbd46" />
</p>

I immediately realized that I faced with a similar issue in previous part. The Normalizer value was affecting the results. So I firstly checked that part.

```cpp
if (tm->normalizer > 0.0f && tm->normalizer != 255.0f && tm->normalizer != 1.0f) {
  float scaleFactor = 255.0f / tm->normalizer;
  texColor = texColor.scale(scaleFactor);
}
```

This part was not considering if the texture is HDR or LDR. So I added a flag in Texture class to indicate if the image is HDR or not, and I modified this part as follows:

```cpp
if (tm->normalizer > 0.0f) {
    if (tm->image->isHDR) {
        texColor = texColor.scale(1.0f / tm->normalizer); // For HDR, normalize to [0,1] range
    } else {
        texColor = texColor.scale(255.0f / tm->normalizer); // For LDR, normalize to [0,255] range
    }
}
```

After this fix, result looked better but still not correct:

<p align="center">
    <img alt="veachajarbug2" src="https://github.com/user-attachments/assets/07cfc57c-82a2-47b1-9e6b-76f10acd2f36" />
</p>

Then I looked at the input file and saw the "degamma" flag under some materials and texture maps. I was not handling that flag yet, so I added degamma flag to Material and TextureMap classes and modified the parsing part as follows:

```cpp
if (materialJson.contains("_degamma")) {
    material.degamma = parseBool(materialJson["_degamma"]);
}

material.ambientReflectance = parseVec3(materialJson["AmbientReflectance"].get<string>());
material.diffuseReflectance = parseVec3(materialJson["DiffuseReflectance"].get<string>());
material.specularReflectance = parseVec3(materialJson["SpecularReflectance"].get<string>());

if (material.degamma) {
    material.ambientReflectance = srgbToLinear(material.ambientReflectance);
    material.diffuseReflectance = srgbToLinear(material.diffuseReflectance);
    material.specularReflectance = srgbToLinear(material.specularReflectance);
}

// ...
// Parsing texture maps
if (j.contains("_degamma")) tex.degamma = parseBool(j["_degamma"]);
else tex.degamma = false;
```

So if the degamma flag is true, I convert the sRGB values in the material to linear space before using them in shading calculations. Then after normalizing the texture color, I also applied degamma if needed:

```cpp
// Only degamma for LDR texture
// No degamma for normal maps or HDR textures
bool affectsColor =
    (tm->decalMode == DecalMode::ReplaceKd) ||
    (tm->decalMode == DecalMode::BlendKd)   ||
    (tm->decalMode == DecalMode::ReplaceKs) ||
    (tm->decalMode == DecalMode::ReplaceAll);

// Apply degamma if needed and texture is LDR
if (material.degamma && affectsColor && tm->image && !tm->image->isHDR) {
    texColor = Vec3(std::pow(std::max(0.0f, std::min(1.0f, texColor.x)), 2.2f),
                    std::pow(std::max(0.0f, std::min(1.0f, texColor.y)), 2.2f),
                    std::pow(std::max(0.0f, std::min(1.0f, texColor.z)), 2.2f));
}
```

But this time, I lost the textures:

<p align="center">
    <img alt="veachajarbug3" src="https://github.com/user-attachments/assets/0e0b0fdc-14af-4e8c-8000-41e43b00eb69" />
</p>


After thinking for a while, I realized that in my implementation, the order of normalization and degamma matters because both operations assume a specific value range. By changing the order, I get a better result:

<p align="center">
    <img alt="veachajarbug3" src="https://github.com/user-attachments/assets/d42d0b1c-ee79-4001-8db1-925be9abd35b" />
</p>


Even though I get better results, there are still issues and my render is still not matching with the expected result. I will investigate further and try to fix the issues in future parts.

### Outputs and Closing Thoughts
I got the expected results (also the chinese dragon for the first time :) for most scenes except some differences on; VeachAjar (even though I tried to implement degamma, I think I am missing something and the problem is related to that), teapot_roughness (I recently refactored my beer's law implementation and there are still some issues remaining, I think the difference is coming from there), audi-tt (I forgot to turn off the backface culling as announced and smooth shading not working with .ply inputs for now), and glass_sphere_env. Also, the 15th .ply file inthe  VeachAjar scene is still causing problems as in previous part (happly library gives error for unsigned int), but I still use the not fixed version and get the expected results. Some scenes like teapot_roughness and dragon_new_ply_with_spot took several hours to render, they are heavy scenes but I think I can speed up my ray tracer. I will try to refactor and optimize my ray tracer further in the next parts.

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
## teapot_roughness
This render took nearly 5.5 hours, and I still could not get the expected result (reason explained in Outputs part) :( but I wanted to place this render at the top of all other renders because of the time taken.

_Time: 14394.7 s_
<p align="center">
<img alt="teapot_roughness" src="https://github.com/user-attachments/assets/573e5709-0a80-4edb-b4af-1603fc5ae2da" />
</p>

---

### cube_directional
_Time: 1.69274 s_
<p align="center">
<img alt="cube_directional" src="https://github.com/user-attachments/assets/933a8170-8750-47c3-b877-28a35bb440a9" />
</p>

---

### cube_point
_Time: 1.75236 s_
<p align="center">
<img alt="cube_point" src="https://github.com/user-attachments/assets/335ba37e-f133-4eaf-a7f1-03f2e02756a6" />
</p>

---

### cube_point_hdr
_Time: 2.98016 s_
<p align="center">
<img alt="cube_point_hdr" src="https://github.com/user-attachments/assets/5c607afa-2b45-49ef-abf9-d05e3219f22e" />
    <br>
    <em>ACES</em>
    <br>
<img alt="cube_point_hdr" src="https://github.com/user-attachments/assets/5ceec10e-1682-42ee-a2db-16ec21ec1f00" />
    <br>
    <em>Filmic</em>
    <br>
<img alt="cube_point_hdr" src="https://github.com/user-attachments/assets/a9e8969e-dfa5-4fbd-95f7-ed616a6c86a6" />
    <br>
    <em>Photographic</em>
    <br>
</p>

---
### dragon_spot_light_msaa
_Time: 79.1459 s_
<p align="center">
<img alt="dragon_spot_light_msaa" src="https://github.com/user-attachments/assets/2ff0d40c-6cbd-480e-aa45-f5e3a681638b" />
</p>

---

### empty_environment_latlong
_Time: 1.22012 s_
<p align="center">
<img alt="empty_environment_latlong_aces" src="https://github.com/user-attachments/assets/3154f4d9-9a05-4113-a3b7-329d1cd8d31f" />
    <br>
    <em>ACES</em>
    <br>
    <img alt="empty_environment_latlong_film" src="https://github.com/user-attachments/assets/0947c597-1005-4167-9f36-fec55047ae6c" />
    <br>
    <em>Filmic</em>
    <br>
<img alt="empty_environment_latlong_phot" src="https://github.com/user-attachments/assets/871f85f0-88d2-4559-9181-fbfef980ac41" />
    <br>
    <em>Photographic</em>
    <br>
</p>

---

### empty_environment_light_probe
_Time: 1.22698 s_
<p align="center">
<img alt="empty_environment_light_probe_aces" src="https://github.com/user-attachments/assets/f96e1ad3-9f9f-4be0-bd4e-3b3254db20c5" />
        <br>
    <em>ACES</em>
    <br>
    <img alt="empty_environment_light_probe_film" src="https://github.com/user-attachments/assets/6e979142-6738-4673-b732-6059d9da1f5f" />
    <br>
    <em>Filmic</em>
    <br>
<img alt="empty_environment_light_probe_phot" src="https://github.com/user-attachments/assets/a18063e3-1a4e-4f0c-b7ed-19e5dad7d243" />
    <br>
    <em>Photographic</em>
    <br>
</p>

---

### glass_sphere_env
_Time: 2.91417 s_
<p align="center">
<img alt="glass_sphere_env_phot" src="https://github.com/user-attachments/assets/27b3b2d3-dc67-4c56-8f7b-79a33800929b" />
        <br>
    <em>Photographic</em>
    <br>
</p>

---

### head_env_light
_Time: 61.3175 s_
<p align="center">
<img alt="head_env_light_phot" src="https://github.com/user-attachments/assets/c45dd861-159a-4f8a-b084-230f242c1efe" />
            <br>
    <em>Photographic</em>
    <br>
</p>

---

### mirror_sphere_env
_Time: 1.71812 s_
<p align="center">
<img alt="mirror_sphere_env_phot" src="https://github.com/user-attachments/assets/25fb5f57-ab43-4058-b29f-6767952c48b7" />
            <br>
    <em>Photographic</em>
    <br>
</p>

---

### sphere_env_light
_Time: 1156.8 s_
<p align="center">
<img alt="sphere_env_light_phot" src="https://github.com/user-attachments/assets/cd4f7724-6694-4344-9e94-767ce5110b12" />
                <br>
    <em>Photographic</em>
    <br>
</p>

---

### sphere_point_hdr_texture
_Time: 2.99435 s_
<p align="center">
<img alt="sphere_point_hdr_texture_aces" src="https://github.com/user-attachments/assets/2b6d6aad-57cd-498f-b628-67af9491fcc8" />
            <br>
    <em>ACES</em>
    <br>
    <img alt="sphere_point_hdr_texture_film" src="https://github.com/user-attachments/assets/61c70423-3cee-40b6-a80a-62663d95455b"  />
    <br>
    <em>Filmic</em>
    <br>
<img alt="sphere_point_hdr_texture_phot" src="https://github.com/user-attachments/assets/c1e7565f-07d5-44a0-a6ac-b9a2d65a7bb3" />
    <br>
    <em>Photographic</em>
    <br>
</p>

---

### dragon_new_ply_with_spot
_Time: 15166.2* s_
<p align="center">
<img alt="dragon_new_ply_with_spot" src="https://github.com/user-attachments/assets/8072ef91-b8d5-42b9-a12d-a025ada42ba0" />
</p>

---

### audi-tt-glacier  
_Time: 7219.37* s_
<p align="center">
<img alt="audi-tt-glacier" src="https://github.com/user-attachments/assets/0b12aad6-646d-4cf3-a9a3-2e8530b4c95a" />
</p>

---

### audi-tt-pisa
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### VeachAjar
_Time: 192.583 s_
<p align="center">
<img alt="VeachAjar_phot_key_0_18_s1_2_burn_1" src="https://github.com/user-attachments/assets/4e4ef41d-0910-4520-86a4-f1ff4d932401" />
    <br>
    <em>Photographic: key 0.18, saturation 1.2, burn 1%</em>
    <br>
    <img alt="VeachAjar_phot_key_0_18_s1_0_burn_1" src="https://github.com/user-attachments/assets/ceae6d73-2329-463c-ab41-f1f13b2af47c" />
    <br>
    <em>Photographic: key 0.18, saturation 1.0, burn 1%</em>
    <br>
    <img alt="VeachAjar_phot_key_0_18_s1_0_burn_0" src="https://github.com/user-attachments/assets/c64d9f79-3728-4bfb-9505-2871cafe0e95" />
    <br>
    <em>Photographic: key 0.18, saturation 1.0, burn 0%</em>
    <br>
    <img alt="VeachAjar_phot_key_0_09_s1_0_burn_1" src="https://github.com/user-attachments/assets/d0e81a88-b191-4726-8a08-6f61179a43c9" />
    <br>
    <em>Photographic: key 0.09, saturation 1.0, burn 1%</em>
    <br>
    <img alt="VeachAjar_film_key_0_18_s1_2_burn_1" src="https://github.com/user-attachments/assets/f3297502-712d-4579-bfc4-ad85deab66a2" />
    <br>
    <em>Filmic: key 0.18, saturation 1.2, burn 1%</em>
    <br>
    <img alt="VeachAjar_film_key_0_18_s1_2_burn_0" src="https://github.com/user-attachments/assets/49002af3-0b44-494e-a0cf-e4ff6a72707d" />
    <br>
    <em>Filmic: key 0.18, saturation 1.2, burn 0%</em>
    <br>
    <img alt="VeachAjar_aces_key_0_18_s1_2_burn_1" src="https://github.com/user-attachments/assets/ad686ba9-c77a-4f02-95c0-0afc31e2adf0" />
    <br>
    <em>ACES: key 0.18, saturation 1.2, burn 1%</em>
    <br>
    <img alt="VeachAjar_aces_key_0_18_s1_2_burn_0" src="https://github.com/user-attachments/assets/abb036bd-382f-4020-aa87-225369f7511a" />
        <br>
    <em>ACES: key 0.18, saturation 1.2, burn 0%</em>
    <br>
</p>

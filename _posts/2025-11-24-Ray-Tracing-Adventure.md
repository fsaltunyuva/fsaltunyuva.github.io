---
layout: post
title: "Ray Tracing Adventure Part 3 (Homework 3)"
date: 2025-11-24
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 3, focusing on implementing features:

- **Multisampling**
- **Depth of Field**
- **Area Lights**
- **Motion Blur**
- **Material Roughness**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz at Middle East Technical University.

### Bugs on Part 2 
To start with the chinese dragon model, as I mentioned in my previous post, the problem was the very small coordinates of the model. Therefore, I thought that multisampling would help. However, after implementing multisampling (and trying 100 samples), I could not see any difference. I will continue to investigate this issue in the future using higher samples or different techniques.

<p align="center">
<img alt="chinese dragon bug" src="https://github.com/user-attachments/assets/8eb29623-5e2f-463d-8507-d6b27c3c4704" />
</p>

Also, as I mentioned in first part, this project also contributes my C++ skills so I want to mention a small (but surprisingly effective) optimization I did in the intersection code.

I was initializing Intersector for both isInShadow and traceRay functions separately. I knew that this was not optimal but I thought the performance gain would be negligible.

```cpp
static bool isInShadow(const Scene& scene, const Vec3& shadowRayOrigin, const Vec3& shadowRayDir, float lightDistance) {
    Intersector intersector(scene); 

    IntersectionInfo info = intersector.findClosestIntersection(shadowRayOrigin, shadowRayDir, true);
    //...
}

Vec3 traceRay(const Scene& scene, const Vec3& rayOrigin, const Vec3& rayDir, int depth) {
    Intersector intersector(scene); 

    IntersectionInfo info = intersector.findClosestIntersection(rayOrigin, rayDir, false);
    //...
}
```

Instead of this, I created Intersector once in the main rendering loop and passed it as a reference to these functions. Of course I should also consider the thread-safety when doing this in a parallel program, so I created one Intersector instance per thread.

```cpp
#pragma omp parallel
{
  Intersector threadIntersector(scene); // Thread-local intersector for parallel safety

#pragma omp for schedule(dynamic)
  for (int y = 0; y < height; ++y) {
      for (int x = 0; x < width; ++x) {
        // ...
```

Even though I did not expect a big performance gain, rendering time decreased from 80.6 seconds to 55.5 seconds in chinese_dragon scene. So, this became a nice example for me showing how much using references can affect performance :).

### Multisampling
Multisampling is a technique used to reduce aliasing artifacts in computer graphics. It works by taking multiple samples per pixel and averaging the results to produce a smoother final image. In ray tracing, we can implement multisampling by sending multiple rays per pixel, each with a slightly different direction. This helps to capture more detail and reduces jagged edges. For my implementation, I used a simple jittered grid sampling method, where I divided each pixel into a grid and randomly sampled within each grid cell. 

Here are some results with different sample sizes (36 vs 100 samples per pixel) on the scene metal_glass_plates, noise reduce can be clearly seen (even though GIF itself has some compression artifacts):

![multisampling GIF](https://github.com/user-attachments/assets/3b5d5781-e24a-44a8-a3d1-e98add70bd13)

For each grid cell, I generated jittered offsets. Shortly, I divided the pixel into SxS grid, then for each cell (sx, sy), I generated random offsets using a uniform distribution in the range [0, 1). Finally, I calculated the jittered coordinates (jx, jy) as follows:

```cpp
float jx = (sx + dist(rng)) / S;
float jy = (sy + dist(rng)) / S;
```

As we discussed in the class, I shuffled the samples to avoid correlation between samples.

```cpp
shuffle(jitterSamples.begin(), jitterSamples.end(), rng);
```

### Depth of Field
Depth of Field is a technique used to simulate the effect of a camera lens focusing on a specific distance, causing objects closer or farther away to appear blurred. In ray tracing, we can implement this by simulating a lens aperture and generating rays that originate from different points on the lens.

The input scene gives us two values "ApertureSize" for the size of the square lens and "FocusDistance" for the distance from the camera where the image should be in focus. In pinhole rendering, all primary rays originate from a single point.
With depth of field, rays originate from random points on the aperture, and pass through a focal point.

I generated two random numbers and mapped them to square lens and calculated the lens point as follows:

```cpp
float e1 = dist(rng);
float e2 = dist(rng);

float lensX = (e1 - 0.5f) * cam.apertureSize;
float lensY = (e2 - 0.5f) * cam.apertureSize;

//...

Vec3 lensPoint = camPos + u_vec * lensX + v_vec * lensY;
```

The primary ray does not start at camPos anymore. It starts from the sampled aperture point and goes through the focal point:

```cpp
rayOrigin = lensPoint;
rayDir = (p_focal - lensPoint).normalize();
```

Which matches the final ray equation, $r(t)=a+t(p−a)$.

<p align="center">
<img alt="spheres_dof" src="https://github.com/user-attachments/assets/f9628780-ce81-4321-8801-ed3c2c1941d9" />
</p>

### Area Lights
Area lights are light sources that have a defined shape and size, as opposed to point lights which emit light from a single point. Area lights produce softer shadows and more realistic lighting effects. In input files, area lights are defined by their position, normal, size, and radiance. To simulate area lights, instead of evaluating a single lighting direction, we must sample many random points on the square surface.

To sample a square area light, I first build an orthonormal basis (u,v) on the light’s surface using its normal:

```cpp
static void buildAreaLightBasis(const Vec3& nL, Vec3& u, Vec3& v) {
    Vec3 n = nL.normalize(); // Ensure normal is normalized
    Vec3 tmp = (fabs(n.x) > 0.9f ? Vec3(0,1,0) : Vec3(1,0,0)); // Choose a helper vector not parallel to n to avoid degenerate cross product
    u = tmp.cross(n).normalize(); // First basis vector
    v = n.cross(u).normalize(); // Second basis vector
}
```

Then I generate random offsets inside the square [−s/2, s/2]:

```cpp
float half = light.size * 0.5f; // Half size of the square light

// Random offsets in the range [-half, half]
float sx = (dist(rng) * 2.0f - 1.0f) * half;
float sy = (dist(rng) * 2.0f - 1.0f) * half;

Vec3 lightPoint = light.position + u * sx + v * sy; // Sampled point on the area light
```

Even though radiance itself does not decrease with distance, the received energy still depends on solid angle. Formula for the geometry term is as follows:

$$E(x)=L\frac{area⋅cosα​}{d^{2}}dω$$

where α is the angle between the light normal and the direction toward the shading point and d is the distance between them.

```cpp
float cosLight = std::max(0.0f, light.normal.normalize().dot(-wi));
float geom = (A * cosLight) / (d2 * numSamples);
Vec3 eff = light.radiance * geom;
```

After implementing area lights, I tested with chessboard_arealight and wine_glass scenes, but results were not as expected. 

<p align="center">
<img alt="chessboard_arealight" src="https://github.com/user-attachments/assets/409b3921-ddfa-4173-b9f7-a25d4f262355" />
</p>

<p align="center">
<img  alt="wine_glass" src="https://github.com/user-attachments/assets/90a8e170-4106-4b5c-add2-2ce41ffe2337" />
</p>

Firstly, I tried to change the number of samples for the area light but nothing changed. This indicated that the issue was probably not related to sampling, but to the lighting logic itself.

Then I noticed the following lines inside the area-light shading loop:

```cpp
// ...
float cosLight = std::max(0.0f, light.normal.normalize().dot(wi.scale(-1.0f)));

if (cosLight <= 0.0f) continue;
// ...
```

That means, if the angle between the light normal and the direction toward the shading point is negative, skip this sample. At first glance, this seems correct because light sources are usually one-sided. However, when I checked the input json file for chessboard_arealight, I saw that the area light normal was actually pointing away from the scene:

```
"AreaLight": {
  "_id": "1",
  "Position": "3.263 -9.2106 5.90386",
  "Normal": "0.591 -0.010 0.806",
  "Size": "1",
  "Radiance": "10000 10000 10000"
}
```

Light is above the scene but normal has a positive z component, meaning it points upwards, away from the scene. Therefore, all samples were being skipped because dot(light.normal, -wi) becomes negative almost everywhere.

To check this , I rewrote cosLight calculation as follows:

```cpp
float cosLight = std::max(0.0f, -light.normal.normalize().dot(wi.scale(-1.0f)));
```

It fixed the issue and produced correct lighting results, but it broke the area light in cornellbox_area.

<p align="center">
<img alt="cornellbox_area_bugged" src="https://github.com/user-attachments/assets/edc16074-4160-4f15-9ada-51ac9439bcd3" />
</p>

Before this issue, cornellbox_area area light was working correctly but the white area light surface on upper plane was not visible. After this change, the area light surface became visible but the lighting effect was incorrect. So I realized that I should treat the light as double-sided ignoring the normal direction.

```cpp
float cosLight = std::max(0.0f, std::fabs(light.normal.normalize().dot(wi.scale(-1.0f)))); // Use absolute value to treat light as double-sided

if (cosLight <= 0.0f) continue;
```

This change produced correct results (can be found in Outputs section) in both scenes.

### Motion Blur

Motion blur is a technique used to simulate the effect of objects moving during the exposure time of a camera. In a ray tracer, this is achieved by letting each ray carry a random time parameter t ∈ [0,1]. My tracer will be assume only translational movements for simplicity.

When shooting primary rays, I generate a random time value per ray:

```cpp
float rayTime = dist(rng); // random in [0,1]
```

I added the time parameter to the necessary functions:

```cpp
Vec3 traceRay(const Scene& scene, const Vec3& rayOrigin, const Vec3& rayDir, int depth, Intersector& intersector, float rayTime);

static bool isInShadow(const Scene& scene, const Vec3& shadowRayOrigin, const Vec3& shadowRayDir, float lightDistance, Intersector& intersector, float rayTime);

IntersectionInfo Intersector::findClosestIntersection(const Vec3& rayOrigin,const Vec3& rayDir, bool isShadowRay, float rayTime);
```

I basically created a blur offset for each moving object based on its velocity and the ray time and subtracted it from the original ray position as can be seen in plane example below:

```cpp
// Motion blur
Vec3 blurOffset = plane.motionBlur.scale(rayTime);
Vec3 rayOriginBlur = rayOrigin.subtract(blurOffset);

if (RayPlane(rayOriginBlur, rayDir, Vec3(p0_world), Vec3(N_world), t))
  // ...
```

<p align="center">
<img alt="cornellbox_boxes_dynamic" src="https://github.com/user-attachments/assets/2e6568f8-9bdf-4c47-aede-f45b3c8459bc" />
</p>

### Material Roughness
Material roughness defines the roughness of the mirrors, conductors, and dielectrics. A rough surface causes incoming rays to scatter in many directions, resulting in blurry reflections. 

Given the ideal direction idealDir, I apply a roughness offset using:

```cpp
static Vec3 perturbDirection(const Vec3& idealDir, float roughness) {
    if (roughness <= 0.0f) // No perturbation for perfectly smooth surfaces
        return idealDir;

    float r = std::min(roughness, 1.0f);
    // float r = roughness;

    Vec3 randVec = randomInUnitSphere();
    Vec3 perturbed = idealDir.add(randVec.scale(r));

    return perturbed.normalize();
}
```

The line `float r = std::min(roughness, 1.0f);` ensures that the roughness value does not exceed 1.0, which would produce preventing the perturbed reflection direction from deviating too far away from the ideal one. Without this clamp, extremely large roughness values could push the random offset to dominate the ideal direction completely. This would cause the resulting vector to lose its correlation with the surface reflection direction and produce unrealistic results.

Here is the difference in the scene metal_glass_plates with `float r = std::min(roughness, 1.0f);` instead of `float r = roughness;`:

<p align="center">
<img alt="roughness difference" src="https://github.com/user-attachments/assets/83487702-8a9a-4ce3-95d1-e6fd9abfe390" />
</p>

In the shading part, I used this function to perturb the reflection and refraction directions:

```cpp
Vec3 idealReflectDir = reflect(rayDir, hitNormal).normalize();
Vec3 reflectDir = perturbDirection(idealReflectDir, material.roughness);
```

Some results with different roughness values for the scene cornellbox_brushed_metal (left is 5 roughness, right is 15):

<p align="center">
<img alt="roughness5left15right" src="https://github.com/user-attachments/assets/db9241d9-08c5-495f-9a61-5ed17ece92f8" />
</p>

And difference between roughness 5 and 15:

<p align="center">
<img alt="roughnes5left diff" src="https://github.com/user-attachments/assets/cb850832-3708-4c97-9d55-3d921f85cb37" />
</p>

### Outputs
Chinese dragon still produces wrong results (which is strange with multisampling, it should have improved a little bit), therefore focusing_dragons scene is also affected. Also I encountered with a strange black screen render issue on cornellbox_boxes_dynamic scene. It is fixed when I turned off the BVH acceleration structure. I suspect there is a bug in my BVH construction code that causes this issue. To investigate this issue later on, I added a line that uses BVH on larger models only (more than 300 triangles), then I was rendering the other scenes, I realized that mirror reflection from the scene metal_glass_plates changed. I again tested with and without BVH, and confirmed that with BVH, reflections were correct but without BVH, reflections were wrong. Then I rechecked my brute force mesh intersection code and found a bug in the reflection calculation.

Left reflection is the correct one, right reflection is wrong.

<p align="center">
<img alt="left correct-right wrong" src="https://github.com/user-attachments/assets/92474478-8a90-4cab-9077-bb1c2146992a" />
</p>

```cpp
if (RayTriangle(rayOriginBlur, rayDir, v0, v1, v2, t, performCulling, triN)) {
    if (t > intersectionTestEpsilon && t < closestT) {
        // Updating closest intersection
        // ...

        // normal transformation (inverse-transpose of M)
        glm::vec3 nWorld = glm::normalize(glm::vec3(normalM * glm::vec4(triN.x, triN.y, triN.z, 0.0f)));
        info.hitNormal = Vec3(nWorld);

        if (info.hitNormal.dot(rayDir) > 0)
            info.hitNormal = info.hitNormal.scale(-1.0f);
```

Here, I was transforming the normal using the normalM matrix, which is the transpose of the inverse of the model matrix. However, triN is already computed in world space (because the triangle vertices are transformed with M before calling RayTriangle), so applying normalM again transforms the normal twice. After removing this extra transformation, the reflections became correct in both cases:

```cpp
if (RayTriangle(rayOriginBlur, rayDir, v0, v1, v2, t, performCulling, triN)) {
    if (t > intersectionTestEpsilon && t < closestT) {
        // Updating closest intersection
        // ...

        info.hitNormal = triN.normalize();

        if (info.hitNormal.dot(rayDir) > 0)
            info.hitNormal = info.hitNormal.scale(-1.0f);
```

Also, I am getting more reflective results than expected results since the first part (as can be seen in chessboard_arealight, there is no reflection in the original image) as in the case of some of my friends, I am suspecting that I am calculating an additional reflection term somewhere in the code, I am investigating this issue as well.

As in previous parts, I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance, and Ramazan Tokay for contributions to the 3D models.

You can see the rendering times of this part below:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| cornellbox_area                 | 260.282 s.  |
| cornellbox_boxes_dynamic (500x400 resolution to speed up)                 | 1691.55 (without BVH bc. of the bug)  |
| cornellbox_brushed_metal                 | 425.659  |
| dragon_dynamic                 | 1322.23  |
| focusing_dragons                 | 169.435  |
| metal_glass_plates                 | 206.233 s.  |
| spheres_dof                 | 129.346  |
| chessboard_arealight                 | 441.501  |
| chessboard_arealight_dof                 | 458.703  |
| chessboard_arealight_dof_glass_queen                 | 948.822  |
| deadmau5                 | 1187.27  |
| wine_glass                 | 7955.46*  |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

**Used CPU: AMD Ryzen 5 7640HS 6-Core Processor (4.30 GHz)*

Also, I merged tap water input json files' render into a .mp4 video (using [FFmpeg](https://www.ffmpeg.org/)). I downscaled the resolution to 500x500 for faster rendering. Each frame nearly took 16 seconds to render on Ryzen 5 7640HS 4.30 GHz.

---

### Tap Water
<video width="100%" controls autoplay muted loop playsinline>
  <source src="{{ site.baseurl }}/videos/tapwater.mp4" type="video/mp4">
</video>

---
### cornellbox_area
_Time: 41.2385 s_
<p align="center">
<img alt="cornellbox_area" src="https://github.com/user-attachments/assets/f0e86fb8-fe4d-453d-86b6-7fcff19b24e1" />
</p>

---
### cornellbox_boxes_dynamic
_Time: 41.2385 s_
<p align="center">
<img alt="cornellbox_boxes_dynamic" src="https://github.com/user-attachments/assets/2393c821-a141-4bea-98d3-0b149fa138f2" />
</p>

---
### cornellbox_brushed_metal
_Time: 41.2385 s_
<p align="center">
<img  alt="cornellbox_brushed_metal" src="https://github.com/user-attachments/assets/e4939c3c-f663-47bc-a193-7bd0204a8a81" />
</p>

---
### dragon_dynamic
_Time: 41.2385 s_
<p align="center">
<img alt="dragon_dynamic" src="https://github.com/user-attachments/assets/f760b6c5-5ffc-4db9-9ccf-bd361f69e753" />
</p>

---
### focusing_dragons
_Time: 41.2385 s_
<p align="center">
<img alt="focusing_dragons" src="https://github.com/user-attachments/assets/8eaef1f8-4a64-4ae3-b7e1-c085a7ec4207" />
</p>

---
### metal_glass_plates
_Time: 41.2385 s_
<p align="center">
<img alt="metal_glass_plates" src="https://github.com/user-attachments/assets/4040962f-2807-4a42-8cca-ed328a94e5da" />
</p>
---
### spheres_dof
_Time: 41.2385 s_
<p align="center">
<img alt="spheres_dof" src="https://github.com/user-attachments/assets/65f0c527-3749-43cd-9cd0-cd05e5b65a62" />
</p>

---
### chessboard_arealight
_Time: 41.2385 s_
<p align="center">
<img alt="chessboard_arealight" src="https://github.com/user-attachments/assets/aa246bdf-9015-4148-bb05-14717e6f5274" />
</p>

---
### chessboard_arealight_dof
_Time: 41.2385 s_
<p align="center">
<img alt="chessboard_arealight_dof" src="https://github.com/user-attachments/assets/191f2b5f-17c6-4173-ab9e-1f36f8890849" />
</p>

---
### chessboard_arealight_dof_glass_queen
_Time: 41.2385 s_
<p align="center">
<img alt="chessboard_arealight_dof_glass_queen" src="https://github.com/user-attachments/assets/5f229211-ecad-4c07-8a4c-a4cbf52694e9" />
</p>

---

### deadmau5
_Time: 41.2385 s_
<p align="center">
<img alt="deadmau5" src="https://github.com/user-attachments/assets/4d22cf6b-778b-4bb1-ad56-4181530819cf" />
</p>

---

### wine_glass
_Time: 41.2385 s_
<p align="center">
 <img alt="wine_glass" src="https://github.com/user-attachments/assets/37fa4a15-0147-43a5-971a-636d26f947f1" />
</p>

---

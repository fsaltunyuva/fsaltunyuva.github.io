---
layout: post
title: "Ray Tracing Adventure Part 6 (Homework 6)"
date: 2026-01-17
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 6, focusing on implementing features:

- **Bidirectional Reflectance Distribution Function (BRDF)**
- **Object Lights**
- **Path Tracing**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz, at Middle East Technical University.

### Bugs from Previous Parts 
Before starting this part, I decided to optimize my ray tracer because I had been getting progressively slower results after each homework part, and also because I saw in the homework file that the VeachAJar scene could reach rendering times of around 36 hours. While making additions to my code in previous parts, I had been taking notes on unnecessary extra operations I noticed in some places or sections that could be handled more efficiently, but I was afraid to refactor, after all, the unwritten rule of engineering is 'if it ain't broke, don't fix it' :). However, the most suitable time to try these optimizations was before starting this homework because I knew how costly the path tracing process would be, and I was determined to get those beautiful path tracing renders that would crown all the results we achieved since the beginning of the semester in time for the homework deadline. Anyway, as I mentioned, I started first with the codes I had previously written and marked with notes like ```// TODO: Not used?``` and ```// TODO: Slows the process?```. With each change I made, I tested various input files from previous parts to check whether I was getting the same results. Although I managed to reduce my renders from around 15 seconds to 12-13 seconds as a result of these changes, what really sped up my ray tracer were the compiler optimization techniques that I thought I had already implemented and configured correctly. Almost none of the optimization methods that I thought I had configured correctly since practically the first homework were actually working. After realizing this, I first felt sad about the time I had wasted waiting unnecessarily in past parts, but then I was glad that I had at least solved this problem before this homework's renders :) This way, I managed to reduce my rendering times by almost half.

### Bidirectional Reflectance Distribution Function (BRDF)
Up to this point, my ray tracer was able to trace rays correctly and compute intersections, but the actual appearance of surfaces was still limited. Every surface responded to light in a very basic way, without considering the complex interactions between light and material properties. To address this, I implemented a Bidirectional Reflectance Distribution Function (BRDF) system. 

A BRDF defines how light is reflected at a surface point, given two directions, the incoming light direction (wi) and the outgoing view direction (wo). More precisely, a BRDF describes how much of the incoming radiance from a given direction is scattered toward another direction. This abstraction is powerful because it separates geometry, lighting, and material behavior into cleanly defined components.

<p align="center">
    <img width="40%" alt="brdf-wiki" src="https://github.com/user-attachments/assets/6397a03d-af95-4e2f-9435-8619bf3dd6ff" />
</p>

As stated in the homework file the possible BRDF models to implement were:
- Original Blinn-Phong
- Original Phong
- Modified Blinn-Phong
- Modified Phong
- Torrance Sparrow

and BRDF field in JSON files included the options ```_normalized``` to indicate whether the BRDF should be normalized or not and ```Exponent``` to define the shininess of the surface. Also, Torrance Sparrow model required an additional parameter ```_kdfresnel``` to define the Fresnel reflectance at normal incidence, I will explain this parameter in more detail below.

As in previous parts, I implemented BRDF models as common interface because they all input the same parameters (wi, wo, normal) and output the reflectance value. The shading logic does not need to know which BRDF is being used, it simply calls the BRDF’s evaluation function. In this way, making changes to the BRDF models or adding new ones becomes straightforward without affecting other parts of the code.

Before starting the implementation of the BRDF models, I normalized all direction vectors (wi, wo, normal) at the beginning of each BRDF evaluation function to ensure consistent calculations (it became muscle memory at this point :)). Then I caculated cosine terms between the surface normal and the incoming and outgoing directions, which are essential for determining how much light is reflected based on the angle of incidence and reflection.

```cpp
Vec3 n = nRaw.normalize();
Vec3 wi = wiRaw.normalize();
Vec3 wo = woRaw.normalize();

float cosI = n.dot(wi);
float cosO = n.dot(wo);
```

For all BRDF models, the diffuse component is initalized as Lambertian term:

```cpp
Vec3 diffuse = kd.scale(1.0f / PI);
```

In Torrance Sparrow model, it will be modified based on the Fresnel reflectance parameter.

Depending on the BRDF type, the specular component uses either the reflection vector or the half vector. To support both Phong and Blinn-Phong models, helper functions are defined as follows:

Phong uses the cosine between the reflection direction and the view direction (cosAlphaR):

```cpp
Vec3 r = reflect(wi.scale(-1.0f), n).normalize();
float cosAlphaR = clamp01(r.dot(wo));
```

Where reflect is a helper function that computes the reflection direction of an incoming vector about a normal.

Blinn-Phong uses the cosine between the surface normal and the half vector (cosAlphaH):

```cpp
Vec3 wh = wi.add(wo).normalize();
float cosAlphaH = clamp01(wh.dot(n));
```


#### Original Blinn-Phong
The Original Blinn-Phong BRDF corresponds to the classic shading model commonly used in computer graphics. Instead of using the perfect reflection direction, it computes specular reflection based on the half vector between the incoming light direction and the view direction. It is defined as:

<p align="center">
    <img alt="original-blinn-phong" src="https://github.com/user-attachments/assets/5b4eac2e-c341-41f1-9a22-bfb2907abf19" />
</p>

```cpp
float c = pow(cosAlphaH(), exponent);
specScalar = c / cosI;
```

#### Original Phong
The Original Phong BRDF is similar but it computes the specular term using the reflection direction of the incoming light instead of the half vector. The highlight intensity depends on the alignment between this reflection direction and the view direction.

<p align="center">
    <img alt="original-phong" src="https://github.com/user-attachments/assets/d0cd0a92-8f96-4f47-bdfd-1619e6924633" />
</p>

```cpp
float c = pow(cosAlphaR(), exponent);
specScalar = c / cosI;
```

#### Modified Phong and Modified Blinn-Phong
The Modified Phong and Modified Blinn-Phong BRDFs extend their originals by optionally applying normalization. When the ```_normalized``` flag is enabled, a normalization factor derived from the exponent is applied to the specular term. This ensures that the total reflected energy does not exceed the incoming energy, preventing materials from becoming unrealistically bright as the exponent increases.

Modified Phong:

<p align="center">
    <img alt="modified-phong" src="https://github.com/user-attachments/assets/c35aeb1b-4d98-4fe8-8885-24b2bf95dd4f" />
</p>

Modified Blinn-Phong:

<p align="center">
    <img alt="modified-phong" src="https://github.com/user-attachments/assets/fa7658d9-5fe5-45ee-abbb-7733a0f62e11" />
</p>

```cpp
case BRDFType::ModifiedPhong: {
    Vec3 diffuse = kd.scale(INV_PI); // 1 / pi for lambertian

    if (normalized) {
        float c = pow(cosAlphaR(), exponent);
        specScalar = ((exponent + 2.0f) / (2.0f * PI)) * c; // normalized as in the formula
    }
    else {
        float c = pow(cosAlphaR(), exponent);
        specScalar = c;
    }
    break;
}

case BRDFType::ModifiedBlinnPhong: {
    Vec3 diffuse = kd.scale(INV_PI); // 1 / pi for lambertian

    if (normalized) {
        float c = pow(cosAlphaH(), exponent); // brdf.pdf uses cosAlphaR but I think it is a typo
        specScalar = ((exponent + 8.0f) / (8.0f * PI)) * c; // normalized as in the formula
    }
    else {
        float c = pow(cosAlphaH(), exponent);
        specScalar = c;
    }
    break;
}
```

#### Torrance-Sparrow
The Torrance Sparrow BRDF models surface reflection using a microfacets and is normalized by definition. In this model, the ```Exponent``` parameter corresponds to the p term in the microfacet distribution function, controlling the roughness of the surface.

<p align="center">
    <img alt="torrance-sparrow" src="https://github.com/user-attachments/assets/76e367e7-9f53-4f37-9c46-9b2b5af786be" />
</p>

The ```kdfresnel``` parameter defines the Fresnel reflectance at normal incidence. When enabled, the diffuse component is scaled by:

<p align="center">
    <img width="30%" alt="torrance-sparrow2" src="https://github.com/user-attachments/assets/bf335eed-e2cd-4be8-9088-86d8a11c00d3" />
</p>

instead of the usual ```kd / π``` term. This adjustment accounts for the fact that some portion of the incoming light is reflected at the surface interface due to Fresnel effects, reducing the amount of light available for diffuse reflection.

<p align="center">
    <img alt="figure" src="https://github.com/user-attachments/assets/49613b6e-11ca-4a40-8c93-f6f2fc53c700" />
    <br>
    <em>Figure 3: Configuration for deriving the normalizing factor of the micro-facet distribution function from BRDF Summary prepared by Professor Ahmet Oğuz Akyüz.</em>
</p>

I followed the steps outlined in the lecture notes to implement the Torrance-Sparrow BRDF as follows:

1. *Compute the half vector (wh) between the incoming (wi) and outgoing (wo).*
2. *Compute the angle α as wh • n.*
3. *Compute the probability of this α using D(α) function (Blinn's distrubiton in our case).*

    <p align="center">
        <img width="20%" alt="D(α)" src="https://github.com/user-attachments/assets/50194788-6e5a-4e50-ab40-31ed013d4004" />
    </p>
    
    ```cpp
    diffuse = kd.scale(1.0f / PI); // Inverse pi for lambertian
    
    Vec3 wh = wi.add(wo).normalize();
    float nDotWh = clamp01(n.dot(wh));
    float D = ((exponent + 2.0f) / (2.0f * PI)) * pow(nDotWh, exponent);
    ```

4. *Compute the geometry term G(wi, wo).*

    <p align="center">
        <img width="80%" alt="G(wi, wo)" src="https://github.com/user-attachments/assets/66af4f3a-a9ee-44a5-a03c-511d2e1ef227" />
    </p>
    
    ```cpp
    float denomG = max(epsilon, woDotWh); // Prevent division by zero
    float g1 = (2.0f * nDotWh * cosO_clamped) / denomG;
    float g2 = (2.0f * nDotWh * cosI_clamped) / denomG;
    float G = min(1.0f, min(g1, g2));
    G = max(0.0f, G);
    ```

5. *Compute the Fresnel reflectance using Shlick's approximation.*

    <p align="center">
        <img width="45%" alt="F" src="https://github.com/user-attachments/assets/6d459bec-e693-463e-87ff-a7bbbae807ae" />
    </p>

#### Smooth Shading Bug Fix
After implementing BRDF, I tried killeroo_torrancesparrow scene, but I get the following render, where there are black artifacts on the object:

<p align="center">
    <img width="50%" alt="buggedkilleroo" src="https://github.com/user-attachments/assets/4104e207-f558-4b62-b473-be67b013fa37" />
</p>

I was confused at first because I thought my BRDF implementation had some bugs, but after double checking everything, I realized that the problem was also in my HW4 render too. Then I realized that there is a bug in my smooth shading implementation, it was caused by computing barycentric coordinates in world space while using mesh local vertex positions and by indexing per vertex normals with global vertex indices. The problem was fixed by transforming the hit point back into local space before computing barycentric coordinates.

<p align="center">
    <img width="50%" alt="fixedkilleroo" src="https://github.com/user-attachments/assets/2880d6e9-12c5-4c99-82c3-cf3a24bd1f29" />
</p>

#### Comparison

Here you can see some comparisons between different BRDF models implemented in killeroo_closeup_phot scene:

<p align="center">
    <img alt="brdf_comparison" src="https://github.com/user-attachments/assets/ae966782-d2ba-4a01-8489-c53e28246f7c" />
</p>

Also, you can see the render times for each BRDF model in the table below:

| BRDF Model               | Time (seconds) |
| ----------------------- | --------------- |
| Original Blinn-Phong    | 91.3994        |
| Original Phong          | 97.8805        |
| Modified Blinn-Phong    | 105.885         |
| Modified Phong          | 99.3751         |
| Torrance-Sparrow        | 90.5573         |

### Object Lights
Object lights will allow any geometric object to act as a light source by assigning it a radiance value. With this extension, regular scene objects such as spheres or meshes can directly emit light. The only difference than the previous light implementations is that it carries an additional radiance attribute.

<p align="center">
    <img width="50%" alt="fixedkilleroo" src="https://github.com/user-attachments/assets/7b5c232e-694d-403a-8875-b4cf25741704" />
</p>

For example, for Light Sphere, it means that whenever a ray intersects this sphere, it is also interacting with a light emitting surface. The sphere still participates in intersection tests just like any other object. Same applies to light meshes. Therefore, I modified my Intersector with the new Light Object types. Only difference than their original versions is these additional lines:

```cpp
intersectionInfo.isEmissive = true;
intersectionInfo.emission = lightObject.radiance;
```

In my ray tracing loop, I immediately return the emission when the closest hit is emissive because light sources contribute directly to the radiance along the ray without further bounces.

```cpp
if (info.isEmissive) {
    return info.emission;
}
```

But in path tracing loop, I added emission only when it should contribute:

```cpp
if (info.isEmissive) {
    if (lastSpecular || !cam.pathTracingOptions.nextEventEstimation) {
        L = L.add(beta.multiply(info.emission)); // Add emission from light source
    }
    break;
}
```

If Next Event Estimation (NEE) is disabled we can add the emission directly because we are not sampling lights separately. However, if NEE is enabled, we only add the emission if the last bounce was specular. This prevents double counting light contributions from light sources when we are already sampling them explicitly.

I will explain path tracing and next event estimation in more detail below in the Path Tracing section.

#### Additional Light Fix
After implementing object lights, I tried cornellbox_sphere_light scene but I got the following render:

<p align="center">
    <img alt="shadowbug" src="https://github.com/user-attachments/assets/335a8157-d903-488f-8586-5f06d3e5ca64" />
</p>

There is an additional light on the ceiling that should not be there. Same issue was also occured in HW3 where I was trying to implement area lights. The issue was caused by double sided lighting. My code was using ```fabs``` for the cosine calculation, which caused the light mesh to emit light both downwards (into the room) and upwards (onto the ceiling).

To fix this, I removed the absolute value to make the light source one sided, ensuring it only emits light in the direction of the normal (downwards):

```cpp
// float cosL = max(0.0f, fabs(nTri.dot(wi.scale(-1.0f))));
float cosL = max(0.0f, nTri.dot(wi.scale(-1.0f)));
```

Additionally, I enabled back-face culling for light meshes in the intersector. This ensures that when rays hit the back of the light (the side facing the ceiling), they pass through it instead of being blocked, preventing a black square artifact on the ceiling.

### Path Tracing
Here we come to the most satisfying renders of the entire ray tracing adventure. Up to this point, my renderer was able to compute direct illumination correctly using ray tracing and explicit light sampling. However, this approach alone cannot capture important global illumination effects such as indirect lighting or soft interreflections. To address this limitations, we will implement a path tracer.

Before taking this course and computer graphics course, I only heard about path tracing in the game Cyberpunk 2077, and most of the videos were showing how realistic reflections and lighting effects could be achieved with path tracing ([This video by MxBenchmarkPC](https://www.youtube.com/watch?v=7NQqWlYZ2RA) is one of them). However, when I saw these comparisons between ray tracing and path tracing in the game, I thought that, "Wait a minute, isn't ray tracing should be doing all these realistic lighting effects already? Why do we need path tracing then?". Also, there were still some discussions about whether ray tracing should be a thing for video games, with this kind of performance cost, so is there a future for path tracing in games?

So, this homework part was a great opportunity for me to understand the differences between ray tracing and path tracing, the performance costs and benefits of each method, and why path tracing is considered the gold standard for realistic rendering.

To describe the idea simply, I will use the example from the video ["How Path Tracing Makes Computer Graphics Look Awesome" by Computerphile](https://www.youtube.com/watch?v=3OKj0SQ_UTw) (they also have videos about ray tracing for anyone interested). As described in the video, let's say that we have a corridor like this:

<p align="center">
    <img alt="pathtracing1" src="https://github.com/user-attachments/assets/165c21a8-b818-494d-964f-9cfe48dd70d6" />
</p>

In ray tracing, let's say we shoot a ray as in above, then we cast another ray towards the light source to see if it is visible from the intersection point, and it is occluded by the wall, so we get no contribution from the light source. However, in real life, the light from the light source would bounce off the walls and illuminate the corridor indirectly, as in this Blender render from the video:

<p align="center">
    <img alt="pathtracing2" src="https://github.com/user-attachments/assets/4919eff6-9ecc-45f0-b3bb-eb580f793efe" />
</p>

To capture these indirect lighting effects, we need to trace additional rays that bounce around the scene, gathering light contributions from multiple bounces. This is where path tracing comes in. In path tracing, instead of just casting a single shadow ray to each light source, we recursively trace rays that bounce off surfaces, simulating the complex interactions of light in the scene.

<p align="center">
    <img alt="pathtracing3" src="https://github.com/user-attachments/assets/d74eae1b-6f21-46bb-925f-5649cbd9a590" />
</p>

Let's say we shoot five rays from the intersection point, we also calculate their contributions as shown in the image above, then we average these contributions (this is the main idea of Monte Carlo Integration) to get the final color for that pixel (in a real path tracer, we also do the same process for new rays). This way, we can capture both direct illumination from light sources and indirect illumination from light bouncing off other surfaces. And as we get further away from the light source, we can see that the indirect illumination becomes less intense, just like in real life. (Here you can see that light intensity is lowered to 20%)

<p align="center">
    <img alt="pathtracing4" src="https://github.com/user-attachments/assets/10dade22-4e19-456a-b780-091486b8d995" />
</p>

But how to choose where the rays go? We randomly choose rays around the hemisphere above the intersection point, weighted by the BRDF of the surface. This way, we are more likely to sample directions that contribute more light based on the material properties.

To implement path tracing in my ray tracer, I also added tracePath function in addition to the existing traceRay function. It maintains 2 important variables, *pathThroughput* which accumulates BRDF values, cosine terms, and probability densities and *L* the accumulated radiance returned by the path. Each bounce updates pathThroughput and any emmited or directly sampled light contributions are added to L.

Basically, the main loop is an iterative loop that continues until a maximum depth is reached or the path terminates via Russian Roulette. At each iteration, we perform the following steps:

1. Check for emissive hits
2. Optionally perform Next Event Estimation (NEE)
3. Sample a new direction
4. Update the throughput

Before going to details, it is useful to briefly clarify how light sources are sampled in the renderer. Since the same idea already discussed in a previous blog posts, the same sampling strategies are reused here. The only extension in this homework is that light spheres and light meshes are treated as additional area emitters. From the perspective of the path tracer, all light sources are handled uniformly. A single emitter is selected at random, a point is sampled on its surface, and the corresponding probability density is converted from area measure to solid angle measure. The resulting direction, emitted radiance, and PDF are then used by the path tracing estimator.

After a valid surface intersection, material properties are evaluated at the hit point. The implementation reuses the same texture mapping idea as in traceRay, ensuring consistent appearance between ray tracing and path tracing. And the specular materials are handled as delta distributions, meaning they generate a single deterministic outgoing direction but their approach is still the same Mirror materials reflect ray direction, Conductors additionally apply Fresnel reflectance, Dielectrics handle both reflection and refraction based on Fresnel equations. In all of these cases, the path throughput is updated accordingly and the path continues without performing hemisphere sampling:

```cpp
pathThroughput *= reflectance or Fresnel term;
rd = reflected or refracted direction;
lastSpecular = true;

continue; // Skip hemisphere sampling for delta materials
```

For nonspecular materials (diffuse and glossy), the renderer evaluates direct illumination from point lights and then continues the path by sampling a new direction over the hemisphere. The outgoing direction is sampled either uniformly or using cosine weighted hemisphere sampling. The path throughput is then updated using the standard Monte Carlo estimator formula:

<p align="center">
    <img width="50%" alt="formula" src="https://github.com/user-attachments/assets/2e4d3be5-36bf-413c-9e26-96aa5c45a7b7" />
</p>

which corresponds to:

```cpp
pathThroughput = pathThroughput.multiply(f).scale(cosI / pdf);
```

#### Russian Roulette Termination
To prevent infinite paths while keeping the estimator unbiased, Russian roulette is applied after a minimum depth is reached:

```cpp
if (cam.pathTracingOptions.russianRoulette && depth >= cam.minRecursionDepth) {
    float p = max(pathThroughput.x, max(pathThroughput.y, pathThroughput.z));
    p = min(0.99f, max(0.05f, p));

    if (dist01(rng) > p) 
        break;

    pathThroughput = pathThroughput.scale(1.0f / p);
}
```

Paths with low expected contribution are terminated early, while surviving paths are reweighted to preserve energy.

#### Importance Sampling

In the basic Monte Carlo estimator, we update the path throughput using:

<p align="center">
    <img width="50%" alt="formula" src="https://github.com/user-attachments/assets/2e4d3be5-36bf-413c-9e26-96aa5c45a7b7" />
</p>

The main idea behind importance sampling is to choose a sampling distribution that resembles the function being integrated. For diffuse terms, function being integrated contains a cosThetai term, so sampling directions with a cosine weighted distribution reduces variance compared to uniform hemisphere sampling. In my implementation, outgoing direction is sampled with uniform sampling of the hemisphere 1 / 2π, or cosine weighted sampling cosTheta / π.

```cpp
Vec3 wi = sampleHemisphere(hitNormal,
                            cam.pathTracingOptions.importanceSampling ? EnvSampler::Cosine : EnvSampler::Uniform,
                            dist01(rng), dist01(rng),
                            pdf);
```

In the end, enabling importance sampling changes only the PDF and sampling distribution, but keeps the estimator unbiased.

#### Next Event Estimation (NEE)

Next Event Estimation is a technique used in path tracing to reduce noise and improve convergence by explicitly sampling direct illumination from light sources at each bounce. Even with importance sampling, a randomly sampled hemisphere direction may take a long time to hit a light source, especially when lights are small or far away, so, instead of relying only on random hemisphere sampling to eventually hit light sources, we directly sample the contribution from lights at each intersection point.

<p align="center">
<img alt="NEEComparison" src="https://github.com/user-attachments/assets/2fea623f-2fa4-4945-bd09-c5f105f09816" />
            <br>
    <em>Importance Sampling - Importance Sampling, NEE, MIS</em>
    <br>
</p>


In the lecture notes, the direct lighting term is written as an integral over the hemisphere (or equivalently over light surfaces). With NEE, we estimate it by sampling a direction toward a light and using the Monte Carlo estimator:

<p align="center">
    <img width="50%" alt="formula3" src="https://github.com/user-attachments/assets/c00443e2-6265-4e6b-9836-ae4e9445f184" />
</p>

In my code, this is implemented by sampling one emitter (including light spheres and light meshes) and computing:

```cpp
LightSample lightSample = sampleOneEmitter(...);

Vec3 contribution = lightSample.Li.multiply(f).scale(cosI_light * w / lightSample.pdfW);

L = L.add(pathThroughput.multiply(directLight));
```

So NEE simply adds an additional direct lighting estimate at each bounce, which helps to capture direct illumination more efficiently.

#### Multiple Importance Sampling (MIS)
For the same sampled direction, there may be multiple ways to generate it. MIS compares the probability of generating that direction with each method and then gives more weight to the method that sampled it more naturally.

For example, using the balance heuristic, the light sampled contribution is weighted by:

<p align="center">
    <img width="30%" alt="formula3" src="https://github.com/user-attachments/assets/8f68847a-e26f-467d-ada9-6f2d6f1f62a2" />
</p>

```cpp
float pdfBsdf = 0.0f;

// Determine the PDF of sampling the light direction
if (cam.pathTracingOptions.importanceSampling)
    pdfBsdf = cosI_light / (float) M_PI; // Cosine weighted hemisphere sampling
else
    pdfBsdf = 1.0f / (2.0f * (float) M_PI); // Uniform hemisphere sampling

// Calculate MIS weight
float w = lightSample.isDelta
                ? 1.0f // Delta lights get full weight
                : misWeight(lightSample.pdfW, pdfBsdf, cam.pathTracingOptions.mis); // MIS weight

// Final contribution
Vec3 contribution = lightSample.Li.multiply(f).scale(cosI_light * w / lightSample.pdfW);
directLight = directLight.add(contribution);
```

#### Clamping
Scenes that contain glasses and mirrors can cause problems due to their generation of high radiance regions. These high radiance regions can cause "fireflies" in the render, which are bright pixels that stand out from the surrounding area.

<p align="center">
<img alt="fireflies" src="https://github.com/user-attachments/assets/327800e5-dda6-4aa7-aa45-39a0702b71d7" />
            <br>
    <em>Firefly example from blenderguru.com</em>
    <br>
</p>

Increasing the sample count can help reduce fireflies (as in almost all of our problems :)), but it also increases render times significantly. To address this, we use clamping, which limits the maximum contribution from any single sample. This helps to reduce variance and fireflies without requiring more samples.

#### A classic bug of every part: the forgotten epsilon
I forget to add or subtract epsilon so often when calling functions (especially isInShadow) that I can now spot epsilon related bugs instantly :). Instead of manually adding or subtracting epsilons in the parameters, I should handle this directly inside the functions. Below is the bug caused by the forgotten epsilon in this section, along with its fixed version:

```cpp
// if (isInShadow(scene, shadowOrigin, wi, d, intersector, rayTime)) 
//    return out;

if (isInShadow(scene, shadowOrigin, wi, d - scene.shadowRayEpsilon, intersector, rayTime)) 
    return out;
```

<p align="center">
  <img src="https://github.com/user-attachments/assets/50cb8b5d-fc5b-48b2-9ad8-a501239a5a4f" alt="bugepsilon">
</p>

### Outputs and Closing Thoughts
There are some minor differences in killeroo scene and I am suspecting that it is because of a different bug from previous parts because I had these issues in previous killeroo renders as well. As I mentioned in previous parts, I broke some things in Beer's Law implementation while refactoring so there are some differences due to that in cornell_glass_mirror and veach_ajar scene. Other than these, I think my renders are mostly correct.

Even though it is the last homework, there are still so much more interesting topics to explore in ray and path tracing. The reason I call this a ray tracing _adventure_ is that I feel like I have just started to scratch the surface of this field. There are so many more advanced techniques and optimizations that can be implemented to further improve the realism and efficiency of the renderer. Ideas, improvements, and features my friends presented during our term project presentations had really opened my eyes to the vastness of this field. 

I still have the same enthusiasm during the rendering of the stanford bunny (and repeatedly saying "please work, please work" :)) in the first homework part, and I am amazed by how far I have come since then. Before taking this course, all my friends that took this course earlier warned me about the workload of the homeworks, but I could clearly see their excitements when they showed me their blog posts and renders. I finally understood their enthusiasm after experiencing this adventure myself :). Of course there are some parts that I struggled with or not completely clicked with me, but these blogs became a notepad for me to revisit each of these concepts and implementations later on.

Although there were some moments like caused me to beg my computer to finish rendering sooner, laughing at my failed renders, or scratching my head over a complete black renders, I truly enjoyed this adventure and I am grateful for the opportunity to learn and experiment with these concepts throughout this course. And of course, I would like to thank to Professor Ahmet Oğuz Akyüz for his guidance and support throughout this _adventure_ and Akif Uslu, Ramazan Tokay, and Akın Aydemir for their contributions to the scene files.

I uploaded .exr and .hdr files to [this folder in repository](http://github.com/fsaltunyuva/fsaltunyuva.github.io/tree/main/images/2026-01-17-Ray-Tracing-Adventure), I used [GIMP](https://www.gimp.org/) to view them but there are other softwares for that purpose.

Here are my final renders and their render times:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| killeroo_blinnphong                 | 77.2764  |
| killeroo_blinnphong (killeroo_blinnphong_closeup)                | 91.3994  |
| killeroo_torrancesparrow                 | 71.9786  |
| killeroo_torrancesparrow (killeroo_torrancesparrow_closeup)                 | 90.5573  |
| cornellbox_jaroslav_diffuse                 | 113.283  |
| cornellbox_jaroslav_diffuse_area                 | 79.1459  |
| cornellbox_jaroslav_glossy                 | 4.42081  |
| cornellbox_jaroslav_glossy_area                 | 112.296   |
| cornellbox_jaroslav_glossy_area_ellipsoid                 | 155.784  |
| cornellbox_jaroslav_glossy_area_small                 | 212.469  |
| cornellbox_jaroslav_glossy_area_sphere                 | 168.635  |
| cornell_diffuse (diffuse_cornell_box_default)                | 12.1427  |
| cornell_diffuse (diffuse_cornell_box_importance)                | 11.9678  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance)                | 21.8582  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_clamping)                | 21.628  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting)                | 1.12045  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting_clamp)                | 1.08141  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian)                | 20.3728  |
| cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian_1600)                | 284.931  |
| cornell_glass_mirror (cornell_box_default)                | 11.3779  |
| cornell_glass_mirror (cornell_box_importance)              | 10.2495  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance)               | 147.712  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance_clamping)                | 148.752  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting)                | 6.16006  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting_clamp)                | 6.19947  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance_russian)                | 136.45  |
| cornell_glass_mirror (cornell_box_importance_nee_mis_balance_2500x2)                | 3963.64  |
| cornellbox_prism_light                 | 113.818  |
| cornellbox_sphere_light                 | 72.9574  |
| sponza_direct                 | 104.626  |
| sponza_path                 | 3047.49  |
| VeachAjar                 | 71627.6* (approximately 20 hours)   |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

**Used CPU: AMD Ryzen 5 7640HS 6-Core Processor (4.30 GHz)*

---
## VeachAjar
This render took nearly 20 hours, and although the render is not completely matched with expected outuput, I think it did fine. So, as in previous parts, I wanted to place this render at the top of all other renders because of the time taken.

_Time: 71627.6* s_
<p align="center">
<img alt="VeachAjar_phot" src="https://github.com/user-attachments/assets/15e9eae7-33ff-4ba6-8451-0f5a336611a4" />
</p>

---

### killeroo_blinnphong
_Time: 77.2764 s_
<p align="center">
<img alt="killeroo_blinnphong_phot" src="https://github.com/user-attachments/assets/404cd6d4-7256-4829-ba1d-589125d8f81e" />

</p>

---

### killeroo_blinnphong (killeroo_blinnphong_closeup) 
_Time: 91.3994 s_
<p align="center">
<img alt="killeroo_torrancesparrow_closeup_phot" src="https://github.com/user-attachments/assets/54df27fc-1286-415b-b7a1-83d0c6351f3a" />
</p>

---

### killeroo_torrancesparrow  
_Time: 71.9786 s_
<p align="center">
<img alt="killeroo_torrancesparrow_phot" src="https://github.com/user-attachments/assets/b8fa408a-f428-4847-9ab1-1ba9fb963e49" />

</p>

---
### killeroo_torrancesparrow (killeroo_torrancesparrow_closeup)
_Time: 90.5573 s_
<p align="center">
<img alt="killeroo_torrancesparrow_closeup_phot" src="https://github.com/user-attachments/assets/278e4f50-d9ea-4a06-b115-11202198356f" />

</p>

---

### cornellbox_jaroslav_diffuse
_Time: 113.283 s_
<p align="center">
<img alt="cornellbox_jaroslav_diffuse_phot" src="https://github.com/user-attachments/assets/3b414a41-f84a-4748-9579-23754d6f5f81" />

</p>

---

### cornellbox_jaroslav_diffuse_area
_Time: 79.1459 s_
<p align="center">
<img alt="cornellbox_jaroslav_diffuse_area_phot" src="https://github.com/user-attachments/assets/2d26bb58-b861-42a0-a59f-b45ade806e4a" />
</p>

---

### cornellbox_jaroslav_glossy
_Time: 4.42081 s_
<p align="center">
<img alt="cornellbox_jaroslav_glossy_phot" src="https://github.com/user-attachments/assets/f334d6c9-c594-4f75-96d5-4dacd7dc745f" />

</p>

---

### cornellbox_jaroslav_glossy_area
_Time: 112.296 s_
<p align="center">
<img alt="cornellbox_jaroslav_glossy_area_phot" src="https://github.com/user-attachments/assets/9e8a902b-6a7a-4f20-9ca1-d8b6c20fa96a" />
</p>

---

### cornellbox_jaroslav_glossy_area_ellipsoid
_Time: 155.784 s_
<p align="center">
<img alt="cornellbox_jaroslav_glossy_area_ellipsoid_phot" src="https://github.com/user-attachments/assets/ff9755c2-4b0e-486b-8834-e661e512caab" />

</p>

---

### cornellbox_jaroslav_glossy_area_small
_Time: 212.469 s_
<p align="center">
<img alt="cornellbox_jaroslav_glossy_area_small_phot" src="https://github.com/user-attachments/assets/4e725906-041f-4a11-b49b-858e0d51afff" />

</p>

---

### cornellbox_jaroslav_glossy_area_sphere
_Time: 168.635 s_
<p align="center">
<img alt="cornellbox_jaroslav_glossy_area_sphere_phot" src="https://github.com/user-attachments/assets/2a28565c-42fb-4bde-9338-6bf31ed854af" />

</p>

---

### cornell_diffuse (diffuse_cornell_box_default)
_Time: 12.1427 s_
<p align="center">
<img alt="diffuse_cornell_box_default_phot" src="https://github.com/user-attachments/assets/de743b50-99b2-41fa-b7a3-bdf2ed03f629" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance) 
_Time: 11.9678 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_phot" src="https://github.com/user-attachments/assets/3aa861a8-a4ca-409f-b369-fa962646e6f2" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance)
_Time: 21.8582 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_phot" src="https://github.com/user-attachments/assets/7701db06-7c35-4d6c-8cd9-549624cc1f73" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_clamping)
_Time: 21.628 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_clamping_phot" src="https://github.com/user-attachments/assets/ecc75d00-e745-4eaa-95ee-7a77f28ec85e" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting) 
_Time: 1.12045 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_splitting_phot" src="https://github.com/user-attachments/assets/2db609af-f97b-41e9-bc34-d5be517349da" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 1.08141 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_splitting_clamp_phot" src="https://github.com/user-attachments/assets/2f54b34d-d8eb-491b-8624-fccf854e79b5" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian)
_Time: 20.3728 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_russian_phot" src="https://github.com/user-attachments/assets/ece96d60-3f33-41d5-b767-f327bf0021d3" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian_1600)
_Time: 284.931 s_
<p align="center">
<img alt="diffuse_cornell_box_importance_nee_mis_balance_russian_1600_phot" src="https://github.com/user-attachments/assets/ebfc05b4-5921-4730-ab52-baf97fc969c6" />
</p>

---

### cornell_glass_mirror (cornell_box_default) 
_Time: 11.3779 s_
<p align="center">
<img alt="cornell_box_default_phot" src="https://github.com/user-attachments/assets/48a790f6-a042-4626-9707-8d62c1e40cc3" />
</p>

---

### cornell_glass_mirror (cornell_box_importance)
_Time: 10.2495 s_
<p align="center">
<img alt="cornell_box_importance_phot" src="https://github.com/user-attachments/assets/0f593af1-45c1-4d66-b647-89b59127b571" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance) 
_Time: 147.712 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_phot" src="https://github.com/user-attachments/assets/8abf29a5-52af-4b08-a463-680ce77b1f44" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_clamping)
_Time: 148.752 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_clamping_phot" src="https://github.com/user-attachments/assets/623da4f8-cccc-4e8e-83c0-fb2f03e5c248" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting)
_Time: 6.16006 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_splitting_phot" src="https://github.com/user-attachments/assets/8fa5868f-f5ea-4ad4-a619-775daa0ff746" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 6.19947 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_splitting_clamp_phot" src="https://github.com/user-attachments/assets/f6ad87c0-5c14-4d0d-a70f-6be294dfea25" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_russian)  
_Time: 136.45 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_russian_phot" src="https://github.com/user-attachments/assets/8e55551f-5270-467e-8442-ab8595dd46b7" />

</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_2500x2) 
_Time: 3963.64 s_
<p align="center">
<img alt="cornell_box_importance_nee_mis_balance_2500x2_phot" src="https://github.com/user-attachments/assets/66a79ff7-cf67-436b-a2d6-fde4c27bc319" />
</p>

---

### cornellbox_prism_light
_Time: 113.818 s_
<p align="center">
<img alt="cornellbox_prism_light_phot" src="https://github.com/user-attachments/assets/924f01c8-467c-4a57-a74d-b002dcfd8a06" />
</p>

---

### cornellbox_sphere_light
_Time: 72.9574 s_
<p align="center">
<img alt="cornellbox_sphere_light_phot" src="https://github.com/user-attachments/assets/a4da6f91-b0b0-44c5-bc31-4f2c0ff276ab" />
</p>

---

### sponza_direct
_Time: 104.626 s_
<p align="center">
<img alt="sponza_direct_phot" src="https://github.com/user-attachments/assets/09404b84-e983-4d3b-a824-d9bbeb33180c" />
</p>

---

### sponza_path
_Time: 3047.49 s_
<p align="center">
<img alt="sponza_path_phot" src="https://github.com/user-attachments/assets/b27ae8cb-ff76-4e44-a7ad-02a68c50a99c" />
</p>

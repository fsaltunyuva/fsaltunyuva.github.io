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

Here you can see some comparisons of render times before and after optimizations:

| Scene                 | Before (seconds) | After (seconds) |
| --------------------- | ---------------- | --------------- |

### Bidirectional Reflectance Distribution Function (BRDF)
Up to this point, my ray tracer was able to trace rays correctly and compute intersections, but the actual appearance of surfaces was still quite limited. Every surface responded to light in a very basic way, without considering the complex interactions between light and material properties. To address this, I implemented a Bidirectional Reflectance Distribution Function (BRDF) system. 

A BRDF defines how light is reflected at a surface point, given two directions, the incoming light direction (wi) and the outgoing view direction (wo). More precisely, a BRDF describes how much of the incoming radiance from a given direction is scattered toward another direction. This abstraction is powerful because it separates geometry, lighting, and material behavior into cleanly defined components.

[BRDF IMAGE HERE]

As stated in the homework file the possible BRDF models to implement were:
- Original Blinn-Phong
- Original Phong
- Modified Blinn-Phong
- Modified Phong
- Torrance Sparrow

and BRDF field in JSON files included the options ```_normalized``` to indicate whether the BRDF should be normalized or not and ```Exponent``` to define the shininess of the surface. Also, Torrance Sparrow model required an additional parameter ```_kdfresnel``` to define the Fresnel reflectance at normal incidence, I will explain this parameter in more detail below.

As in previous parts, I implemented BRDF models as common interface because they all input the same parameters (wi, wo, normal) and output the reflectance value. The shading logic does not need to know which BRDF is being used, it simply calls the BRDF’s evaluation function. In this way, making changes to the BRDF models or adding new ones becomes straightforward without affecting other parts of the code.

Before starting the implementation of the BRDF models, I normalized all direction vectors (wi, wo, normal) at the beginning of each BRDF evaluation function to ensure consistent calculations. Then I caculated cosine terms between the surface normal and trhe incoming and outgoing directions, which are essential for determining how much light is reflected based on the angle of incidence and reflection.

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
    <img alt="original-blinn-phong" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

```cpp
float c = pow(cosAlphaH(), exponent);
specScalar = c / cosI;
```

#### Original Phong
The Original Phong BRDF is similar but it computes the specular term using the reflection direction of the incoming light instead of the half vector. The highlight intensity depends on the alignment between this reflection direction and the view direction.

<p align="center">
    <img alt="original-phong" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

```cpp
float c = pow(cosAlphaR(), exponent);
specScalar = c / cosI;
```

#### Modified Phong and Modified Blinn-Phong
The Modified Phong and Modified Blinn-Phong BRDFs extend their originals by optionally applying normalization. When the ```_normalized``` flag is enabled, a normalization factor derived from the exponent is applied to the specular term. This ensures that the total reflected energy does not exceed the incoming energy, preventing materials from becoming unrealistically bright as the exponent increases.

Modified Phong:

<p align="center">
    <img alt="modified-phong" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

Modified Blinn-Phong:

<p align="center">
    <img alt="modified-phong" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
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
        float c = pow(cosAlphaR(), exponent); // brdf.pdf uses cosAlphaR
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

The ```_kdfresnel``` parameter defines the Fresnel reflectance at normal incidence, When enabled, the diffuse component is scaled by:

<p align="center">
    <img alt="torrance-sparrow" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

instead of the usual ```(1 - kd)``` term. This adjustment accounts for the fact that some portion of the incoming light is reflected at the surface interface due to Fresnel effects, reducing the amount of light available for diffuse reflection.

<p align="center">
    <img alt="figure" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

I followed the steps outlined in the lecture notes to implement the Torrance-Sparrow BRDF as follows:

1. *Compute the half vector (wh) between the incoming (wi) and outgoing (wo).*
2. *Compute the angle α as wh • n.*
3. *Compute the probability of this α using D(α) function (Blinn's distrubiton in our case).*

<p align="center">
    <img alt="D(α)" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

```cpp
diffuse = kd.scale(1.0f / PI); // Inverse pi for lambertian

Vec3 wh = wi.add(wo).normalize();
float nDotWh = clamp01(n.dot(wh));
float D = ((exponent + 2.0f) / (2.0f * PI)) * pow(nDotWh, exponent);
```

4. *Compute the geometry term G(wi, wo).*

<p align="center">
    <img alt="G(wi, wo)" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
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
    <img alt="F" src="https://github.com/user-attachments/assets/6ce5c2c5-b590-4794-9bd4-9e8d23d724ac" />
    <br>
</p>

### Object Lights


### Path Tracing


### Outputs and Closing Thoughts

I uploaded .exr and .hdr files to [this folder in repository](https://github.com/fsaltunyuva/fsaltunyuva.github.io/tree/main/images/2025-12-27-Ray-Tracing-Adventure), I used [GIMP](https://www.gimp.org/) to view them but there are other softwares for that purpose.

As in previous parts, I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance. Here are my final renders and their render times:

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
<img alt="teapot_roughness" src="https://github.com/user-attachments/assets/573e5709-0a80-4edb-b4af-1603fc5ae2da" />
</p>

---

### killeroo_blinnphong
_Time: 77.2764 s_
<p align="center">
<img alt="cube_directional" src="https://github.com/user-attachments/assets/933a8170-8750-47c3-b877-28a35bb440a9" />
</p>

---

### killeroo_blinnphong (killeroo_blinnphong_closeup) 
_Time: 91.3994 s_
<p align="center">
<img alt="cube_point" src="https://github.com/user-attachments/assets/335ba37e-f133-4eaf-a7f1-03f2e02756a6" />
</p>

---

### killeroo_torrancesparrow  
_Time: 71.9786 s_
<p align="center">
<img alt="cube_point_hdr" src="https://github.com/user-attachments/assets/5c607afa-2b45-49ef-abf9-d05e3219f22e" />
</p>

---
### killeroo_torrancesparrow (killeroo_torrancesparrow_closeup)
_Time: 90.5573 s_
<p align="center">
<img alt="dragon_spot_light_msaa" src="https://github.com/user-attachments/assets/2ff0d40c-6cbd-480e-aa45-f5e3a681638b" />
</p>

---

### cornellbox_jaroslav_diffuse
_Time: 113.283 s_
<p align="center">
<img alt="empty_environment_latlong_aces" src="https://github.com/user-attachments/assets/3154f4d9-9a05-4113-a3b7-329d1cd8d31f" />
</p>

---

### cornellbox_jaroslav_diffuse_area
_Time: 79.1459 s_
<p align="center">
<img alt="empty_environment_light_probe_aces" src="https://github.com/user-attachments/assets/f96e1ad3-9f9f-4be0-bd4e-3b3254db20c5" />
</p>

---

### cornellbox_jaroslav_glossy
_Time: 4.42081 s_
<p align="center">
<img alt="glass_sphere_env_phot" src="https://github.com/user-attachments/assets/27b3b2d3-dc67-4c56-8f7b-79a33800929b" />
</p>

---

### cornellbox_jaroslav_glossy_area
_Time: 112.296 s_
<p align="center">
<img alt="head_env_light_phot" src="https://github.com/user-attachments/assets/c45dd861-159a-4f8a-b084-230f242c1efe" />
            <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area_ellipsoid
_Time: 155.784 s_
<p align="center">
<img alt="mirror_sphere_env_phot" src="https://github.com/user-attachments/assets/25fb5f57-ab43-4058-b29f-6767952c48b7" />
</p>

---

### cornellbox_jaroslav_glossy_area_small
_Time: 212.469 s_
<p align="center">
<img alt="sphere_env_light_phot" src="https://github.com/user-attachments/assets/cd4f7724-6694-4344-9e94-767ce5110b12" />
                <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area_sphere
_Time: 168.635 s_
<p align="center">
<img alt="sphere_point_hdr_texture_aces" src="https://github.com/user-attachments/assets/2b6d6aad-57cd-498f-b628-67af9491fcc8" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_default)
_Time: 12.1427 s_
<p align="center">
<img alt="dragon_new_ply_with_spot" src="https://github.com/user-attachments/assets/8072ef91-b8d5-42b9-a12d-a025ada42ba0" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance) 
_Time: 11.9678 s_
<p align="center">
<img alt="audi-tt-glacier" src="https://github.com/user-attachments/assets/0b12aad6-646d-4cf3-a9a3-2e8530b4c95a" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance)
_Time: 21.8582 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_clamping)
_Time: 21.628 s_
<p align="center">
<img alt="VeachAjar_phot_key_0_18_s1_2_burn_1" src="https://github.com/user-attachments/assets/4e4ef41d-0910-4520-86a4-f1ff4d932401" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting) 
_Time: 1.12045 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 1.08141 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian)
_Time: 20.3728 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian_1600)
_Time: 284.931 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_default) 
_Time: 11.3779 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance)
_Time: 10.2495 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance) 
_Time: 147.712 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_clamping)
_Time: 148.752 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting)
_Time: 6.16006 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 6.19947 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_russian)  
_Time: 136.45 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_2500x2) 
_Time: 3963.64 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornellbox_prism_light
_Time: 113.818 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornellbox_sphere_light
_Time: 72.9574 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### sponza_direct
_Time: 104.626 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### sponza_path
_Time: 3047.49 s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>
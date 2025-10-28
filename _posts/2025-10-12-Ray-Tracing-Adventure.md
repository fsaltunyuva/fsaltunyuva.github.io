---
layout: post
title: "Ray Tracing Adventure Part 1 (Homework 1)"
date: 2025-10-12
categories: [ray-tracing, graphics, adventure]
---
Hi, I'm Furkan Safa Altunyuva. This blog will be about the bugs, interesting situations, and steps I've encountered while developing my own ray tracer.

The programming language I'll be using is C++. Although this will be my first *advanced* project in C++, I believe this process will teach me a lot about the intricacies of C++, in addition to ray tracing techniques. Of course, there will be many bugs I'm unaware of, including out of bounds and potential memory leaks (I'll briefly discuss these in my blog posts; some of them create interesting renderings :)), but as I said, I'm confident this project will also teach me a lot about C++.

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz at Middle East Technical University.

If I were to briefly mention what the first assignment required of us:
- Parse and use inputs given as JSON files
- Save results to a file in PNG format
- Ray Intersection Tests
- Blinn-Phong shading model
- Fresnel reflection rules for conductors, dielectrics, and simple mirrors
- Beer's law for attenuation for dielectrics
- Point and ambient light sources
- Back face culling
- Degenerate Triangle handling
- Shaded Mesh rendering

I will briefly talk about the first two items (JSON parsing and PNG saving), because both parts are mostly done by certain libraries and are not very related to the ray tracing process itself.

To parse JSON files, I used the most famous library for this task in C++, the [json library](https://github.com/nlohmann/json?tab=readme-ov-file#license) developed by Niels Lohmann. Similarly, I used another popular library, [stb](https://github.com/nothings/stb), to write the calculated results to a PNG file.

And yes, it's finally time for ray tracing. And the ingredients we need for a ray tracer are a camera, an image plane, and objects. Let's start with the camera.

<p align="center">
  <img alt="camera" src="https://github.com/user-attachments/assets/b3583edf-6f0f-475a-b4e0-d3edfed71525"/>
  <br/>
  <em>From <strong>Fundamentals of Computer Graphics</strong> by Stephen Marschner and Peter Shirley</em>
</p>

I set w to be the exact opposite of gaze (it is given in JSON file), because w should be the exact opposite of the direction the camera is looking. Then I set u to be the cross product of up and w, because the u vector should point to the right of the camera, and the cross product of these two vectors gives exactly that direction. Then I made the vector v the cross product of w and u, because the cross product of these two vectors should give the upward direction.

```cpp
Vec3 w = gaze.scale(-1.0f); // -gaze
Vec3 u_vec = (up.cross(w)).normalize();
Vec3 v_vec = w.cross(u_vec);
```

Then we entered two nested for loops where everything will be calculated, everything I will talk about from now on will be inside these nested for loops, in short, we will calculate the calculations I will talk about by sending rays to each pixel.

```cpp
// Parsed height and width from JSON
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        // All calculations will be here
    }
}
```

This loop will run specifically for each camera provided in the input JSON file, generating an image for each one. In other words, it will create as many images as there are cameras.

Before sending our ray, we need to normalize the coordinates we have to -1 and 1.

```cpp
float su = left + (right - left) * (x + 0.5f) / width;
float sv = bottom + (top - bottom) * (y + 0.5f) / height;
```

In this part of the code, I normalized the pixel ranges we will move through according to the near plane values ​​given to us. For example, in an image given a 300x300, if our ray tracer is given near plane values ​​of 1 and -1, instead of advancing to 300 one by one, my ray tracer will move from -1 to 1 in intervals divided by 300. In this way, we will provide the actual field of view value we want.

When we subtract w * nearDist from the camera’s position (i.e., camPos - w * nearDist), we reach the center of the image plane. To find the exact point on the image plane where our ray should be cast, we then add the horizontal and vertical offsets — u * su and v * sv — to this value. This gives us the precise point from which the ray should be generated.

```cpp
Vec3 P = camPos
    .subtract(w.scale(nearDist))   // camPos - w * nearDist
    .add(u_vec.scale(su))          // + u * su
    .add(v_vec.scale(sv));         // + v * sv
```

In short, this P value is the D value in the ray equation R(t) = O + tD, where O is the origin (camPos in our case) and D is the direction (P - camPos).

I'll fast forward a little bit and come back shortly. After this step, I sent the rays and made my calculations and got the following result in my first attempt (since I forgot to save the result I got at that time, I will show a similar scenario with the output I got from the latest version of my ray tracer).

<p align="center">
<img alt="flipped_spheres" src="https://github.com/user-attachments/assets/d34549f7-21a2-45a7-bbb7-20a8d8306b87" />
</p>

Although the first thing that came to my mind when I saw it was a wrong origin or a wrong calculation, I later realized that in PNG files, the point (0,0) is the upper left corner. So I flipped my bottom left corner (0,0) by doing:

```cpp
int flippedY = height - 1 - y;
int index = (flippedY * width + x) * 3; // This value will be used as index to set pixel colors
```

Then I was able to get the correct result.

<p align="center">
<img alt="spheres" src="https://github.com/user-attachments/assets/c4d04551-1a95-4aad-bc42-7fae8bebc31b" />
</p>

Okay, let's continue where we left off. After completing the project to this point, I thought the first thing I needed to do was intersection tests. So, I directly adapted the formulas of sphere, triangle and plane intersections that we learned in class into the code.

Even though I struggled a bit with small syntax errors here and there, I eventually got it working. Then, I thought to myself for a moment, "Intersection tests were supposed to be the easy part, there are much harder things ahead." While this idea actually scared me, I mentioned to a friend that I had just finished the intersection test part of the assignment. My friend said, "Well, then you can render anything now!"

At that moment, I realized I could actually render *everything*, and of course, to make things harder for myself, I decided to start with the famous Stanford Bunny. When I saw the output image (after 57 minutes :)), a smile appeared on my face.

<p align="center">
<img alt="bunny2" src="https://github.com/user-attachments/assets/d7d97b94-98be-4ceb-ac8f-c5e56b966bc4" />
</p>

For a while, I kept thinking: *“I just told the program where the camera and the objects should be, and it did all the necessary calculations to produce an image for me. What could possibly be more exciting than that?”* 

But of course, it was only just beginning :). After the sweet smell of success, I immediately started working on the shading part.

If the ray I threw intersected with an object and that object had a material, I first started by adding ambient color to the color of that pixel.

<p align="center">
<img alt="bunny_ambient" src="https://github.com/user-attachments/assets/de44a631-3bfe-496c-9c0f-b89fe8843987" />
</p>

Then I added diffuse and specular for each light source.

<p align="center">
<img alt="bunny_diffuse_specular" src="https://github.com/user-attachments/assets/46f98a85-aac2-486c-a99f-8eb749e75cf9" />
</p>

Although seeing the results was really satisfying, the render times (which could easily exceed 50 minutes) were becoming a bit of a problem. Sure, I could have tried simpler scenes, but once my eyes had been captivated by the Stanford Bunny, a few basic shapes just wouldn’t cut it anymore :).

That’s when I remembered that ray tracing can run incredibly fast on GPUs. Of course, I didn’t have enough knowledge to suddenly move all computations to the GPU, but the fact that it could run there made me realize something: my ray tracer should be parallelizable. After all, each iteration was completely independent, no data needed to be shared between them.

In C++, I found a library that would let me parallelize this process with minimal effort: [OpenMP](https://www.openmp.org/). All I had to do was add a simple directive before my for loop, and the iterations would start executing in parallel.

```cpp
#pragma omp parallel for schedule(dynamic)
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        // All calculations will be here
    }
}
```

With this simple change, I was able to reduce my render times down to around 5 minutes. As we discussed in class, I knew there were techniques capable of bringing these times down to milliseconds, but before diving into those, I wanted to see what else I could improve.

That’s when I decided to focus on the back-face culling method, which was also mentioned in the assignment.

Back-face culling is a technique used to improve rendering efficiency by not drawing triangles that face away from the camera. In short, the goal is to avoid unnecessarily including triangles in the rendering calculations if they’re not visible anyway.

The normals of such triangles naturally point in the same direction as the camera’s viewing direction. Therefore, if the dot product of the ray’s direction and the triangle’s normal is positive, it means the triangle is facing away from the camera, and we can safely skip it during rendering. Also,we need to take into account that we should not make this calculation on shadow rays.

After adding the back-face culling logic to my code, I tried rendering the Stanford Bunny again, this time without using parallelization.
Surprisingly, the render time actually increased from around 57 minutes to about 65 minutes.

Then I realized my mistake: I was performing the back-face culling check after computing the ray–triangle intersection. In other words, instead of skipping unnecessary intersection calculations by checking the triangle’s orientation beforehand, I was doing the opposite, adding an extra computation after the intersection test.

Once I moved the back-face culling check inside my intersection function (so that it happens before the intersection is calculated), the render time dropped to around 35 minutes, just as I had expected.

```cpp
static bool rayIntersectsTriangle( bool backFaceCull, // some parameters ) {
    // Calculate triangle normal N

    float NdotD = N.dot(rayDir);

    if (fabs(NdotD) < 1e-8) return false; // Plane and ray are parallel

    if (backFaceCull && NdotD >= 0.0f) return false; // Backface culling

    // Continue with intersection calculations
}
```

Also, before performing the intersection test, I added a small check to skip degenerate triangles, triangles whose area is effectively zero (when all three vertices lie on the same line).

To detect these, I calculate the squared length of the triangle’s normal vector. Because getting the square root is computationally expensive and unnecessary for this check. If this value is smaller than a tiny threshold (epsilon), it means the triangle’s area is too small to be valid, so the function simply returns false and skips the intersection:

```cpp
// Calculate triangle normal N
float N_len_sq = N.length_squared();

if (N_len_sq < 1e-12f) {
    return false; // Degenerate triangle
}
```

Now that my renders can run faster, we can go back to the ray tracing part. After implementing basic shading, I added shadow rays to determine if a point is in shadow. The idea is simple, for every intersection point, I cast a secondary ray from the hit point toward the light source. If this ray intersects any other object before reaching the light, that means the point is in shadow, so its diffuse and specular contributions from that light are skipped. To avoid self-intersections, each shadow ray starts slightly offset from the surface point by adding a small epsilon value along the surface normal.

```cpp
static bool isInShadow(const Scene& scene,
                       const Vec3& shadowRayOrigin,
                       const Vec3& shadowRayDir,
                       float lightDistance) {
    IntersectionInfo info = findClosestIntersection(scene, shadowRayOrigin, shadowRayDir, true); // true for shadow ray, it sets the epsilon appropriately

    if (info.hit) {
        if (info.t > 0.0f && info.t < lightDistance) {
            return true; // It is in shadow!
        }
    }

    return false; // Not in shadow
}
```

With all these changes, I rendered the Stanford Bunny again, and this time, the shadows added a nice touch of realism to the scene.

<p align="center">
<img alt="bunny_with_plane_shadow" src="https://github.com/user-attachments/assets/74eefbca-549f-4a6b-9d6c-eb3a5df220d9" />
</p>

For the reflection calculations, I relied on the Fresnel equations we covered in class. Using those formulas, I implemented separate reflection behaviors for dielectrics, conductors, and simple mirrors.

The goal was to make light interactions more physically accurate: dielectric materials (like glass or water) both reflect and refract light depending on the viewing angle, while metals reflect light with color and intensity determined by their complex refractive indices. Simple mirrors, on the other hand, follow the ideal reflection rule without any refraction or absorption.

<p align="center">
<img alt="bunny_with_plane_fresnel" src="https://github.com/user-attachments/assets/513f98ab-8397-43b5-a051-7a0e3ef683cd" />
</p>

For dielectric materials, I also implemented light attenuation using Beer's Law, as discussed in class.
This law models how light intensity decreases as it travels through a transparent medium, depending on the material’s absorption coefficient and the distance the light travels inside it.

$$L(x) = L(0)e^{-cx}$$

In my ray tracer, this means that when a ray refracts through materials like glass, the transmitted color becomes slightly darker and more tinted the thicker the object is, producing a more realistic look for transparent surfaces. 

The difference that beer's law makes can be seen in the following GIF, Beer's law is not applied when the top black bar is full, beer's law is applied when the top bar is empty (white).

![beers law](https://github.com/user-attachments/assets/e4651326-5bbd-4e88-9158-4662e92908c6)

After implementing flat shading for each triangle, I added smooth shading to make curved surfaces look continuous and natural.

Instead of using a single face normal for the entire triangle, I compute the interpolated normal at the intersection point using the triangle’s vertex normals and their barycentric coordinates.

<p align="center">
<img alt="smooth shading" src="https://github.com/user-attachments/assets/955bbf95-71fb-4748-b9cb-dec3d14af299" />
</p>

With this, I successfully completed all the required features for the assignment. For the upcoming ones, I designed my code to follow object-oriented principles, so that I can easily extend or modify it later. I also organized the logic into dedicated functions for each operation.

I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance, and Akif Uslu and Deniz Sayın for their contributions to the 3D models.

You can see the rendering times in the following table:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| bunny                 | 18.0517  |
| bunny_with_plane      | 229.144  |
| chinese_dragon        | 457.066  |
| cornellbox            | 0.399039 |
| cornellbox_recursive  | 0.588336 |
| other_dragon          | PLY      |
| scienceTree           | 43.5769  |
| scienceTree_glass     | 63.807   |
| simple                | 0.217648 |
| spheres               | 0.254147 |
| spheres_mirror        | 0.31197  |
| spheres_with_plane    | 0.220229 |
| two_spheres           | 0.142726 |
| berserker_smooth      | 44.4589  |
| Car_front_smooth      | 632.8    |
| Car_smooth            | 314.33   |
| low_poly_scene_smooth | 139.563  |
| ton_Roosendaal_smooth | PLY      |
| tower_smooth          | 165.598  |
| trex_smooth           | PLY      |
| windmill_smooth       | 171.286  |
| lobster               | PLY      |
| David (100x100 Resolution to speed up)                 | —        |
| raven                 | 230.513  |
| UtahTeapotMugCENG (100x100 Resolution to speed up)     | —        |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

I could not manage to import and render PLY files yet, so those are marked as PLY.

Back Face Culling Experiments on Bunny With Plane Scene:
- Without Back Face Culling and Parallelization: 2329.69 seconds
- With Wrong Back Face Culling (after intersection test) and Without Parallelization: 3469.64 seconds
- With Correct Back Face Culling (before intersection test) and Without Parallelization: 2104.23 seconds
- With Correct Back Face Culling and With Parallelization: 254.837 seconds

# Outputs:

Other than having some issues in chinese_dragon and raven inputs, my ray tracer worked perfectly for all other scenes.

---

### Bunny  
_Time: 18.0517 s_
<p align="center">
<img alt="bunny" src="https://github.com/user-attachments/assets/89f932af-5554-4523-9eb5-ff0999cd154a" />
</p>

---

### Bunny_with_plane  
_Time: 229.144 s_
<img alt="bunny_with_plane" src="https://github.com/user-attachments/assets/98858d36-843e-4641-8968-01acb18e14b7" />

---

### Chinese_dragon  
_Time: ???_

---

### Cornellbox  
_Time: 0.399039 s_
<img alt="cornellbox" src="https://github.com/user-attachments/assets/d4f780e7-b3cf-4ba1-b00c-03bb46646f04" />

---

### Cornellbox_recursive  
_Time: 0.588336 s_
<img alt="cornellbox_recursive" src="https://github.com/user-attachments/assets/3f0411b1-8fd2-4eb3-af2c-1a56979b08be" />

---

### ScienceTree  
_Time: 43.5769 s_
<img alt="scienceTree" src="https://github.com/user-attachments/assets/80b28a05-5825-4068-9313-8c9ff1fe1d4e" />

---

### ScienceTree_glass  
_Time: 63.807 s_
<img alt="scienceTree_glass" src="https://github.com/user-attachments/assets/39efbef9-9dcd-44e6-9b43-c3854aa0a393" />

---

### Simple  
_Time: 0.217648 s_
<img alt="simple" src="https://github.com/user-attachments/assets/93675a27-0667-4117-8478-0e2b437500cc" />

---

### Spheres  
_Time: 0.254147 s_
<img alt="spheres" src="https://github.com/user-attachments/assets/aba0d8c1-4f70-421c-b989-9a4bd53b78d5" />

---

### Spheres_mirror  
_Time: 0.31197 s_
<img alt="spheres_mirror" src="https://github.com/user-attachments/assets/f21e24e5-8cb2-4b08-bf6e-bae84cbc8c21" />

---

### Spheres_with_plane  
_Time: 0.220229 s_
<img alt="spheres_with_plane" src="https://github.com/user-attachments/assets/7979d72c-e57d-4410-b94b-6ee664799de1" />

---

### Two_spheres  
_Time: 0.142726 s_
<img alt="two_spheres" src="https://github.com/user-attachments/assets/dc96e70c-c1b7-4c98-ae6a-0134010f8f95" />

---

### Berserker_smooth  
_Time: 44.4589 s_
<img  alt="berserker_smooth" src="https://github.com/user-attachments/assets/d5183262-d503-47a0-884e-af6527e3b3f4" />

---

### Car_front_smooth  
_Time: 632.8 s_
<img alt="Car_front_smooth" src="https://github.com/user-attachments/assets/31cf4660-ec37-45de-b3b8-b43e71834adc" />

---

### Car_smooth  
_Time: 314.33 s_
<img alt="Car_smooth" src="https://github.com/user-attachments/assets/4ce46bd0-d710-4640-9220-7b6d97c27781" />

---

### Low_poly_scene_smooth  
_Time: 139.563 s_
<img alt="low_poly_scene_smooth" src="https://github.com/user-attachments/assets/523f65b3-6ca3-4b22-8b89-036bb3693cf2" />

---

### Tower_smooth  
_Time: 165.598 s_
<img alt="tower_smooth" src="https://github.com/user-attachments/assets/3a9e108c-bc27-4b1d-bbfb-6ebf5dddaa6a" />

---

### Windmill_smooth  
_Time: 171.286 s_
<img alt="windmill_smooth" src="https://github.com/user-attachments/assets/5f59163f-a16a-4455-b4d7-26e82897e7b5" />

---

### David (100x100 Resolution to speed up)   
_Time: —_

---

### Raven  
_Time: 230.513 s_
<img alt="raven" src="https://github.com/user-attachments/assets/5d27a28a-f741-43ab-98b0-6517bf7388db" />

---

### UtahTeapotMugCENG (100x100 Resolution to speed up)  
_Time: —_

---
layout: post
title: "Ray Tracing Adventure Part 2 (Homework 2)"
date: 2025-11-11
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 2, focusing on implementing features:

- **Transformations** (translation, scaling, rotation)
- **Mesh Instancing**
- **Bounding Volume Hierarchies (BVH)**
- **Look At Camera**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz at Middle East Technical University.

### Bugs on Part 1
But, talking about new features, I would like to mention the bugs I have encountered on Part 1 and understand some of them. Let's start with the PLY import bug. 

<p align="center">
<img alt="ton_roosendal_scary" src="https://github.com/user-attachments/assets/f2769745-6945-406c-ab0d-a9c8a6eaac4d" />
</p>

I had been suspecting that the strange results I was getting with the PLY models were due to incorrectly computed normals or the vertex data being interpreted differently than I expected. However, it turned out the issue was much simpler than that: in the JSON scenes we use, the vertex indices start from 1 rather than 0, so I had been adding +1 when indexing the vertex data. But when importing PLY files, I forgot to apply this adjustment, meaning every vertex was shifted by one position.

So that’s why my ton_Roosendaal_smooth result ended up looking unintentionally artistic (and terrifying) :).
<p align="center">
<img alt="ton_Roosendaal_smooth" src="https://github.com/user-attachments/assets/b01e97a9-fc6b-41f3-a756-583a9d2b9461" />
</p>

Another issue, as shown below, was with the Chinese dragon model. The potential cause for this problem actually came to my mind while Oğuz Hoca was explaining multisampling in class. Before starting the topic, he mentioned that since we shoot only a single ray per pixel, and we cast it directly through the center, we can miss intersections when there is a very small triangle or object located within that pixel.
<p align="center">
<img alt="chinese dragon bug" src="https://github.com/user-attachments/assets/8eb29623-5e2f-463d-8507-d6b27c3c4704" />
</p>

At that moment, I remembered that the Chinese dragon model has extremely small vertex coordinates, and I realized that this was most likely the reason behind the issue I was seeing. I haven’t found a proper solution yet, other than the brute-force approach of simply scaling up the coordinate values, but at least now I understand the root cause of the problem.

Alright, now let's get back to this week's requirements. Before starting, I should have mentioned that I used the [GLM library](https://github.com/g-truc/glm) for necessary vector and matrix operations for transformations because they are well optimized and easy to use.

## Transformations
Transformations will allow us to position, rotate, and scale objects within the scene without modifying their original geometry. 

I started with transformations for both cameras and point lights before moving on to objects. Camera can be rotated, translated, and scaled, but point lights can only be translated because scaling and rotation do not make sense for point lights.

```cpp
// Apply transformations to cameras
for (auto& cam : scene.cameras) {
    // Apply transformation to position vector
    glm::vec4 pos = cam.modelMatrix * glm::vec4(cam.position.x, cam.position.y, cam.position.z, 1.0f);
    cam.position = Vec3(pos.x, pos.y, pos.z);

    // Apply transformation to gaze and up vectors
    glm::vec4 gazeDir = cam.modelMatrix * glm::vec4(cam.gaze.x, cam.gaze.y, cam.gaze.z, 0.0f);
    glm::vec4 upDir = cam.modelMatrix * glm::vec4(cam.up.x, cam.up.y, cam.up.z, 0.0f);

    cam.gaze = Vec3(gazeDir.x, gazeDir.y, gazeDir.z).normalize();
    cam.up = Vec3(upDir.x, upDir.y, upDir.z).normalize();
}

// Apply transformations to point lights
for (auto& light : scene.pointLights) {
    // Apply transformation to position vector
    glm::vec4 transformed = light.modelMatrix * glm::vec4(light.position.x, light.position.y, light.position.z, 1.0f);
    light.position = Vec3(transformed.x, transformed.y, transformed.z);
}
```

Transforming cameras and point lights was straightforward, but things got a bit more complicated when it came to objects.
At first, I considered simply applying transformations to all objects directly, which is probably what most people think of initially. However, I quickly realized that this approach wouldn’t work as expected. In order to transform an object properly, I would need to separate the vertex data per object.

After applying a transformation, the modified vertex positions must be stored in independent vertex buffers, since the original vertex data might be shared among multiple objects. If I overwrite vertex data globally, transforming one object could unintentionally affect another that references the same mesh.

Of course, there had to be a *better* approach.

During the lectures, we discussed the idea of transforming rays instead of objects, which cleanly solves this problem.
While I understood the core concept in class, I grasped it much more clearly after reading [Eric Arnebäck’s excellent blog post](https://erkaman.github.io/posts/ray_trace_inverse_ray.html). His visual explanations helped me to understand how inverse transformations can be used to bring rays into an object’s local space.

Given a ray $$r(t) = o + tD$$ in world space,  
we can transform it into object space using the inverse of the object’s model matrix:


$$o' = M^{-1} o, \quad D' = M^{-1} D $$


```cpp
glm::mat4 invModel = glm::inverse(object.modelMatrix);
glm::vec4 localOrigin = invModel * glm::vec4(ray.origin.x, ray.origin.y, ray.origin.z, 1.0f);
glm::vec4 localDir    = invModel * glm::vec4(ray.direction.x, ray.direction.y, ray.direction.z, 0.0f);
```

After finding the intersection point in the object’s local space, I needed to transform the surface normal back into world space.
This part is subtle but essential, normals cannot be transformed with the same matrix as positions, because non-uniform scaling would distort them. Instead, they must be multiplied by the inverse transpose of the model matrix:

```cpp
glm::vec3 localN = glm::normalize(localHit - glm::vec3(centerWorld));
glm::vec3 worldN = glm::normalize(
    glm::vec3(glm::transpose(glm::inverse(M)) * glm::vec4(localN, 0.0f))
);
```

This was a small but crucial step to maintain accurate shading and lighting calculations.

In my implementation, I applied this inverse ray method only to spheres, since it’s cheap for simple primitives and effective for ellipsoid like shapes. For triangles and meshes, I instead transform their vertices into world space once before rendering, avoiding the cost of applying inverse matrices for every ray. Also, unlike spheres, there is no concept of an “ellipsoid” case for triangles or polygonal meshes.

After adding the transformation support, I tested the scene with the mirror_room.json input and got the following result:

<p align="center">
<img alt="mirror_room_noise" src="https://github.com/user-attachments/assets/b7e16376-1dbc-4844-b04d-871b1a589fa4" />
</p>

The noise immediately hinted that something was wrong with my shadow rays. Because it was looking like self-intersection noise. To confirm this, I temporarily disabled shadow ray computations and the noise disappeared. This clearly meant that the issue was somewhere inside my isInShadow() function.

```cpp
static bool isInShadow(const Scene& scene, const Vec3& shadowRayOrigin, const Vec3& shadowRayDir, float lightDistance) {
    IntersectionInfo info = findClosestIntersection(scene, shadowRayOrigin, shadowRayDir, true);

    if (info.hit) {
        if (info.t > 0.0f && info.t < lightDistance) {
            return true; // It is in shadow
        }
    }

    return false; // Not in shadow
}
```

The problem was caused by floating-point precision near the light and surface intersection points. When the shadow ray started exactly on a surface, it could re-intersect the same geometry due to numerical error, producing random noise on surfaces.
In my code I was already adding a small epsilon offset to the shadow ray origin inside findClosestIntersection, but I forgot to include the same epsilon in the distance comparison inside isInShadow.
To fix this, I added shadow ray epsilon both at the origin and in the comparison.

```cpp
static bool isInShadow(const Scene& scene, const Vec3& shadowRayOrigin, const Vec3& shadowRayDir, float lightDistance) {
    IntersectionInfo info = findClosestIntersection(scene, shadowRayOrigin, shadowRayDir, true);

    if (info.hit) {
        if (info.t > scene.shadowRayEpsilon && info.t < lightDistance - scene.shadowRayEpsilon) {
            return true; // It is in shadow
        }
    }

    return false; // Not in shadow
}
```

After this change, the noise completely disappeared, and the reflection looked clean and correct:

<p align="center">
<img alt="mirror_room_fixed" src="https://github.com/user-attachments/assets/7582c115-a49e-4710-99d5-1321c72b4a2f" />
</p>

### Mesh Instancing
A mesh instance is not a new mesh, it’s simply a reference to an existing base mesh that can have its own transformation and material. Instead of duplicating the same vertex data multiple times in memory, each instance stores only a transformation matrix and an optional material, allowing the same geometry to appear multiple times in the scene at different positions, orientations, or scales.

When I first learned about mesh instancing in class, it immediately reminded me of the static batching concept in Unity ([this video](https://www.youtube.com/watch?v=8D8A6gabwlU) by LlamAcademy perfectly explains it).
In the Unity engine, if multiple GameObjects in a scene share the same mesh, they can be rendered with a single draw call.
This improves performance by reducing the number of calls made to the GPU.

Similarly, mesh instancing in ray tracing also reuses the same mesh data multiple times, but instead of duplicating geometry, it applies different transformation matrices to a single shared copy of the mesh. However, unlike Unity’s batching, it doesn’t provide the same level of performance gain, because in ray tracing, each instance still requires its own intersection tests. In other words, even though the geometry is shared, the rays must be tested against every instance individually. But still, it’s a great way to save memory and manage complex scenes with many repeated objects.

Basically, mesh instancing process was looking like this:

```cpp
for (const auto& instance : scene.objects.meshInstances) {
    const Mesh* baseMesh = instance.resolvedBaseMesh; // Get the base mesh
    glm::mat4 M = instance.modelMatrix; // Instance transformation matrix of instance

    // Transform vertex positions into world space
    Vec3 v0 = Vec3(M * glm::vec4(baseMesh->vertices[i0], 1.0f));
    Vec3 v1 = Vec3(M * glm::vec4(baseMesh->vertices[i1], 1.0f));
    Vec3 v2 = Vec3(M * glm::vec4(baseMesh->vertices[i2], 1.0f));

    // Perform intersection test using the transformed vertices
    if (RayTriangle(rayOrigin, rayDir, v0, v1, v2, t, !isShadowRay, triN)) {
        // Use instance.material for shading if provided
        // ...
    }
}
```

Here, each mesh instance applies its own transformation matrix before testing for intersections. The base mesh data stays the same, only the transform changes per instance. 

For smooth shading, I updated the normal computation for mesh instances just like I did for regular meshes. To test my changes, I used the two_berserkers.json scene, since it contained both transformed mesh instances and smooth shading.
It was the perfect test case to verify whether my normal interpolation and transformation logic worked correctly.

However, the result wasn’t quite what I expected:

<p align="center">
<img alt="two_berserkers_strange_smoothing" src="https://github.com/user-attachments/assets/8c1a0d81-35bb-459a-bd22-885b0897f48b" />
</p>
At first, I spent a long time searching for the cause of the issue inside the smooth shading logic.
Everything seemed correct, also the base mesh rendered perfectly with smooth shading, and the interpolated normals looked fine.

Then I remembered something we discussed during the lectures: the mirror scaling problem, where an object’s vertex order flips from counter-clockwise (CCW) to clockwise (CW).

Just like in the lecture example, imagine a triangle with vertices a, b, and c, where a is the top vertex, b is the bottom-left, and c is the bottom-right.
Normally, we compute the triangle’s normal using:

$$N=(b−a)×(c−a)$$

In this case, the triangle is defined in counter-clockwise (CCW) order, so the normal correctly points outward (towards the viewer).

However, if we apply a mirror scaling for example, scaling by -1 along the Y-axis, the vertex order effectively flips to clockwise (CW).
Now, the same cross product 

$$(b−a)×(c−a)$$ produces a normal pointing in the opposite direction.

Here is my (not so well :)) drawing to illustrate this:

![CENG795 Drawing](https://github.com/user-attachments/assets/7219b87f-72ee-42e0-95d1-ed6275418d74)


As a result, the normal vectors were inverted, which led to incorrect results in several parts, most noticeably in back-face culling, where some visible faces were being incorrectly discarded. 

To confirm this, I temporarily disabled back-face culling, and as expected, the result immediately made sense:

<p align="center">
<img alt="two_berserkers" src="https://github.com/user-attachments/assets/c700aaa1-ac39-48e7-8857-af521bfecc6d" />
</p>
Since back-face culling improves performance in most cases, disabling it globally wasn’t a good solution.
Instead, I added a small check to automatically detect when an object is mirrored and selectively disable culling only in those cases:

```cpp
glm::mat4 M = instance.modelMatrix;

// if determinant is negative, it's a mirrored transform (0, -1, 0 scale for example)
bool isMirrored = glm::determinant(M) < 0;

bool performCulling = !isShadowRay && !isMirrored;
```

This way, back-face culling remains enabled for regular objects to keep rendering efficient, but it’s automatically disabled for mirrored mesh instances (and, of course, for shadow rays), ensuring the image renders correctly without flipped normals.

### Look At Camera
Look at camera is a common camera model that allows you to specify a target point in the scene. They use a vertical field of view (fovY) instead of an explicit near-plane definition. Some scenes, such as dragon_metal.json, required this setup. 

Given the distance from the camera to the near plane (nearDist), the top edge of the near plane can be computed as:

$$top = nearDist * tan(fovY / 2)$$

Since the camera's frustum is symmetric:
bottom = -top, right = top * aspect, left = -right

As implemented in my code:

```cpp
if (cam.hasFovY) { // look-at camera
    float fovY = glm::radians(cam.fovY); // convert to radians
    float aspect = (float) width / (float) height; // aspect ratio
    top = nearDist * tan(fovY / 2.0f); // top of the near plane
    bottom = -top; // bottom of the near plane
    right = top * aspect; // right of the near plane
    left = -right; // left of the near plane
} else { // using NearPlane values
    left = cam.nearPlane[0];
    right = cam.nearPlane[1];
    bottom = cam.nearPlane[2];
    top = cam.nearPlane[3];
}
```

### Bounding Volume Hierarchies (BVH)
Bounding Volume Hierarchies (BVH) are the most popular acceleration structure used in ray tracing to speed up intersection tests. By organizing the scene's geometry into a tree of bounding volumes, we can quickly eliminate large portions of the scene from consideration when tracing rays. [Sebastian Lague's this video](https://www.youtube.com/watch?v=C1H4zIiCOaI) about BVH construction and traversal was very helpful in understanding the concept visually.

Basic idea is to recursively divide the scene's geometry into two groups, creating a binary tree where each node contains a bounding box that encloses all the geometry in its subtree. Before ray tracing begins, we build the BVH by partitioning the objects based on their spatial distribution. When tracing a ray, we first test for intersection with the bounding boxes, and only if the ray intersects a box do we proceed to test the actual geometry inside.

During BVH construction, I recursively split the set of triangles along the longest axis of their bounding box:

```cpp    
// Compute bounding box for this node
for (int triIdx : tris) {
    // Getting vertex indices i0, i1, i2 for the triangle
    // ...

    // Expand the bounding box to include the triangle vertices
    node->bbox.expand(vertices[i0]);
    node->bbox.expand(vertices[i1]);
    node->bbox.expand(vertices[i2]);
}

// Base case: if few triangles or max depth reached, make leaf
if (tris.size() <= 4 || depth > 25) { // 25 is arbitrary max depth
    node->triangleIndices = tris;
    return node;
}

// Then split along the longest axis 
// ...
```

When tracing a ray, I first test it against the node’s bounding box. If it intersects, the ray is tested against the BVH instead of every triangle individually:

```cpp
for (const auto& mesh : scene.objects.meshes) {
    if (mesh.bvh) {
        if (mesh.bvh->intersect(rayOrigin, rayDir, tCandidate, info)) {
            // Necessary setting of intersection info
            // ...
        }
        continue; // Skip per-triangle tests if BVH is present
    }
}
```

Without BVH, I was checking each ray against every triangle in the scene, which resulted in O(n) complexity per ray. With BVH, the average complexity drops to roughly O(log n), so when triangle counts are high, the performance improvement becomes more and more significant. Performance gains can be seen in the following table:
    
| Scene | Time Without BVH (sec) | Time With BVH (sec) |
|-------|------------------|----------------|
| bunny_with_plane      | 229.144  | 4.05434     |
| Car_front_smooth      | 632.8    | 4.59678     |
| Car_smooth            | 314.33   | 6.93229     |
| two_berserkers          | 822.561    | 1.438    |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

Although render times are decreased with BVH, in some complex scenes like marching_dragons, BVH construction time itself becomes significant. It is still nothing compared no BVH case, but it also comes with a cost. For example, in marching_dragons scene, BVH construction took around 21.2038 seconds, while the actual rendering took only 5.67193 seconds. In the rendering time table below, I also included the BVH construction times for better comparison between other parts.

Even though BVH works, I could not manage to share BVH structures between mesh instances yet. So each instance still has to build its own BVH, which is not optimal. I will try to improve this in the future parts.

### Outputs
As I mentioned in the beginning, chinese dragon model had very small vertex coordinates, which caused intersection issues. I could not find a proper solution for this yet, therefore results for marching_dragons is still not perfect. Also, in dragon_new_ply.json input, I could not get a result after 45 minutes of rendering even though I confirmed it creates BVH correctly. I will investigate these issues further in the next part. And in mirror_room.json I get an additional light in compared to the reference image, it is fixed when I remove one of the point lights.

Other than that, everything else seems to be working fine now.

As in previous part, I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance, and Akif Uslu for contributions to the 3D models.

You can see the rendering times of this part below:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| dragon_metal                 | 15.0101  |
| ellipsoids                 | 2.13959  |
| marching_dragons                 | 26.2038  |
| metal_glass_plates                 | 7.52798  |
| mirror_room                 | 25.5161  |
| simple_transform                 | 0.454879  |
| spheres                 | 1.39373  |
| glaring_davids                 | 1.60047  |
| two_berserkers                 | 1.20802  |
| grass_desert                 | 41.2385  |

*Used CPU: AMD Ryzen 5 5600X 6-Core Processor (3.70 GHz)*

Also, I merged raven input json files' render into a .mp4 video (using [FFmpeg](https://www.ffmpeg.org/)). Each frame nearly took 2 seconds to render.

---

### Camera Around David
<p align="center">
<img alt="output1" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/videos/output.mp4" />
</p>

<video>
  <source src="videos/output.mp4">
</video>

![video](https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/videos/output.mp4)
---

### Light Around David
<p align="center">
<img alt="output2" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/videos/output2.mp4" />
</p>
---

Here are some renders from test scenes:

---

### Dragon Metal
_Time: 15.0101 s_
<p align="center">
<img alt="dragon_metal" src="https://github.com/user-attachments/assets/18cabe23-5584-494a-b279-a929197088d2" />
</p>

---

### Ellipsoids
_Time: 2.13959 s_
<p align="center">
<img alt="ellipsoids" src="https://github.com/user-attachments/assets/7a4cf67e-78c0-46cd-aeb8-14f9d9dcd98b" />
</p>

---

### Marching Dragons
_Time: 26.2038 s_
<p align="center">
<img alt="marching_dragons" src="https://github.com/user-attachments/assets/921bc7da-f1d8-4b0d-b63c-66242e73c768" />
</p>

---

### Metal Glass Plates
_Time: 7.52798 s_
<p align="center">
<img alt="metal_glass_plates" src="https://github.com/user-attachments/assets/97f70f62-65c0-438c-9def-2576096ded8b" />
</p>

---

### Mirror Room
_Time: 25.5161 s_
<p align="center">
<img alt="mirror_room" src="https://github.com/user-attachments/assets/f6ac9dcc-f734-4773-8459-943f407280b2" />
</p>

---

### Simple Transform
_Time: 0.454879 s_
<p align="center">
<img alt="simple_transform" src="https://github.com/user-attachments/assets/4457a564-dcec-4e4d-bb93-2c331083f274" />
</p>

---

### Spheres
_Time: 1.39373 s_
<p align="center">
<img alt="spheres" src="https://github.com/user-attachments/assets/1ae73c89-2038-4f14-9989-c9932a8060b3" />
</p>

---

### Glaring Davids
_Time: 1.60047 s_
<p align="center">
<img alt="glaring_davids" src="https://github.com/user-attachments/assets/e948810f-aba6-4556-969c-732fb972f586" />
</p>

---

### Two Berserkers
_Time: 1.20802 s_
<p align="center">
<img alt="two_berserkers" src="https://github.com/user-attachments/assets/d047c863-9015-4f23-b739-affc13b9104f" />
</p>

---

### Grass Desert
_Time: 41.2385 s_
<p align="center">
<img alt="grass_desert" src="https://github.com/user-attachments/assets/006820bc-e684-4bcb-ad98-60bb7b706015" />
</p>

---

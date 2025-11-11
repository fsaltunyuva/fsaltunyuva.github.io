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

### Bugs on Part 1
But, talking about new features, I would like to mention about the bugs I have encountered on Part 1 and understood some of them. Let's start with PLY import bug. 

[TON ROOSENDAL SCARY RENDER]

I had been suspecting that the strange results I was getting with the PLY models were due to incorrectly computed normals or the vertex data being interpreted differently than I expected. However, it turned out the issue was much simpler than that: in the JSON scenes we use, the vertex indices start from 1 rather than 0, so I had been adding +1 when indexing the vertex data. But when importing PLY files, I forgot to apply this adjustment, meaning every vertex was shifted by one position.

So that’s why my ton_Roosendaal_smooth result ended up looking unintentionally artistic (and terrifying) :).

[TON ROOSENDAL FIXED RENDER]

Another issue, as shown below, was with the Chinese dragon model. The potential cause for this problem actually came to my mind while Oğuz Hoca was explaining multisampling in class. Before starting the topic, he mentioned that since we shoot only a single ray per pixel, and we cast it directly through the center we can miss intersections when there is a very small triangle or object located within that pixel.

[CHINESE DRAGON DOTTED]

At that moment, I remembered that the Chinese dragon model has extremely small vertex coordinates, and I realized that this was most likely the reason behind the issue I was seeing. I haven’t found a proper solution yet, other than the brute-force approach of simply scaling up the coordinate values, but at least now I understand the root cause of the problem.

Alright, now let's get back to this week's requirements. Before starting I should have mentioned that I used the [GLM library](https://github.com/g-truc/glm) for necessary vector and matrix operations for transformations because they are well optimized and easy to use.

## Transformations
Transformations will allow us to position, rotate, and scale objects within the scene without modifying their original geometry. 

[BELKİ BİR ÖRNEK]

I started with transformations for both cameras and point lights before moving on to objects. Camera can be rotated, translated, and scaled but point lights can only be translated because scaling and rotation do not make sense for point lights.

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

Given a ray \( r(t) = O + tD \) in world space,  
we can transform it into object space using the inverse of the object’s model matrix:

\[
O' = M^{-1} O, \quad D' = M^{-1} D
\]

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

[mirror_room_noise.png]

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
In my code I was already adding a small epsilon offset to the shadow ray origin inside findClosestIntersection, but I forgot to include the same epsilon in the distance comparison inside isInShadow. As a result, even with the offset origin, hits with t ≈ 0 or t ≈ lightDistance were still being counted as occluders.
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

[mirror_room_fixed.png]

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

[two_berserkers_strange_smoothing.png]

At first, I spent a long time searching for the cause of the issue inside the smooth shading logic.
Everything seemed correct, also the base mesh rendered perfectly with smooth shading, and the interpolated normals looked fine.

Then I remembered something we discussed during the lectures: the mirror scaling problem, where an object’s vertex order flips from counter-clockwise (CCW) to clockwise (CW).

Just like in the lecture example, imagine a triangle with vertices a, b, and c, where a is the top vertex, b is the bottom-left, and c is the bottom-right.
Normally, we compute the triangle’s normal using:

N=(b−a)×(c−a)

In this case, the triangle is defined in counter-clockwise (CCW) order, so the normal correctly points outward (towards the viewer).

However, if we apply a mirror scaling for example, scaling by -1 along the Y-axis, the vertex order effectively flips to clockwise (CW).
Now, the same cross product 

(b−a)×(c−a) produces a normal pointing in the opposite direction.

Here is my (not so well :)) drawing to illustrate this:

[PAINTTEN ÇİZDİĞİN VEYA TABLETTEN YENİ ÇİZDİĞİN]

As a result, the normal vectors were inverted, which led to incorrect results in several parts, most noticeably in back-face culling, where some visible faces were being incorrectly discarded. 

To confirm this, I temporarily disabled back-face culling, and as expected, the result immediately made sense:

[two_berserkers.png]

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

top = nearDist * tan(fovY / 2)

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
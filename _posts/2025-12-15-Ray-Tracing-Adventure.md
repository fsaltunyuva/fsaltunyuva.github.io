---
layout: post
title: "Ray Tracing Adventure Part 4 (Homework 4)"
date: 2025-12-15
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 4, focusing on implementing features:

- **Texture Mapping**
- **Normal and Bump Mapping**
- **Diffuse and Specular Maps**
- **Perlin Noise**
- **Checkerboard Patterns**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz, at Middle East Technical University.

### Bugs from Previous Parts 
As I mentioned in previous sections, my renders were showing excessive reflections, making the scene appear more reflective than it actually was. I easily found the source of the problem by referring to a blog post where Professor Oğuz answered. I was applying reflection directly if any coordinate value of the material's mirror reflectance vector was greater than 0, but it turned out that mirror reflectance should only be applied if the material's _type value is 'mirror'. As Oğuz Hoca indicated, checking via _type solved the problem, and the renders were the same as expected.

```cpp
// else if (material.mirrorReflectance.x > 0 || material.mirrorReflectance.y > 0 || material.mirrorReflectance.z > 0)
else if (material.type == "mirror")
```
![car reflectance GIF](https://github.com/user-attachments/assets/9b708cb1-33fa-4f82-9d08-5cbaed8501d6)

[belki beers law]

### Texture Mapping
Texture mapping is a fundamental technique in computer graphics used to add surface detail to 3D models. Instead of assigning a single flat color to an object, we map a 2D image (or a procedural pattern) onto the 3D surface using UV coordinates. This allows us to simulate complex materials like wood, stone, or earth without increasing the geometric complexity of the mesh. This technique is widely used in video games because it give good looking results without requiring high processing power by keeping the geometry simple even though the textures can be very detailed.

There are two main types of texture mapping I implemented: Image Textures and Procedural Textures. Image textures use images like PNG to define surface colors, while procedural textures generate patterns algorithmically (like Perlin Noise for some patterns as in sphere_perlin_bump render).

Also there are different types of texture maps that serve various purposes, Normal Maps and Bump Maps modify surface normals to simulate detail, while Diffuse and Specular Maps define how light interacts with the surface. These maps usually looks like this:

[eart and eartnormalmap image]

[woodbox diffuse and specular map image]

We map textures using UV coordinates, which are 2D coordinates that correspond to points on the texture image. The U coordinate typically runs horizontally across the texture, while the V coordinate runs vertically. These coordinates are normalized between 0 and 1, and they are assigned to each vertex of the 3D model. During rendering, the UV coordinates are interpolated across the surface of the polygon to determine which part of the texture image corresponds to each pixel on the screen.

But choosing which texel (texture element) to sample is crucial for quality. Many games or game engines ask the user for different sampling methods to avoid artifacts like aliasing or blurriness. For example in Unity, you can choose which method to use for texture filtering.

[Unity Texture Filtering Options Image]

There are several sampling techniques, but I focused on two primary methods: Nearest Neighbor and Bilinear Interpolation.

#### Nearest Neighbor
The Nearest Neighbor method is the simplest way to sample a texture. It involves rounding the U and V coordinates to the nearest integer pixel in the texture image. This method is fast and easy to implement, but it can lead to noticeable artifacts, especially when the texture is viewed up close or at steep angles. The result can appear blocky or pixelated because it doesn't consider any neighboring texels.

I implemented Nearest Neighbor sampling as follows:

```cpp
Vec3 Image::sampleNearest(const Vec2& uv) const {
    if (width <= 0 || height <= 0 || pixels.empty()) return Vec3(0,0,0);

    // Clamp [0,1]
    float u = clamp01(uv.u);
    float v = clamp01(uv.v);

    v = 1.0f - v; // Flip V coordinate for image origin at top-left

    // Find nearest texel (pixel center)
    int x = (int) std::floor(u * (width - 1) + 0.5f);
    int y = (int) std::floor(v * (height - 1) + 0.5f);

    // Clamp to image boundaries
    x = std::max(0, std::min(width - 1, x));
    y = std::max(0, std::min(height - 1, y));

    // Get pixel color
    int idx = (y * width + x) * channels;
    float r = pixels[idx + 0] / 255.0f;
    float g = pixels[idx + 1] / 255.0f;
    float b = pixels[idx + 2] / 255.0f;

    return Vec3(r, g, b);
}
```

When I run this, I see a render like this and I thought that it is the same as expected result.

[Image.cpp samplenearest problem fix commit, bugged image]

Then after some time, I also wanted to test my results with a comparing tool, and I found out that my implementation was slightly off.

[Image.cpp samplenearest problem fix commit, original image]

Upon closer inspection, I realized the culprit was a standard convention I had blindly adopted: ```v = 1.0f - v;```.

The problem was a simple assumption I made about where the "starting point" of the image was. I assumed that the texture coordinates (0,0) should start at the top-left corner of the image. To match this assumption, I added the line v = 1.0f - v; to flip the vertical coordinate upside down.

It turned out my assumption was wrong. The input data was already set up correctly, so by flipping the V coordinate, I was actually breaking the alignment by inverting it unnecessarily. So I removed that line, and got the correct results.

Even though this method is simple to implement and solves the problem, for image textures, simply picking the nearest pixel can result in "blocky" artifacts when the camera is close to the surface.

[Nearest Neighbor Example Image]

### Bilinear Interpolation

To solve this, we can use Bilinear Interpolation. In this method, instead of taking the single nearest texel, we sample the four surrounding texels and interpolated their colors based on the exact UV position.

[https://www.gamedevelopment.blog/wp-content/uploads/2017/11/nearest-vs-linear-texture-filter.png]

Here is how I calculated the weighted average of the four neighbors:

```cpp
// Calculate weights based on position within the pixel grid
float dx = x_coord - x0;
float dy = y_coord - y0;

// ... fetch c00, c10, c01, c11 neighbors ...

// Interpolate along X
Vec3 top = c00.scale(1.0f - dx).add(c10.scale(dx));
Vec3 bot = c01.scale(1.0f - dx).add(c11.scale(dx));

// Interpolate along Y for the final color
return top.scale(1.0f - dy).add(bot.scale(dy));
```

We can also see the difference in my renders between Nearest Neighbor and Bilinear Interpolation in the `plane_nearest` and `plane_bilinear` renders.

[Bilinear Interpolation Example Image]

There are more advanced techniques like Mipmapping and Anisotropic Filtering, but for this project, Nearest Neighbor and Bilinear Interpolation were sufficient for me to achieve good quality textures.

### Bump Mapping
While texture mapping changes the color, Bump Mapping changes how light interacts with the surface to simulate depth and wrinkles. It works by perturbing the surface normal based on the intensity (or height) of a texture map. This creates the illusion of geometric detail, like the ridges on a spaceship or the grout between tiles, without actually moving any vertices.

[Bump Mapping Örneği]

To implement this, I had to calculate the gradient (slope) of the texture at the hit point. Crucially, this requires taking the derivative of the texture height in the U and V directions. By moving a tiny amount (du, dv) in texture space, I calculated the new normal relative to the surface's tangent space (TBN):

```cpp
// Calculate derivatives (gradients)
float dH_du = (h_u - h) / du;
float dH_dv = (h_v - h) / dv;

// Scale by bump factor
float displacement_u = dH_du * bumpTM->bumpFactor;
float displacement_v = dH_dv * bumpTM->bumpFactor;

// Perturb the normal in Tangent Space
Vec3 nTS = Vec3(-displacement_u, -displacement_v, 1.0f).normalize();
```

In the code snippet above, you might have noticed the bumpFactor variable. This is a crucial parameter because the raw intensity values from a texture image (0 to 255) do not have an inherent physical height. Without a scaling factor, the calculated gradients might be too steep or too shallow for the object's scale. While a high value creates deep grooves and sharp ridges, making the surface look very rough, a low value creates subtle imperfections, like the grain on wood or slight scratches on metal.

Not all scenes included bump factor value, so I used 0.01f as a default value for those scenes. Here you can see some different bump factor values and their effects on the final render.

[Bump Factor Comparison Image]

### Diffuse & Specular Reflectance Mapping
While Bump Mapping handles the geometry's "feel," Diffuse Mapping handles its "look." This is the most common form of texturing, where an image is mapped onto the 3D surface to define its color.

In my implementation, I supported two distinct modes for diffuse textures, controlled by the DecalMode enum: Replace and Blend. In Replace mode, the texture completely overrides the material's base color. This is used when the texture represents the object's skin entirely, like the map of the Earth or a wooden floor. In Blend mode, the texture color is averaged with the material's existing color. This is useful for adding detail to a base material without losing its original hue.

However, realistic materials are rarely uniform in their shininess. A rusty metal plate, for example, is shiny where the metal is exposed but dull where the rust has formed. To simulate this, I implemented Specular Mapping.

A specular map (usually a grayscale image) controls the specular coefficient. Brighter parts of the texture make the surface more reflective to light, while darker parts make it matte.

Here is how I integrated these modes into the shading loop. Before calculating the Phong lighting model, I check the texture mode and update the material coefficients ($k_d$ and $k_s$) dynamically:

```cpp
// Sample the texture color (using bilinear or nearest)
Vec3 texColor = tm->image->sample(info.hitUV);

// Apply the Decal Mode logic
if (tm->decalMode == DecalMode::ReplaceKd) {
    // Completely replace diffuse color
    kd = texColor;
} 
else if (tm->decalMode == DecalMode::BlendKd) {
    // Average the texture with the base material color
    kd = kd.add(texColor).scale(0.5f);
} 
else if (tm->decalMode == DecalMode::ReplaceKs) {
    // Update specular coefficient based on texture intensity
    ks = texColor;
}
```

By modifying kd and ks before the lighting calculation, the rest of the ray tracer (shadows, light attenuation, etc.) works automatically with these new, detailed material properties.

### Perlin Noise
Finally, I implemented procedural generation using the most popular procedural texture technique,
Perlin Noise. Unlike images, procedural textures are calculated on the fly using mathematical functions. Perlin Noise is a gradient noise function that produces smooth, natural-looking patterns. It is widely used for simulating organic textures like clouds, marble, wood grain, and terrain. Because of the easiness and amazing results of Perlin Noise, Ken Perlin even won an Academy Award for Technical Achievement for his invention in 1997.

[Perlin Noise Örneği]

[The video "What is Perlin Noise?" by Acerola](https://www.youtube.com/watch?v=DxUY42r_6Cg) about the Perlin Noise was very fun and informative to watch, it helped me to understand the concept better and, I strongly recommend it to anyone interested.

Perlin Noise works by defining a grid of random gradient vectors and interpolating between them based on the input coordinates. This creates a continuous noise function that can be sampled at any point in 3D space. 

I used an existing implementation of Perlin Noise from [here](https://mrl.nyu.edu/~perlin/noise/) and integrated it into my ray tracer.

In original Perlin Noise, we sum multiple octaves of noise to create fractal patterns. Each octave has a different frequency and amplitude, allowing for both large-scale and fine details.

I implemented support for "Turbulence," which creates marble-like or "snake-like" veins. The key difference here is taking the absolute value of the noise before accumulating it into the sum. This creates sharp creases in the pattern where the value crosses zero.

```cpp
for (int k = 0; k < K; ++k) {
    Vec3 q = p.scale(freq);
    float n = gPerlin.noise(q.x, q.y, q.z); 

    if (tm.conversion == NoiseConversion::AbsVal) {
        // Absolute value for turbulence (snake-like patterns)
        s += amp * std::fabs(n);
    } else {
        // Standard summation (cloud-like patterns)
        s += amp * n;
    }
    
    ampSum += amp;
    freq *= 2.0f;
    amp *= 0.5f;
}
```

### Normalizer

There is also a field named "Normalizer" in input files for some textures. I interpreted this as a divisor for the standard 8-bit color range, applying a scaling factor of 255.0f / Normalizer.

```cpp
if (tm->normalizer != 255.0f && tm->normalizer > 0.0f) {
    float scaleFactor = 255.0f / tm->normalizer;
    texColor = texColor.scale(scaleFactor);
}
```

But after trying this on ellipsoids_texture scene, I realized that something is wrong. Also, at this time, I was trying to figure out why the painting on the wall is completely white in the veachajar scene.

[veachajar wall bugged image]

[ellipsoids_texture normalizer bugged image]

When I checked the Normalizer value in the ellipsoids_texture scene, it was set to 1.0f, which means no scaling should be applied. However, I mistakenly applied the scaling factor regardless of the Normalizer value, thus multiplying the texture colors by 255.0f, leading to completely white colors. So I also checked for the 1f value and skipped scaling in that case, which fixed the problem.

```cpp
if (tm->normalizer > 0.0f && tm->normalizer != 255.0f && tm->normalizer != 1.0f) {
    float scaleFactor = 255.0f / tm->normalizer;
    texColor = texColor.scale(scaleFactor);
}
```

### Outputs and Closing Thoughts
Even though this part was including straightforward implementations of well-known techniques, testing the features and debugging them took a lot of time because wrong texture mapping results cannot be easily diagnosed by human eyes.

[Wood box wrong texture mapping example]

Therefore I used digital tools a lot for debugging, such as [Diffchecker](https://www.diffchecker.com/image-compare/).

Sphere inputs was not including bump factor value, so I tried different values to see which one is giving the expected results and used 10 as the bump factor for those scenes. Also, in veachajar scene, I got a PLY read error while importing the models/Mesh015_fixed.ply file, but when I used the original models/Mesh015.ply file it worked fine, The [happly](https://github.com/nmwsharp/happly) library was giving an error about unsigned int usage in the fixed file, so I just used the original file.

Other than these, I get somewhat different result in killeroo_bump_walls scene, I could not figure out the exact reason, but I thought it might be related to bump factor but even different bump factor values did not result in an exact match with expected output, I will investigate this issue further in the next parts. Here are my final renders and their render times:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| brickwall_with_normalmap                 | 1.60214  |
| bump_mapping_transformed                 | 3.24855  |
| cube_cushion                 | 1.53794  |
| cube_perlin                 | 1.47713  |
| cube_perlin_bump                 | 1.56417  |
| cube_wall                 | 1.57439   |
| cube_wall_normal                 | 1.566830  |
| cube_waves                 | 1.63458  |
| ellipsoids_texture                 | 2.92964  |
| galactica_dynamic                 | 449.681  |
| galactica_static                 | 5.75577  |
| killeroo_bump_walls                 | 26.6139  |
| plane_bilinear                 | 1.09835  |
| plane_nearest                 | 1.13796  |
| sphere_nearest_bilinear                 | 2.16499  |
| sphere_nobump_bump                 | 1.9995  |
| sphere_nobump_justbump                 | 1.94817  |
| sphere_normal                 | 117.432  |
| sphere_perlin                 | 2.4421  |
| sphere_perlin_bump                 | 2.55343  |
| sphere_perlin_scale                 | 2.40968  |
| wood_box                 | 1.58755  |
| wood_box_all                 | 1.60994  |
| wood_box_no_specular                 | 1.5577  |
| dragon_new                 | 14394.7* (3.686.400 pixels vs. my ray tracer :))  |
| mytap_final                 | PLY Read Error  |
| 1 frame of tunnel_of_doom (400x400 resolution to speed up)                 | (approximately) 14*  |
| VeachAjar                 | 174.211  |



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

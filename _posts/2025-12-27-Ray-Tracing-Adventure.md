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


### Outputs and Closing Thoughts


As in previous parts, I would like to thank Professor Ahmet Oğuz Akyüz for all the course materials and guidance, and Akın Aydemir for contributions to the 3D models. Here are my final renders and their render times:

| Scene                 | Time (seconds) |
| --------------------- | -------- |
| cube_directional                 | 1.69274  |
| cube_point                 | 1.75236  |
| cube_point_hdr                 | 2.98016  |
| dragon_spot_light_msaa                 | 83.7131  |
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

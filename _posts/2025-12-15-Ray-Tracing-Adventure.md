---
layout: post
title: "Ray Tracing Adventure Part 4 (Homework 4)"
date: 2025-12-15
categories: [ray-tracing, graphics, adventure]
---

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
<img alt="dragon_new" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/dragon_new.png" />
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
<img alt="brickwall_with_normalmap" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/brickwall_with_normalmap.png" />
</p>

---
### bump_mapping_transformed
_Time: 3.24855 s_
<p align="center">
<img alt="bump_mapping_transformed" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/bump_mapping_transformed.png" />
</p>

---
### cube_cushion
_Time: 1.53794 s_
<p align="center">
<img alt="cube_cushion" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_cushion.png" />
</p>

---
### cube_perlin
_Time: 1.47713 s_
<p align="center">
<img alt="cube_perlin" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_perlin.png" />
</p>

---
### cube_perlin_bump
_Time: 1.56417 s_
<p align="center">
<img alt="cube_perlin_bump" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_perlin_bump.png" />
</p>

---
### cube_wall
_Time: 1.57439 s_
<p align="center">
<img alt="cube_wall" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_wall.png" />
</p>

---
### cube_wall_normal
_Time: 1.566830 s_
<p align="center">
<img alt="cube_wall_normal" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_wall_normal.png" />
</p>

---
### cube_waves
_Time: 1.63458 s_
<p align="center">
<img alt="cube_waves" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/cube_waves.png" />
</p>

---
### ellipsoids_texture
_Time: 2.92964 s_
<p align="center">
<img alt="ellipsoids_texture" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/ellipsoids_texture.png" />
</p>

---
### galactica_dynamic
_Time: 449.681 s_
<p align="center">
<img alt="galactica_dynamic" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/galactica_dynamic.png" />
</p>

---
### galactica_static
_Time: 5.75577 s_
<p align="center">
<img alt="galactica_static" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/galactica_static.png" />
</p>

---
### killeroo_bump_walls
_Time: 26.6139 s_
<p align="center">
<img alt="killeroo_bump_walls" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/killeroo_bump_walls.png" />
</p>

---
### plane_bilinear
_Time: 1.09835 s_
<p align="center">
<img alt="plane_bilinear" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/plane_bilinear.png" />
</p>

---
### plane_nearest
_Time: 1.13796 s_
<p align="center">
<img alt="plane_nearest" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/plane_nearest.png" />
</p>

---
### sphere_nearest_bilinear
_Time: 2.16499 s_
<p align="center">
<img alt="sphere_nearest_bilinear" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nearest_bilinear.png" />
</p>

---
### sphere_nobump_bump
_Time: 1.9995 s_
<p align="center">
<img alt="sphere_nobump_bump" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nobump_bump.png" />
</p>

---
### sphere_nobump_justbump
_Time: 1.94817 s_
<p align="center">
<img alt="sphere_nobump_justbump" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_nobump_justbump.png" />
</p>

---
### sphere_normal
_Time: 117.432 s_
<p align="center">
<img alt="sphere_normal" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_normal.png" />
</p>

---
### sphere_perlin
_Time: 2.4421 s_
<p align="center">
<img alt="sphere_perlin" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin.png" />
</p>

---
### sphere_perlin_bump
_Time: 2.55343 s_
<p align="center">
<img alt="sphere_perlin_bump" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin_bump.png" />
</p>

---
### sphere_perlin_scale
_Time: 2.40968 s_
<p align="center">
<img alt="sphere_perlin_scale" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/sphere_perlin_scale.png" />
</p>

---
### wood_box
_Time: 1.58755 s_
<p align="center">
<img alt="wood_box" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box.png" />
</p>

---
### wood_box_all
_Time: 1.60994 s_
<p align="center">
<img alt="wood_box_all" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box_all.png" />
</p>

---
### wood_box_no_specular
_Time: 1.5577 s_
<p align="center">
<img alt="wood_box_no_specular" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/wood_box_no_specular.png" />
</p>

---
### VeachAjar
_Time: 174.211 s_
<p align="center">
<img alt="VeachAjar" src="https://github.com/fsaltunyuva/fsaltunyuva.github.io/blob/main/images/2025-12-15-Ray-Tracing-Adventure/VeachAjar.png" />

</p>
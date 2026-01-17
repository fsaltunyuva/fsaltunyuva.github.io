---
layout: post
title: "Ray Tracing Adventure Part 6 (Homework 6)"
date: 2026-01-17
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will continue my ray tracing adventure with Part 6, focusing on implementing features:

- **BRDF**
- **Object Lights**
- **Path Tracing**

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz, at Middle East Technical University.

### Bugs from Previous Parts 

### Outputs and Closing Thoughts

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

_Time: 14394.7 s_
<p align="center">
<img alt="teapot_roughness" src="https://github.com/user-attachments/assets/573e5709-0a80-4edb-b4af-1603fc5ae2da" />
</p>

---

### killeroo_blinnphong
_Time: 1.69274 s_
<p align="center">
<img alt="cube_directional" src="https://github.com/user-attachments/assets/933a8170-8750-47c3-b877-28a35bb440a9" />
</p>

---

### killeroo_blinnphong (killeroo_blinnphong_closeup) 
_Time: 1.75236 s_
<p align="center">
<img alt="cube_point" src="https://github.com/user-attachments/assets/335ba37e-f133-4eaf-a7f1-03f2e02756a6" />
</p>

---

### killeroo_torrancesparrow  
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
### killeroo_torrancesparrow (killeroo_torrancesparrow_closeup)
_Time: 79.1459 s_
<p align="center">
<img alt="dragon_spot_light_msaa" src="https://github.com/user-attachments/assets/2ff0d40c-6cbd-480e-aa45-f5e3a681638b" />
</p>

---

### cornellbox_jaroslav_diffuse
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

### cornellbox_jaroslav_diffuse_area
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

### cornellbox_jaroslav_glossy
_Time: 2.91417 s_
<p align="center">
<img alt="glass_sphere_env_phot" src="https://github.com/user-attachments/assets/27b3b2d3-dc67-4c56-8f7b-79a33800929b" />
        <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area
_Time: 61.3175 s_
<p align="center">
<img alt="head_env_light_phot" src="https://github.com/user-attachments/assets/c45dd861-159a-4f8a-b084-230f242c1efe" />
            <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area_ellipsoid
_Time: 1.71812 s_
<p align="center">
<img alt="mirror_sphere_env_phot" src="https://github.com/user-attachments/assets/25fb5f57-ab43-4058-b29f-6767952c48b7" />
            <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area_small
_Time: 1156.8 s_
<p align="center">
<img alt="sphere_env_light_phot" src="https://github.com/user-attachments/assets/cd4f7724-6694-4344-9e94-767ce5110b12" />
                <br>
    <em>Photographic</em>
    <br>
</p>

---

### cornellbox_jaroslav_glossy_area_sphere
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

### cornell_diffuse (diffuse_cornell_box_default)
_Time: 15166.2* s_
<p align="center">
<img alt="dragon_new_ply_with_spot" src="https://github.com/user-attachments/assets/8072ef91-b8d5-42b9-a12d-a025ada42ba0" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance) 
_Time: 7219.37* s_
<p align="center">
<img alt="audi-tt-glacier" src="https://github.com/user-attachments/assets/0b12aad6-646d-4cf3-a9a3-2e8530b4c95a" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_clamping)
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

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting) 
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_diffuse (diffuse_cornell_box_importance_nee_mis_balance_russian_1600)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_default) 
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance) 
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_clamping)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_splitting_clamp)
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_russian)  
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornell_glass_mirror (cornell_box_importance_nee_mis_balance_2500x2) 
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornellbox_prism_light
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### cornellbox_sphere_light
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### sponza_direct
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>

---

### sponza_path
_Time: 7209.42* s_
<p align="center">
<img alt="audi-tt-pisa" src="https://github.com/user-attachments/assets/2b6f7c09-6944-41bc-8f5a-e5fcbfcd79db" />
</p>
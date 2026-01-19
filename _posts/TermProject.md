---
layout: post
title: "Foveated Rendering in Ray Tracing (Term Project)"
date: 2026-01-20
categories: [ray-tracing, graphics, adventure]
---

Hello again, I will be focusing on Foveated Rendering in Ray Tracing for my term project. Although I know about the concept of Foveated Rendering and its applications, I have been wanting to implement it from scratch for some time to discover different methods or perspectives that could be added to or contribute to the technique, and my ray tracer project was a great opportunity for that. 

Before I begin, I should mention that this blog and project are part of the [Advanced Ray Tracing](https://catalog.metu.edu.tr/course.php?prog=571&course_code=5710795) course given by my professor Ahmet Oğuz Akyüz, at Middle East Technical University.

Let's start with the question:

## What is Foveated Rendering?
Basically, it is a rendering technique that tries to mimic the human eye's way of perceiving the world. The human eye has a small area called the fovea, which is responsible for sharp central vision. The rest of the visual field is perceived with less detail. 

[Foveated Rendering Image]

Foveated rendering takes advantage of this by rendering high detail images only in the area where the viewer is looking (the foveal region) and lower detail images in the peripheral areas. This approach can significantly reduce computational load while maintaining visual quality where it matters most.

It is commonly used in virtual reality (VR) because it can help improve performance and reduce latency, which helps to prevent [cybersickness](https://en.wikipedia.org/wiki/Virtual_reality_sickness) during VR experiences. Another reason why it is popular in VR is that VR headsets also tries to mimic and render based on our visual system, and most of the modern VR headsets have eye tracking capabilities, which makes it easier to implement foveated rendering.

## Brief History of Foveated Rendering
The idea is firstly proposed by Levoy and Whitaker in 1990 in their paper [“Gaze-Directed Rendering”](https://graphics.stanford.edu/papers/gaze-i3d90/gaze-i3d90-searchable.pdf). They were also trying to improve their ray tracing algorithm's performance for volumetric rendering.

[Gaze-Directed Rendering Image]

Then, in 1996, Ohshima et al. published a paper titled ["Gaze-Directed Adaptive Rendering for Interacting with Virtual Space"](https://ieeexplore.ieee.org/document/490517). They tampered the main idea "we do not need do render everything in high detail" a bit further and proposed a method that lowers or increases the level of detail (triangle count) of objects in a scene based on the viewer's gaze direction.

[Gaze-Directed Adaptive Rendering Image]

Even though, there were some other studies and papers on the topic, idea did not merged with the human visual system directly until 1998, when Geisler and Perry published ["A real-time foveated multiresolution system for low-bandwidth video communication"](https://svi.cps.utexas.edu/spie1998.pdf). They were trying to propose the same method for low-bandwidth video communication, but they directly used the human visual system's characteristics to determine the foveal and peripheral regions this time.

[Real-time foveated multiresolution system Image]

In 2016, NVIDIA Researchers published a paper titled ["Towards foveated rendering for gaze-tracked virtual reality"](https://cwyman.org/papers/siga16_gazeTrackedFoveatedRendering.pdf). They proposed a method that almost completely solves the tunnel vision problem that is commonly seen in foveated rendering techniques.

[Tunnel Vision Problem Image]

They applied a post process contrast enhancement filter to the peripheral regions to make them more visible and less blurry. This method was a great improvement for foveated rendering in VR and made Foveated Rendering more practical for VR applications.

## Types of Foveated Rendering

There are 2 main types of foveated rendering techniques:

### Static Foveated Rendering
In static foveated rendering, the foveal region is fixed and does not change based on the viewer's gaze direction. This method is simpler to implement but can lead to lower performance gain, because to achieve the overall best quality, the foveal region needs to be larger than necessary. Therefore, more pixels need to be rendered in high detail.

[Static Foveated Rendering Image]

### Dynamic Foveated Rendering
In dynamic foveated rendering, the foveal region changes based on the viewer's gaze direction. This method is more complex to implement, as it requires eye tracking technology to determine where the viewer is looking. However, it can lead to significant performance gains, as only the necessary pixels are rendered in high detail and foveal region can be smaller.

Even though it is not my problem for this term project, dynamic foveated rendering also come with the challenge of latency. If there is a delay between the viewer's gaze direction input from the eye tracker and the rendering process, it can lead to a mismatch between the foveal region and the viewer's actual gaze direction, which can cause discomfort and reduce visual quality, especially in quick eye movements (saccades).


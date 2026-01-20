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

[Dynamic Foveated Rendering Image]

## Implementation in Ray Tracing
I think we should start by asking, how can we reduce image quality in peripheral regions in ray tracing to improve performance? There are several other ideas for this, but the most straightforward and common way is to reduce the sample size in peripheral regions. As I described in [Part 3](https://fsaltunyuva.github.io/ray-tracing/graphics/adventure/2025/11/24/Ray-Tracing-Adventure.html), in ray tracing, we usually shoot multiple rays per pixel to achieve anti-aliasing and improve image quality. By reducing the number of rays shot per pixel in peripheral regions, we can significantly reduce the computational load.

In my implementation, I will use 3 areas:

1. Fovea Region: This is the central area where the viewer is looking. In this region, we will use the maximum sample size to achieve the highest image quality.

2. Blend Region: This is the area surrounding the fovea region. In this region, we will gradually decrease (but how exactly?) the sample size from the maximum to the minimum sample size as we move away from the fovea region.

3. Peripheral Region: This is the outer area where the viewer is not looking. In this region, we will use the minimum sample size to reduce computational load.

[Image I created for presentation]

Of course there are other ways to divide the regions, because the human visual system is continuous, so there are multiple ways to approximate it. But for simplicity and easily differ the falloff methods, I will use 3 regions.

Okay, until now, we understood the idea. In short, we will reduce the image quality (sample size) in the region where our eyes are not focused on (peripheral region) and increase the image quality (sample size) in the (foveal region). But how do we determine the sample size (in Blend Region) for each pixel based on its distance from the foveal region that will not cause noticeable quality loss and tunnel vision problem? This takes us to:

### Falloff Methods
Let's say we have determined 64 as our maximum sample size, and 4 as our minimum sample size. Now, we need to determine how to decrease the sample size from 64 to 4 in the Blend Region. There are several methods to do this, but before that, I should define some terms that will be used in the calculations:

#### Definitions
- Eccentricity (𝑒): The distance from the center of the fovea to a given point in the visual field, measured in degrees of visual angle. When eccentricity increases, the sample size should decrease.
    It can be easily calculated for each pixel using the following formula:

    𝑒(x, y) = arccos(dgaze DOT dpixel) * 180 / pi
    
    where dgaze is the normalized direction vector of the gaze, dpixel is the ray passing through the pixel. (We simply take the arccos of the dot product of these two vectors to get the angle between them in radians, then convert it to degrees by multiplying with 180 / pi.)

- Reference Eccentricity (𝑒0): A small constant (in degrees) used to avoid singularities at the fovea center and to control how aggressively the falloff begins.

- Cortical Magnification (M(𝑒)): A function that models how much cortical area is allocated to one degree of visual angle at eccentricity 𝑒.

- Cutoff eccentricity (𝑒c): The eccentricity up to which a linear approximation is considered reasonable (often treated as a “central vision” range in practical models).

#### Log Acuity Model
This model has been found that the excitation of the cortex can be approximated by a log-polar mapping of the eye’s retinal image [CITE Meng2018Foveated.pdf]. This model is proposed by Eric Schwartz in his paper ["Anatomical and Physiological Correlates of Visual Computation from Striate to Infero-Temporal Cortex"](https://ieeexplore.ieee.org/document/6313208) in 1984. The calculation model of this method is cheap and fast.

The key idea behind the log acuity model is *cortical magnification*. Physiological studies show that a disproportionately large area of the visual cortex is devoted to processing the foveal region, while peripheral regions are represented more compactly. This behavior can be approximated by the following cortical magnification function:

M(𝑒) = K / (𝑒 + 𝑒0)

where K is a scaling constant that determines the overall level of detail.

Near the fovea (small 𝑒), cortical magnification M(𝑒) is large, indicating high detail. As 𝑒 increases, M(𝑒) decreases, reflecting lower detail towards peripheral region.

Schwartz showed that integrating this magnification function leads to a logarithmic mapping between retinal space and cortical space. In other words, equal distances on the cortex correspond to exponentially increasing distances in the visual field. This is why the model is often referred to as a log-polar or log acuity model.

But how can we use this function to determine the sample size for each pixel in the Blend Region? If visual acuity is proportional to cortical magnification M(e) (as in paper), then the linear resolution should scale with M(e), and the sampling density over an image area should scale with M(e)^2.

Therefore, we can define the sample size N(𝑒) at eccentricity 𝑒 as:

N(𝑒) = N_max * (M(𝑒) / M(0))^2

Substituting the cortical magnification function into this equation gives:

N(𝑒) = N_max * (e0 / (e + e0))^2

So we can use this equation in the Blend Region to determine the sample size for each pixel:

N(𝑒) = clamp(Nmin, Nmax, N_max * (e0 / (e + e0))^2).

As we said earlier, if Nmax = 64, Nmin = 4, and e0 = 1 degree, function will look like this (I will also upload the codes I used to create these plots):

[FoveatedRenderingFalloffs.ipynb Image]

#### Linear Acuity Model
Another commonly used model is the linear acuity model. In this model, visual acuity is assumed to decrease approximately linearly with eccentricity within the central visual field. This model matches both anatomical data and is applicable for many low-level vision tasks [CITE Hans Strasburger, Ingo Rentschler, and Martin J¨uttner. Peripheral vision andpattern recognition: a review. Journal of vision, 11(5):13–13, 2011.]

The model is based on the concept of Minimum Angle of Resolution (MAR), which represents the smallest angular separation at which two features can be distinguished. Experimental studies on human observers indicate that MAR increases roughly linearly with eccentricity for central vision, leading to the following formulation:

MAR(𝑒) = a + b𝑒

where a = MAR(0) is the foveal resolution limit, and b controls the rate at which acuity degrades with eccentricity.

By this idea, we assume that the sampling density should be proportional to the amount of resolvable visual detail.

N(e) = N_max * (MAR(0) / MAR(𝑒))

Substituting the linear MAR function into this equation gives this calculation for sample size at eccentricity 𝑒:

N(𝑒) = clamp(Nmax * a / (a+b*e), Smin, Smax)

While this model is well supported by human-subject experimental data (to determine a and b values), its validity is largely limited to central vision, typically within an angular radius of approximately 8°. Beyond this region, MAR has been shown to increase more steeply than predicted by a linear model [CITE Foveated 3D graphics. Brian Guenter, Mark Finch, Steven Drucker, Desney Tan, and John Snyder. 2012. ]

In the peripheral visual field, photoreceptor density decreases rapidly, and the visual system becomes increasingly limited by the Nyquist limit of the retinal sampling lattice rather than optical blur alone. As a result, spatial frequencies that are theoretically detectable become aliased, leading to a regime where a linear falloff no longer accurately reflects perceived visual quality.

This behavior is illustrated by the widening gap between the detectable without aliasing and detectable but aliased regions in the sample falloff diagram. The linear model effectively balances quality and performance in the central field but tends to overestimate perceptual sensitivity in the far periphery.



## Future Work
Our peripheral vision changes also by the speed.
https://media.istockphoto.com/id/1597773385/tr/vekt%C3%B6r/safety-car-driving-rules-and-tips-peripheral-vision-while-driving-vector-illustration.jpg?s=612x612&w=is&k=20&c=2iJJ95UChOUSgmE4nm6aESAXqRIcF62QrcU4baCW3hc=

As I mentioned, our view area can be divided into more than 3 regions.
https://en.wikipedia.org/wiki/Peripheral_vision#/media/File:Peripheral_vision.svg
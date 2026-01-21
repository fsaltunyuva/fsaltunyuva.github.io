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

As we said earlier, if Nmax = 64, Nmin = 4, and e0 = 1 degree, function will look like this (Python scripts used to generate the plots can be found [here](https://github.com/fsaltunyuva/RayTracer/tree/main/Foveated%20Rendering)):

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

#### Mixed Acuity Model

The mixed acuity model is motivated by the fact that different biological bottlenecks dominate visual resolution depending on eccentricity. As shown in here:

[Figure 1.1 from https://3dvar.com/Meng2018Foveated.pdf]

Photoreceptor density (cones/rods) and ganglion cell (RGC) density follow different trends across the retina. In the fovea, ganglion cell density is high and tends to match photoreceptor density, enabling high spatial resolution. However, away from the fovea, ganglion cell density drops much more rapidly, meaning that many photoreceptors effectively map to the same ganglion cell (i.e., spatial pooling increases with eccentricity). This relationship is supported by human retinal topography measurements reported by Curcio et al.

Because of this, a single pure acuity model (only photoreceptors or only cortical/log mapping) may not capture the full picture. Instead, mixed acuity models treat acuity as being limited by the most restrictive stage at each eccentricity.

In reality, when we get close to the fovea, resolution is primarily limited by photoreceptor sampling (cones dominate), so acuity remains high. In the periphery, ganglion cell density becomes the limiting factor due to increasing pooling; even if photoreceptors exist, the neural output bandwidth is reduced, so effective resolution drops faster.

So we can define two eccentricity dependent resolution limits in MAR form:

- Photoreceptor limited MAR: MARphoto(𝑒)
- Ganglion limited MAR: MARganglion(𝑒)

A conservative mixed model can be expressed by taking the maximum MAR (worst acuity) at each eccentricity (it can be improved further by smooth blending, but I will keep it simple here):

MARmixed(𝑒) = max(MARphoto(𝑒), MARganglion(𝑒))

Then we follow the same reasoning as before to derive sample size:

N(𝑒) = clamp(Nmax * (MARmixed(0) / MARmixed(𝑒))^2, Nmin, Nmax)

Here is how the mixed acuity model looks like with the following assumptions.

- Photoreceptor limited MAR: MARphoto(𝑒) = 0.02 + 0.01𝑒 (Same linear MAR as in linear acuity model)

- Ganglion limited MAR: MARganglion(𝑒) = 0.02 + 0.015 * log(1 + 0.08 * 𝑒) (Steeper linear MAR to reflect faster drop in ganglion cell density)

[Mixed Acuity Model Image]

### Falloff Comparison
Here is a comparison of the 3 falloff methods I described above:

[Falloff Comparison Image]

I will show the results of these falloff methods in the last section.

#### Fovea Radius
Another important parameter in foveated rendering is the fovea radius, which determines the size of the foveal region. A larger fovea radius will result in a larger area being rendered in high detail, while a smaller fovea radius will result in a smaller area being rendered in high detail.

We can define the fovea radius in degrees of visual angle. Then we can convert this angle to a distance on the screen based on the viewer's distance from the screen (as discussed in Definitions section).

A common value for the fovea radius is around 2 degrees of visual angle [CITE Brian Guenter, Mark Finch, Steven Drucker, Desney Tan, and John Snyder.
Foveated 3d graphics. ACM Trans. Graph., 31(6):164:1–164:10, November 2012].

#### Blend Region Radius
The outer boundary of the blend region is commonly chosen to be several times larger than the fovea radius. In practice, a value between 6 and 10 degrees of visual angle works well for many applications. A typical and well-balanced choice is around 8 degrees of visual angle.

#### Peripheral Region
The peripheral region encompasses all areas beyond the blend region. In this region, the sample size is set to the minimum value to maximize performance savings.

### Code and Results
Let's start with static foveated rendering implementation. At first, I implemented the most basic version of foveated rendering with static fovea region and constant sample sizes for each region.

I explained the multisampling logic in [Part 3](https://fsaltunyuva.github.io/ray-tracing/graphics/adventure/2025/11/24/Ray-Tracing-Adventure.html), so I will not go into details here. To start, I modified the sampling logic like this for a fixed regions and sample sizes:

```cpp
for (int y = 0; y < height; ++y) {
    for (int x = 0; x < width; ++x) {
        // Calculate Eccentricity (e) as I explained before in Definitions section
        float u_center = (x + 0.5f) / width;
        float v_center = (y + 0.5f) / height;

        float su_center = left + (right - left) * u_center;
        float sv_center = bottom + (top - bottom) * v_center;

        // Compute pixel position in world space
        Vec3 P_center = camPos.subtract(w.scale(nearDist))
                                .add(u_vec.scale(su_center))
                                .add(v_vec.scale(sv_center));

        // Pixel Direction
        Vec3 pixelDir = P_center.subtract(camPos).normalize();

        // Dot product of Gaze and Pixel Direction
        Vec3 gazeNorm = cam.gaze.normalize();
        float dotVal = max(-1.0f, min(1.0f, gazeNorm.dot(pixelDir)));
        float angleRad = acos(dotVal);
        float eccentricity = angleRad * 180.0f / (float) M_PI; // Convert to degrees as explained

        // Determine Sample Count (N)
        int N_foveated = 1; // Default peripheral

        // Configuration to see the regions clearly
        float foveaRadius = 20.0f;
        float blendRadius = 30.0f;

        int maxSamples = cam.numSamples; // 100 in metal_glass_plates.json
        int blendSamples = 16;
        int minSamples = 1;

        int flippedY = height - 1 - y;

        if (eccentricity <= foveaRadius) {
            N_foveated = maxSamples;
            // DEBUG: RED
            // hdr[flippedY * width + x] = Vec3(255.0f, 0.0f, 0.0f);
            // continue;
        }
        else if (eccentricity <= blendRadius) {
            N_foveated = blendSamples;
            // DEBUG: GREEN
            // hdr[flippedY * width + x] = Vec3(0.0f, 255.0f, 0.0f);
            // continue;
        }
        else {
            N_foveated = minSamples;
            // DEBUG: BLUE
            // hdr[flippedY * width + x] = Vec3(0.0f, 0.0f, 255.0f);
            // continue;
        }

        int N = N_foveated;
        int S = (int) sqrt((float) N); // S x S grid
        // ... (rest of the sampling logic)
    }
}
```

When I debugged this with colored regions (with commented codes), I got this result for metal_glass_plates.json scene with 800x800 resolution:

[Static Foveated Rendering Debug Image]

Even though there are some problems in my renderer with attenuation, I wanted to use metal_glass_plates.json because I thought it would be a good scene to see the sample differences in different regions. I also modified the camera position and gaze direction a bit to get a better view for foveated rendering. In the following render, I used 100 samples for fovea region, 16 samples for blend region, and 1 samples for peripheral region. Here is the result compared to normal rendering with 100 samples per pixel:

[Static Foveated Rendering Result Image]

108.196 seconds vs 20.7135 seconds!

Now, we can implement the falloff methods I explained before. But I was implemented Stratified Random Sampling in my ray tracer, so I needed to modify the sampling logic a bit to accommodate non-square number of samples per pixel. As we discussed in class, I will use N-Rooks Sampling for this purpose.

```cpp
// Arrange samples along the diagonal randomly
vector<float> xCoords(N);
vector<float> yCoords(N);

for (int i = 0; i < N; ++i) {
    xCoords[i] = (i + dist(rng)) / (float)N;
    yCoords[i] = (i + dist(rng)) / (float)N;
}

// Shuffle their x-coordinates and y-coordinates independently
shuffle(xCoords.begin(), xCoords.end(), rng);
shuffle(yCoords.begin(), yCoords.end(), rng);

for (int i = 0; i < N; ++i) {
    jitterSamples.push_back({xCoords[i], yCoords[i]});
}
```

So, we can now implement the falloff methods in the sampling logic. Here is the modified sampling logic with Log Acuity Model:

```cpp
float e0 = 1.0f; // Reference eccentricity

if (eccentricity <= foveaRadius) {
    N_foveated = maxSamples;
}
else if (eccentricity <= blendRadius) {
    // N = N_max * (e0 / (e + e0))^2
    float term = e0 / (eccentricity + e0);
    float falloff = (float) maxSamples * (term * term);

    N_foveated = max(minSamples, min(maxSamples, (int) falloff));
}
else {
    N_foveated = minSamples;
}
```

But when I tested this, I encountered a problem. The standard Log Acuity model assumes the falloff starts immediately from the center ($e=0$). However, in my implementation, I am keeping the Fovea Region at maximum quality up to a certain radius ( $20^\circ$).

If the blend region starts at $20^\circ$ and I feed this value directly into the formula, the function calculates the drop-off as if we are already far away from the center. For example, with $e=20$ and $e_0=1$, the multiplier becomes $(1/21)^2$, which is tiny. This caused the sample count to plummet instantly from maxSamples to minSamples at the fovea boundary, creating a sharp artifact instead of a smooth transition.

[og log acuity model render image]

To fix this, I needed to shift the eccentricity so that the falloff calculation treats the edge of the fovea as its starting point ($0$). I also introduced a local $e_0$ parameter to better control the slope of the falloff within the blend region.

```cpp
else if (eccentricity <= blendRadius) {
    // Shift the eccentricity so the falloff curve starts at 0 
    float shiftedEccentricity = eccentricity - foveaRadius; 
    
    // A larger e0 here ensures the drop-off isn't too steep initially
    float blend_e0 = 10.0f; 

    // N = N_max * (e0 / (shifted_e + e0))^2
    float term = blend_e0 / (shiftedEccentricity + blend_e0);
    float falloff = (float) maxSamples * (term * term);

    N_foveated = max(minSamples, min(maxSamples, (int) falloff));
}
```

[fixed instant drop commit image]

When I encountered this, I thought that my idea of using these falloff methods only for the Blend Region will not help me to compare different falloff methods properly, and also it is not how these models are intended to be used to. Therefore, I used these falloff methods from the center of the fovea (0 degrees) to the outer edge of the blend region. By doing this, I got more natural looking falloffs and less region transition artifacts.

[falloff from fovea center to blend commit image]

So I continued with this approach and implemented the Linear Acuity Model:

```cpp
float a = 0.02f; // MAR(0) - Foveal intercept
float b = 0.04f; // MAR slope

// ...
else {
    // MAR(e) = a + b * e
    float mar_e = a + b * eccentricity;

    // Ratio = MAR(0) / MAR(e) = a / (a + b*e)
    float ratio = a / mar_e;
    float falloff = (float)maxSamples * ratio;

    N_foveated = max(minSamples, min(maxSamples, (int) falloff));
}
```

[linear acuity model render image]

and finally, I implemented the Mixed Acuity Model:

```cpp
// ...
else {
    // 1. Photoreceptor Limited MAR (Linear)
    // Formula: 0.02 + 0.01 * e
    float mar_photo = 0.02f + 0.01f * eccentricity;

    // 2. Ganglion Limited MAR (Logarithmic-like)
    // Formula: 0.02 + 0.015 * log(1 + 0.08 * e)
    float mar_ganglion = 0.02f + 0.015f * std::log(1.0f + 0.08f * eccentricity);

    // 3. Mixed MAR (Worst Case / Maximum Angle)
    float mar_mixed = std::max(mar_photo, mar_ganglion);

    // 4. Calculate Sample Count
    // Both formulas intersect at e = 0, MAR(0) = 0.02
    float mar0 = 0.02f;

    float ratio = mar0 / mar_mixed;
    float falloff = (float)maxSamples * (ratio * ratio);

    N_foveated = std::max(minSamples, std::min(maxSamples, (int)falloff));
}
```

[Mixed Acuity Model Render Image]

## Results Comparison
Due to a lot of parameters are involved in foveated rendering, it is hard to say which falloff method is the best only by looking at one render. Also, there are several purposes of foveated rendering (some want to maximize performance, some want to minimize quality loss, some want to avoid tunnel vision problem, etc.), so it is hard to conclude a method as the best one overall. But I will share my observations and results here.

Used parameters for all methods:
- Scene: metal_glass_plates.json (with modified camera position and gaze direction, I will share the used json)
- Fovea Radius: 20 degrees (Instead of the common 2 degrees, I increased it to see the differences more clearly)
- Blend Radius: 30 degrees
- Max Samples: 100
- Min Samples: 1
- Blend Samples: 16 (only for Static Foveated Rendering)
- e0 (Log Acuity Model): 5 degrees 
- a and b (Linear Acuity Model): 0.02 and 0.04 (MAR(0) and slope)
- Photoreceptor MAR (Mixed Acuity Model): 0.02 + 0.01 * e
- Ganglion MAR (Mixed Acuity Model): 0.02 + 0.015 * log(1 + 0.08 * e)

Static Foveated Rendering and Log Acuity Model with Fovea-Blend-Peripheral Regions results:

[Static Foveated Rendering vs Log Acuity Model Image]

3 falloff methods from center of fovea to edge of blend region results:

[Falloff Methods Comparison Image]

1.5x zoomed in images to see the differences better:

[1.5x Zoomed In Falloff Methods Comparison Image]

[Comparison GIF]

I also uploaded the renders to here, if you want to see them in full resolution.

Also, you can see the render times for each method here:

| Falloff Method              | Time (seconds) |
| ----------------------- | --------------- |
| Static Foveated Rendering    | 19.9914        |
| Log Acuity Falloff with Fovea-Blend-Peripheral Regions          | 25.6729        |
| Log Acuity Falloff From Center to Blend Edge    | 2.76325         |
| Linear Acuity Falloff From Center to Blend Edge          | 1.71995         |
| Mixed Acuity Falloff From Center to Blend Edge        | 1.56458         |
| No Foveated Rendering - 100 Sample Size        | 99.8554         |

## Comments
When I look at the render times, I understood why the Foveated Rendering is popular in VR applications. Even with a simple static foveated rendering implementation, I was able to achieve almost 5x speedup compared to normal rendering with 100 samples per pixel. And with more advanced falloff methods (and fine-tuning the parameters), I could have achieved even better performance.

When I look at the images, I can say that all falloff methods produced acceptable results, but there are some differences in quality and artifacts.

In Static Foveated Rendering, the transitions between regions are quite noticeable, especially at the boundaries. This became a good example for tunnel vision problem, as the peripheral region looks very low quality compared to the fovea region. But it is expected, as we are using constant sample sizes for each region.

In all falloff methods, the transitions are much smoother, and there are less noticeable artifacts. In Log Acuity Model, the falloff is quite steep (fovea region has high samples, but it quickly drops to low samples in peripheral region), which can lead more noticeable quality loss when eccentricity increases. In Linear Acuity Model, the falloff is more gradual, which helps to maintain better quality when eccentricity increases. In Mixed Acuity Model, even though it requires biological data and more complex calculations, it provides the best balance between quality and performance due to its consideration of multiple biological bottlenecks.

When I looked at render times of each falloff method, I wondered why Log Acuity Model is taking more time than others, it should be the fastest one due to its steep falloff. But then I realized that it is because of e0 value I used (5 degrees). With this value, in 5 degree angle, I get:

N(e) = 100 * (5 / (5 + 5))^2 = 25 samples

Which in Linear Acuity Model, at 5 degree angle, I get:

N(e) = 100 * (0.02 / (0.02 + 0.04 * 5)) = 9 samples

This is also a good example of how parameters can affect the results and performance of foveated rendering. I get 1.50144 seconds with e0 = 2 degrees for Log Acuity Model, which is faster than other falloff methods.

## Conclusions and Future Work
Even though my main goal was to stick to the parameters from the papers I read, I realized that keeping those parameters will not help me to compare the falloff methods properly. So I changed some parameters (like Fovea Radius, e0 value) but I think it cause a chain effect on other parameters so I could not keep everything consistent and as in the papers. But I think I achieved my main goal of implementing foveated rendering in ray tracing and exploring different falloff methods and their approach to the problem.

For future work, I can explore the following ideas:
- Parameters that I used can be inputted by the json files to make it easier to test different configurations.

- If the gaze point of the user is given (from json file or the eye tracker), fovea region can be moved based on that point instead of fixed camera gaze direction. By doing this, dynamic foveated rendering can be implemented.

- Our peripheral vision is also affected by the speed of objects in our visual field. For instance, when we are driving a car, when speed increases, our peripheral vision decreases. This can be explored in foveated rendering by adjusting the fovea and blend region sizes based on the speed of the camera or objects in the scene. 
https://media.istockphoto.com/id/1597773385/tr/vekt%C3%B6r/safety-car-driving-rules-and-tips-peripheral-vision-while-driving-vector-illustration.jpg?s=612x612&w=is&k=20&c=2iJJ95UChOUSgmE4nm6aESAXqRIcF62QrcU4baCW3hc=

- As I mentioned, our view area can be divided into more than 3 regions.
https://en.wikipedia.org/wiki/Peripheral_vision#/media/File:Peripheral_vision.svg

- Some geometric falloff methods can also be explored for falloff calculations.

## Great Papers and Articles to Read
- [Foveated 3D Graphics](https://www.microsoft.com/en-us/research/wp-content/uploads/2012/11/foveated_final15.pdf)

- [FOVEATED RENDERING TECHNIQUES IN MODERN COMPUTER GRAPHICS](https://3dvar.com/Meng2018Foveated.pdf#page=66&zoom=100,149,346)

- [Foveated Path Tracing with Configurable Sampling and Block-Based Rendering](https://arxiv.org/pdf/2406.07981)

- [Fooling Around with Foveated Rendering](https://www.peterstefek.me/focused-render.html)
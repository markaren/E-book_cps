# Virtual Environments

Testing a robot's perception and control in the real world is slow, expensive, and sometimes dangerous — you cannot crash a real robot a thousand times to tune a controller. A **virtual environment** simulates the world in software, so you can develop and test algorithms before deploying them, and generate **synthetic training data** for the [vision models](deep_vision.md) of the previous chapters. This chapter is a short tour of how those worlds are made and why they matter for a cyber-physical system.

---

## Why simulate

A virtual environment buys you three things:

- **Safe, fast iteration.** Test a control loop or a perception pipeline against a simulated robot — no hardware, no risk, and far faster than real time if you want.
- **A repeatable test suite.** A simulated scene runs identically every time, so a regression test for your [control](../Chapter3/real_time.md) or vision code is possible in a way the messy real world never allows.
- **Synthetic data.** Render labelled images of objects in varied poses and lighting to *train* a model when real labelled data is scarce — the labels come for free because the simulator knows exactly what it drew.

The pattern is a loop: develop against the **virtual world**, then deploy the *same* control and vision code to the **real world**. How faithful the simulation needs to be depends on the task — sometimes a coarse world is plenty, sometimes you need photo-realism.

---

## Rendering: turning a model into an image

**Rendering** is generating a 2D image from a 3D model — handling geometry, lighting, and texture. Two broad techniques matter:

- **Rasterization** is the fast technique behind real-time graphics and games. It projects the 3D model's vertices to 2D screen coordinates, assembles them into triangles, and fills in pixels, then shades each pixel. The pipeline — vertex processing → primitive assembly → rasterization → fragment (pixel) processing — is what GPUs are built to do at high frame rates. It is an *approximation* of how light behaves, which is why games render in milliseconds.
- **Ray tracing** and **path tracing** simulate light physically: cast rays from the camera (and from light sources) and follow their bounces, reflections, and refractions. The result is far more realistic but far heavier — historically offline-only, though since ~2019 GPUs have hardware acceleration for it. Path tracing (many rays per pixel, statistically sampled) is more accurate still and heavier again.

The trade is the familiar one: **rasterization for speed, ray/path tracing for realism.** A coarse rasterized world is often enough to test logic; photo-realism (ray tracing, often paired with a physics engine) matters when you are generating training data that must look like the real camera's output.

---

## Graphics APIs and engines

Rendering is done through a **graphics API** that talks to the GPU:

| API | Note |
|-----|------|
| **OpenGL** | Long-standing, cross-platform; development ended, but still common in education and hobby projects |
| **Vulkan** | OpenGL's successor — far more powerful and lower-level (and more work to use) |
| **Direct3D / Metal** | Microsoft / Apple platform APIs |
| **WebGL / WebGPU** | Graphics in the browser; WebGPU is the modern successor |

Above the raw APIs sit **engines and frameworks** that handle the rendering for you: **NVIDIA Omniverse** and **Unreal Engine** offer photo-realism with ray tracing and integrated physics, suited to high-fidelity robotics simulation. For lighter, C++-native rendering — keeping everything in the language the rest of this book uses — **[Threepp](https://github.com/markaren/threepp)** is a C++ port of the popular three.js library, enough to put a robot and its sensors in a simple simulated world alongside your control code.

!!! tip "Match fidelity to purpose"
    Do not reach for photo-realism by default. Testing whether a state machine or a control loop behaves correctly needs only a **coarse** world — simple shapes, basic rasterization — which is quick to build and fast to run. Save ray tracing and physics engines for when you specifically need **realistic sensor data** (e.g. synthetic images that must fool a vision model trained on real photos). The right simulation is the *simplest* one that exercises what you are testing.

---

## Where it fits

A virtual environment closes the development loop for the whole book: the [concurrency](../Chapter2/processes_threads.md) and [real-time](../Chapter3/real_time.md) control code, the [communication](../Chapter4/serialization.md) layer, and the [vision pipeline](onnx.md) can all be exercised against a simulated robot before any of it touches hardware — and the same code then deploys to the real device. For vision specifically, synthetic rendered data feeds back into the [training](deep_vision.md) step, and a simulated camera lets you test the [inference pipeline](onnx.md) frame-by-frame with ground truth you control.

---

## Summary

- A **virtual environment** lets you develop and test control and perception **safely and repeatably** before deploying to real hardware, and **generates synthetic, automatically-labelled training data**.
- **Rendering** turns a 3D model into an image. **Rasterization** is fast (the games/real-time technique: vertices → triangles → pixels); **ray/path tracing** is physically realistic but heavy (hardware-accelerated since ~2019).
- Rendering uses a **graphics API** (OpenGL, Vulkan, WebGPU, …); higher-level **engines** (Omniverse, Unreal) give photo-realism + physics, while **Threepp** offers lightweight C++-native rendering.
- **Match fidelity to purpose**: a coarse world tests logic; photo-realism is for realistic sensor data. The simplest simulation that exercises your code is the right one.
- A virtual world closes the develop-then-deploy loop for the whole system. Next: the [exercises](exercises.md) for this part.

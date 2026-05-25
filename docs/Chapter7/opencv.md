# OpenCV in C++

A cyber-physical system that *sees* — a robot following a line, detecting an obstacle, reading a marker — needs **computer vision**. The standard tool is **OpenCV**, and although most tutorials (and the lectures) use its Python interface, OpenCV is a **C++ library** at heart. That matters here: the perception code you prototype in Python is the same code you can run, fast, in C++ on the robot. This part takes that C++ and on-device angle.

!!! note "Where the work actually happens"
    Be honest about the split. Vision *training* and quick experiments are usually done in **Python** (numpy, PyTorch). Vision *deployment* — running on the Pi, in a real-time loop, alongside your control code — is where **C++** earns its place. This part focuses on the C++ side; [Model Deployment & ONNX](onnx.md) and [Calling C++ from Python](../Chapter6/python_interop.md) connect the two worlds.

---

## What OpenCV is

OpenCV (Open Source Computer Vision Library) is a mature, open-source library for real-time computer vision: image and video processing, feature detection, object and face detection, and — through its `dnn` module — running deep-learning models. It is written in C++ and provides bindings for Python, Java, and more, so an algorithm is implemented once and used from several languages.

Add it to a CMake project like any other dependency (via [vcpkg](../Chapter6/dependencies.md), `vcpkg install opencv`):

```cmake
find_package(OpenCV REQUIRED)

add_executable(vision main.cpp)
target_link_libraries(vision PRIVATE ${OpenCV_LIBS})
```

---

## An image is a matrix

The central type is **`cv::Mat`** — a matrix of pixels. Understanding how it stores an image explains most of OpenCV:

- A **grayscale** image is a single-channel matrix; each pixel is one intensity from `0` (black) to `255` (white).
- A **color** image is a three-channel matrix. OpenCV's channel order is **BGR** (Blue, Green, Red) — **not** RGB. This is the single most common OpenCV gotcha: load an image, display it with a library expecting RGB, and everything looks blue-tinted.
- **Depth** is the bits per pixel value — usually 8-bit (`0–255`), sometimes 16-bit.

| Property | Meaning |
|----------|---------|
| `mat.rows`, `mat.cols` | Image height and width in pixels |
| `mat.channels()` | 1 (grayscale) or 3 (BGR color) |
| `mat.type()` | Element type + channels, e.g. `CV_8UC3` = 8-bit unsigned, 3 channels |

On the Python side, a `cv::Mat` surfaces as a **numpy array** — which is why Python CV code slices images like arrays. In C++ you index with `image.at<cv::Vec3b>(row, col)` for a colour pixel, though you rarely loop over pixels by hand: OpenCV's functions operate on whole matrices, which is far faster.

---

## Core operations

A typical pipeline reads an image, converts and filters it, and extracts something. Here is a complete edge-detection program — it needs OpenCV linked, so it will not run on Compiler Explorer, but it is the shape of almost every classic-vision task:

<!-- no-ce -->
```cpp
#include <opencv2/opencv.hpp>

int main() {
    cv::Mat image = cv::imread("input.jpg");           // load (BGR, 8-bit, 3 channels)
    if (image.empty()) {
        return 1;                                      // imread returns an empty Mat on failure
    }

    cv::Mat gray;
    cv::cvtColor(image, gray, cv::COLOR_BGR2GRAY);     // 3 channels → 1

    cv::Mat blurred;
    cv::GaussianBlur(gray, blurred, cv::Size(5, 5), 1.5);   // reduce noise

    cv::Mat edges;
    cv::Canny(blurred, edges, 50, 150);               // detect edges

    cv::imwrite("edges.png", edges);                  // save the result
}
```

The operations you will reach for most:

| Task | Function |
|------|----------|
| Read / write | `cv::imread`, `cv::imwrite` |
| Resize | `cv::resize` |
| Colour conversion | `cv::cvtColor` (e.g. `COLOR_BGR2GRAY`, `COLOR_BGR2HSV`) |
| Blur / denoise | `cv::GaussianBlur`, `cv::medianBlur` |
| Edges | `cv::Canny` |
| Threshold | `cv::threshold`, `cv::adaptiveThreshold` |
| Contours | `cv::findContours` |
| Display (desktop) | `cv::imshow` + `cv::waitKey` |

A common task — finding a coloured object — converts BGR to **HSV** (where "colour" is one axis, robust to lighting), thresholds the hue range with `cv::inRange`, and takes the centroid of the result. That colour-tracking exercise is a classic first OpenCV project.

---

## Cameras and video

A live system reads frames from a camera with `cv::VideoCapture`, which yields one `cv::Mat` per frame:

<!-- no-ce -->
```cpp
#include <opencv2/opencv.hpp>

int main() {
    cv::VideoCapture camera(0);                        // open the default camera
    if (!camera.isOpened()) return 1;

    cv::Mat frame;
    while (camera.read(frame)) {                       // grab the next frame
        // ... process `frame` here ...
        if (cv::waitKey(1) == 27) break;               // Esc to quit
    }
}
```

That `while (camera.read(frame))` loop is the spine of every vision application — and on a robot it runs on its **own thread**, feeding results to the control loop, so a slow frame never stalls the rest of the system. Structuring that is the subject of the [pipeline section in Model Deployment](onnx.md#a-real-time-vision-pipeline), and it is exactly the [producer/consumer](../Chapter2/condition_variables.md) and [real-time](../Chapter3/real_time.md) machinery from Parts 2 and 3.

!!! warning "`imshow` needs a desktop"
    `cv::imshow` opens a GUI window — fine on your laptop, but a headless Raspberry Pi over SSH has no display. On-device you typically *don't* show frames; you process them and send results over a [socket](../Chapter4/sockets.md) or log them. Guard any `imshow`/`waitKey` so it is easy to compile out for the embedded build.

---

## Prototype in Python, deploy in C++

The honest workflow for this course: **explore in Python**, where numpy slicing and instant feedback make experimentation quick, then **port the settled pipeline to C++** for the robot, where you need real-time performance and integration with your control and [communication](../Chapter4/serialization.md) code. Because it is the *same library*, a `cv::cvtColor` in C++ does exactly what `cv2.cvtColor` did in Python — the translation is mechanical. When only part of the pipeline needs C++, the [Python↔C++ bridge](../Chapter6/python_interop.md) lets the two coexist.

---

## Summary

- **OpenCV** is a C++ computer-vision library with bindings for other languages; add it with `find_package(OpenCV)` and [vcpkg](../Chapter6/dependencies.md). The same code runs in Python (for prototyping) and C++ (for deployment).
- An image is a **`cv::Mat`** — a pixel matrix. Grayscale is 1 channel (`0–255`); colour is **3 channels in BGR order** (not RGB — the classic gotcha). Depth is bits per value (usually 8-bit).
- Work on **whole matrices** with built-in functions (`imread`, `cvtColor`, `GaussianBlur`, `Canny`, `threshold`, `findContours`) rather than per-pixel loops; HSV + `inRange` is the go-to for colour tracking.
- A live system reads frames with **`cv::VideoCapture`** in a loop — run it on its **own thread** ([Part 2](../Chapter2/condition_variables.md)/[Part 3](../Chapter3/real_time.md)) so vision never stalls control. Avoid `imshow` on a headless Pi.
- **Prototype in Python, deploy in C++** — it is one library, so the port is mechanical. Next: [Camera Calibration](calibration.md), making the geometry of those images trustworthy.

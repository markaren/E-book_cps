# Camera Calibration

A camera lens bends light, and a lens is never perfect: straight lines bow outward near the edges, and the pixel coordinates you read do not map cleanly to angles in the real world. **Camera calibration** measures these imperfections so you can correct for them. If your robot only needs to spot a coloured blob, you can skip it; if it needs to *measure* — distances, angles, where an object is in space — calibration is what makes the numbers trustworthy.

---

## What calibration recovers

Calibration estimates two sets of parameters.

**Intrinsic parameters** describe the camera and lens themselves — properties that do not change as the camera moves:

- **Focal length** and **optical (principal) point** — how the lens projects the 3D world onto the sensor, captured in a 3×3 **camera matrix** `K`.
- **Distortion coefficients** — how the lens bends the image away from an ideal projection (the barrel distortion that bows straight lines near the edges).

**Extrinsic parameters** describe where the camera *is*: its **rotation** and **translation** relative to some world coordinate system. These change whenever the camera moves, and they are what let you turn "a pixel in the image" into "a direction in the room" — essential for navigation and for fusing a camera with other sensors.

| Parameter set | Captures | Changes when the camera moves? |
|---------------|----------|-------------------------------|
| **Intrinsic** | Focal length, optical centre, lens distortion | No — fixed for a given camera + lens |
| **Extrinsic** | Camera position and orientation in the world | Yes |

---

## Camera models

The parameters are defined with respect to a **camera model** — an idealised description of how light forms an image:

- **Pinhole model** — assumes an ideal camera where light passes through a single point and projects onto the sensor. It is the standard model for most computer-vision work, where lens distortion is modest.
- **Fisheye model** — for very wide field-of-view lenses (up to ~195°), where the pinhole model breaks down. Used for panoramic and surveillance cameras.

The pinhole model projects a 3D point onto a 2D pixel through the camera matrix `K`; the distortion coefficients then account for the lens bending that the ideal pinhole ignores.

---

## When you need it (and when you don't)

| Calibrate when… | Skip it when… |
|-----------------|---------------|
| You need accurate **measurements** or spatial reasoning | The task is qualitative (is the line left or right?) |
| Lens **distortion** is significant and affects results | Distortion is minimal and harmless |
| You use **multiple cameras** that must align | A single camera, casual detection |

For deep-learning [object detection](deep_vision.md) on a normal lens, you often do not calibrate — the network tolerates mild distortion. But for measuring how far away a detected object is, or stitching multiple views, calibration is the foundation.

---

## The calibration procedure

The standard method uses a **known pattern** — usually a printed **checkerboard**, whose square size and corner layout you know exactly. The procedure:

1. **Capture** many images of the checkerboard from different angles and distances.
2. **Detect corners** in each image — their pixel positions are the *image points*; their known real-world positions are the *object points*.
3. **Estimate** the intrinsic matrix and distortion coefficients by solving for the parameters that best explain how the known 3D pattern maps to the observed 2D corners across all views.
4. **Apply** the parameters to undistort images (or to project between world and image coordinates).

OpenCV provides this end to end (needs OpenCV linked, so not runnable here):

<!-- no-ce -->
```cpp
#include <opencv2/opencv.hpp>

// objectPoints: the checkerboard's 3D corner coordinates, one set per view
// imagePoints:  the detected 2D corners in each image (from cv::findChessboardCorners)

cv::Mat K, distCoeffs;                       // outputs: camera matrix + distortion
std::vector<cv::Mat> rvecs, tvecs;           // per-view extrinsics (rotation, translation)

double rmsError = cv::calibrateCamera(
    objectPoints, imagePoints, imageSize,
    K, distCoeffs, rvecs, tvecs);            // estimate everything at once

// Use the result to straighten a new image:
cv::Mat undistorted;
cv::undistort(image, undistorted, K, distCoeffs);
```

`cv::findChessboardCorners` detects the corners for step 2; `cv::calibrateCamera` does steps 3; `cv::undistort` does step 4.

!!! tip "Check the reprojection error"
    `cv::calibrateCamera` returns the **RMS reprojection error** — how far, on average, the model's predicted corner positions fall from the detected ones, in pixels. A good calibration is well under one pixel. A large error means bad input: too few views, all from similar angles, a blurry or partially-hidden board. Capture 15–20 views covering different angles, distances, and image regions.

---

## Where it fits

Calibration sits at the *front* of a vision pipeline: undistort the frame first, then run [detection](deep_vision.md) or measurement on a geometrically faithful image. The **extrinsic** parameters connect the camera to the rest of the robot — once you know the camera's pose, a detected object's pixel position becomes a direction (and, with depth or known geometry, a position) the [control](../Chapter3/real_time.md) system can act on. That bridge from "pixels" to "where things are" is the reason calibration matters for a cyber-physical system, not just a photo.

---

## Summary

- **Calibration** measures a camera's imperfections so image coordinates map faithfully to the real world — needed for **measurement** and spatial reasoning, skippable for purely qualitative tasks.
- It recovers **intrinsic** parameters (focal length, optical centre, **distortion coefficients** — fixed for a camera+lens) and **extrinsic** parameters (the camera's **pose** in the world — changes as it moves).
- Parameters are defined against a **camera model**: **pinhole** for normal lenses, **fisheye** for very wide ones.
- The procedure: capture many **checkerboard** views, detect corners, estimate with `cv::calibrateCamera`, and correct with `cv::undistort`. Check the **RMS reprojection error** (well under a pixel is good).
- Undistort *before* detection; the **extrinsics** turn detected pixels into directions the [control system](../Chapter3/real_time.md) can use. Next: [Deep Vision](deep_vision.md), what modern detection actually does.

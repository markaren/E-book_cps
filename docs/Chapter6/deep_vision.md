# Deep Vision

The [OpenCV](opencv.md) operations so far — edges, thresholds, colour filters — are **classic** computer vision: a human expert designs the steps. **Deep vision** flips that: a neural network *learns* what to look for from examples. Over the last decade it has gone from research curiosity to the default for any non-trivial perception task, and it is what powers a robot recognising objects, people, or signs. This chapter is conceptual — it explains *what* deep vision does and *where* it fits — because the heavy lifting is done with Python frameworks, while C++'s job is to *run* the result ([next chapter](onnx.md)).

---

## Classic vs deep vision

The difference is **who designs the features** — the visual patterns the system keys on.

- **Classic machine vision** uses *hand-crafted* features: edge detectors, corner detectors, and descriptors like SIFT or HOG, combined by an algorithm an engineer wrote. It is fast, needs little or no data, and runs on a microcontroller — but it is brittle, struggling with changing light, occlusion, and cluttered scenes.
- **Deep vision** uses a neural network that *learns* features automatically from a large labelled dataset. It handles complex, unstructured scenes and adapts to new tasks by retraining — but it needs that data, a training step, and more compute.

| | Classic machine vision | Deep vision |
|---|------------------------|-------------|
| Features | Hand-crafted (edges, SIFT, HOG) | Learned from data |
| Data needed | Little | Large labelled datasets |
| Robustness | Brittle to lighting/occlusion | Handles variation well |
| Compute | Low (runs on small devices) | Higher; a training step |
| Best for | Structured tasks: barcodes, template matching | Unstructured scenes: detection, segmentation |
| Examples | OpenCV pipelines, rule-based systems | YOLO, Faster R-CNN, DETR, U-Net |

Neither is obsolete. For a controlled task — read a barcode, find a bright marker on a known background — classic vision is simpler, faster, and needs no training. Reach for deep vision when the scene is variable and the thing you are detecting is hard to describe with rules.

---

## The vision tasks

"Computer vision" is several distinct tasks, and knowing which you need shapes everything downstream:

| Task | Output |
|------|--------|
| **Classification** | A label for the whole image ("this is a cat") |
| **Object detection** | **Bounding boxes** + labels for each object ("cat here, dog there") |
| **Segmentation** | A class label for **every pixel** (semantic: all cats; instance: this cat vs that cat) |
| **Pose / keypoint** | Specific points — joints of a person, corners of a marker |
| **Tracking** | A consistent ID for each object **across frames** |

For a mobile robot, **object detection** is the workhorse: "what is around me, and where in the frame is it?" — boxes you can act on.

---

## CNNs and YOLO

The engine behind deep vision is the **Convolutional Neural Network (CNN)**. A convolutional layer slides small **filters** across the image, each responding to a local pattern — an edge, a texture, a shape. Stacking layers builds a **hierarchy**: early layers detect edges, later ones combine those into parts, and deeper ones into whole objects. The crucial point is that the network *learns these filters from data* rather than being told what to look for — automatic feature learning is exactly what makes deep vision so effective on messy real-world images.

For real-time robotics the standout model family is **YOLO** ("You Only Look Once"). YOLO treats detection as a **single regression**: it predicts all bounding boxes and class probabilities from the whole image in **one pass** through the network, which makes it fast enough to run on live video. Other notable architectures — **Faster R-CNN** (accurate, two-stage), **DETR** (transformer-based), **EfficientNet** (efficient classification) — trade speed for accuracy differently, but YOLO's single-pass design is why it dominates real-time use.

---

## Training: mostly Python

Training a model is a **Python** activity, and worth understanding even though you will rarely do it in C++:

1. **Start from a pre-trained model.** Models trained on big datasets (e.g. **COCO**, 80 common object classes) already detect everyday objects. You rarely train from scratch.
2. **Annotate a dataset** for your *own* classes if COCO does not cover them — draw bounding boxes, assign labels, in a tool that exports a standard format.
3. **Augment** (flip, rotate, adjust brightness) to enlarge and diversify the data, and **split** it roughly 70 % train / 20 % validation / 10 % test.
4. **Train**, ideally on a **GPU**, with a framework — **PyTorch**, **TensorFlow**/Keras, or **Ultralytics** for YOLO. This is **transfer learning**: you fine-tune a pre-trained network on your data, which needs far less data and time than starting fresh.

The output is a trained model file in the framework's format. Getting that file to *run on the robot* — in C++, in real time — is the deployment problem the [next chapter](onnx.md) solves.

!!! note "C++'s role is inference, not training"
    You train in Python because that is where the frameworks, the GPU tooling, and the fast iteration live. You **deploy** in C++ because that is where the robot's real-time loop, control code, and [communication](../Chapter4/serialization.md) live. The handoff between them is a portable model format — [ONNX](onnx.md) — covered next. Trying to *train* in C++ is swimming upstream; running *inference* in C++ is exactly right.

---

## Summary

- **Classic** vision uses **hand-crafted** features (edges, SIFT, HOG) — fast, data-free, but brittle; **deep** vision **learns** features from data — robust to variation, but needs data, training, and compute. Use classic for controlled tasks, deep for messy ones.
- The tasks differ: **classification** (whole-image label), **object detection** (bounding boxes — the robot workhorse), **segmentation** (per-pixel), **pose** (keypoints), **tracking** (IDs across frames).
- **CNNs** slide learned **filters** to build a hierarchy of features automatically; **YOLO** detects everything in a **single pass**, making it fast enough for live video.
- **Training is a Python job** — start from a pre-trained model (e.g. COCO), annotate/augment/split your data, and fine-tune on a GPU (PyTorch, TensorFlow, Ultralytics) via **transfer learning**.
- **C++'s role is inference/deployment**, not training. Next: [Model Deployment & ONNX](onnx.md) takes a trained model and runs it on the robot in C++.

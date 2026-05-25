# Chapter 6 Exercises

Work through these after reading Chapter 6 (Computer Vision). **Try each one yourself before revealing the solution.** Type the code into CLion and run it.

When you open a solution it appears **blurred** — click it once more to reveal it.

The first three are runnable programs that need **no OpenCV** — they exercise the *maths and structure* behind a vision pipeline, which is where the real bugs live. The last is a **design** exercise.

---

## 1. Intersection over Union

*Practises: [Deep Vision](deep_vision.md)*

Detectors produce overlapping boxes for the same object, and *non-maximum suppression* removes duplicates by measuring how much two boxes overlap — the **Intersection over Union** (IoU): the area they share divided by the area they jointly cover. Write `iou(a, b)` for two axis-aligned boxes and test it on two overlapping squares.

> Hint: the intersection rectangle has corners `max(x1)`, `max(y1)`, `min(x2)`, `min(y2)`; its width/height clamp to `0` if the boxes do not overlap. Union = areaA + areaB − intersection.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <algorithm>
    #include <iostream>

    struct Box { double x1, y1, x2, y2; };   // top-left and bottom-right corners

    double iou(const Box& a, const Box& b) {
        double ix1 = std::max(a.x1, b.x1);
        double iy1 = std::max(a.y1, b.y1);
        double ix2 = std::min(a.x2, b.x2);
        double iy2 = std::min(a.y2, b.y2);

        double iw = std::max(0.0, ix2 - ix1);     // 0 if no horizontal overlap
        double ih = std::max(0.0, iy2 - iy1);
        double intersection = iw * ih;

        double areaA = (a.x2 - a.x1) * (a.y2 - a.y1);
        double areaB = (b.x2 - b.x1) * (b.y2 - b.y1);
        double uni = areaA + areaB - intersection;

        return uni > 0.0 ? intersection / uni : 0.0;
    }

    int main() {
        Box a{0, 0, 2, 2};      // area 4
        Box b{1, 1, 3, 3};      // area 4, overlapping in a 1×1 corner
        std::cout << iou(a, b) << "\n";   // 1 / (4 + 4 - 1) = 0.142857
    }
    ```

    The two squares share a 1×1 corner (intersection = 1) and jointly cover 7 units (union = 4 + 4 − 1), so IoU ≈ `0.143`. Clamping the intersection width and height to `0` is what makes non-overlapping boxes return `0` instead of a spurious negative area. A detector keeps the highest-confidence box and discards any other whose IoU with it exceeds a threshold (say 0.5) — that is non-maximum suppression, the post-processing step from [Model Deployment](onnx.md).

    </div>

---

## 2. Decode a YOLO box

*Practises: [Deep Vision](deep_vision.md), [Model Deployment & ONNX](onnx.md)*

YOLO outputs each box as a **normalised centre and size**: `(cx, cy, w, h)`, all in `0..1` relative to the image. To draw or act on it you need **pixel corners** `(xmin, ymin, xmax, ymax)`. Write the conversion for a given image width and height.

> Hint: the top-left corner is `(cx − w/2, cy − h/2)`, still normalised; multiply by the image dimensions to get pixels. The box width in pixels is `w * imageWidth`.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>

    struct PixelBox { int xmin, ymin, xmax, ymax; };

    PixelBox yoloToPixels(double cx, double cy, double w, double h, int imgW, int imgH) {
        double x = (cx - w / 2.0) * imgW;        // left edge, in pixels
        double y = (cy - h / 2.0) * imgH;        // top edge, in pixels
        double boxW = w * imgW;
        double boxH = h * imgH;
        return { static_cast<int>(x), static_cast<int>(y),
                 static_cast<int>(x + boxW), static_cast<int>(y + boxH) };
    }

    int main() {
        // A box centred in the image, half its width and a third of its height, on 640×480:
        PixelBox p = yoloToPixels(0.5, 0.5, 0.5, 0.33, 640, 480);
        std::cout << p.xmin << "," << p.ymin << " - " << p.xmax << "," << p.ymax << "\n";
        // 160,160 - 480,319
    }
    ```

    Converting centre+size to corner+size, then scaling normalised coordinates by the image dimensions, is exactly the post-processing every YOLO deployment does between `net.forward()` and drawing a box. Getting it wrong — forgetting the `/2`, or mixing up normalised and pixel units — is the classic "the model works but the boxes are in the wrong place" bug from [Model Deployment](onnx.md).

    </div>

---

## 3. A frame pipeline that never stalls

*Practises: [Model Deployment & ONNX](onnx.md), [Condition Variables](../Chapter2/condition_variables.md)*

Model the capture→inference pipeline with **no OpenCV**: a producer "captures" frames (just integers) and a consumer "processes" them, connected by a thread-safe queue, so the producer never waits for the slow consumer. The producer pushes five frames then closes the queue; the consumer drains it and exits cleanly.

> Hint: this is the [producer/consumer](../Chapter2/condition_variables.md) pattern. The consumer `pop`s in a loop; have `pop` return a `std::optional<int>` that is empty once the queue is closed *and* drained, so the consumer's loop ends naturally.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <condition_variable>
    #include <iostream>
    #include <mutex>
    #include <optional>
    #include <queue>
    #include <thread>

    class FrameQueue {
    public:
        void push(int frame) {
            { std::lock_guard<std::mutex> lock(mutex_); queue_.push(frame); }
            cv_.notify_one();
        }
        void close() {
            { std::lock_guard<std::mutex> lock(mutex_); closed_ = true; }
            cv_.notify_all();
        }
        std::optional<int> pop() {                  // empty when closed and drained
            std::unique_lock<std::mutex> lock(mutex_);
            cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
            if (queue_.empty()) return std::nullopt;
            int frame = queue_.front();
            queue_.pop();
            return frame;
        }
    private:
        std::mutex mutex_;
        std::condition_variable cv_;
        std::queue<int> queue_;
        bool closed_ = false;
    };

    int main() {
        FrameQueue frames;

        std::jthread inference([&] {
            while (auto frame = frames.pop()) {     // stops when pop() returns nullopt
                std::cout << "processed frame " << *frame << "\n";
            }
        });

        for (int i = 1; i <= 5; ++i) {
            frames.push(i);                          // "capture"
        }
        frames.close();                              // no more frames coming
    }
    ```

    The capture loop pushes frames and never blocks on inference — exactly the structure of the real [vision pipeline](onnx.md), with `cv::Mat` swapped for `int`. The `std::optional` return lets the consumer's `while` loop end the instant the queue is closed and empty, so the `jthread` joins cleanly. For a *real-time* system you would go one step further and **drop stale frames** — cap the queue and discard the oldest when full — so the consumer always works on a recent frame rather than a growing backlog ([Real-Time & Timing](../Chapter3/real_time.md)).

    </div>

---

## 4. Where does inference run?

*Practises: [Model Deployment & ONNX](onnx.md)*

A robot has a 30 FPS camera and must detect obstacles for navigation. Your YOLO model takes about **150 ms per frame** on the Raspberry Pi's CPU (≈ 6–7 FPS). The camera produces frames far faster than the Pi can process them. Decide how to deploy the perception, and justify the trade-offs. There is no single right answer.

> Hint: 7 FPS of processing against 30 FPS of frames means you *cannot* run the model on every frame. Think about: making inference faster, processing fewer frames, or moving inference elsewhere — and what each costs.

??? success "Show discussion"

    <div class="spoiler" markdown title="Click to reveal">

    A good answer weighs three levers and picks for *this* task (obstacle avoidance):

    - **Make inference faster on the Pi.** Use a smaller/quantised model and a smaller input size (e.g. 320×320 instead of 640×640). This raises FPS at some cost to accuracy — often a fine trade for spotting nearby obstacles, where you do not need to read fine detail.
    - **Process fewer frames, newest first.** You physically cannot do 30 FPS, so **drop stale frames** and always run on the latest one ([the pipeline warning](onnx.md)). Detecting on every 4th–5th frame at low latency is far better for navigation than a growing backlog of old detections — *fresh-but-coarse beats accurate-but-late*.
    - **Offload inference.** Send frames (or a downscaled stream) over the [network](../Chapter4/sockets.md) to a laptop or a Jetson and get detections back. This buys accuracy and frame rate but adds **network latency and bandwidth**, and a dependency on the link staying up — risky for a safety function if the network drops.

    For obstacle avoidance, the defensible default is **on-device, small model, drop-stale frames**: latency and reliability matter more than precision, and you do not want obstacle detection to fail when the Wi-Fi does. If the task instead needed fine recognition (reading a label, say) and tolerated latency, **offloading** to a stronger machine — or putting a **Jetson with TensorRT** on the robot — would be the better call. The reasoning — matching where inference runs to the task's latency, accuracy, and reliability needs — is the real deliverable.

    </div>

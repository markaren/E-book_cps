# Camera & Detections

Your simulator has a [camera](../Chapter6/virtual_environments.md#the-camera-is-a-sensor-too) producing `cv::Mat`s, and [Part 4](../Chapter4/serialization.md#streaming-images) taught you to compress and stream them as JPEG bytes. This page puts those same frames onto the **ROS2 graph** — where a teammate's node, a recording tool, or a stock viewer can subscribe without knowing anything about your simulator — and publishes back what the [detector](../Chapter6/onnx.md) finds. It is the [talker/listener](ros2_workspace.md) flow again, carrying the project's real payload.

---

## The message: `CompressedImage`

ROS2's standard image types live in `sensor_msgs`:

| Type | What it carries | When |
|---|---|---|
| `sensor_msgs/msg/Image` | Raw pixels + width/height/encoding | Same-host pipelines where no decode is wanted |
| `sensor_msgs/msg/CompressedImage` | **Encoded bytes** (JPEG/PNG) + a `format` string | Anything that crosses a network — your case |

`CompressedImage` is [Part 4's payload](../Chapter4/serialization.md#streaming-images) in a typed envelope: a `header` (timestamp + `frame_id`), a `format` (`"jpeg"`), and `data` — a plain `std::vector<uint8_t>` that `cv::imencode` can fill **directly**. Do the raw-image arithmetic from Part 4 before choosing `Image` for anything networked: ~900 KB per frame, twenty times a second. (Tutorials often route images through the `cv_bridge` helper package; with `CompressedImage` you do not need it — the bytes are just bytes.)

---

## Publishing the camera

A node that reads the newest frame from the simulator's [`Mailbox`](../Chapter2/condition_variables.md#a-latest-value-mailbox) and publishes at the sensor rate:

<!-- no-ce -->
```cpp
#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/compressed_image.hpp"
#include <opencv2/imgcodecs.hpp>
using namespace std::chrono_literals;

class CameraPublisher : public rclcpp::Node {
public:
    explicit CameraPublisher(Mailbox<cv::Mat>& frames)
        : Node("camera_publisher"), frames_(frames) {
        pub_ = create_publisher<sensor_msgs::msg::CompressedImage>(
            "camera/image/compressed", rclcpp::SensorDataQoS());   // best-effort, keep-last
        timer_ = create_wall_timer(50ms, [this] { tick(); });      // 20 Hz — the sensor rate
    }
private:
    void tick() {
        const auto frame = frames_.latest();    // newest frame, or nullopt before the first
        if (!frame) return;
        sensor_msgs::msg::CompressedImage msg;
        msg.header.stamp = now();
        msg.header.frame_id = "camera_link";
        msg.format = "jpeg";
        cv::imencode(".jpg", *frame, msg.data,  // encode straight into the message
                     {cv::IMWRITE_JPEG_QUALITY, 80});
        pub_->publish(msg);
    }
    Mailbox<cv::Mat>& frames_;
    rclcpp::Publisher<sensor_msgs::msg::CompressedImage>::SharedPtr pub_;
    rclcpp::TimerBase::SharedPtr timer_;
};
```

Three deliberate choices:

- **`rclcpp::SensorDataQoS()`.** ROS2's ready-made QoS profile for exactly the judgement you made in [exercise 1](exercises.md): best-effort, shallow keep-last history — fresh-or-nothing, never a backlog of stale frames. A `RELIABLE` deep-queue image topic is the textbook way to watch a subscriber fall thirty seconds behind reality.
- **Encode into `msg.data` directly.** `CompressedImage::data` is a `std::vector<uint8_t>`, so `cv::imencode` writes the JPEG straight into the message — no intermediate buffer, no copy.
- **The `Mailbox` is the boundary.** The timer callback runs on the **executor thread** ([concepts](ros2_concepts.md)); the render thread `set()`s frames. Two threads, one shared object, already mutex-guarded — [Part 2's rule](../Chapter2/sharing_data.md) holding the design together.

!!! note "Two loops, one process"
    The simulator's render loop owns `main` — `canvas.animate` does not return until the window closes — and `rclcpp::spin` also wants to own a thread. Give the executor its own:

    <!-- no-ce -->
    ```cpp
    int main(int argc, char** argv) {
        rclcpp::init(argc, argv);
        Mailbox<cv::Mat> frames;

        auto node = std::make_shared<CameraPublisher>(frames);
        std::jthread ros([node] { rclcpp::spin(node); });   // executor on its own thread

        runSimulator(frames);     // the Part 6 render loop — blocks until the window closes
        rclcpp::shutdown();       // makes spin() return, so the jthread can join
    }
    ```

    From this point on, *every callback in the node runs concurrently with your render and physics code* — which is why the mailbox, and nothing else, is shared between them.

---

## Watching the stream

The payoff of the graph is that **tools you did not write** can consume your camera:

```powershell
ros2 topic hz /camera/image/compressed     # is it really 20 Hz?
ros2 topic bw /camera/image/compressed     # how many bytes/s is the JPEG stream?
ros2 run rqt_image_view rqt_image_view     # pick the topic — your car's view, live
```

`rqt_image_view` decodes and displays compressed topics directly. Watching your own simulator's camera in a stock ROS2 tool, with zero code, is the "adopting the conventions buys the ecosystem" argument from [concepts](ros2_concepts.md) made visible.

---

## Detections onto the graph

[Part 6's pipeline](../Chapter6/onnx.md#a-real-time-vision-pipeline) turns frames into boxes; publish those too, so the planner — or a teammate's node — can react to them. The standard type is `vision_msgs/msg/Detection2DArray`: an array of detections, each a bounding box plus scored class hypotheses:

<!-- no-ce -->
```cpp
#include "vision_msgs/msg/detection2_d_array.hpp"

vision_msgs::msg::Detection2DArray msg;
msg.header.stamp = imageStamp;            // the FRAME's stamp — see below
msg.header.frame_id = "camera_link";
for (const auto& box : boxes) {           // your detector's output
    vision_msgs::msg::Detection2D d;
    d.bbox.center.position.x = box.cx;    // pixel coordinates in the camera image
    d.bbox.center.position.y = box.cy;
    d.bbox.size_x = box.w;
    d.bbox.size_y = box.h;
    auto& hyp = d.results.emplace_back();
    hyp.hypothesis.class_id = box.label;  // e.g. "cone"
    hyp.hypothesis.score = box.confidence;
    msg.detections.push_back(d);
}
detectionsPub_->publish(msg);
```

Note the **stamp**: it is the *image's* timestamp, not `now()`. A detection describes the world as it was when the frame was captured, and inference took tens of milliseconds since. Copy the `header` from the `CompressedImage` the detector consumed, and every subscriber can pair boxes with the frames they belong to.

Detections are small and low-rate, so QoS matters less here — the default (reliable, keep-last 10) is fine; the freshness argument bites on the megabyte-scale image stream, not on a handful of boxes. And reusing the standard type instead of inventing your own is the [`.msg` reuse rule](ros2_concepts.md) paying out: `rqt`, `ros2 bag`, and anyone else's node already understand a `Detection2DArray`.

!!! tip "Adding `vision_msgs` to the toolchain"
    `vision_msgs` is its own package: add `ros-jazzy-vision-msgs` to the [RoboStack environment](ros2_windows.md)'s `pixi.toml`, then declare `vision_msgs` in your package's `package.xml` and `CMakeLists.txt` like any other dependency.

---

## Summary

- Networked images ride **`sensor_msgs/CompressedImage`** — [Part 4's JPEG bytes](../Chapter4/serialization.md#streaming-images) in a typed envelope (`header`, `format`, `data`); `cv::imencode` fills `msg.data` directly, no `cv_bridge` needed.
- Publish with **`rclcpp::SensorDataQoS()`** — best-effort, keep-last: the [exercise 1](exercises.md) judgement, available as a named profile.
- The executor runs on **its own `std::jthread`** beside the render loop; the **`Mailbox`** is the only shared state between them — [Part 2's rules](../Chapter2/sharing_data.md) at the boundary.
- Verify with **`ros2 topic hz`/`bw`**, and watch live in **`rqt_image_view`** — stock tools consuming your simulator is the ecosystem payoff.
- Publish detections as **`vision_msgs/Detection2DArray`**, stamped with the **image's** time, so boxes stay paired with the frames they describe. Next: the [exercises](exercises.md).

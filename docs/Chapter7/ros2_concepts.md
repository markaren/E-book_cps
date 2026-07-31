# ROS2 Concepts

You already know how to make programs on different machines talk: [sockets](../Chapter4/sockets.md), [serialization](../Chapter4/serialization.md), [MQTT and RPC](../Chapter4/mqtt_rpc.md). **ROS2** (Robot Operating System 2) is not a competitor to those — it is a whole **robotics middleware** built on the same ideas, packaged with the conventions, tools, and message types the robotics world has standardised on. This chapter teaches it as a **delta from Part 4**: almost everything here is a concept you already met, wearing a robotics name.

!!! note "It is not an operating system"
    Despite the name, ROS2 is a set of **libraries, tools, and conventions** that run on top of Linux, Windows, or macOS — a framework for building distributed robot software, not a kernel.

---

## The one-table translation

| You know (Part 4) | ROS2 calls it | Difference that matters |
|---|---|---|
| MQTT **topic** | ROS2 **topic** | Same publish/subscribe by name — but **typed** |
| MQTT **broker** | **DDS discovery** | **No broker** — nodes find each other peer-to-peer |
| MQTT **QoS** (0/1/2) | **DDS QoS** (reliability, history, depth, …) | Richer, per-topic |
| **`.proto`** + `protoc` | **`.msg`** + IDL codegen | Same "define a schema, generate code" workflow |
| gRPC **RPC call** | ROS2 **service** | Request/response, one reply |
| (no direct match) | ROS2 **action** | A long-running request with **feedback** and cancellation |
| A **program** | A **node** | A participant in the graph |
| A CMake project | A **package** | Unit of build + distribution |

If you understand [MQTT and RPC](../Chapter4/mqtt_rpc.md), you understand 80% of ROS2 already. The rest is the ecosystem.

---

## Nodes and the graph

A **node** is a single participant in the system — one process, or one object inside a process — that does one job: read a sensor, run a controller, drive the wheels. Nodes are the units you compose a robot from. A running robot is a **graph** of nodes exchanging data.

Because nodes are separate participants, this is a **concurrency** story, and everything from [Part 2](../Chapter2/processes_threads.md) applies. When messages arrive, ROS2 runs your **callbacks** on an **executor** — a thread (or a pool of them) that pulls incoming messages and invokes the matching callback. *Your callback runs on an executor thread, not on `main`.* So two callbacks touching the same data can race exactly like any other threads, and the [shared-data](../Chapter2/sharing_data.md) rules — a [mutex](../Chapter2/sharing_data.md), or handing work across a [queue/mailbox](../Chapter2/condition_variables.md) — are the same ones you already use. (More in [ROS2 Workspace](ros2_workspace.md).)

---

## Topics: typed publish/subscribe, no broker

A **topic** is a named channel. Publishers send messages to it; subscribers receive them; neither addresses the other. This is [MQTT's pub/sub](../Chapter4/mqtt_rpc.md) — with two differences.

**No broker.** MQTT routes everything through a central broker you must run (Mosquitto). ROS2 uses **DDS** (Data Distribution Service) underneath, which is **brokerless**: nodes **discover** each other automatically over the network — each announces the topics it publishes and subscribes to, and DDS wires up matching pairs directly, peer-to-peer. There is no central process to start or to be a single point of failure. The trade is that discovery traffic is chattier and configuration lives per-node rather than in one broker.

**Typed.** An MQTT payload is just bytes (you [serialize](../Chapter4/serialization.md) them yourself, often as JSON). A ROS2 topic carries a **specific message type**, and both ends are generated from the same schema — you cannot accidentally publish the wrong shape.

### Messages: `.msg` is `.proto` in a robotics hat

You define a message type in a **`.msg`** file — fields and types, one per line:

```
# Twist.msg (simplified) — a velocity command
Vector3 linear
Vector3 angular
```

At build time an **IDL code generator** turns each `.msg` into a C++ struct (and a Python class, etc.), exactly like [`protoc` turns a `.proto`](../Chapter4/mqtt_rpc.md) into code. The generated type is the single source of truth both ends share — a C++ publisher and a Python subscriber built from the same `.msg` interoperate for free. ROS2 ships a large library of **standard messages** (`std_msgs`, `sensor_msgs`, `geometry_msgs`) so common data — an image, a laser scan, a pose — has an agreed type you should reuse rather than reinvent.

---

## QoS: richer than MQTT's three levels

MQTT gives you three [QoS levels](../Chapter4/mqtt_rpc.md) (0/1/2). DDS exposes QoS as a set of independent **policies** per topic; the two you will actually set are:

| Policy | Options | Analogy |
|---|---|---|
| **Reliability** | `BEST_EFFORT` / `RELIABLE` | MQTT QoS 0 vs QoS 1 — retransmit lost samples, or don't |
| **History + depth** | `KEEP_LAST(n)` / `KEEP_ALL` | How many recent samples to buffer for a slow subscriber |

The choice is the same "is a lost message acceptable?" judgement from [Part 4](../Chapter4/exercises.md). High-rate lidar or camera data → **best-effort, keep-last small**: a dropped frame is replaced in milliseconds, and you never want a slow consumer stalling the producer with a backlog of stale frames. A one-off command or a map → **reliable**: it must arrive. Publisher and subscriber QoS must be **compatible**, or DDS refuses to connect them — a common first-time gotcha.

!!! warning "Incompatible QoS = silent no-connection"
    If a publisher is `BEST_EFFORT` and a subscriber demands `RELIABLE`, they will **not** match and no data flows — with no error thrown. When a subscription is silent, check the QoS on both ends first.

---

## Services and actions

Topics are for **streaming state** (the MQTT case). For **"do X and tell me the result"** — the [RPC](../Chapter4/mqtt_rpc.md) case — ROS2 has two shapes:

- A **service** is a plain request/response: call it, get one reply. This is a gRPC call by another name — "compute a path", "reset odometry". Defined in a `.srv` file (request fields, then `---`, then response fields).
- An **action** is a service for work that **takes time**: it streams **feedback** while running and can be **cancelled**. "Drive to this waypoint" is an action — you want progress updates and the ability to abort, which a one-shot service cannot give.

Rule of thumb, same as Part 4: **topic** to broadcast state to anyone interested, **service** to invoke a quick operation and get its result, **action** when that operation is long-running and you want feedback.

---

## A minimal publisher and subscriber (rclcpp)

`rclcpp` is the C++ client library. A node is a class deriving from `rclcpp::Node`. A publisher counts up and sends a string every 500 ms:

<!-- no-ce -->
```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
using namespace std::chrono_literals;

class Talker : public rclcpp::Node {
public:
    Talker() : Node("talker") {
        pub_ = create_publisher<std_msgs::msg::String>("chatter", 10);   // topic, depth 10
        timer_ = create_wall_timer(500ms, [this] { tick(); });           // periodic callback
    }
private:
    void tick() {
        std_msgs::msg::String msg;
        msg.data = "hello " + std::to_string(count_++);
        RCLCPP_INFO(get_logger(), "Publishing: '%s'", msg.data.c_str());
        pub_->publish(msg);
    }
    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
    rclcpp::TimerBase::SharedPtr timer_;
    size_t count_ = 0;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<Talker>());   // executor: run callbacks until shutdown
    rclcpp::shutdown();
}
```

The subscriber registers a callback on the same topic:

<!-- no-ce -->
```cpp
class Listener : public rclcpp::Node {
public:
    Listener() : Node("listener") {
        sub_ = create_subscription<std_msgs::msg::String>(
            "chatter", 10,
            [this](const std_msgs::msg::String& msg) {                  // runs on executor thread
                RCLCPP_INFO(get_logger(), "I heard: '%s'", msg.data.c_str());
            });
    }
private:
    rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};
```

Note the pieces you recognise: `create_publisher`/`create_subscription` name a **topic** and a **depth** (that `10` is the QoS history depth); `create_wall_timer` is the [periodic task](../Chapter3/real_time.md) from Part 3; the shared-pointer / `create` idiom mirrors [threepp](../Chapter6/virtual_environments.md) and Part 1's [smart pointers](../Chapter1/smart_pointers.md); and `rclcpp::spin` is the **executor** driving callbacks — the concurrency boundary where Part 2's rules kick in.

---

## Why ROS2 for robots

You could build all of this on raw MQTT and sockets — so why adopt ROS2? The **ecosystem**. Because everyone in robotics shares the same node/topic/message conventions, an enormous body of ready-made tooling just works with your graph:

- **TF** — tracks the moving coordinate frames of a robot (wheel → chassis → world) and converts between them, so "where is this lidar point in world space?" is a library call, not maths you rewrite.
- **RViz** — a 3D visualiser that shows any standard topic live: point clouds, camera images, poses, paths. Invaluable for debugging perception.
- **Nav2**, MoveIt, and hundreds of driver and algorithm packages — navigation, arm motion planning, sensor drivers — all speaking the same topics.

Adopting the conventions is the price of admission to that ecosystem, and for a real robot it is well worth paying.

---

## Summary

- **ROS2** is a robotics **middleware** (libraries + tools + conventions), not an OS. Read it as a **delta from Part 4**: topics, typed messages, QoS, and RPC-style calls you already understand.
- **Nodes** are participants in a **graph**; incoming messages run **callbacks on an executor thread** — a concurrency boundary, so [Part 2](../Chapter2/sharing_data.md)'s shared-data rules apply.
- **Topics** are [pub/sub like MQTT](../Chapter4/mqtt_rpc.md) but **typed** and **brokerless** — **DDS** discovers nodes peer-to-peer. **Messages** are `.msg` files + IDL codegen, the [`.proto`/`protoc`](../Chapter4/mqtt_rpc.md) workflow.
- **QoS** is DDS policies (**reliability**, **history/depth**) — richer than MQTT's three levels, and must be **compatible** on both ends or no data flows.
- **Services** = one-shot [RPC](../Chapter4/mqtt_rpc.md); **actions** = long-running with feedback + cancel.
- ROS2 earns its place through its **ecosystem** — TF, RViz, Nav2, MoveIt, drivers — all speaking the same graph. Next: [running ROS2 on Windows](ros2_windows.md).

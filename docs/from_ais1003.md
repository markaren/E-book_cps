# From AIS1003 to AIS2203

AIS1003 taught you to write a correct C++ program: one that takes input, computes a result, and produces output, all in a single, predictable sequence of steps. AIS2203 is about what happens when that neat picture breaks down — when several things run **at the same time**, when work happens on **another machine**, and when the answer has to arrive **before a deadline**.

This page is a bridge. It lists what the course assumes you already own, and previews the three new ideas the rest of the book is built on.

---

## What you should already know

This book does not re-teach the AIS1003 material. You should be comfortable with all of the following; if any is shaky, review the linked AIS1003 chapter before going on.

| You should be able to… | AIS1003 reference |
|------------------------|-------------------|
| Write classes with private state and a clean public interface | [Classes](https://markaren.github.io/E-book_cpp/Chapter4/classes/) |
| Use `std::vector`, `std::map` and the standard algorithms | [Standard Library](https://markaren.github.io/E-book_cpp/Chapter3/standard_library/) |
| Explain RAII and why you rarely write `new`/`delete` | [RAII](https://markaren.github.io/E-book_cpp/Chapter4/raii/) |
| Tell a value from a reference from a pointer, and use `const` correctly | [Values, References & Pointers](https://markaren.github.io/E-book_cpp/Chapter4/types_refs_ptrs/) |
| Configure a project with CMake and link a library | [CMake](https://markaren.github.io/E-book_cpp/Chapter2/cmake_intro/) |
| Handle errors with exceptions and `std::optional` | [Error Handling](https://markaren.github.io/E-book_cpp/Chapter6/error_handling/) |
| Write a test with Catch2 | [Testing](https://markaren.github.io/E-book_cpp/Chapter6/testing/) |

We **extend** several of these. [Part 1](Chapter1/ownership.md) revisits ownership, move semantics and smart pointers — not as review, but because concurrency forces you to think about *who owns what* far more carefully than a single-threaded program ever does.

---

## Three new ideas

### 1. Things happen at the same time

In AIS1003, your program had a single thread of control: one instruction followed another, and you always knew which line ran next. From now on, two pieces of your code can run **simultaneously** — on different CPU cores, or interleaved so finely that they may as well be.

That breaks an assumption you never knew you were making. This is fine single-threaded, and broken with two threads:

<!-- no-ce -->
```cpp
int counter = 0;

void tick() {
    counter = counter + 1;   // read counter, add 1, write it back
}
```

If two threads run `tick()` at once, both can read the same old value before either writes back, and one increment vanishes. The bug is not in the logic — it is in the *timing*. [Part 2](Chapter2/processes_threads.md) is about seeing these hazards and shutting them down.

### 2. The other end is a different program

A cyber-physical system is rarely one program. It is a sensor node talking to a controller, a robot talking to an operator's laptop, an embedded board talking to the cloud. Those programs do not share variables — they share **messages**, sent over a wire or a network.

That raises questions a single program never faces: how do you turn an object into bytes and back ([serialization](Chapter4/serialization.md))? What happens when a message is lost or arrives late ([TCP vs UDP](Chapter4/sockets.md))? How do two machines agree on what the bytes *mean* ([Modbus](Chapter4/modbus.md), [MQTT & RPC](Chapter4/mqtt_rpc.md))? [Part 4](Chapter4/serialization.md) answers them.

### 3. Late is wrong

A desktop program that takes an extra 50 ms is slightly annoying. A balance controller that takes an extra 50 ms tips the robot over. In a **real-time** system, a correct answer delivered too late is a wrong answer.

You will learn to think about deadlines, latency and jitter, to run periodic tasks on a schedule, and to react to asynchronous events promptly. [Part 3](Chapter3/real_time.md) makes timing a first-class concern rather than an afterthought.

---

## Where the code runs

AIS1003 mentioned Arduino — a microcontroller with no operating system, where your code *is* the whole show. Much of the robotics world instead runs on **embedded Linux**, on boards like the Raspberry Pi — a middle ground between the Arduino and a desktop PC:

| | Arduino (bare metal) | Embedded Linux (Pi) | Desktop PC |
|---|---|---|---|
| Operating system | None | Linux | Windows/Linux/macOS |
| Threads & processes | No real OS threads | Full POSIX threads | Full |
| Networking | Add-on shields | Built-in | Built-in |
| Your program | The only thing running | One process among many | One process among many |

This year the project runs in a **3D simulator** on your PC rather than on real hardware, so you write and test everything on your desktop. The table still matters, though: a Pi runs the *same* C++ and standard library as your laptop, so the code you write for the simulated robot is the code a real one would run. The [Embedded Linux](embedded_linux.md) reference covers what deploying to actual hardware would involve — cross-compiling, hardware pins, running as a service — as background for when you leave the simulator.

---

## Mindset shift

The single biggest change from AIS1003 is this: **a program can be wrong only sometimes.** A single-threaded bug is reliably reproducible — same input, same crash. A concurrency or networking bug may appear one run in a thousand, on one machine but not another, only under load. Learning to reason about *what could happen* rather than *what did happen this time* is the real skill this course builds.

Up next: [Part 1](Chapter1/ownership.md) sharpens the C++ tools you will need, starting with ownership.

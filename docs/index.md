# AIS2203 E-book

Welcome to the e-book for the course [AIS2203 — Computer Engineering for Cyber-Physical Systems](https://www.ntnu.no/studier/emner/AIS2203#tab=omEmnet).

This book is about making programs do **more than one thing at once**, and making programs on **different machines talk to each other**. Those two ideas — concurrency and communication — are what turn a single-file exercise into a robot that reads its sensors, runs a control loop, streams telemetry to an operator, and reacts to commands: all at the same time, and all on time.

Use the sidebar to navigate, or jump straight to a topic with the search box above. New here? Start with [Getting Started](getting_started.md), then read [From AIS1003 to AIS2203](from_ais1003.md) to see what this course assumes you already know.

!!! note "This book builds on AIS1003"
    AIS2203 assumes you can already write, build and debug a modern C++ program with classes, `std::vector`, RAII and CMake — the content of the [AIS1003 e-book](https://markaren.github.io/E-book_cpp/). If any of that feels shaky, review it first; this book moves faster and leans on all of it.

!!! note "Norwegian translation"
    Part of this book is available in Norwegian (Bokmål) — use the language selector in the top bar. The translation is incomplete; pages not yet translated are shown in English.

## What you will learn

The book follows the arc of the course:

| Part | What it covers |
|------|----------------|
| **1. Modern C++ Toolkit** | The language features concurrency and communication lean on: ownership, move semantics, smart pointers, templates, lambdas, and `std::chrono`. |
| **2. Concurrency Fundamentals** | Processes and threads, race conditions, mutexes, condition variables and atomics — sharing data without corrupting it. |
| **3. Asynchronous & Real-Time** | Futures, thread pools, parallel algorithms, coroutines, and meeting deadlines in a real-time system. |
| **4. Data Communication** | Serialization, sockets, TCP/UDP, serial lines, Modbus, and higher-level patterns like MQTT and RPC. |
| **5. Embedded Linux** | Cross-compiling for a Raspberry Pi, driving GPIO and buses from Linux, and deploying your program as a service. |
| **6. Building Larger Projects** | Multi-target CMake, third-party dependencies with vcpkg, and bridging C++ with Python. |

Each chapter ends with **exercises**. As in AIS1003, the solutions are blurred — try each one honestly before revealing the answer. Type the code into CLion and run it; reading is not the same as knowing.

## How to use this book

Read a section, then write code. The examples are short on purpose; type them by hand, break them, and watch what the compiler and the runtime do. Concurrency in particular punishes copy-paste: a program that *looks* correct can still be wrong one time in a thousand, and the only way to build intuition is to run things yourself.

Where an example is a complete program you can run, a **▶ Run on Compiler Explorer** link appears beneath it. Use it to experiment without leaving the page.

When this book gives a recommendation, follow it unless you have a specific reason not to. C++ offers many ways to do everything, and the fastest way to drown is to try to learn all of them at once. Learn the one good way first.

## Additional resources

This book supports your learning; it is not a complete reference. Supplement it.

### Books

- **C++ Concurrency in Action, Second Edition** (Anthony Williams). The definitive book on the C++ thread library and memory model — the main companion for Part 2 and Part 3.
- **A Tour of C++** (Bjarne Stroustrup). A concise overview of modern C++ for people who already know the basics.
- **Effective Modern C++** (Scott Meyers). Move semantics, smart pointers and concurrency, explained item by item.

### Online

- [**cppreference**](https://en.cppreference.com/w/). Reliable, up-to-date documentation of the language and standard library — especially the [`<thread>`](https://en.cppreference.com/w/cpp/thread) section.
- [**Boost.Asio**](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html). The de facto C++ library for networking and asynchronous I/O.
- [**Stack Overflow**](https://stackoverflow.com/). Most programming questions have already been asked and answered.

## Quick links

- [Common terms](terms.md)
- [Frequently asked questions (FAQ)](faq.md)

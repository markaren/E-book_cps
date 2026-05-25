# Common Terms

A quick reference for the vocabulary used throughout this book. Each term links to the chapter that covers it in depth.

---

## Processes and threads

| Term | Meaning |
|------|---------|
| **Process** | A running program with its own isolated memory and resources. ([Processes & Threads](Chapter2/processes_threads.md)) |
| **Thread** | A schedulable sequence of execution *inside* a process, sharing the process's memory with sibling threads. ([Processes & Threads](Chapter2/processes_threads.md)) |
| **Concurrency** | A program structured as multiple tasks *in progress* at once (not necessarily simultaneous). ([Processes & Threads](Chapter2/processes_threads.md)) |
| **Parallelism** | Tasks executing *literally* at the same instant on multiple CPU cores. |
| **Context switch** | The OS pausing one thread to run another; cheaper between threads than between processes. |
| **Scheduler** | The OS component that decides which thread runs on which core, and when. |

## Synchronization and hazards

| Term | Meaning |
|------|---------|
| **Race condition** | A bug where the result depends on the timing/interleaving of threads. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Data race** | Two threads accessing the same memory with at least one writing, unsynchronised — **undefined behaviour**. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Critical section** | Code accessing a shared resource that must not run on two threads at once. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Mutex** | A lock giving mutual exclusion: one thread in the critical section at a time. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Lock guard / scoped lock** | RAII wrappers that lock a mutex on construction and unlock on destruction. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Deadlock** | Threads each holding a resource and waiting for another's, so none proceeds. ([Sharing Data](Chapter2/sharing_data.md)) |
| **Condition variable** | A primitive to *wait* until a condition becomes true, without busy-waiting. ([Condition Variables](Chapter2/condition_variables.md)) |
| **Spurious wakeup** | A `wait` returning without a notify — why you always re-check the predicate. ([Condition Variables](Chapter2/condition_variables.md)) |
| **Semaphore** | A counting permit controlling access by up to N threads (C++20). ([Condition Variables](Chapter2/condition_variables.md)) |
| **Atomic** | A variable whose operations are indivisible and data-race-free, without a lock. ([Atomics](Chapter2/atomics.md)) |
| **Busy-wait / spinlock** | Looping to check a condition instead of sleeping — wastes a CPU core. |

## Asynchronous and real-time

| Term | Meaning |
|------|---------|
| **Future / promise** | A handle to a result that will exist later; the promise is the writing end. ([Futures & Promises](Chapter3/futures.md)) |
| **Thread pool** | A fixed set of reusable worker threads fed tasks from a queue. ([Thread Pools](Chapter3/thread_pools.md)) |
| **Execution policy** | The `seq`/`par`/`par_unseq` argument that parallelises a standard algorithm. ([Parallel Algorithms](Chapter3/parallel_algorithms.md)) |
| **Coroutine** | A function that can suspend and resume; cooperative, lightweight concurrency. ([Coroutines](Chapter3/coroutines.md)) |
| **Real-time** | A system where a correct result delivered too late is wrong; hard / firm / soft. ([Real-Time & Timing](Chapter3/real_time.md)) |
| **Latency / jitter** | The delay of a response, and the *variation* in that delay. ([Real-Time & Timing](Chapter3/real_time.md)) |

## Modern C++

| Term | Meaning |
|------|---------|
| **RAII** | Tying a resource's lifetime to an object's, so the destructor always releases it. ([Ownership & RAII](Chapter1/ownership.md)) |
| **Ownership** | Who is responsible for releasing a resource: unique, shared, or borrowed. ([Ownership & RAII](Chapter1/ownership.md)) |
| **Move semantics** | Transferring a resource instead of copying it; leaves the source empty. ([Move Semantics](Chapter1/move_semantics.md)) |
| **Smart pointer** | A type expressing ownership: `unique_ptr` (one owner), `shared_ptr` (many). ([Smart Pointers](Chapter1/smart_pointers.md)) |
| **Template** | A compile-time blueprint generating a concrete type/function per type used. ([Templates](Chapter1/templates.md)) |
| **Lambda** | An inline callable with a capture list. ([Lambdas & std::function](Chapter1/lambdas.md)) |
| **`std::function`** | A wrapper holding any callable of a given signature. ([Lambdas & std::function](Chapter1/lambdas.md)) |
| **`std::chrono`** | The library of durations, time points and clocks. ([Time with std::chrono](Chapter1/chrono.md)) |

## Communication

| Term | Meaning |
|------|---------|
| **Serialization** | Converting an object to a portable byte sequence (and back). ([Serialization](Chapter4/serialization.md)) |
| **Endianness** | The byte order of multi-byte numbers; the wire uses big-endian. ([Serialization](Chapter4/serialization.md)) |
| **Socket** | An endpoint of a network link, addressed by IP + port. ([Sockets, TCP & UDP](Chapter4/sockets.md)) |
| **TCP / UDP** | Reliable ordered stream vs unreliable connectionless datagrams. ([Sockets, TCP & UDP](Chapter4/sockets.md)) |
| **Framing** | Marking message boundaries in a byte stream (length prefix or delimiter). ([Sockets, TCP & UDP](Chapter4/sockets.md)) |
| **Serial / UART** | Sending data one bit at a time over TX/RX; baud rate, start/stop/parity. ([Serial Communication](Chapter4/serial.md)) |
| **Modbus** | An industrial protocol over TCP or serial; 16-bit big-endian registers. ([Modbus](Chapter4/modbus.md)) |
| **MQTT** | Lightweight publish/subscribe messaging through a central broker. ([MQTT & RPC](Chapter4/mqtt_rpc.md)) |
| **RPC** | Calling a function on another machine as if it were local (gRPC, Thrift). ([MQTT & RPC](Chapter4/mqtt_rpc.md)) |

## Build and tooling

| Term | Meaning |
|------|---------|
| **Target** | A thing CMake builds — a library or an executable. ([CMake](Chapter6/cmake.md)) |
| **Visibility (PUBLIC/PRIVATE/INTERFACE)** | Whether a target's dependency propagates to things that link it. ([CMake](Chapter6/cmake.md)) |
| **Static / shared library** | Linked into the binary vs a separate `.dll`/`.so` found at run time. ([Dependencies](Chapter6/dependencies.md)) |
| **vcpkg** | A C++ package manager; declare dependencies in `vcpkg.json`. ([Dependencies](Chapter6/dependencies.md)) |
| **`extern "C"`** | Gives a C++ function C linkage (no name mangling) so other languages can call it. ([Calling C++ from Python](Chapter6/python_interop.md)) |
| **Cross-compiling** | Building on one architecture (host) a binary for another (target). ([Embedded Linux](embedded_linux.md)) |
| **ThreadSanitizer (TSan)** | A runtime tool that detects data races. ([Debugging Concurrent Programs](debugging_concurrency.md)) |
| **Undefined behaviour (UB)** | Code with no defined meaning; may "work" then break — not portable. ([Portability](portability.md)) |

## Computer vision

| Term | Meaning |
|------|---------|
| **OpenCV / `cv::Mat`** | The C++ computer-vision library; an image is a matrix of pixels (BGR). ([OpenCV in C++](Chapter7/opencv.md)) |
| **Calibration** | Recovering a camera's intrinsic (lens) and extrinsic (pose) parameters. ([Camera Calibration](Chapter7/calibration.md)) |
| **CNN / YOLO** | Convolutional network; YOLO detects objects in a single pass. ([Deep Vision](Chapter7/deep_vision.md)) |
| **ONNX** | A framework-agnostic model format for running inference in C++. ([Model Deployment & ONNX](Chapter7/onnx.md)) |

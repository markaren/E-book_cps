# Getting Started

AIS2203 uses the same core tools as AIS1003 — **CLion**, **CMake**, and a C++ compiler — so if you finished that course you are most of the way set up. This page covers what is new: the language standard, a few platform details that matter once you start threading, and an optional Linux environment for the Unix-flavoured examples.

If CLion, CMake or Git are not installed, follow the AIS1003 [Getting Started](https://markaren.github.io/E-book_cpp/getting_started/) guide first, then come back.

---

## The language standard: C++20

This book targets **C++20**. That is not a stylistic choice — several tools you will use this semester only exist from C++20 onward:

| Feature | Introduced | Used in |
|---------|------------|---------|
| `std::jthread`, stop tokens | C++20 | [Creating Threads](Chapter2/threads.md) |
| `std::counting_semaphore`, `std::binary_semaphore` | C++20 | [Condition Variables](Chapter2/condition_variables.md) |
| `std::latch`, `std::barrier` | C++20 | [Real-Time & Timing](Chapter3/real_time.md) |
| Coroutines (`co_await`, `co_return`) | C++20 | [Coroutines](Chapter3/coroutines.md) |

Set the standard in every project's `CMakeLists.txt`:

```cmake
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

Use a recent compiler. **GCC 11+**, **Clang 14+**, or **MSVC 19.3+ (Visual Studio 2022)** all have the C++20 thread features this book uses. Older compilers will fail to find `<jthread>` and the C++20 synchronisation types.

---

## Linking the thread library

On Windows with MSVC, threading works out of the box. On **Linux and the Raspberry Pi**, the thread library is separate and you must link it, or your program will compile and then fail at link time with errors like `undefined reference to pthread_create`.

The portable way to do this in CMake is `Threads`:

```cmake
find_package(Threads REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE Threads::Threads)
```

`Threads::Threads` expands to `-pthread` on Linux and to nothing on platforms that do not need it, so the same `CMakeLists.txt` works everywhere. Add it to every target that uses `<thread>`.

---

## A Linux environment

This year the project runs in a 3D simulator on your desktop, so you do **not** need a Raspberry Pi. A Linux environment is still handy, though — several communication examples are Unix-flavoured, and it makes the [Embedded Linux](embedded_linux.md) background concrete. Three options, in rough order of convenience:

| Option | Good for | Notes |
|--------|----------|-------|
| **WSL2** (Windows Subsystem for Linux) | Everyday Linux on a Windows laptop | `wsl --install` from an admin PowerShell, then install `build-essential cmake git`. |
| **A Raspberry Pi** | Optional real hardware | Not needed this year; only if you want to try the [Embedded Linux](embedded_linux.md) material for real. |
| **A Linux VM or dual boot** | A full desktop Linux | Heaviest to set up; rarely necessary for this course. |

You do not need all three — and you may need none. WSL2 is more than enough to follow every chapter, and on Windows you can build and run the course work directly; pick a Linux option only if you want a Unix shell handy.

!!! tip "CLion talks to all of these"
    CLion can build and debug on a remote machine over SSH — WSL2 or a Pi — through **Settings → Build, Execution, Deployment → Toolchains**, so you edit on your laptop while the compiler runs on Linux. The [Embedded Linux](embedded_linux.md) reference sketches the rest (cross-compiling, deploying as a service) for when real hardware is in play.

---

## Third-party libraries

C++ has no networking in its standard library (see [Networking in C++](Chapter4/networking.md)), so Part 4 uses external libraries such as **Boost.Asio**. This course manages dependencies with **vcpkg**, covered in [Dependencies with vcpkg](Chapter6/dependencies.md). You do not need it yet — install it when a chapter first asks for a library.

---

## Sanity check

Build and run this program. It starts a second thread, so it exercises both the compiler's C++20 support and your thread-library linking:

```cpp
#include <iostream>
#include <thread>

int main() {
    std::jthread worker([] {
        std::cout << "Hello from the worker thread\n";
    });
    std::cout << "Hello from main\n";
    // worker joins automatically when it goes out of scope (it is a jthread)
}
```

If this compiles, links and runs — printing two lines in either order — your setup is ready. If it fails to *link* on Linux, you forgot `Threads::Threads`. If `std::jthread` is *not found*, your compiler is too old.

Up next: [From AIS1003 to AIS2203](from_ais1003.md) maps what carries over from last year and what is genuinely new.

# Getting Started

AIS2203 uses the same core tools as AIS1003 — **CLion**, **CMake**, and a C++ compiler — so if you finished that course you are most of the way set up. This page covers what is new: the language standard, a few platform details that matter once you start threading, and the Linux environment you will need for [Part 5](Chapter5/intro.md).

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

Part 5 of this book is about **embedded Linux**, and several communication examples assume a Unix system. You have three good options, in rough order of convenience:

| Option | Good for | Notes |
|--------|----------|-------|
| **WSL2** (Windows Subsystem for Linux) | Everyday Linux on a Windows laptop | `wsl --install` from an admin PowerShell, then install `build-essential cmake git`. |
| **A Raspberry Pi** | The real embedded target | What [Part 5](Chapter5/intro.md) deploys to. A Pi 4 or Pi Zero 2 W is plenty. |
| **A Linux VM or dual boot** | A full desktop Linux | Heaviest to set up; rarely necessary for this course. |

You do not need all three. WSL2 is enough to follow every chapter except the parts that touch real GPIO pins; for those you want a physical Pi.

!!! tip "CLion talks to all of these"
    CLion can build and debug on a remote machine over SSH, including WSL2 and a Raspberry Pi, through **Settings → Build, Execution, Deployment → Toolchains**. You edit on your laptop and the compiler runs on Linux. [Cross-Compiling for the Pi](Chapter5/cross_compiling.md) covers the faster alternative — building ARM binaries on your PC.

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

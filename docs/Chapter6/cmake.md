# CMake for Multi-Target Projects

In AIS1003 a `CMakeLists.txt` was usually one `add_executable` line. Your AIS2203 project is bigger: a core of [concurrency](../Chapter2/processes_threads.md) and [control](../Chapter3/real_time.md) logic, a [communication](../Chapter4/serialization.md) layer, maybe a [vision](../Chapter7/onnx.md) pipeline, third-party libraries, and tests — all of which want to be built and reused cleanly. This chapter takes CMake from "compile one file" to "organise a real project" around its central idea: **targets**.

It assumes the CMake basics from the [AIS1003 CMake chapter](https://markaren.github.io/E-book_cpp/Chapter2/cmake_intro/) (configure/build, `add_executable`, `CMAKE_CXX_STANDARD`).

---

## Think in targets

A **target** is a thing CMake builds: an executable (`add_executable`) or a library (`add_library`). The shift from beginner to competent CMake is to stop thinking about *global* compiler flags and start attaching everything — sources, include directories, language features, and dependencies — to the **target that needs it**. Each target carries its own properties, and those properties compose when one target links another.

Here is the smallest multi-target project: a reusable library plus an app that uses it.

```cmake
cmake_minimum_required(VERSION 3.16)
project(robot CXX)

add_library(robot_core
    src/sensor.cpp
    src/controller.cpp)
target_include_directories(robot_core PUBLIC include)
target_compile_features(robot_core PUBLIC cxx_std_20)

add_executable(robot_app src/main.cpp)
target_link_libraries(robot_app PRIVATE robot_core)
```

`robot_core` is a library; `robot_app` is a program that **links** it. Because `robot_app` links `robot_core`, it automatically inherits whatever `robot_core` exposes — its public headers (`include/`) and its C++20 requirement. You did not repeat the include path or the standard for the app; linking the library propagated them. That propagation is the whole point of modern CMake, and it is governed by one keyword you must understand: visibility.

---

## Visibility: PUBLIC, PRIVATE, INTERFACE

When you give a target an include directory or a dependency, you say whether it is part of that target's **interface** (visible to things that link it) or just its **implementation** (private to it). Three keywords:

| Keyword | Used to build this target? | Propagated to things that link it? | Means |
|---------|:--:|:--:|-------|
| `PRIVATE` | yes | no | An implementation detail of this target |
| `PUBLIC` | yes | yes | Part of this target's interface *and* used internally |
| `INTERFACE` | no | yes | Part of the interface only (e.g. a header-only library) |

The test is simple: **does using this target force consumers to need the thing too?**

- If `robot_core`'s **public headers** `#include <nlohmann/json.hpp>`, then anyone using `robot_core` also needs that header → link it **PUBLIC**.
- If `robot_core` uses a library only *inside* its `.cpp` files, consumers never see it → link it **PRIVATE**.

```cmake
target_link_libraries(robot_core
    PUBLIC  nlohmann_json::nlohmann_json   # appears in robot_core's headers → consumers need it
    PRIVATE fmt::fmt)                       # used only in robot_core.cpp → consumers don't
```

Getting this right keeps your build honest: a `PRIVATE` dependency stays an implementation detail you can swap later without touching anyone who links you, while a `PUBLIC` one correctly forces consumers to see it. Over-using `PUBLIC` leaks implementation details and slows builds; the habit is **PRIVATE by default, PUBLIC only when the header demands it.**

!!! tip "Per-target standard, not global"
    `target_compile_features(robot_core PUBLIC cxx_std_20)` requires C++20 *for that target and everything that links it* — cleaner than the global `set(CMAKE_CXX_STANDARD 20)`, because the requirement travels with the library. The thread library is the same idea: `find_package(Threads REQUIRED)` then `target_link_libraries(robot_core PUBLIC Threads::Threads)` ([Getting Started](../getting_started.md)).

---

## Adding a test target

Tests are just another executable that links your library — which is exactly why you split the logic into a library in the first place: the app *and* the tests can both link `robot_core` and exercise the same code. Pull in a test framework with **`FetchContent`**, which downloads and builds a dependency at configure time:

```cmake
include(FetchContent)
FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG        v3.5.2)
FetchContent_MakeAvailable(Catch2)

add_executable(tests tests/test_controller.cpp)
target_link_libraries(tests PRIVATE robot_core Catch2::Catch2WithMain)

enable_testing()
add_test(NAME unit COMMAND tests)        # so `ctest` runs it
```

Now `robot_core`, `robot_app`, and `tests` are three targets in one tree: the library holds the logic, the app runs it, the tests check it. `FetchContent` is convenient for a single test dependency; for application libraries you will usually prefer [vcpkg](dependencies.md), the next chapter.

---

## A sensible layout

CMake does not enforce a directory structure, but a conventional one keeps a growing project navigable:

```
robot/
├── CMakeLists.txt
├── include/robot/        # PUBLIC headers — the library's interface
│   ├── sensor.hpp
│   └── controller.hpp
├── src/                  # implementation (.cpp) + private headers
│   ├── sensor.cpp
│   ├── controller.cpp
│   └── main.cpp
└── tests/
    └── test_controller.cpp
```

The split mirrors the visibility idea physically: `include/` holds the **public** interface (the directory you mark `PUBLIC`), while `src/` holds the implementation. Consumers see only `include/`.

---

## Build options

Let users turn parts of the build on or off with `option`, a cached boolean they can flip from the command line:

```cmake
option(ROBOT_BUILD_TESTS "Build the unit tests" ON)

if(ROBOT_BUILD_TESTS)
    # ... the test target from above ...
endif()
```

```bash
cmake -S . -B build -DROBOT_BUILD_TESTS=OFF      # configure without tests
```

This is how a library lets a consumer skip building its tests, or how you gate an optional feature (say, the vision pipeline) behind a flag.

---

## Summary

- Modern CMake is built on **targets** (`add_library`, `add_executable`); attach sources, includes, features, and dependencies to the **target that needs them**, not to global flags. Linking a target **propagates** its interface.
- **Visibility** controls propagation: **`PRIVATE`** (implementation detail), **`PUBLIC`** (interface + used internally), **`INTERFACE`** (interface only). Rule of thumb: **PRIVATE by default; PUBLIC only when your public headers expose the dependency.**
- Prefer **per-target** requirements — `target_compile_features(... cxx_std_20)`, `target_link_libraries(... Threads::Threads)` — over global settings.
- Split logic into a **library** so the **app** and the **tests** can both link it; pull a test framework in with **`FetchContent`** and register it with `add_test`/`ctest`.
- Keep a conventional **layout** (`include/` public headers, `src/` implementation, `tests/`) and gate optional parts behind **`option`** flags.
- Next: [Dependencies with vcpkg](dependencies.md), for the third-party libraries this book relies on.

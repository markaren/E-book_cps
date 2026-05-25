# Chapter 5 Exercises

Work through these after reading Chapter 5 (Building Larger Projects). Unlike the earlier chapters, these are about **project structure and tooling**, so the "solutions" are `CMakeLists.txt` files, manifests, and small multi-file setups rather than single programs — build them in CLion and confirm they configure and run.

When you open a solution it appears **blurred** — click it once more to reveal it.

---

## 1. Split into library, app, and test

*Practises: [CMake for Multi-Target Projects](cmake.md)*

Take a one-file program and restructure it into **three targets**: a `stats` *library* holding a `mean(const std::vector<double>&)` function, a `stats_app` *executable* that calls it, and a `tests` *executable* (Catch2 via `FetchContent`) that checks it. Use the conventional `include/` + `src/` + `tests/` layout.

> Hint: `add_library(stats src/stats.cpp)` with `target_include_directories(stats PUBLIC include)`; both the app and the tests `target_link_libraries(... PRIVATE stats)`. Pull Catch2 with `FetchContent`.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    Layout:

    ```
    stats/
    ├── CMakeLists.txt
    ├── include/stats.hpp     // declaration:  double mean(const std::vector<double>&);
    ├── src/stats.cpp         // definition
    ├── src/main.cpp          // calls mean(), prints the result
    └── tests/test_stats.cpp  // Catch2 TEST_CASE checking mean()
    ```

    `CMakeLists.txt`:

    ```cmake
    cmake_minimum_required(VERSION 3.16)
    project(stats CXX)

    # --- the library: the logic, reusable by both app and tests ---
    add_library(stats src/stats.cpp)
    target_include_directories(stats PUBLIC include)
    target_compile_features(stats PUBLIC cxx_std_20)

    # --- the application ---
    add_executable(stats_app src/main.cpp)
    target_link_libraries(stats_app PRIVATE stats)

    # --- the tests ---
    include(FetchContent)
    FetchContent_Declare(
        Catch2
        GIT_REPOSITORY https://github.com/catchorg/Catch2.git
        GIT_TAG        v3.5.2)
    FetchContent_MakeAvailable(Catch2)

    add_executable(tests tests/test_stats.cpp)
    target_link_libraries(tests PRIVATE stats Catch2::Catch2WithMain)

    enable_testing()
    add_test(NAME unit COMMAND tests)
    ```

    The point is that the logic lives in **one** target, `stats`, which both `stats_app` and `tests` link — so the tests exercise exactly the code the app runs, with no duplication. `include/` is marked **`PUBLIC`** on the library, so linking `stats` automatically puts its headers on the app's and tests' include paths. Run the app from the dropdown; run the tests with `ctest` (or directly). This is the same library-plus-app-plus-tests shape every non-trivial project converges on.

    </div>

---

## 2. Pull in a dependency with vcpkg

*Practises: [Dependencies with vcpkg](dependencies.md)*

Add **nlohmann/json** to a project through vcpkg manifest mode and use it to print a small JSON object. Write the `vcpkg.json`, wire it into CMake, and configure with the vcpkg toolchain file.

> Hint: declare `nlohmann-json` (vcpkg package name) in `vcpkg.json`; in CMake use `find_package(nlohmann_json CONFIG REQUIRED)` (the CMake target name) and link `nlohmann_json::nlohmann_json`. Configure with `-DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake`.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    `vcpkg.json`:

    ```json
    {
      "name": "json-demo",
      "version": "0.1.0",
      "dependencies": [ "nlohmann-json" ]
    }
    ```

    `CMakeLists.txt`:

    ```cmake
    cmake_minimum_required(VERSION 3.16)
    project(json_demo CXX)

    find_package(nlohmann_json CONFIG REQUIRED)

    add_executable(json_demo main.cpp)
    target_compile_features(json_demo PRIVATE cxx_std_20)
    target_link_libraries(json_demo PRIVATE nlohmann_json::nlohmann_json)
    ```

    `main.cpp`:

    ```cpp
    #include <iostream>
    #include <nlohmann/json.hpp>

    int main() {
        nlohmann::json j;
        j["sensor"] = "boiler";
        j["value"]  = 42.5;
        std::cout << j.dump() << "\n";   // {"sensor":"boiler","value":42.5}
    }
    ```

    Configure (vcpkg reads the manifest and builds the dependency on first run):

    ```bash
    cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=<path-to-vcpkg>/scripts/buildsystems/vcpkg.cmake
    cmake --build build
    ```

    Note the two different names: **`nlohmann-json`** is the vcpkg *package*, **`nlohmann_json::nlohmann_json`** is the CMake *target*. The link is **`PRIVATE`** because `nlohmann/json.hpp` is used only in `main.cpp` — if instead it appeared in a library's public header, it would need to be `PUBLIC` (exercise 4). In CLion, set the toolchain file once under **Settings → Build → CMake** and you never type it again.

    </div>

---

## 3. Call a C++ function from Python

*Practises: [Calling C++ from Python](python_interop.md)*

Expose a C++ function to Python. Write a `square(int)` behind a C interface, build it as a **shared** library, and call it from Python with `ctypes`.

> Hint: wrap the declaration in `extern "C"` so it is not name-mangled; build with `add_library(... SHARED ...)` and `set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)`; in Python, `ctypes.CDLL` the library and set `argtypes`/`restype` before calling.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    `mathlib.hpp` — the C interface:

    ```cpp
    extern "C" {
        int square(int x);
    }
    ```

    `mathlib.cpp` — ordinary C++ behind it:

    ```cpp
    #include "mathlib.hpp"
    int square(int x) { return x * x; }
    ```

    `CMakeLists.txt`:

    ```cmake
    cmake_minimum_required(VERSION 3.16)
    project(mathlib CXX)

    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)   # export symbols on Windows
    add_library(mathlib SHARED mathlib.cpp)
    ```

    `use.py`:

    ```python
    import ctypes

    lib = ctypes.CDLL("./libmathlib.so")   # "mathlib.dll" on Windows
    lib.square.argtypes = [ctypes.c_int]
    lib.square.restype  = ctypes.c_int

    print(lib.square(7))                    # 49
    ```

    `extern "C"` gives `square` a plain, unmangled symbol name that `ctypes` can find — without it, the C++ compiler would emit something like `_Z6squarei` and the lookup would fail. The library must be **`SHARED`** (Python loads a binary at run time, not a header), and the `argtypes`/`restype` declarations are essential: omit them and `ctypes` assumes `int` and would mishandle anything else. For a real API you would wrap this in a Python class so callers never touch `ctypes` — or reach for **pybind11** to skip the C interface entirely.

    </div>

---

## 4. Public or private?

*Practises: [CMake for Multi-Target Projects](cmake.md)*

Your `robot_core` library depends on two things: **spdlog**, used only *inside* its `.cpp` files for logging, and **Eigen**, whose matrix types appear *in `robot_core`'s public headers*. How should each be linked — `PUBLIC`, `PRIVATE`, or `INTERFACE` — and why does it matter?

> Hint: the test is "does someone who links `robot_core` also need this library?" If it only appears in `.cpp` files, the answer is no; if it appears in a header consumers `#include`, the answer is yes.

??? success "Show discussion"

    <div class="spoiler" markdown title="Click to reveal">

    ```cmake
    target_link_libraries(robot_core
        PUBLIC  Eigen3::Eigen     # appears in robot_core's public headers
        PRIVATE spdlog::spdlog)   # used only inside robot_core's .cpp files
    ```

    **Eigen is `PUBLIC`** because `robot_core`'s headers expose Eigen types — so any code that `#include`s those headers must also see Eigen's headers. Marking it `PUBLIC` propagates Eigen's include paths to every consumer automatically; mark it `PRIVATE` and consumers would get *"`Eigen/Dense` not found"* the moment they used `robot_core`.

    **spdlog is `PRIVATE`** because it appears only in the implementation — no consumer ever sees it. Keeping it private means it stays a swappable implementation detail (you could replace spdlog with another logger without touching anyone who links `robot_core`), and consumers are not forced to compile against a library they do not use.

    Why it matters: visibility is how a library states its **true interface**. Over-marking things `PUBLIC` leaks implementation details, forces unnecessary recompilation on consumers, and makes the library harder to change; the discipline is **`PRIVATE` by default, `PUBLIC` only when a public header genuinely exposes the dependency**. (`INTERFACE` is the third case — a dependency the target does not use itself but its consumers must, typical of a header-only library that has no `.cpp` of its own.)

    </div>

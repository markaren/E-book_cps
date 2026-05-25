# Dependencies with vcpkg

Your project needs libraries the standard library does not provide: [Boost.Asio](../Chapter4/networking.md) for networking, [nlohmann/json](../Chapter4/serialization.md) for serialization, [OpenCV](../Chapter7/opencv.md) for vision, [TBB](../Chapter3/parallel_algorithms.md) for parallel algorithms, Catch2 for tests. In a language like Python (`pip`) or Rust (`cargo`), pulling these in is one command. In C++ it has historically been a genuine chore — this chapter is about the tool that fixes that for this course: **vcpkg**.

---

## Why C++ dependencies are hard

C++ has no built-in package manager, and for decades adding a library meant doing by hand what `pip` does for you:

1. Download the library's source.
2. Build it — with the *right* compiler, flags, and options to match your project.
3. Find the resulting headers and compiled binaries.
4. Tell your build where they are (include paths, link paths).
5. Repeat for every **transitive** dependency the library itself needs.
6. Do it again on every platform and every teammate's machine.

Miss a flag or a version and you get cryptic link errors. This friction is why "just use a library" was never as easy in C++ as elsewhere — and why a package manager is worth the small setup.

---

## What you are actually consuming

Before the tool, know the three forms a C++ library comes in, because they behave differently:

| Form | What you get | At link time | At run time |
|------|--------------|--------------|-------------|
| **Header-only** | Just headers (`.hpp`) | Compiled into your code | Nothing extra — it's in your binary |
| **Static** (`.lib`, `.a`) | Headers + a static archive | Copied into your binary | Nothing extra — self-contained |
| **Shared** (`.dll`, `.so`, `.dylib`) | Headers + a shared binary | A reference is recorded | The library file must be **found at run time** |

- **Header-only** (e.g. nlohmann/json) is the easiest to use — `#include` and go — at the cost of longer compile times, since the whole library recompiles into every file that uses it.
- **Static** linking folds the library into your executable: bigger binary, but it runs anywhere with no extra files.
- **Shared** linking keeps the library separate: smaller binary, shared between programs, but the `.dll`/`.so` must be present and findable when the program runs (the classic "DLL not found" error). Shared libraries can themselves depend on other shared libraries.

A package manager hides most of this, but it explains *why* a program builds fine and then fails to start (a missing shared library) — and why static linking is often simpler for deployment.

---

## vcpkg in manifest mode

**vcpkg** is a C++ package manager (from Microsoft) that builds libraries from source for your exact toolchain and hands them to CMake. This course uses its **manifest mode**, where a `vcpkg.json` file in your project declares its dependencies — checked in, so every teammate gets the same versions.

A manifest is small:

```json
{
  "name": "robot",
  "version": "0.1.0",
  "dependencies": [
    "nlohmann-json",
    "boost-asio",
    "tbb",
    "catch2"
  ]
}
```

The one-time link to CMake is a **toolchain file**: point CMake at vcpkg's, and from then on the declared libraries are simply *there* for `find_package`.

```bash
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=<path-to-vcpkg>/scripts/buildsystems/vcpkg.cmake
```

On that first configure, vcpkg reads `vcpkg.json`, downloads and builds the dependencies (and their transitive dependencies) for your platform, and makes them findable. Your `CMakeLists.txt` then uses them with the `find_package` / `target_link_libraries` pattern from the [previous chapter](cmake.md):

```cmake
find_package(nlohmann_json CONFIG REQUIRED)
find_package(Boost REQUIRED COMPONENTS system)

target_link_libraries(robot_core
    PUBLIC  nlohmann_json::nlohmann_json   # used in public headers
    PRIVATE Boost::system)                 # used only internally
```

That is the whole workflow: **declare in `vcpkg.json`, configure with the toolchain file, `find_package` + link.** No manual downloads, no hunting for binaries, and the same result on every machine.

!!! tip "Find the exact package and target names"
    Two names matter and are not always identical: the **vcpkg package** name (in `vcpkg.json`, e.g. `nlohmann-json`) and the **CMake target** name (`nlohmann_json::nlohmann_json`). `vcpkg install <pkg>` prints the exact `find_package`/`target_link_libraries` lines to copy — the fastest way to get them right.

!!! note "Targeting other platforms"
    vcpkg builds for a **triplet** (architecture + OS + linkage, e.g. `x64-windows`, `x64-linux`). It can build for other targets too — relevant if you ever deploy to a Raspberry Pi ([Embedded Linux](../embedded_linux.md)) — but since this year's project runs in the [simulator](../Chapter7/virtual_environments.md) on your own machine, the default host triplet is all you need.

---

## The alternatives

vcpkg is not the only option, and you will meet the others:

| Tool | What it is | When |
|------|-----------|------|
| **vcpkg** | Package manager, manifest + CMake toolchain | This course's default |
| **Conan** | The other major C++ package manager | Common in industry; an alternative to vcpkg |
| **FetchContent** | Built into CMake; downloads + builds source at configure time | A handful of dependencies, no extra tooling ([previous chapter](cmake.md)) |
| **Git submodules** | Vendor a dependency's repo inside yours | Header-only libs, or pinning an exact source |
| **Vendoring** | Copy the source straight into your tree | Tiny, rarely-changing dependencies |

For one or two dependencies, [`FetchContent`](cmake.md) is often enough and needs no separate tool. For a project pulling in OpenCV, Boost, and friends, a real package manager like **vcpkg** is far less painful. Mixing vcpkg and Conan in one project is a headache — pick one.

---

## Summary

- C++ has **no built-in package manager**, so adding libraries by hand (download → build with the right flags → locate → link → repeat for transitive deps → per platform) is the historical pain a tool removes.
- Libraries come **header-only** (easy, slower compiles), **static** (folded into your binary, self-contained), or **shared** (separate `.dll`/`.so` that must be **found at run time**).
- **vcpkg manifest mode**: declare dependencies in **`vcpkg.json`**, configure CMake with vcpkg's **toolchain file**, then use them via **`find_package` + `target_link_libraries`** — reproducible across machines. `vcpkg install <pkg>` prints the exact CMake names.
- Alternatives: **Conan** (the other package manager), **`FetchContent`** (built-in, good for a few deps), **submodules**/**vendoring** (pin or copy source). Use `FetchContent` for one or two deps, **vcpkg** for many; don't mix vcpkg and Conan.
- Next: [Calling C++ from Python](python_interop.md), bridging your fast C++ to the Python world.

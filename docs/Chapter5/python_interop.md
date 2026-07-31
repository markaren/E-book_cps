# Calling C++ from Python

A recurring theme of this book — most explicitly in [Computer Vision](../Chapter6/opencv.md) — is "**prototype in Python, deploy in C++**." Sometimes you want both at once: a Python program that is easy to write but calls into **fast C++** for the heavy lifting, or an existing C++ library you want to drive from a Python script. This chapter shows how to bridge the two languages — and why the bridge is shaped the way it is.

---

## Why bridge at all

Python and C++ have complementary strengths, and combining them gives you the best of each:

- **Performance.** For compute-heavy work, C++ is roughly 10–100× faster than pure Python and uses less memory. Move the hot loop to C++ and keep the rest in Python.
- **Existing libraries.** A great deal of fast, mature code exists only in C/C++; calling it from Python beats reimplementing it.
- **Low-level access.** C++ reaches hardware, memory, and OS facilities that are awkward from Python.
- **Reuse.** You can give an existing C++ codebase a friendly Python front-end without rewriting it.

The pattern is to write the **convenient** parts in Python and the **fast or low-level** parts in C++ — which is exactly what libraries like NumPy and OpenCV already do internally.

---

## The catch: you can call C, not C++

Here is the fact that shapes everything: **you cannot call C++ functions directly from Python** (or from most other languages). You can call **C** functions. The reason is **name mangling**.

To support overloading (several functions with the same name but different parameters), a C++ compiler encodes each function's name *together with its parameter types* into a single, compiler-specific symbol — `add(int, int)` might become `_Z3addii`. Those mangled names are unpredictable and differ between compilers, so nothing outside C++ can reliably find them. C, which has no overloading, uses plain, stable symbol names — `add` stays `add`.

The fix is to expose a **C interface** to your C++ code using `extern "C"`, which tells the C++ compiler to give those functions **C linkage** — plain, unmangled names:

```cpp
// library.hpp — a strictly-C interface to C++ code
extern "C" {
    int add(int a, int b);
    double mean(const double* data, int n);
}
```

```cpp
// library.cpp — the implementation is ordinary C++
#include "library.hpp"

int add(int a, int b) {
    return a + b;
}

double mean(const double* data, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; ++i) {
        sum += data[i];
    }
    return n > 0 ? sum / n : 0.0;
}
```

Inside the `.cpp` you can use *all* of C++ — classes, the STL, threads — but the functions you **expose** must use C-compatible types (numbers, pointers, not `std::string` or `std::vector` directly) and be wrapped in `extern "C"`.

---

## ctypes: load a shared library from Python

Python's built-in **`ctypes`** module loads a shared library and calls its C functions. The steps:

**1. Build the C++ as a *shared* library** (`.so`/`.dll`) — it cannot be header-only, because Python loads a binary at run time. On Windows you must export the symbols:

```cmake
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)   # export symbols on Windows

add_library(mylib SHARED src/library.cpp)
target_include_directories(mylib PUBLIC include)
```

**2. Load it and declare the signatures** in Python. `ctypes` does not know the argument or return types, so you tell it — otherwise it assumes `int` and silently corrupts anything else:

```python
import ctypes

lib = ctypes.CDLL("mylib.dll")            # "libmylib.so" on Linux/macOS

lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype  = ctypes.c_int
print(lib.add(3, 4))                       # 7

data = (ctypes.c_double * 4)(1.0, 2.0, 3.0, 4.0)
lib.mean.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int]
lib.mean.restype  = ctypes.c_double
print(lib.mean(data, len(data)))           # 2.5
```

!!! warning "Finding the DLL on Windows"
    Three Windows details bite here, and CLion's defaults make all three likely:

    - **No `lib` prefix.** CMake names the file `mylib.dll` on Windows — not `libmylib.so` as on Linux. Load it under the name it actually has.
    - **It lands in the build folder.** CLion writes the DLL to `cmake-build-debug/` (or `cmake-build-release/`), *not* next to your Python script. Either pass the **absolute path** to `CDLL`, or run Python from that folder, or copy the DLL over — `CDLL("mylib.dll")` only works when the file is on the DLL search path.
    - **Python 3.8+ changed the DLL search.** Since Python 3.8, Windows no longer searches `PATH` for a DLL's **own dependent DLLs**. If `mylib.dll` itself depends on other DLLs (a vcpkg-built library, say), add their folder with `os.add_dll_directory(r"C:\path\to\dlls")` before loading, or the load fails with a cryptic "DLL load failed" even though the file exists.

**3. Hide the C interface behind a Pythonic wrapper** so callers never see `ctypes`. You would normally write a small Python class that loads the library once and presents clean methods:

```python
class FastMath:
    def __init__(self, path="mylib.dll"):      # "libmylib.so" on Linux/macOS
        self._lib = ctypes.CDLL(path)
        self._lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
        self._lib.add.restype  = ctypes.c_int

    def add(self, a, b):
        return self._lib.add(a, b)
```

`ctypes` is transparent — there is no magic, just a library and some type declarations — which makes it easy to understand and debug. The cost is the manual `argtypes`/`restype` boilerplate and the restriction to C-compatible types at the boundary.

---

## pybind11: expose C++ directly

Writing C wrappers and `ctypes` declarations by hand gets tedious for a real API. **pybind11** is a header-only library that exposes C++ functions *and classes* to Python as a proper Python module, handling the type conversions (including `std::string`, `std::vector`, and more) automatically:

```cpp
#include <pybind11/pybind11.h>

int add(int a, int b) { return a + b; }

PYBIND11_MODULE(mymodule, m) {
    m.def("add", &add, "Add two integers");
}
```

```python
import mymodule
print(mymodule.add(3, 4))      # 7 — just a normal Python module
```

No `extern "C"`, no manual signatures, and you can bind whole C++ classes so they look like native Python objects. Get it via [vcpkg](dependencies.md) (add `pybind11` to `vcpkg.json`). The build wiring is a `pybind11_add_module` target rather than a plain `add_library`:

```cmake
find_package(pybind11 CONFIG REQUIRED)

pybind11_add_module(mymodule mymodule.cpp)   # builds mymodule.pyd on Windows
```

`pybind11_add_module` produces an **importable Python extension** — `mymodule.pyd` on Windows (`.so` on Linux), not a plain `.dll` — which Python imports directly once it is on `sys.path`.

!!! warning "The module is bound to one specific Python"
    A pybind11 extension is compiled **against a particular Python version and installation**, and it will only import into *that* interpreter. This is the common trap on Windows: if you build against **vcpkg's** Python but run your script under a **conda** Python (or vice-versa), the import fails with a version/ABI mismatch. Make sure the Python that CMake finds at build time is the **same** interpreter you run the module with. `ctypes`, by contrast, is not tied to a build-time Python at all — which is one reason it is the path the [exercises](exercises.md) assess.

The trade-off is exactly the usual one: **pybind11** does more for you but hides more machinery, while **ctypes** is more verbose but fully transparent. For a small, stable C interface, `ctypes` keeps everything in plain sight; for a rich C++ API you want to expose comfortably, pybind11 is far less work.

| | `ctypes` | pybind11 |
|---|----------|----------|
| Extra library | No (built into Python) | Yes (header-only, via vcpkg) |
| Boilerplate | Manual `extern "C"` + signatures | Minimal |
| C++ classes / STL types | Awkward (C interface only) | Native support |
| Transparency | Full — no magic | More hidden machinery |
| Best for | A small C interface | A rich C++ API as a Python module |

---

## Where this fits

For AIS2203 the bridge shows up wherever the two worlds meet: a [vision](../Chapter6/deep_vision.md) model explored in Python with a [C++ inference core](../Chapter6/onnx.md) for speed, or a Python script driving a C++ control library. The guiding idea is the one from the CV part — **keep the fast, real-time, low-level work in C++ and the glue in Python** — and either bridge lets you draw that line wherever it makes sense.

---

## Summary

- Bridging Python and C++ gives you Python's convenience with C++'s **speed, existing libraries, and low-level access** — the "fast core, easy glue" split.
- You **cannot call C++ directly** because of **name mangling**; expose a **C interface** with `extern "C"` (plain, stable symbol names). The implementation is still full C++, but the *exposed* functions use C-compatible types.
- **`ctypes`** (built into Python) loads a **shared** library (export symbols on Windows) and calls it once you declare `argtypes`/`restype`; wrap it in a Pythonic class. Transparent, but manual.
- **pybind11** exposes C++ functions *and classes* as a real Python module with minimal boilerplate and automatic type conversion — more convenient, more hidden machinery. Get it via [vcpkg](dependencies.md).
- Choose **`ctypes`** for a small, transparent C interface; **pybind11** for a rich C++ API. Either way, keep the **fast/low-level work in C++ and the glue in Python** — the [prototype-in-Python, deploy-in-C++](../Chapter6/opencv.md) idea made concrete.

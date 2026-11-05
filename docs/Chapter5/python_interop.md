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

---

## A worked example: `Vector3` and `Matrix4`

A single `add` function does not show what pybind11 is really for: **classes**. Here is a small 3D math library — a `Vector3` and a 4×4 homogeneous transform `Matrix4`, the two types every robotics or graphics project ends up with — exposed to Python so they behave like native Python objects. The project is three source files, and it follows one rule worth copying: **the library knows nothing about Python**. Only the bindings file includes pybind11.

```
vecmath/
├── CMakeLists.txt
├── vcpkg.json
├── include/vecmath.hpp     # plain C++ — no Python in sight
├── src/bindings.cpp        # the ONLY file that includes pybind11
└── use_vecmath.py
```

### The C++ library

`vecmath.hpp` is ordinary C++ — the same header you would `#include` from a C++ `main`. `Vector3` is a plain struct with public fields and a few methods; `Matrix4` stores 16 doubles row-major and offers factory functions for a translation and a rotation about *z*, matrix multiplication, and a point transform:

```cpp
// include/vecmath.hpp
#pragma once

#include <array>
#include <cmath>

struct Vector3 {
    double x{0.0}, y{0.0}, z{0.0};

    Vector3() = default;
    Vector3(double x, double y, double z) : x(x), y(y), z(z) {}

    double length() const { return std::sqrt(x * x + y * y + z * z); }

    Vector3 normalized() const {
        const double len = length();
        return len > 0.0 ? Vector3{x / len, y / len, z / len} : *this;
    }

    double dot(const Vector3& o) const { return x * o.x + y * o.y + z * o.z; }

    Vector3 cross(const Vector3& o) const {
        return {y * o.z - z * o.y, z * o.x - x * o.z, x * o.y - y * o.x};
    }

    Vector3 operator+(const Vector3& o) const { return {x + o.x, y + o.y, z + o.z}; }
    Vector3 operator-(const Vector3& o) const { return {x - o.x, y - o.y, z - o.z}; }
    Vector3 operator*(double s) const { return {x * s, y * s, z * s}; }
};

// 4x4 homogeneous transform, stored row-major: elements_[row * 4 + col]
class Matrix4 {
public:
    Matrix4() { setIdentity(); }

    void setIdentity() {
        elements_.fill(0.0);
        for (int i = 0; i < 4; ++i) (*this)(i, i) = 1.0;
    }

    double& operator()(int row, int col) { return elements_[row * 4 + col]; }
    double operator()(int row, int col) const { return elements_[row * 4 + col]; }

    static Matrix4 translation(double x, double y, double z) {
        Matrix4 m;
        m(0, 3) = x;
        m(1, 3) = y;
        m(2, 3) = z;
        return m;
    }

    static Matrix4 rotationZ(double angle) {   // radians
        Matrix4 m;
        const double c = std::cos(angle), s = std::sin(angle);
        m(0, 0) = c; m(0, 1) = -s;
        m(1, 0) = s; m(1, 1) = c;
        return m;
    }

    Matrix4 operator*(const Matrix4& o) const {
        Matrix4 r;
        for (int i = 0; i < 4; ++i) {
            for (int j = 0; j < 4; ++j) {
                double sum = 0.0;
                for (int k = 0; k < 4; ++k) sum += (*this)(i, k) * o(k, j);
                r(i, j) = sum;
            }
        }
        return r;
    }

    // Transform a point (w = 1): rotation AND translation apply
    Vector3 transformPoint(const Vector3& v) const {
        const auto& m = *this;
        return {m(0, 0) * v.x + m(0, 1) * v.y + m(0, 2) * v.z + m(0, 3),
                m(1, 0) * v.x + m(1, 1) * v.y + m(1, 2) * v.z + m(1, 3),
                m(2, 0) * v.x + m(2, 1) * v.y + m(2, 2) * v.z + m(2, 3)};
    }

private:
    std::array<double, 16> elements_{};
};
```

Notice the C++ naming — `rotationZ`, `transformPoint`, `setIdentity`. Python code expects `snake_case`; the bindings are where that gets translated.

### The bindings

`bindings.cpp` describes, member by member, what the Python classes should look like. It is declarative: you list what to expose and pybind11 generates the glue.

```cpp
// src/bindings.cpp — the only file that knows about Python
#include <pybind11/pybind11.h>
#include <pybind11/operators.h>   // py::self + py::self, py::self * double(), ...

#include <sstream>

#include "vecmath.hpp"

namespace py = pybind11;

PYBIND11_MODULE(vecmath, m) {              // module name == CMake target name
    m.doc() = "A tiny 3D math library implemented in C++";

    py::class_<Vector3>(m, "Vector3")
        .def(py::init<>())
        .def(py::init<double, double, double>(), py::arg("x"), py::arg("y"), py::arg("z"))
        .def_readwrite("x", &Vector3::x)
        .def_readwrite("y", &Vector3::y)
        .def_readwrite("z", &Vector3::z)
        .def("length", &Vector3::length)
        .def("normalized", &Vector3::normalized)
        .def("dot", &Vector3::dot, py::arg("other"))
        .def("cross", &Vector3::cross, py::arg("other"))
        .def(py::self + py::self)              // __add__
        .def(py::self - py::self)              // __sub__
        .def(py::self * double())              // __mul__ (Vector3 * float)
        .def("__repr__", [](const Vector3& v) {
            std::ostringstream ss;
            ss << "Vector3(" << v.x << ", " << v.y << ", " << v.z << ")";
            return ss.str();
        });

    py::class_<Matrix4>(m, "Matrix4")
        .def(py::init<>())
        .def_static("translation", &Matrix4::translation, py::arg("x"), py::arg("y"), py::arg("z"))
        .def_static("rotation_z", &Matrix4::rotationZ, py::arg("angle"))     // renamed
        .def("transform_point", &Matrix4::transformPoint, py::arg("point"))  // renamed
        .def(py::self * py::self)              // Matrix4 * Matrix4
        .def("__getitem__", [](const Matrix4& m, std::pair<int, int> rc) {   // m[row, col]
            auto [r, c] = rc;
            if (r < 0 || r > 3 || c < 0 || c > 3) throw py::index_error("Matrix4 index out of range");
            return m(r, c);
        })
        .def("__setitem__", [](Matrix4& m, std::pair<int, int> rc, double value) {
            auto [r, c] = rc;
            if (r < 0 || r > 3 || c < 0 || c > 3) throw py::index_error("Matrix4 index out of range");
            m(r, c) = value;
        })
        .def("__repr__", [](const Matrix4& m) {
            std::ostringstream ss;
            ss << "Matrix4(\n";
            for (int r = 0; r < 4; ++r) {
                ss << "  ";
                for (int c = 0; c < 4; ++c) ss << m(r, c) << (c < 3 ? ", " : "");
                ss << "\n";
            }
            ss << ")";
            return ss.str();
        });
}
```

The pieces, in the order they appear:

- **`PYBIND11_MODULE(vecmath, m)`** defines the module's entry point. The name must match the CMake target (and therefore the file name Python imports); `m` is the module object you add things to.
- **`py::class_<Vector3>(m, "Vector3")`** opens a Python class wrapping the C++ type. Everything chained onto it with `.def…` becomes a member of that class.
- **`py::init<…>()`** binds a constructor. `py::arg("x")` names the parameters, which gives keyword arguments (`Vector3(x=1, y=0, z=0)`) and a readable signature in `help()`.
- **`def_readwrite`** turns a public field into a read/write attribute: `v.z = 5.0` writes straight into the C++ struct.
- **`def`** binds a method from a pointer-to-member. pybind11 reads the C++ signature and does the argument conversion — pass a string where a `Vector3` is expected and you get a Python `TypeError`, not silent corruption.
- **`def_static`** binds a static function, so `Matrix4.translation(10, 0, 0)` is called on the class, just like a Python `@staticmethod`.
- **Operators** come from `<pybind11/operators.h>`: `py::self + py::self` becomes `__add__`, `py::self * double()` becomes `__mul__` with a float on the right.
- **Python protocols** are just methods with the dunder name. A lambda implements `__repr__`, and `__getitem__`/`__setitem__` give `m[row, col]` indexing — Python passes the tuple `(row, col)`, which pybind11 converts to a `std::pair<int, int>`. Throwing `py::index_error` surfaces as a Python `IndexError`.
- **Renaming at the boundary.** `rotationZ` becomes `rotation_z`, `transformPoint` becomes `transform_point`. The binding layer is the right place to translate conventions, so each language's users see what they expect.

??? note "The same `Vector3` through `ctypes`"

    For contrast, this is the most `ctypes` can offer: a C struct and free functions with C linkage.

    ```cpp
    extern "C" {
        struct Vec3 { double x, y, z; };
        double vec3_length(Vec3 v);
        Vec3   vec3_cross(Vec3 a, Vec3 b);
    }
    ```

    ```python
    import ctypes

    class Vec3(ctypes.Structure):
        _fields_ = [("x", ctypes.c_double), ("y", ctypes.c_double), ("z", ctypes.c_double)]

    lib = ctypes.CDLL("vec3lib.dll")           # "libvec3lib.so" on Linux/macOS
    lib.vec3_cross.argtypes = [Vec3, Vec3]
    lib.vec3_cross.restype  = Vec3

    c = lib.vec3_cross(Vec3(1, 0, 0), Vec3(0, 1, 0))
    print(c.x, c.y, c.z)                        # 0.0 0.0 1.0
    ```

    No methods, no operators, no `__repr__` — unless you write a Python wrapper class by hand — and every `Matrix4` operation would become a free function taking an array of 16 doubles. This is exactly the "rich API" case where pybind11 pays for itself.

### Building it

The `CMakeLists.txt` is short. `pybind11_add_module` creates a `MODULE` library, links it against the Python it found, and names the output with Python's ABI tag — `vecmath.cp310-win_amd64.pyd` on Windows, `vecmath.cpython-312-x86_64-linux-gnu.so` on Linux — so the file only ever imports into the interpreter it was built for:

```cmake
cmake_minimum_required(VERSION 3.16)
project(vecmath CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(pybind11 CONFIG REQUIRED)

pybind11_add_module(vecmath src/bindings.cpp)   # -> vecmath.*.pyd / vecmath.*.so
target_include_directories(vecmath PRIVATE include)
```

```json
{
  "dependencies": [ "pybind11" ]
}
```

!!! tip "Getting pybind11 from `pip` instead of vcpkg"
    pybind11 is also a `pip` package, and that route has one big advantage: the module is guaranteed to be built against the Python you will run it with. The vcpkg port, by contrast, depends on vcpkg's own `python3` port — so the first configure builds a complete CPython (and OpenSSL) from source, which took close to half an hour on a laptop — and the Python the module binds to is simply whichever one CMake finds first, which may not be the one you run.

    ```bash
    pip install pybind11
    cmake -S . -B build -Dpybind11_DIR="$(python -m pybind11 --cmakedir)"
    cmake --build build --config Release
    ```

    `python -m pybind11 --cmakedir` prints the folder containing `pybind11Config.cmake`, so the same `find_package(pybind11 CONFIG REQUIRED)` works unchanged — just drop `vcpkg.json`. In CLion, put the `-Dpybind11_DIR=…` in **Settings → Build, Execution, Deployment → CMake → CMake options**.

### Using it from Python

With the build folder on `sys.path` (see below), the classes are indistinguishable from ones written in Python:

```python
# use_vecmath.py
import math
from vecmath import Vector3, Matrix4

p = Vector3(1.0, 0.0, 0.0)
print(p, p.length())                      # Vector3(1, 0, 0) 1.0

q = p + Vector3(0.0, 2.0, 0.0)
print(q, q.normalized())                  # Vector3(1, 2, 0) Vector3(0.447214, 0.894427, 0)
print(p.cross(Vector3(0.0, 1.0, 0.0)))    # Vector3(0, 0, 1)

p.z = 5.0                                 # attributes are read/write
print(p.dot(Vector3(0.0, 0.0, 1.0)))      # 5.0

# Rotate 90 degrees about z, THEN translate by (10, 0, 0)
t = Matrix4.translation(10.0, 0.0, 0.0) * Matrix4.rotation_z(math.pi / 2)
print(t.transform_point(Vector3(1.0, 0.0, 0.0)))   # Vector3(10, 1, 0)
print(t[0, 3])                            # 10.0 — the translation column
```

Every call crosses into compiled C++ and back. Ask Python for help and the signatures it prints are derived from the C++ types:

```python
>>> help(Vector3.dot)
dot(self: vecmath.Vector3, other: vecmath.Vector3) -> float
>>> Matrix4()[4, 0]
IndexError: Matrix4 index out of range
```

!!! warning "Python finds the module by `sys.path`, not by where you are"
    The `.pyd`/`.so` lands in the **build folder** — `cmake-build-debug/` or `cmake-build-release/` in CLion, `build/Release/` with the commands above — and `import vecmath` fails with `ModuleNotFoundError` until Python can see it. Python searches `sys.path`, which starts with the **folder of the script you are running** (not the current directory) and then `site-packages`. So either:

    - run with the build folder on the path: `PYTHONPATH=build/Release python use_vecmath.py` (PowerShell: `$env:PYTHONPATH="build\Release"; python use_vecmath.py`),
    - add it at the top of the script: `sys.path.insert(0, r"C:\path\to\build\Release")`, or
    - copy the `.pyd` next to the script.

    The file name's ABI tag (`cp310-win_amd64`) is checked on import: a module built for Python 3.10 does not import into 3.12, however it is found.

!!! warning "The module is bound to one specific Python"
    A pybind11 extension is compiled **against a particular Python version and installation**, and it will only import into *that* interpreter. This is the common trap on Windows: if you build against **vcpkg's** Python but run your script under a **conda** Python (or vice-versa), the import fails with a version/ABI mismatch. Make sure the Python that CMake finds at build time is the **same** interpreter you run the module with — the `pip install pybind11` route above makes that automatic. `ctypes`, by contrast, is not tied to a build-time Python at all — which is one reason it is the path the [exercises](exercises.md) assess.

---

## Choosing between them

The trade-off is exactly the usual one: **pybind11** does more for you but hides more machinery, while **ctypes** is more verbose but fully transparent. For a small, stable C interface, `ctypes` keeps everything in plain sight; for a rich C++ API you want to expose comfortably — anything with classes, like `Vector3` and `Matrix4` above — pybind11 is far less work.

| | `ctypes` | pybind11 |
|---|----------|----------|
| Extra library | No (built into Python) | Yes (header-only, via vcpkg or `pip`) |
| Boilerplate | Manual `extern "C"` + signatures | Minimal |
| C++ classes / STL types | Awkward (C interface only) | Native support |
| Type errors | Silent corruption if signatures are wrong | `TypeError` at the call |
| Transparency | Full — no magic | More hidden machinery |
| Tied to a Python version | No | Yes (built per interpreter) |
| Best for | A small C interface | A rich C++ API as a Python module |

---

## Where this fits

For AIS2203 the bridge shows up wherever the two worlds meet: a [vision](../Chapter6/deep_vision.md) model explored in Python with a [C++ inference core](../Chapter6/onnx.md) for speed, or a Python script driving a C++ control library — where types like `Vector3` and `Matrix4` are exactly what crosses the boundary. The guiding idea is the one from the CV part — **keep the fast, real-time, low-level work in C++ and the glue in Python** — and either bridge lets you draw that line wherever it makes sense.

---

## Summary

- Bridging Python and C++ gives you Python's convenience with C++'s **speed, existing libraries, and low-level access** — the "fast core, easy glue" split.
- You **cannot call C++ directly** because of **name mangling**; expose a **C interface** with `extern "C"` (plain, stable symbol names). The implementation is still full C++, but the *exposed* functions use C-compatible types.
- **`ctypes`** (built into Python) loads a **shared** library (export symbols on Windows) and calls it once you declare `argtypes`/`restype`; wrap it in a Pythonic class. Transparent, but manual.
- **pybind11** exposes C++ functions *and classes* as a real Python module: `py::class_` plus `py::init`, `def`, `def_readwrite`, `def_static`, operator overloads, and dunder methods like `__repr__` and `__getitem__` make a C++ `Vector3`/`Matrix4` look native. Keep the library Python-free and do all binding (and renaming to `snake_case`) in one file.
- The built `.pyd`/`.so` lives in the build folder and is **tied to one Python version**; put the folder on `sys.path` and build with the same interpreter you run (`pip install pybind11` + `-Dpybind11_DIR` makes that automatic).
- Choose **`ctypes`** for a small, transparent C interface; **pybind11** for a rich C++ API. Either way, keep the **fast/low-level work in C++ and the glue in Python** — the [prototype-in-Python, deploy-in-C++](../Chapter6/opencv.md) idea made concrete.

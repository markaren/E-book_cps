# Frequently Asked Questions

Short answers to questions that come up every year. Most link to the chapter with the full story.

---

### My threaded program gives a different answer every run (or crashes sometimes).

Classic symptom of a **[data race](Chapter2/sharing_data.md)**: two threads touch the same data with no synchronisation. Protect the shared data with a [mutex](Chapter2/sharing_data.md) or make it [atomic](Chapter2/atomics.md), and confirm it is gone by building with [ThreadSanitizer](debugging_concurrency.md) (`-fsanitize=thread`). "It usually works" is exactly what a race looks like.

### My program hangs and never finishes.

Two usual causes: a **[deadlock](Chapter2/sharing_data.md)** (two threads each waiting for a lock the other holds — fix with a consistent lock order or `std::scoped_lock`), or a [condition variable](Chapter2/condition_variables.md) waiting for a notify that never comes (often a missing `done` flag in the predicate). Attach a debugger and read every thread's stack ([Debugging](debugging_concurrency.md)) to see who is stuck where.

### `undefined reference to pthread_create` at link time.

On Linux/the Pi the thread library is separate — link it. In CMake: `find_package(Threads REQUIRED)` then `target_link_libraries(app PRIVATE Threads::Threads)`. ([Getting Started](getting_started.md))

### `std::jthread` / `std::counting_semaphore` / `std::latch` not found.

Your compiler is too old or not set to C++20. These are C++20 features; set `target_compile_features(app PUBLIC cxx_std_20)` and use GCC 11+, Clang 14+, or MSVC 2022. ([Getting Started](getting_started.md))

### `terminate called` right as my program ends, with a thread.

You created a `std::thread` and neither `join()`ed nor `detach()`ed it before it was destroyed. Use **[`std::jthread`](Chapter2/threads.md)**, which joins itself automatically.

### My socket `recv()` returns half a message, or two messages stuck together.

[TCP is a byte stream, not a message queue](Chapter4/sockets.md) — it does not preserve your `send()` boundaries. You must **frame** messages yourself with a length prefix or a delimiter (e.g. a newline after each [JSON](Chapter4/serialization.md) object). UDP and [MQTT](Chapter4/mqtt_rpc.md) preserve boundaries; TCP and [serial](Chapter4/serial.md) do not.

### My `std::execution::par` algorithm isn't faster (or won't link).

On GCC/Clang the [parallel algorithms](Chapter3/parallel_algorithms.md) need **Intel TBB** linked (`find_package(TBB)` + `TBB::tbb`), or `par` silently runs sequentially. And parallelism has overhead — it only pays off on *large* data; on small ranges the sequential version wins. Measure.

### Should I use threads, `std::async`, or coroutines?

Rough guide ([Part 3](Chapter3/futures.md)): a few long-lived activities → [threads](Chapter2/threads.md); compute a result on another thread → [`std::async`](Chapter3/futures.md); the same operation over a big collection → [parallel algorithms](Chapter3/parallel_algorithms.md); many short tasks → a [thread pool](Chapter3/thread_pools.md); thousands of I/O-bound waits → [coroutines](Chapter3/coroutines.md) (via a library).

### Is `std::shared_ptr` thread-safe?

Its **reference count is** — copying/destroying `shared_ptr`s across threads is safe. The **object it points to is not** — concurrent mutation of the pointee still needs a [mutex](Chapter2/sharing_data.md). ([Smart Pointers](Chapter1/smart_pointers.md))

### How do I add a library like OpenCV, Boost, or nlohmann/json?

Use **[vcpkg](Chapter6/dependencies.md)**: declare it in `vcpkg.json`, configure CMake with vcpkg's toolchain file, then `find_package` + `target_link_libraries`. For one or two dependencies, CMake's [`FetchContent`](Chapter6/cmake.md) also works.

### Where did my output file (or `readings.txt`) go?

Your program runs from its **working directory**, which in CLion is the *build* folder (e.g. `cmake-build-debug/`), not your project folder. That is where output files appear and where input files are looked for — put them there or set the working directory under **Run → Edit Configurations**.

### Do I need a Raspberry Pi or other hardware?

**No.** This year the project runs in a [3D simulator](Chapter7/virtual_environments.md) on your own machine. The [Embedded Linux](embedded_linux.md) reference is background — the same C++ you write for the simulated robot would run on a real Pi, but you do not need one for the coursework.

### Can I call my C++ code from Python?

Yes. Expose a C interface with [`extern "C"`](Chapter6/python_interop.md) and call it with `ctypes`, or use **pybind11** to expose C++ functions and classes directly as a Python module. ([Calling C++ from Python](Chapter6/python_interop.md))

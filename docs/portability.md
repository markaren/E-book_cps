# Portability

Your code runs in more places than you might think: on your Windows laptop, on a teammate's Linux machine or WSL2, against the [simulator](Chapter7/virtual_environments.md), and — conceptually — on a Raspberry Pi ([Embedded Linux](embedded_linux.md)). C++ *source* can be portable across all of these, but only if you write it that way. This page collects the habits that keep one codebase building and behaving the same everywhere, with a focus on the concurrency and communication code this book is about.

---

## Source is portable; binaries are not

The foundation, restated: a compiled binary is machine code for **one** operating system and CPU architecture — a Windows `.exe` does not run on Linux, and an x86-64 build does not run on ARM. What *is* portable is the **source code**, recompiled on each platform. [CMake](Chapter6/cmake.md) is what makes that recompilation uniform: one `CMakeLists.txt` configures and builds the same project on Windows, Linux, and macOS.

But "compiles on my machine" is not the same as "portable." Three things conspire against you, and the rest of this page is how to handle each.

---

## Compilers and standard libraries differ

C++ has three major compilers — **MSVC** (Windows), **GCC** (Linux), **Clang** (macOS and elsewhere) — each with its own quirks, its own warnings, and its own implementation of the standard library. Code that compiles cleanly on one can warn, error, or subtly misbehave on another:

- One compiler may accept a non-standard extension another rejects.
- A bug that relies on [undefined behaviour](#undefined-behaviour-is-not-portable) may "work" under one and break under another.
- Standard-library details left unspecified by the standard (hash ordering, small-string buffer sizes) can differ.

The practical defence is to **build on more than one compiler**. With [WSL2](getting_started.md) you can compile your project with both MSVC and GCC on the same laptop — and a difference that shows up on the second compiler is a portability bug you would otherwise have shipped.

!!! tip "Turn warnings on and take them seriously"
    Build with `-Wall -Wextra` (GCC/Clang) or `/W4` (MSVC) — set per target in [CMake](Chapter6/cmake.md). A large fraction of portability problems surface first as a warning on one compiler. Treat warnings as errors you have not hit yet.

---

## Prefer portable libraries over OS APIs

The biggest portability win is to talk to the **standard library** and **cross-platform libraries**, never directly to the operating system. The OS APIs differ; the libraries hide that difference behind one interface:

| Instead of… | Use… |
|-------------|------|
| Win32 threads / pthreads | `std::thread` / [`std::jthread`](Chapter2/threads.md) |
| Winsock2 vs `sys/socket` | [Boost.Asio or SimpleSocket](Chapter4/networking.md) |
| Manual `\` vs `/` path handling | `std::filesystem::path` |
| Platform sleep/timer calls | [`std::chrono`](Chapter1/chrono.md) + `sleep_for` |

This is exactly the argument the [Networking chapter](Chapter4/networking.md) made: writing raw sockets means `#include <winsock2.h>` and `WSAStartup` on Windows versus `<sys/socket.h>` on Linux, and a library exists to spare you that. The same holds for threads, files, and time. When you find yourself reaching for an `#ifdef _WIN32`, stop and look for a library that already abstracts the difference — a codebase peppered with platform `#ifdef`s is a maintenance trap.

The thread library is the one piece you must still wire up per platform, and [CMake](Chapter6/cmake.md) handles even that: `find_package(Threads REQUIRED)` + `target_link_libraries(... Threads::Threads)` expands to `-pthread` on Linux and nothing on Windows, so the *same* `CMakeLists.txt` links correctly everywhere ([Getting Started](getting_started.md)).

---

## Types, bytes, and the wire

When data crosses between machines — over a [socket](Chapter4/sockets.md), a [serial line](Chapter4/serial.md), or [Modbus](Chapter4/modbus.md) — small representation differences become real bugs:

- **Integer sizes vary.** `int` is usually 32-bit, but `long` is 32-bit on Windows and 64-bit on Linux. When the exact size matters — and on the wire it always does — use the **fixed-width types** from `<cstdint>`: `std::int32_t`, `std::uint16_t`, and so on.
- **Endianness varies.** The byte order of multi-byte numbers differs between architectures; the agreed order *on the wire* is big-endian ([network byte order](Chapter4/serialization.md)). Never send a number's raw bytes assuming the other end has the same order — this is the [serialization](Chapter4/serialization.md) and [Modbus](Chapter4/modbus.md) lesson.
- **`char` signedness varies**, and **struct padding differs** between compilers — which is exactly why you [never send a raw struct over the wire](Chapter4/serialization.md) and use a serialization format instead.

For storage and text, `std::filesystem::path` handles separators, and line endings differ (`\n` vs `\r\n`) — open files in binary mode when you need the bytes exactly as written.

---

## Undefined behaviour is not portable

A program with **undefined behaviour** has no guaranteed meaning, so the compiler may do *anything* — and different compilers, optimization levels, and platforms do different things. UB is the deepest portability trap because it can "work" for years and then break when you change compiler or turn on optimization. Two kinds you have already met:

- A [**data race**](Chapter2/sharing_data.md) is undefined behaviour — which is why a threaded program can be correct on one machine and wrong on another.
- Reading a [moved-from](Chapter1/move_semantics.md) object, a dangling reference, an out-of-bounds access — all UB.

The [sanitizers](debugging_concurrency.md) (TSan, ASan, UBSan) exist precisely to catch UB at runtime; run them, because UB that happens to work today is a portability failure waiting for tomorrow.

---

## Summary

- C++ **source** is portable; **binaries** are not. [CMake](Chapter6/cmake.md) makes the *build* uniform across Windows/Linux/macOS, and [vcpkg](Chapter6/dependencies.md) supplies dependencies per platform.
- **Compilers and standard libraries differ** (MSVC/GCC/Clang) — build on more than one (WSL2 makes this easy) and keep `-Wall -Wextra` / `/W4` on.
- **Prefer the standard library and cross-platform libraries** (`std::thread`, `std::filesystem`, `std::chrono`, [Asio/SimpleSocket](Chapter4/networking.md)) over OS APIs and `#ifdef` spaghetti. Link the thread library portably with `Threads::Threads`.
- On the **wire**, use **fixed-width `<cstdint>` types**, mind **endianness** (big-endian on the wire), and never send raw structs — use a [serialization format](Chapter4/serialization.md).
- **Undefined behaviour is not portable** — a [data race](Chapter2/sharing_data.md) or dangling access may work on one platform and fail on another; catch it with the [sanitizers](debugging_concurrency.md).

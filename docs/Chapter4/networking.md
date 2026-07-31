# Networking in C++

[Sockets](sockets.md) showed the raw POSIX API — `socket`, `bind`, `accept`, `send`, `recv` — and ended with a warning: it is verbose, error-prone, and *different* on Windows (Winsock2) versus Linux (`sys/socket`). Because the C++ standard library offers **no** networking, you have to bridge that gap with a library. This chapter surveys the ones this course uses and the one design decision they force on you: **blocking** or **asynchronous** I/O.

---

## Why a library at all

Re-read the raw echo server from [Sockets](sockets.md) and count the hazards: `reinterpret_cast` to `sockaddr*`, manual `htons`, a different header and a `WSAStartup` call on Windows, an `int` file descriptor you must remember to `close`, and no error handling. None of that is your program's logic — it is boilerplate the platform forces on you. A networking library wraps all of it behind a clean, cross-platform, RAII interface so you can write the part that matters.

Three libraries cover almost everything you will need:

| Library | What it is | Reach for it when |
|---------|-----------|-------------------|
| **Boost.Asio** | The de-facto C++ networking & async I/O library | Serious networking, especially asynchronous servers |
| **SimpleSocket** | A minimal cross-platform socket wrapper, no dependencies | Straightforward client/server work in course projects |
| **libcurl** | A battle-tested client for HTTP and many other protocols | Fetching from web URLs / talking to HTTP APIs |

---

## SimpleSocket: the small option

For most course work you do not need an industrial async framework — you need a TCP connection that works the same on Windows and the Pi without ceremony. [SimpleSocket](https://github.com/markaren/SimpleSocket) is a thin cross-platform wrapper around Win/Unix sockets with no third-party dependencies. The whole [raw echo pair from Sockets](sockets.md) — which only ran on Linux/WSL2 — becomes this, and it **builds and runs in CLion on Windows out of the box**:

**Server:**

<!-- no-ce -->
```cpp
#include <simple_socket/TCPSocket.hpp>
#include <iostream>
#include <string>

using namespace simple_socket;

int main() {
    TCPServer server(8080);                       // claim port 8080 and listen
    std::cout << "listening on port 8080...\n";

    auto conn = server.accept();                  // block until a client connects

    std::string buffer(1024, '\0');
    int n = conn->read(buffer);                   // read whatever arrived
    conn->write(std::vector<unsigned char>(buffer.begin(), buffer.begin() + n));  // echo it back
    // conn and server close themselves via RAII
}
```

**Client:**

<!-- no-ce -->
```cpp
#include <simple_socket/TCPSocket.hpp>
#include <iostream>
#include <string>

using namespace simple_socket;

int main() {
    TCPClientContext ctx;
    auto conn = ctx.connect("127.0.0.1", 8080);   // reach out to the server

    conn->write("hello");

    std::string buffer(1024, '\0');
    int n = conn->read(buffer);
    std::cout << "server replied: " << buffer.substr(0, n) << "\n";  // server replied: hello
    // conn closes itself when it goes out of scope (RAII)
}
```

Run the server in one CLion configuration, the client in another, and the client prints `server replied: hello`. Compare with the [raw version](sockets.md): no `WSAStartup`, no `reinterpret_cast`, no `SOCKET`-vs-`int`, no manual `close` — and *identical* source on Windows, Linux, and the Pi. `read` returns the byte count (a `read` may return fewer bytes than a full message — the [framing](serialization.md) problem again; `readExact` loops until a known number of bytes arrive). SimpleSocket also bundles a [Modbus TCP](modbus.md) client and an optional [MQTT](mqtt_rpc.md) client, which is why the next chapters lean on it. For a robot streaming telemetry to a laptop, this is usually all you need.

---

## Boost.Asio: the serious option

[Boost.Asio](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html) is the most widely used C++ networking library, and the basis of many others. It provides TCP, UDP, timers, and serial ports through one model built around an **`io_context`** — an object that owns the I/O and runs the event loop. Asio supports both styles of I/O, and that choice is the heart of this chapter.

### Blocking vs asynchronous I/O

A **blocking** (synchronous) call waits until it completes: `recv` does not return until data arrives. It is simple to reason about, but one thread can serve only one connection at a time — so handling many connections means **one thread per connection**, which (as [Thread Pools](../Chapter3/thread_pools.md) explained) does not scale to thousands.

An **asynchronous** call returns immediately; you say *what to do when the operation finishes*, and a single thread driving the `io_context` services many connections as their data arrives. No thread sits idle waiting.

```mermaid
flowchart TB
    subgraph blocking["Blocking: one thread per connection"]
      tA["thread A — conn 1 (mostly waiting)"]
      tB["thread B — conn 2 (mostly waiting)"]
      tC["thread C — conn 3 (mostly waiting)"]
    end
    subgraph async["Async: one thread, many connections"]
      loop["io_context event loop"] --> c1["conn 1"] & c2["conn 2"] & c3["conn 3"]
    end
```

Historically you expressed the "what to do next" as a **callback** (a handler function Asio calls on completion) — flexible, but nests into the "callback hell" [Futures & Promises](../Chapter3/futures.md) warned about. Modern Asio lets you instead use C++20 **[coroutines](../Chapter3/coroutines.md)**, so asynchronous code reads like straight-line blocking code while still never blocking a thread:

<!-- no-ce -->
```cpp
using boost::asio::awaitable;
using boost::asio::co_spawn;
using boost::asio::detached;
using boost::asio::ip::tcp;
namespace this_coro = boost::asio::this_coro;

// Asio coroutine: suspends (without blocking the thread) at each co_await
awaitable<void> echo(tcp::socket socket) {
    char data[1024];
    for (;;) {                                    // loops until the peer disconnects
        std::size_t n = co_await socket.async_read_some(boost::asio::buffer(data));
        co_await async_write(socket, boost::asio::buffer(data, n));
    }
}

int main() {
    boost::asio::io_context io;
    tcp::acceptor acceptor(io, {tcp::v4(), 8080});
    tcp::socket socket = acceptor.accept();       // (blocking accept, for brevity)

    co_spawn(io, echo(std::move(socket)), detached);  // schedule the coroutine
    io.run();                                     // drive the event loop until work is done
}
```

Two things to note. The **token-less** `co_await socket.async_read_some(...)` form — no explicit completion token — needs **Boost ≥ 1.86**; on older Boost you pass `boost::asio::use_awaitable` as a final argument to each async call instead. And the `io_context` is what actually *runs* the coroutine: `co_spawn` schedules it and `io.run()` drives the event loop, so nothing happens until you call `run()`. If the peer drops mid-conversation, `async_read_some` fails and **throws out of the coroutine** — you would wrap the body in `try`/`catch` (or spawn with an error-aware completion handler) to clean up that one connection without taking down the server.

This is the payoff promised in the [Coroutines](../Chapter3/coroutines.md) chapter: one or a few threads can drive thousands of connections, each a suspended coroutine costing almost nothing while it waits — the scalable structure for a busy server, with readable code.

!!! tip "Match the model to the load"
    For a handful of connections, **blocking** I/O (with [SimpleSocket](https://github.com/markaren/SimpleSocket) or blocking Asio) is simplest and perfectly fine — a thread per connection is no problem at small scale. Reach for **asynchronous** Asio when you have many connections, or strict [latency](../Chapter3/real_time.md) goals that a thread-per-connection design cannot meet. Don't pay async's complexity tax before you need it.

---

## libcurl: when you just need HTTP

If the task is "fetch this URL" or "POST to this web API", you do not want raw sockets *or* Asio — you want **libcurl**, the library behind the `curl` command. It speaks HTTP/HTTPS (and much more), handles TLS, redirects, and authentication, and is available everywhere. Use it whenever your program is an HTTP *client* talking to a web service; reserve sockets and Asio for custom protocols and servers.

---

## Choosing

| Situation | Use |
|-----------|-----|
| A few TCP connections, course project | **SimpleSocket** (or blocking Asio) |
| Many connections / async / low latency | **Boost.Asio** (coroutines or callbacks) |
| HTTP client, talking to a web API | **libcurl** |
| Industrial device speaking Modbus | [Modbus](modbus.md) library (SimpleSocket has one) |
| Publish/subscribe or remote calls | [MQTT / RPC](mqtt_rpc.md) library |

Whatever you pick, install it through vcpkg rather than by hand — `vcpkg add port boost-asio` / `curl` in your project's manifest — so the dependency is reproducible. (vcpkg and its manifest workflow are what [Part 5 shows how to set up](../Chapter5/dependencies.md).)

---

## Summary

- C++ has **no standard networking**, and the raw OS APIs are verbose and platform-specific — so you use a library that wraps them with a clean, cross-platform, RAII interface.
- **SimpleSocket** is a dependency-free wrapper good for straightforward course client/server work (and it includes [Modbus TCP](modbus.md)); **Boost.Asio** is the serious, full-featured choice; **libcurl** is for HTTP clients.
- The core decision is **blocking vs asynchronous** I/O: blocking is simple but needs **one thread per connection**; asynchronous (Asio's `io_context`) lets **one thread serve many** connections. Modern Asio uses C++20 [coroutines](../Chapter3/coroutines.md) so async code reads like blocking code.
- Use blocking I/O at small scale; reach for async only when connection count or [latency](../Chapter3/real_time.md) demands it.
- Install networking libraries via [vcpkg](../Chapter5/dependencies.md). Next: [Serial Communication](serial.md), for the wire to a microcontroller.

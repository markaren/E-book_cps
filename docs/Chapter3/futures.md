# Futures & Promises

[Creating Threads](../Chapter2/threads.md) left one question open: a thread's callable cannot `return` a value to whoever started it. So how do you run a calculation on another thread and *get the answer back*? The answer is a **future** — a handle to a result that does not exist yet, but will. Futures also solve a second problem that raw threads handle badly: what happens to an **exception** thrown on another thread.

This is the gentlest way into asynchronous programming, and the foundation the rest of Part 3 builds on.

---

## `std::async`: run a function, get a future

The simplest tool is `std::async` (from `<future>`). You hand it a callable; it runs that callable — possibly on another thread — and hands you back a `std::future<T>` representing the eventual result. Call `.get()` on the future to retrieve the value, blocking until it is ready.

```cpp
#include <future>
#include <iostream>

int slowSquare(int x) {
    // imagine this takes a while
    return x * x;
}

int main() {
    std::future<int> result = std::async(slowSquare, 7);

    std::cout << "doing other work while it computes...\n";

    int value = result.get();          // blocks here until slowSquare returns
    std::cout << "7 squared is " << value << "\n";
}
```

The shape is the whole point: you launch the work, carry on doing something else, and only block at the moment you actually need the answer. No manual thread, no `join()`, no shared variable to guard — the future *is* the communication channel.

!!! warning "`get()` can be called only once"
    A `std::future` delivers its result exactly once. After `result.get()`, the future is empty (`valid()` becomes `false`) and calling `get()` again is undefined behaviour. Store the value in a variable if you need it more than once — or use a `std::shared_future` (below) when several places must read the same result.

### Launch policies

`std::async` takes an optional first argument saying *how* to run the task:

| Policy | Meaning |
|--------|---------|
| `std::launch::async` | Run **now**, on a new thread. |
| `std::launch::deferred` | Run **lazily** — the function does not execute until you call `.get()`, and then it runs on *this* thread. |
| default (both) | The implementation chooses either. |

If you actually want concurrency, ask for it explicitly — `std::async(std::launch::async, slowSquare, 7)` — because the default is allowed to defer, in which case nothing runs in parallel at all.

!!! danger "The blocking-destructor gotcha"
    The future returned by `std::async` with the `async` policy has a surprising property: **its destructor blocks** until the task finishes. So `std::async(std::launch::async, work);` on its own line — without storing the future — launches the task and then immediately waits for it at the end of the full expression, making it effectively synchronous. Always keep the future in a named variable that lives as long as you want the work to run in the background.

---

## Exceptions cross the thread boundary

With a raw `std::thread`, an exception that escapes the thread's function calls `std::terminate()` and kills the program — there is no way to catch it from another thread. Futures fix this. An exception thrown inside an async task is **captured**, stored in the future, and **re-thrown when you call `get()`**:

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int risky(int x) {
    if (x < 0) {
        throw std::invalid_argument("x must be non-negative");
    }
    return x * 2;
}

int main() {
    std::future<int> result = std::async(std::launch::async, risky, -1);

    try {
        int value = result.get();      // the exception surfaces here
        std::cout << "got " << value << "\n";
    } catch (const std::exception& e) {
        std::cout << "task failed: " << e.what() << "\n";
    }
}
```

The exception travels from the worker thread to the thread that calls `get()`, where you handle it with an ordinary `try`/`catch`. This is a major reason to prefer futures over hand-rolled threads for anything that can fail: error handling stays where it belongs, using the language feature you already know.

---

## `std::promise`: the manual end of the channel

`std::async` ties the result-producing function and the future together. Sometimes you want to **separate** them — one thread decides the result at some arbitrary later point, and another waits for it. A `std::promise<T>` is the *writing* end of a one-shot channel; the `std::future<T>` you get from it is the *reading* end.

```cpp
#include <future>
#include <iostream>
#include <thread>

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::jthread worker([&promise] {
        // ... compute something ...
        promise.set_value(42);          // fulfil the promise; unblocks future.get()
    });

    std::cout << "waiting for the worker's result...\n";
    std::cout << "worker produced " << future.get() << "\n";
}
```

The producer calls `set_value` (or `set_exception` to deliver a failure) exactly once; the consumer's `get()` blocks until it does. Think of it as a single-use mailbox: the promise drops one value in, the future takes it out. Use a promise when the value becomes available somewhere that is not simply "the return of a function" — for example, deep inside a callback, or when one thread must signal a specific result to another.

---

## `std::packaged_task`: a callable wired to a future

A `std::packaged_task<Sig>` wraps a callable so that calling it stores the return value in an associated future, instead of returning it directly. That makes the task a *movable object you can hand to someone else to run later* — which is exactly what a [thread pool](thread_pools.md) needs.

```cpp
#include <future>
#include <iostream>
#include <thread>

int main() {
    std::packaged_task<int(int, int)> task([](int a, int b) { return a + b; });
    std::future<int> result = task.get_future();

    std::jthread runner(std::move(task), 3, 4);   // run the task on another thread

    std::cout << "sum = " << result.get() << "\n";   // 7
}
```

The task is *moved* onto the thread (a `packaged_task` cannot be copied — it owns the shared result state, like the [move-only types](../Chapter1/move_semantics.md) Part 1 covers). Whoever ends up invoking it could be a worker pulling tasks off a queue; the submitter just keeps the future and reads the answer when it is ready. That separation between *describing* work and *running* it is the basis of the next chapter.

---

## Choosing between them

| Tool | Use it when |
|------|-------------|
| `std::async` | You have a function, you want its result later, and you do not care which thread runs it. The everyday choice. |
| `std::promise` / `std::future` | The result is produced somewhere that is not a clean function return — a callback, an event handler, one thread signalling another. |
| `std::packaged_task` | You want to package work now and let *something else* (a pool, a queue) run it later, while you keep the future. |
| `std::shared_future` | **Several** consumers must each read the same result. A normal `future` allows only one `get()`; copy it into a `shared_future` to fan the value out. |

---

## Summary

- A **`std::future<T>`** is a handle to a result that will exist later; `.get()` blocks until it is ready and returns it (once).
- **`std::async`** runs a callable and hands back a future — the simplest way to compute on another thread and collect the answer. Pass `std::launch::async` to guarantee it actually runs concurrently, and **keep the future in a named variable** (its destructor can block).
- Exceptions thrown in the task are **captured and re-thrown at `get()`**, so failures cross the thread boundary and you handle them with ordinary `try`/`catch` — a key advantage over raw threads.
- **`std::promise`** is the manual writing end of a one-shot channel; pair it with its future when the result appears outside a simple function return.
- **`std::packaged_task`** binds a callable to a future so the work can be queued and run elsewhere — the foundation of the [thread pool](thread_pools.md) in the next chapter.

# Lambdas & std::function

You have been writing lambdas since [Part 2](../Chapter2/threads.md) — they are how you hand work to a thread, a [future](../Chapter3/futures.md), a [thread pool](../Chapter3/thread_pools.md), or a standard algorithm. This chapter makes them precise, with particular attention to the one thing that bites hardest in concurrent code: **what a lambda captures, and how long that capture stays valid.** It then covers `std::function`, the type that lets you *store* a callable — the backbone of task queues and callbacks.

---

## Anatomy of a lambda

A lambda is an unnamed function you can write inline. It has three parts:

```cpp
auto add = [](int a, int b) { return a + b; };
//          ^capture  ^params      ^body
std::cout << add(3, 4) << "\n";   // 7
```

The `[ ]` is the **capture list** — the part that makes lambdas more than just unnamed functions. It lets the lambda grab variables from the surrounding scope and carry them along. That is the powerful and dangerous part.

---

## Captures: by value or by reference

Each captured variable is taken either **by value** (a copy, stored inside the lambda) or **by reference** (an alias to the original). The difference is exactly the difference you would expect — and it is the crux of using lambdas safely with threads.

```cpp
#include <iostream>

int main() {
    int x = 10;
    auto byValue = [x]  { return x; };     // stores a copy of x, taken now
    auto byRef   = [&x] { return x; };     // stores a reference to x

    x = 99;
    std::cout << byValue() << "\n";        // 10 — the snapshot from capture time
    std::cout << byRef()   << "\n";        // 99 — sees the current x
}
```

| Capture | Meaning |
|---------|---------|
| `[x]` | Capture `x` **by value** (a copy) |
| `[&x]` | Capture `x` **by reference** |
| `[=]` | Capture everything used, by value |
| `[&]` | Capture everything used, by reference |
| `[this]` | Capture the enclosing object (access its members) |
| `[p = std::move(ptr)]` | **Init-capture**: move (or compute) a new member into the lambda |

The init-capture form is how you get a **move-only** type like a `unique_ptr` into a lambda — `[d = std::move(data)]` — which you saw transferring ownership into a thread in [Smart Pointers](smart_pointers.md).

!!! tip "Prefer naming captures over `[=]` / `[&]`"
    The catch-all captures `[=]` and `[&]` are convenient and a little dangerous: they hide *what* the lambda depends on, and `[&]` makes it easy to capture a reference by accident. Listing captures explicitly — `[x, &total]` — documents the lambda's dependencies and forces you to think about each one's lifetime. This matters most for the next section.

---

## The capture hazard in concurrent code

Here is the single most important point on this page. A reference capture is only valid as long as the referenced variable is alive. A lambda that **outlives the scope it was created in** — which is exactly what happens when you hand it to a [thread](../Chapter2/threads.md), a [future](../Chapter3/futures.md), or a [pool](../Chapter3/thread_pools.md) — must **not** capture locals by reference:

<!-- no-ce -->
```cpp
// BROKEN: the lambda outlives `local`, leaving a dangling reference
std::jthread startWorker() {
    int local = 42;
    return std::jthread([&local] { use(local); });   // local dies when the function returns
}                                                     // the thread now reads freed memory
```

The thread keeps running after `startWorker` returns, but `local` was destroyed at that `return` — the captured reference now dangles, and reading it is undefined behaviour. The fixes are the ones [Creating Threads](../Chapter2/threads.md) and [Smart Pointers](smart_pointers.md) introduced:

- **Capture by value** (`[local]`) so the lambda owns its own copy; or
- **capture a `std::shared_ptr` by value** when the data is large or must be shared, so the lambda co-owns it and keeps it alive.

```cpp
std::jthread startWorker() {
    int local = 42;
    return std::jthread([local] { use(local); });    // owns a copy — safe
}
```

The rule of thumb: **a lambda that will run later, or on another thread, should capture by value** (or co-own via `shared_ptr`). Reserve reference capture for lambdas used *right here, right now* — like the comparator you pass to `std::sort`, which has finished before the surrounding variables go anywhere.

---

## Mutable and generic lambdas

By default a by-value capture is read-only inside the lambda. Add `mutable` to let the lambda modify its *own copy* (the original is untouched):

```cpp
auto counter = [count = 0]() mutable { return ++count; };
counter();   // 1
counter();   // 2 — the lambda's own count persists across calls
```

A **generic lambda** uses `auto` for its parameters, making it a template in disguise — useful for callbacks that should work with several types:

```cpp
auto print = [](const auto& value) { std::cout << value << "\n"; };
print(42);          // int
print("hello");     // const char*
```

---

## `std::function`: storing any callable

A lambda each has its own unique, unnameable type, so you cannot declare a variable of "lambda type" or put differently-typed lambdas in one container. **`std::function<Signature>`** (from `<functional>`) solves this: it is a wrapper that can hold *any* callable matching a given signature — a lambda, a free function, a functor — behind one uniform type.

```cpp
#include <functional>
#include <iostream>
#include <vector>

int main() {
    // A list of callbacks, each something callable as void(int) — different lambdas, one type.
    std::vector<std::function<void(int)>> callbacks;

    callbacks.push_back([](int n) { std::cout << "log: " << n << "\n"; });

    int limit = 5;
    callbacks.push_back([limit](int n) {
        if (n > limit) std::cout << "alert: " << n << " over " << limit << "\n";
    });

    for (const auto& cb : callbacks) {
        cb(7);                       // calls each registered callback
    }
}
```

This is precisely how the [thread pool](../Chapter3/thread_pools.md) stores its tasks (`std::queue<std::function<void()>>`) and how an observer/callback list works (`std::vector<std::function<void(Event)>>`): you need to keep callables of *different* concrete types in one container, and `std::function` erases their type to a common one. The flexibility costs a little: `std::function` may heap-allocate to store a large captured state, and calling through it is an indirect call — slightly slower than calling a lambda directly.

!!! danger "A stored `std::function` can outlive what it captured"
    The capture hazard is worse for a `std::function` you *store*, because it may live arbitrarily long. A callback that captures `[&]` a local, or `[this]` and then outlives its object, dangles exactly like the thread example above. When you register a callback that will be called later, capture by value or co-own with a `shared_ptr` — and be especially careful with `[this]`: the object must outlive every stored callback that captured it.

---

## `std::function` or a template parameter?

When a function *takes* a callable, you have two choices:

```cpp
template <typename F>
void runTwice(F f) { f(); f(); }              // template: zero overhead, any callable

void runTwice(const std::function<void()>& f) { f(); f(); }   // std::function: stored, uniform
```

- Use a **template parameter** when you just need to *call* the callable now — it inlines, has no overhead, and accepts anything callable. This is what the standard algorithms do.
- Use **`std::function`** when you need to *store* the callable, put it in a container, or expose a stable (non-template) interface across a compilation boundary — accepting the small runtime cost.

The short version: **call now → template; store for later → `std::function`.**

---

## Summary

- A **lambda** is an inline callable; its **capture list** `[ ]` grabs surrounding variables, **by value** (a copy) or **by reference** (an alias).
- The key concurrency rule: a lambda that **runs later or on another thread must capture by value** (or co-own via a [`shared_ptr`](smart_pointers.md)) — capturing a local **by reference** dangles once that local dies. Prefer explicit captures over `[=]`/`[&]`.
- `mutable` lets a lambda modify its own by-value captures; **generic lambdas** (`auto` params) are templates in disguise; **init-capture** (`[p = std::move(ptr)]`) moves move-only types in.
- **`std::function<Sig>`** stores *any* callable of a given signature behind one type — the basis of [thread-pool](../Chapter3/thread_pools.md) task queues and callback lists — at a small runtime cost. A stored `std::function` magnifies the capture hazard; be wary of `[this]`.
- Taking a callable: **template parameter to call now** (zero overhead), **`std::function` to store for later**. Next: [Time with std::chrono](chrono.md), the last toolkit piece — the vocabulary of every timeout and periodic loop.

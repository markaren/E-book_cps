# Templates & Generic Code

AIS1003 introduced templates as the mechanism behind `std::vector<int>` and `std::vector<double>` — write an algorithm or container once, and the compiler stamps out a version for each type you use it with. This chapter assumes that foundation (see the [AIS1003 Templates chapter](https://markaren.github.io/E-book_cpp/Chapter5/templates/) if it is hazy) and focuses on what templates buy you in *this* course: writing your own **generic, reusable concurrency components** — above all, a thread-safe queue you can use with any element type.

---

## A quick recap

A **function template** is parameterised by type; a **class template** is a blueprint for a whole type. The compiler generates concrete versions on demand:

```cpp
template <typename T>
T max3(T a, T b, T c) {
    return std::max(a, std::max(b, c));
}

int main() {
    auto i = max3(3, 7, 2);        // T = int
    auto d = max3(1.5, 0.5, 9.9);  // T = double — a second version, generated for free
}
```

Two properties make templates the right tool for infrastructure code: they are resolved **entirely at compile time** (no runtime cost — `max3<int>` is as fast as a hand-written integer version), and they are **type-safe** (the compiler still checks every instantiation). The cost is compile time and occasionally dense error messages.

---

## The payoff: a generic thread-safe queue

Here is why templates matter for AIS2203. In [Part 2](../Chapter2/condition_variables.md) and [Part 3](../Chapter3/thread_pools.md) you repeatedly needed the same thing: a queue that multiple threads can push to and pop from safely, blocking until an item is available. You do not want to rewrite that for `int`, then for `std::string`, then for a task type. Write it **once, generically**, with `T` as the element type:

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>

template <typename T>
class ThreadSafeQueue {
public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            queue_.push(std::move(value));     // move the element in (works for any T)
        }
        cv_.notify_one();
    }

    T waitAndPop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.empty();
    }

private:
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    std::queue<T> queue_;
};

int main() {
    ThreadSafeQueue<int> q;
    q.push(1);
    q.push(2);
    std::cout << q.waitAndPop() << "\n";   // 1
    std::cout << q.waitAndPop() << "\n";   // 2
}
```

This single class works for `ThreadSafeQueue<int>`, `ThreadSafeQueue<std::string>`, or `ThreadSafeQueue<Task>` — the [mutex](../Chapter2/sharing_data.md) and [condition variable](../Chapter2/condition_variables.md) logic is written once and reused for every element type. It composes the whole toolkit so far: a [template](templates.md) over `T`, RAII [locks](ownership.md), a condition variable, and [`std::move`](move_semantics.md) to put elements in and take them out without copying. This is exactly the structure a [thread pool](../Chapter3/thread_pools.md) is built on — and now you could write the pool's task queue in one line: `ThreadSafeQueue<std::function<void()>>`.

!!! note "Template code usually lives entirely in headers"
    The compiler must see a template's *full definition* to instantiate it for your type, so class and function templates normally live wholly in a `.hpp` header — there is no separate `.cpp`. This is why so many generic libraries (and the standard library) are header-only. It is also why heavy template use slows compilation: every translation unit that uses the template recompiles it.

---

## Deduction and `auto`

You rarely spell out template arguments — the compiler **deduces** them from the call (`max3(3, 7, 2)` deduces `T = int`). The same machinery powers `auto`, which deduces a variable's type from its initialiser:

```cpp
auto count = readings.size();              // std::size_t, deduced
for (const auto& reading : readings) { }   // element type, deduced — the everyday use
```

`auto` is not a runtime feature; it is compile-time deduction, the same engine templates use. Reach for it when the type is long or obvious (iterators, `size()` results, range-`for` elements), and spell the type out when it aids clarity.

---

## Variadic templates: forwarding "any arguments"

Have you wondered how `std::make_unique`, `std::thread`, and the pool's `enqueue` all accept *any* arguments and pass them straight through to a constructor or function? That is a **variadic template** — a template with a parameter *pack* — combined with **perfect forwarding**:

```cpp
#include <memory>
#include <utility>

template <typename... Args>                       // Args is a pack of zero or more types
auto makeSensor(Args&&... args) {                  // forwarding references to all of them
    return std::make_unique<Sensor>(std::forward<Args>(args)...);   // pass them on, unchanged
}
```

The pieces:

- `typename... Args` declares a **parameter pack** — any number of type arguments.
- `Args&&... args` is a pack of **forwarding references**: each can bind to an lvalue or an rvalue.
- `std::forward<Args>(args)...` **perfectly forwards** each argument — preserving whether it was an lvalue or rvalue (see [Move Semantics](move_semantics.md)) — so it reaches `Sensor`'s constructor exactly as the caller passed it (a move stays a move, a copy stays a copy).

You will mostly *use* this machinery rather than write it — it is what lets `std::thread t(fn, a, b)`, `std::async(fn, a, b)`, and `pool.enqueue(fn, a, b)` accept arbitrary arguments. But recognising the `Args&&...` + `std::forward` pattern means those signatures stop looking like noise.

---

## A glimpse of concepts (C++20)

The classic complaint about templates is their error messages: pass a wrong type to `std::sort` and you get a screenful of jargon, because the failure surfaces deep inside the template. **Concepts** (C++20) let you *constrain* a template up front, so a misuse is rejected with a clear message at the call site:

```cpp
#include <concepts>

template <std::integral T>          // T must be an integer type
T addOne(T x) { return x + 1; }

// addOne(3);     // fine
// addOne(2.5);   // error: 2.5 does not satisfy std::integral — a clear, early message
```

You do not need to write concepts to benefit from them — the standard library increasingly uses them, which is why modern template errors are gradually becoming readable. For your own code, a constraint like `std::integral` or `std::floating_point` documents intent and turns a cryptic failure into a precise one. Treat them as a tool to reach for when a template's requirements are worth stating, not as something every template needs.

---

## When to write your own template

For most course code you *use* templates (every standard container is one) far more than you write them. Writing one is the right call in two situations:

- **A container or component generic over its element type** — the `ThreadSafeQueue<T>` above, a fixed-size ring buffer, a generic object pool.
- **An algorithm that does not care about the element type** — a `clamp`, a `findMax`, a parallel reduction.

The tell is duplication: if you are about to write the same class or function again with only the type changed, that is a template waiting to be written.

---

## Summary

- A **template** is a compile-time blueprint; the compiler generates a concrete, fully type-checked, zero-overhead version for each type used. Templates normally live **entirely in headers**.
- The payoff for this course is **generic concurrency components** — above all a `ThreadSafeQueue<T>` written once and reused for any element type, composing templates, RAII locks, a condition variable, and `std::move`.
- The compiler **deduces** template arguments; `auto` is the same deduction applied to variables — use it for obvious or unwieldy types.
- **Variadic templates** with `Args&&...` + `std::forward` are how `std::thread`, `std::async`, `make_unique`, and `enqueue` accept and forward arbitrary arguments — recognise the pattern even if you rarely write it.
- **Concepts** (C++20) constrain templates for clearer errors and self-documenting requirements; use them when a template's requirements are worth stating.
- Write a template when you would otherwise duplicate code that differs only by type. Next: [Lambdas & std::function](lambdas.md), the callables you hand to all this generic machinery.

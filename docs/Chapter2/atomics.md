# Atomics

A [mutex](sharing_data.md) protects a critical section, but locking and unlocking has a cost, and for the simplest shared data — a single counter, a single flag — it is heavier than necessary. For those cases C++ offers **atomic** types: variables whose individual operations are guaranteed indivisible, so several threads can read and write them safely **without a lock**. This chapter shows what they are good for, and — just as importantly — where they are *not* enough.

---

## The counter, without a lock

Recall the broken counter from [Sharing Data](sharing_data.md): two threads each did `++counter` a million times, and the total came out wrong because `++counter` is really a read-modify-write that the scheduler can interrupt. A mutex fixed it. So does making the counter **atomic**:

```cpp
#include <atomic>
#include <iostream>
#include <thread>

std::atomic<int> counter{0};

void increment() {
    for (int i = 0; i < 1'000'000; ++i) {
        ++counter;             // one indivisible read-modify-write, no lock
    }
}

int main() {
    {
        std::jthread t1(increment);
        std::jthread t2(increment);
    }                          // both threads join here
    std::cout << "counter = " << counter << "\n";   // reliably 2000000
}
```

`std::atomic<int>` (from `<atomic>`) guarantees that `++counter` happens as a single, uninterruptible step: no other thread can observe a half-finished update, and no increment is ever lost. The result is `2000000` every run. Compared with the mutex version this is shorter, and for a lone counter it is faster — there is no lock to acquire, and the increment compiles down to a single hardware instruction on most CPUs.

The atomic types support the operations you would expect — `++`, `--`, `+=`, and the explicit `fetch_add`, `fetch_sub`, `exchange`, `load`, and `store`:

```cpp
counter.fetch_add(5);          // add 5 atomically; same as counter += 5
int now = counter.load();      // read the current value atomically
counter.store(0);              // write atomically
```

`load()` and `store()` make the atomic access explicit; `counter` used directly (as in `++counter` or reading it in `std::cout`) does the same thing implicitly.

---

## What "atomic" actually guarantees

Two threads touching the same atomic variable, with at least one writing, is **not** a data race — atomics are the one kind of shared variable you may access concurrently without further synchronisation. An atomic gives you two guarantees:

1. **Indivisibility.** Each operation completes as a whole. No thread ever sees a "torn" value — half-written, half-old.
2. **Visibility and ordering.** A write by one thread becomes visible to others in a well-defined order. This is the subtle part, and it is why a plain variable cannot stand in for an atomic even for something as simple as a flag.

### Why a plain `bool` flag is broken

It is tempting to use an ordinary `bool` to tell a worker thread to stop:

<!-- no-ce -->
```cpp
bool running = true;           // plain bool — WRONG for cross-thread signalling

void worker() {
    while (running) {          // the optimiser may read `running` once, then loop forever
        // ... work ...
    }
}
```

This can hang forever. From the worker thread's point of view, nothing *inside* the loop changes `running`, so the compiler is entitled to load it **once** into a register and never look at memory again — turning `while (running)` into `while (true)`. There is also no guarantee that another thread's write to `running` ever becomes visible to this one. Both problems vanish with an atomic:

```cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>

std::atomic<bool> running{true};

int main() {
    std::jthread worker([]{
        while (running) {                 // a real atomic load every iteration
            // ... work ...
        }
        std::cout << "worker stopped\n";
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    running = false;                      // atomic store; the worker sees it promptly
}
```

`std::atomic<bool>` forces a genuine memory read each time and establishes the ordering that makes one thread's write visible to the other. This is the manual equivalent of the `std::stop_token` you saw in [Creating Threads](threads.md) — and `stop_token` is usually the cleaner choice for a `jthread`, because it also integrates with `request_stop()`. Reach for `std::atomic<bool>` when you need a stop/ready flag outside a `jthread`'s cancellation machinery.

!!! note "`atomic_flag` and spinlocks"
    `std::atomic_flag` is the simplest atomic — just `test_and_set()` and `clear()`. It is the primitive from which a *spinlock* is built. You will rarely use it directly: a spinlock busy-waits (the very thing [condition variables](condition_variables.md) avoid), so it is appropriate only for locks held for a handful of instructions. Know it exists; reach for it almost never.

---

## A glimpse of the memory model

You may see atomic operations written with an explicit **memory order**:

```cpp
counter.fetch_add(1, std::memory_order_relaxed);
```

C++ has a detailed *memory model* that lets experts relax the ordering guarantees for extra performance on weakly-ordered hardware. The default — used whenever you omit the argument — is `std::memory_order_seq_cst` (sequentially consistent), the **strongest and most intuitive** ordering: atomic operations appear to happen in a single global order that all threads agree on.

!!! warning "Use the default ordering"
    For this course, **always use the default** (omit the memory-order argument). Relaxed and acquire/release orderings are a genuine expert topic where small mistakes produce bugs that are essentially impossible to reproduce. If you ever need them, study the memory-model chapters of *C++ Concurrency in Action* (Anthony Williams) first — that book exists largely to explain this one topic properly. Reaching for `relaxed` to "go faster" without that grounding is how subtle, unreproducible bugs are born.

You can ask whether a given atomic is truly lock-free (implemented with a hardware instruction rather than a hidden lock):

```cpp
std::atomic<int> a;
std::cout << a.is_lock_free() << "\n";    // almost always 1 (true) for int
```

Integral and pointer atomics are lock-free on every platform you will use. A large type wrapped in `std::atomic` (say `std::atomic<BigStruct>`) may instead use an internal lock — at which point you have a mutex anyway, just a less visible one.

---

## Atomics or a mutex?

Atomics are not a general replacement for mutexes. They protect a **single variable's individual operations** — they cannot protect an *invariant that spans several variables or several steps*.

The trap is **check-then-act**. Even with atomic operations, a sequence of them is not itself atomic:

<!-- no-ce -->
```cpp
std::atomic<int> size{...};
// Two threads can BOTH pass the check before either pops — still a race.
if (size > 0) {        // atomic read
    pop();             // ... but another thread may have popped in between
    --size;            // atomic write
}
```

Each atomic step is safe, yet the *combination* is not: two threads can both see `size > 0` and both pop. Protecting a compound operation like this needs a [mutex](sharing_data.md) around the whole sequence. The dividing line:

| Use an **atomic** when… | Use a **mutex** when… |
|--------------------------|------------------------|
| The shared state is a single variable (counter, flag, pointer). | An invariant spans several variables or a multi-step update. |
| Each operation stands alone (increment, set a flag, swap). | You must "check then act" as one unit. |
| The type is small and trivially copyable. | The data is a container, string, or other non-trivial object. |
| You want maximum throughput on a hot, simple value. | Correctness of a compound operation matters more than the lock's cost. |

A good rule of thumb: **a lone counter or flag → atomic; anything with an invariant → mutex** (and to *wait* for a condition, a [condition variable](condition_variables.md)).

---

## Summary

- `std::atomic<T>` makes a variable's individual operations **indivisible and lock-free**, fixing data races on simple shared values (a counter, a flag) without the cost of a mutex.
- Atomics give two guarantees: **no torn values**, and **visibility/ordering** of writes across threads. The second is why a plain `bool` flag is broken — the optimiser may cache it and loop forever, and a write may never become visible.
- `std::atomic<bool>` is the manual stop/ready flag; for a `jthread`, prefer the integrated [`std::stop_token`](threads.md). `std::atomic_flag` underlies spinlocks — rarely your tool.
- C++ has a rich **memory model**, but **use the default `seq_cst` ordering**; relaxed orderings are an expert topic — see Williams' *C++ Concurrency in Action*.
- Atomics protect *single operations*, not multi-step invariants: **check-then-act still races**. Lone counter or flag → atomic; invariant across variables → mutex; waiting on a condition → condition variable.

That completes the fundamentals of sharing state safely. [Part 3](../Chapter3/futures.md) shifts from *protecting* shared data to *structuring* concurrent work — futures, thread pools, and meeting real-time deadlines.

# Parallel Algorithms

A [thread pool](thread_pools.md) is the right tool when you have a stream of distinct tasks. But a huge fraction of real work is simpler than that: *do the same operation to every element of a big collection* — sum a million readings, transform an image, sort a vector. For this, C++17 lets you parallelise the standard algorithms you already know by adding one argument: an **execution policy**. No threads, no queue, no locks in your code — the library does it for you.

---

## One extra argument

Most algorithms in `<algorithm>` and `<numeric>` gained overloads that take an execution policy (from `<execution>`) as their first argument. The algorithm is otherwise identical:

<!-- no-ce -->
```cpp
#include <algorithm>
#include <execution>
#include <vector>

std::vector<int> data = /* ... a large vector ... */;

// Sequential — the version you already know
std::sort(data.begin(), data.end());

// Parallel — same sort, spread across cores
std::sort(std::execution::par, data.begin(), data.end());
```

That is the entire change. The second call may run on several cores at once; the library manages the threads on an internal pool you never see.

### The three policies

| Policy | Meaning |
|--------|---------|
| `std::execution::seq` | Sequential — like the no-policy version, but explicit. |
| `std::execution::par` | **Par**allel — work may run on multiple threads. |
| `std::execution::par_unseq` | Parallel **and** vectorised — may use multiple threads *and* interleave/SIMD-vectorise calls within a thread. |

`par` is the one you reach for. `par_unseq` can be faster still but imposes stricter rules on your code (below). `seq` exists so you can switch a policy with a variable without special-casing the no-argument form.

---

## Reductions: `reduce`, not `accumulate`

Summing a range is the classic parallel win, but there is a catch you must understand. The familiar `std::accumulate` is **strictly sequential and ordered** — it folds left to right, element by element, and has no parallel overload. Its parallel-friendly cousin is `std::reduce`:

<!-- no-ce -->
```cpp
#include <execution>
#include <numeric>
#include <vector>

std::vector<double> v = /* ... a million values ... */;

double total = std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
```

`std::reduce` is allowed to combine elements in **any order and any grouping**, which is exactly what lets it split the range across cores and add the partial sums. The price is a requirement on the operation: it must be **associative and commutative**, because the order is not guaranteed.

!!! warning "Floating-point addition is not truly associative"
    Because rounding makes `(a + b) + c` slightly different from `a + (b + c)`, a parallel `reduce` over `double`s can give a result that differs in the last few bits from a sequential `accumulate`, and can vary run to run. For sums of sensor data this is almost always fine — but if you need a *bit-for-bit reproducible* total, use the ordered `accumulate`. Know the trade-off you are making.

`std::transform_reduce` fuses a transform with a reduce — ideal for things like a parallel dot product or "sum of squares":

<!-- no-ce -->
```cpp
// sum of squares, in parallel
double sumSq = std::transform_reduce(
    std::execution::par, v.begin(), v.end(), 0.0,
    std::plus<>{},                          // how to combine
    [](double x) { return x * x; });        // how to transform each element
```

---

## Your callable must be safe to parallelise

When you pass `par`, **you** become responsible for avoiding data races — the same rule as writing threads by hand ([Sharing Data](../Chapter2/sharing_data.md)). The element function the algorithm calls will run on several threads at once, so it must not touch shared mutable state without synchronisation:

<!-- no-ce -->
```cpp
// BROKEN: every parallel invocation writes the same `sum` → data race
double sum = 0.0;
std::for_each(std::execution::par, v.begin(), v.end(),
              [&](double x) { sum += x; });        // UB

// CORRECT: use the algorithm built for reduction
double sum = std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
```

`par_unseq` is stricter still: because calls may be *interleaved within a single thread* (vectorised), the callable may **not** acquire a mutex, allocate memory, or do anything that needs to complete atomically with respect to another call. Reserve `par_unseq` for simple, self-contained element operations.

!!! danger "An escaping exception calls `std::terminate`"
    If an element function throws under a parallel policy and the exception would escape the algorithm, the program **terminates** — exceptions are not propagated out of parallel algorithms the way they are out of [futures](futures.md). Keep parallel callables non-throwing, or handle errors inside them.

---

## The practical catch: linking on GCC and Clang

This is the detail that wastes an afternoon if no one warns you. On **MSVC** the parallel policies work out of the box. On **GCC and Clang**, the parallel algorithms are implemented on top of **Intel TBB**, which you must install and link — otherwise `std::execution::par` silently compiles as sequential, or fails to link.

In CMake:

```cmake
find_package(TBB REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE TBB::tbb)
```

Install TBB through your package manager or [vcpkg](../Chapter6/dependencies.md) (`vcpkg install tbb`). Without it, that beautiful `std::execution::par` may do nothing in parallel at all — which is also why these examples carry no "Run on Compiler Explorer" link: the default online compiler is not set up with TBB.

!!! note "Parallelism has overhead — measure"
    Splitting work across threads costs coordination. For a *small* range, the sequential version wins outright; the parallel one is only faster once the data is large enough to dwarf the overhead. Never assume `par` is faster — benchmark with realistic data, and remember an implementation is always free to fall back to sequential if it judges parallelism not worthwhile.

---

## When to use them

Parallel algorithms are the **first** thing to reach for when the work is "the same operation over a large collection", because they give you parallelism with almost no new code and no concurrency bugs of your own. Use a [thread pool](thread_pools.md) instead when the tasks are *heterogeneous* (different jobs, not one operation over a range), and hand-written [threads](../Chapter2/threads.md) only when you need precise control over a small number of long-lived activities.

---

## Summary

- C++17 adds **execution policies** to the standard algorithms: pass `std::execution::par` as the first argument and the algorithm runs across cores, with the library managing the threads.
- `seq` (sequential), `par` (parallel), `par_unseq` (parallel + vectorised). `par` is the everyday choice; `par_unseq` is faster but forbids locks/allocation in the callable.
- Use **`std::reduce`** / `std::transform_reduce` for parallel sums, not the strictly-sequential `std::accumulate`; the operation must be **associative and commutative**, and floating-point sums may differ slightly run to run.
- Under a parallel policy **you** must avoid data races in the callable, and an escaping **exception calls `std::terminate`**.
- On **GCC/Clang you must link Intel TBB** (`TBB::tbb`) or the parallel policies do nothing; MSVC works out of the box. Parallelism has overhead — **measure** on realistic data.
- Reach for parallel algorithms first for "one operation over a big range"; use a [pool](thread_pools.md) for heterogeneous tasks.

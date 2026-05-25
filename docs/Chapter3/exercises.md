# Chapter 3 Exercises

Work through these after reading Chapter 3. **Try each one yourself before revealing the solution.** Type the code into CLion and run it; for the timing exercise especially, seeing the numbers is the point.

When you open a solution it appears **blurred** — click it once more to reveal it.

Remember to link the thread library on Linux and the Pi (`target_link_libraries(app PRIVATE Threads::Threads)`, see [Getting Started](../getting_started.md)). The parallel-algorithm exercise additionally needs Intel TBB — noted where it applies.

---

## 1. Two halves at once

*Practises: [Futures & Promises](futures.md)*

Sum a large vector by splitting the work: compute one half with `std::async` while the calling thread computes the other, then add the two partial sums. Fill a vector with `1`s so you know the answer is its size.

> Hint: `std::async(std::launch::async, fn, args...)` returns a `std::future`. Pass the vector by `std::cref` — `async` copies its arguments, and you do not want to copy a big vector (the vector outlives the future because you `get()` before `main` ends). Do the second half on the current thread, then add `future.get()`.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <future>
    #include <iostream>
    #include <vector>

    long long sumRange(const std::vector<int>& v, std::size_t begin, std::size_t end) {
        long long sum = 0;
        for (std::size_t i = begin; i < end; ++i) {
            sum += v[i];
        }
        return sum;
    }

    int main() {
        std::vector<int> data(1000, 1);
        std::size_t mid = data.size() / 2;

        // First half runs on another thread; second half runs here, concurrently.
        std::future<long long> first =
            std::async(std::launch::async, sumRange, std::cref(data), 0, mid);
        long long second = sumRange(data, mid, data.size());

        long long total = first.get() + second;     // get() waits for the async half
        std::cout << "total = " << total << "\n";    // 1000
    }
    ```

    The two halves are summed in parallel: one on the async thread, one on `main`'s thread. `get()` blocks only at the moment we need the first half's result — by then `main` has already finished the second. Passing `std::cref(data)` shares the vector instead of copying it (the [argument-copying rule](../Chapter2/threads.md) from Creating Threads); it is safe because `data` outlives the future. Note there is no manual thread, no `join`, and no mutex — the future *is* the synchronisation.

    </div>

---

## 2. Fan out, collect futures

*Practises: [Futures & Promises](futures.md), [Thread Pools](thread_pools.md)*

Launch eight independent computations concurrently and collect their results in order. Store each `std::future` in a `std::vector`, then loop and `get()` them. (This is the fan-out a [thread pool](thread_pools.md) does for you — here you do it by hand with `std::async`.)

> Hint: a `std::future` is move-only, so build a `std::vector<std::future<int>>` and `push_back` each one from `std::async`. Getting the results in a second loop keeps all eight running concurrently first.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <future>
    #include <iostream>
    #include <vector>

    int main() {
        std::vector<std::future<int>> futures;
        for (int i = 1; i <= 8; ++i) {
            futures.push_back(std::async(std::launch::async, [i] { return i * i; }));
        }

        for (auto& f : futures) {
            std::cout << f.get() << " ";     // 1 4 9 16 25 36 49 64
        }
        std::cout << "\n";
    }
    ```

    All eight tasks are launched *before* any `get()`, so they run concurrently rather than one-at-a-time. Collecting the futures in a vector and draining them in order gives results in submission order, even though the tasks may finish in any order. Swapping `std::async` for `pool.enqueue` from the [Thread Pools](thread_pools.md) chapter would give the same shape with bounded thread use — which is what you would do for thousands of tasks instead of eight.

    </div>

---

## 3. A loop that holds its rate

*Practises: [Real-Time & Timing](real_time.md)*

Write a loop that runs five cycles at a fixed **50 ms** period using `sleep_until` against a rolling deadline, and print the measured time since the previous cycle so you can see it stays on rate. Then change it to the naïve `sleep_for(50ms)` and add a `20ms` "work" sleep inside the loop — watch the period drift to ~70 ms.

> Hint: keep a `next` deadline that you advance by the period each cycle, and `sleep_until(next)`. Measure with `steady_clock` (monotonic — the right clock for intervals).

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <chrono>
    #include <iostream>
    #include <thread>

    int main() {
        using namespace std::chrono;
        const auto period = 50ms;

        auto next = steady_clock::now() + period;
        auto last = steady_clock::now();

        for (int cycle = 0; cycle < 5; ++cycle) {
            std::this_thread::sleep_until(next);     // wait for the absolute deadline

            auto now = steady_clock::now();
            auto delta = duration_cast<milliseconds>(now - last).count();
            std::cout << "cycle " << cycle << ": " << delta << " ms since last\n";

            last = now;
            next += period;                          // advance the deadline, no drift
        }
    }
    ```

    Each line prints ≈ 50 ms because `next` advances by exactly one period regardless of how long the cycle's work took. Now contrast the broken version:

    ```cpp
    // Drifts: real period = work time + sleep time, and it accumulates.
    while (cycle < 5) {
        std::this_thread::sleep_for(20ms);   // pretend the work takes 20 ms
        std::this_thread::sleep_for(period); // then sleep the period
        // ... measured interval is ~70 ms, not 50 ms ...
    }
    ```

    The naïve loop sleeps the *period* on top of the *work*, so it runs at ~70 ms and falls further behind the longer it runs. Scheduling against absolute deadlines is what keeps a control loop locked to its intended rate — the core real-time pattern.

    </div>

---

## 4. Sum a million in parallel

*Practises: [Parallel Algorithms](parallel_algorithms.md)*

Sum a large vector with `std::reduce` and the `std::execution::par` policy. Fill it with `1`s so the answer is its size. (On GCC/Clang you must link Intel TBB — see below — or the policy does nothing.)

> Hint: `std::reduce(std::execution::par, begin, end, init)` from `<numeric>` and `<execution>`. Use `std::reduce`, **not** `std::accumulate` — only `reduce` parallelises, because it may combine elements in any order.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <execution>
    #include <iostream>
    #include <numeric>
    #include <vector>

    int main() {
        std::vector<long long> data(10'000'000, 1);

        long long total = std::reduce(std::execution::par,
                                      data.begin(), data.end(), 0LL);

        std::cout << "total = " << total << "\n";   // 10000000
    }
    ```

    `std::reduce` is allowed to add the elements in any grouping, which is what lets it split the range across cores; `std::accumulate` is strictly ordered and has no parallel overload. To build this on GCC or Clang, link TBB:

    ```cmake
    find_package(TBB REQUIRED)
    target_link_libraries(app PRIVATE TBB::tbb Threads::Threads)
    ```

    Without TBB the code may compile but run sequentially. Try timing `par` against `seq` with [`steady_clock`](real_time.md): the parallel version wins only once the data is large enough to outweigh the coordination overhead — on ten million elements it should, on a hundred it will not.

    </div>

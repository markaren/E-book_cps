# Chapter 2 Exercises

Work through these after reading Chapter 2. **Try each one yourself before revealing the solution** — you learn far more from an honest attempt than from reading a finished answer. Type the code into CLion and run it; concurrency especially rewards seeing things actually happen (and occasionally fail).

When you open a solution it appears **blurred** — click it once more to reveal it, so you do not see the answer by accident.

Each program has its own `main()`. Remember to link the thread library on Linux and the Pi (`target_link_libraries(app PRIVATE Threads::Threads)`, see [Getting Started](../getting_started.md)).

---

## 1. Join, or don't crash

*Practises: [Processes & Threads](processes_threads.md), [Creating Threads](threads.md)*

This program crashes — it starts a thread and never joins it, so the `std::thread` destructor calls `std::terminate()`:

<!-- no-ce -->
```cpp
#include <iostream>
#include <thread>

int main() {
    std::thread worker([] {
        std::cout << "work done\n";
    });
    std::cout << "main done\n";
}   // worker is destroyed un-joined → std::terminate()
```

Fix it **two** ways: first by joining explicitly, then by switching to `std::jthread` so the join is automatic. Convince yourself both print two lines and exit cleanly.

> Hint: a `std::thread` must be joined or detached before its destructor runs. `std::jthread` (C++20) joins itself in its own destructor, so you simply delete the problem.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    **Explicit join:**

    ```cpp
    #include <iostream>
    #include <thread>

    int main() {
        std::thread worker([] {
            std::cout << "work done\n";
        });
        std::cout << "main done\n";
        worker.join();           // wait for worker before main returns
    }
    ```

    **`std::jthread` — the join is automatic:**

    ```cpp
    #include <iostream>
    #include <thread>

    int main() {
        std::jthread worker([] {
            std::cout << "work done\n";
        });
        std::cout << "main done\n";
    }   // worker.join() happens here, in jthread's destructor
    ```

    Both print `main done` and `work done` (in either order). The lesson: a bare `std::thread` makes you responsible for the join, and forgetting is fatal; `std::jthread` ties the join to scope via RAII, exactly like a `lock_guard` ties an unlock to scope. Prefer `std::jthread` unless you have a specific reason not to.

    </div>

---

## 2. A thread-safe log

*Practises: [Sharing Data](sharing_data.md)*

Build a class `Log` that wraps a `std::vector<std::string>`. Give it `add(std::string)` and a `size()` getter, both safe to call from many threads at once. In `main`, launch **four** threads that each call `add` a thousand times, join them, and print the size — it must be exactly `4000` every run.

> Hint: keep a `std::mutex` *inside* the class next to the vector, and take a `std::lock_guard` at the top of every method that touches it. Make the mutex `mutable` so the `const` `size()` can still lock it. Capture the `Log` by reference in the thread lambdas (`[&log]`).

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <mutex>
    #include <string>
    #include <thread>
    #include <vector>

    class Log {
    public:
        void add(std::string message) {
            std::lock_guard<std::mutex> lock(mutex_);
            messages_.push_back(std::move(message));
        }

        std::size_t size() const {
            std::lock_guard<std::mutex> lock(mutex_);
            return messages_.size();
        }

    private:
        mutable std::mutex mutex_;
        std::vector<std::string> messages_;
    };

    int main() {
        Log log;
        std::vector<std::jthread> threads;
        for (int t = 0; t < 4; ++t) {
            threads.emplace_back([&log] {
                for (int i = 0; i < 1000; ++i) {
                    log.add("event");
                }
            });
        }
        threads.clear();                 // joins all jthreads (destructors run)
        std::cout << "size = " << log.size() << "\n";   // 4000
    }
    ```

    Without the mutex, four threads calling `push_back` on the same vector at once is a data race — it would corrupt the vector's internal state, not merely miscount, and likely crash. The mutex makes each `add` a critical section. Note the synchronisation lives *inside* `Log`: callers cannot forget to lock, because they never touch the vector directly. `threads.clear()` destroys the `jthread`s, and each destructor joins — so the count is taken only after every thread has finished.

    </div>

---

## 3. Parallel sum — avoid the shared accumulator

*Practises: [Processes & Threads](processes_threads.md), [Sharing Data](sharing_data.md)*

Sum a large `std::vector<long long>` using several threads. The naïve approach gives every thread a shared `total` guarded by a mutex — but then the threads spend all their time queueing for the lock and the "parallel" version is *slower* than a single loop. Instead, give **each thread its own partial sum** over a slice of the vector, with no shared state at all, and add the partials together at the end.

Fill a vector with `1`s (so you know the answer is its size), split it across four threads, and print the total.

> Hint: the fastest synchronisation is *no* synchronisation. If threads never touch shared data, there is nothing to protect. Give each thread an index range and a slot in a `std::vector<long long> partials(4)` to write its own result into — different slots are different objects, so that is not a race. Sum the partials in `main` after joining.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <thread>
    #include <vector>

    int main() {
        const std::size_t n = 8'000'000;
        std::vector<long long> data(n, 1);     // sum should be n

        const int numThreads = 4;
        std::vector<long long> partials(numThreads, 0);
        std::vector<std::jthread> threads;

        const std::size_t chunk = n / numThreads;
        for (int t = 0; t < numThreads; ++t) {
            std::size_t begin = t * chunk;
            std::size_t end = (t == numThreads - 1) ? n : begin + chunk;
            threads.emplace_back([&data, &partials, t, begin, end] {
                long long sum = 0;                  // thread-local, no sharing
                for (std::size_t i = begin; i < end; ++i) {
                    sum += data[i];
                }
                partials[t] = sum;                  // its own slot, not a race
            });
        }
        threads.clear();                            // join all

        long long total = 0;
        for (long long p : partials) {
            total += p;
        }
        std::cout << "total = " << total << "\n";   // 8000000
    }
    ```

    Each thread sums its slice into a **local** variable, touching no shared memory during the hot loop — so there is no lock and no contention, and the work genuinely runs in parallel. Writing each thread's result into its *own* element of `partials` is safe because different elements are distinct objects; only `main` reads them all, and only after every thread has joined. This is the **avoid shared state** strategy: the best way to win a race is not to enter one. (A mutex-protected shared `total` would be correct but could run slower than a single-threaded loop, because every `+=` would serialise on the lock.)

    </div>

---

## 4. Break the deadlock

*Practises: [Sharing Data](sharing_data.md)*

The following "transfer" code can deadlock: two threads lock the two account mutexes in opposite orders, and on an unlucky interleaving each waits forever for the lock the other holds.

<!-- no-ce -->
```cpp
std::mutex mutexA;
std::mutex mutexB;

void aToB() {
    std::lock_guard<std::mutex> first(mutexA);
    std::lock_guard<std::mutex> second(mutexB);
    // ... move funds A → B ...
}

void bToA() {
    std::lock_guard<std::mutex> first(mutexB);
    std::lock_guard<std::mutex> second(mutexA);
    // ... move funds B → A ...
}
```

Rewrite it so it cannot deadlock, no matter how the threads interleave. Then run two threads hammering `aToB` and `bToA` in a loop and confirm the program always finishes.

> Hint: the deadlock exists because the two functions acquire the locks in *different* orders. Either impose one consistent order everywhere, or — cleaner — acquire both mutexes in a single step with `std::scoped_lock`, which is built to do this without deadlocking regardless of argument order.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <mutex>
    #include <thread>

    std::mutex mutexA;
    std::mutex mutexB;

    void aToB() {
        std::scoped_lock lock(mutexA, mutexB);   // both at once, deadlock-free
        // ... move funds A → B ...
    }

    void bToA() {
        std::scoped_lock lock(mutexB, mutexA);   // order here no longer matters
        // ... move funds B → A ...
    }

    int main() {
        std::jthread t1([] { for (int i = 0; i < 100'000; ++i) aToB(); });
        std::jthread t2([] { for (int i = 0; i < 100'000; ++i) bToA(); });
        // both threads join here; the program always reaches this point
        std::cout << "no deadlock\n";
    }
    ```

    `std::scoped_lock` locks all the mutexes you give it as one atomic operation, using an algorithm that never produces the circular wait that causes deadlock — so `aToB` and `bToA` can list the mutexes in whatever order reads naturally and still cooperate. The alternative fix is to make *both* functions lock `mutexA` before `mutexB`; that also works, but relies on every author remembering the convention, while `scoped_lock` makes the safety structural. Either way, you have broken the **circular wait** condition from the chapter.

    </div>

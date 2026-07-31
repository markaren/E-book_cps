# Condition Variables

A [mutex](sharing_data.md) lets one thread into a critical section at a time — it solves *mutual exclusion*. But threads often need something different: to **wait until something becomes true**. A consumer waits until there is data to consume; a worker waits until a job arrives. Mutexes alone cannot express "sleep until notified", and the obvious workaround — checking in a loop — wastes a whole CPU core. The tool for waiting efficiently is the **condition variable**.

---

## The problem with busy-waiting

Suppose one thread produces values and another consumes them from a shared queue. A first attempt has the consumer spin in a loop, repeatedly locking the mutex to check whether the queue has anything:

<!-- no-ce -->
```cpp
// Anti-pattern: busy-waiting ("spinning")
while (true) {
    std::unique_lock<std::mutex> lock(mtx);
    if (!queue.empty()) {
        auto value = queue.front();
        queue.pop();
        lock.unlock();
        process(value);
    }
    // else: loop again immediately, burning CPU
}
```

This *works*, but it is wasteful: when the queue is empty, the consumer thread spins as fast as the CPU allows, doing nothing useful, pinning a core at 100% and starving other threads of it. On a battery-powered robot it also drains power for no benefit. (This is the "busy wait" the [Processes & Threads](processes_threads.md) chapter warned about.) What we want is for the consumer to **sleep** while the queue is empty and be **woken** the instant something arrives. That is exactly what a condition variable provides.

---

## `std::condition_variable`

A condition variable is a signalling mechanism, always used together with a mutex. Three operations matter:

| Operation | Who calls it | Effect |
|-----------|--------------|--------|
| `wait(lock, predicate)` | the waiting thread | Atomically releases the lock and sleeps until the predicate is true (the predicate is re-checked on every wakeup, spurious or not); re-acquires the lock before returning. |
| `notify_one()` | the signalling thread | Wakes **one** waiting thread. |
| `notify_all()` | the signalling thread | Wakes **all** waiting threads. |

The waiting side must hold a `std::unique_lock` (not a `lock_guard`), because `wait` needs to *unlock* the mutex while it sleeps and *re-lock* it on waking — and `unique_lock` is the flexible lock that allows that.

The magic of `wait` is that releasing the lock and going to sleep happen **atomically**: there is no gap in which a notification could slip past unnoticed. While the consumer sleeps, the mutex is free, so the producer can lock it, add data, and notify.

---

## Spurious wakeups: always re-check the condition

A condition variable is allowed to wake a thread **even when no one notified it** — a *spurious wakeup*. The C++ standard explicitly permits this, because forbidding it would make condition variables slower on real hardware. The consequence is a rule you must never break:

!!! danger "Never wait without re-checking the condition"
    A bare `wait` can return when the thing you are waiting for is *not* actually ready:

    ```cpp
    // WRONG — a spurious wakeup returns here with the queue still empty
    if (queue.empty()) {
        cv.wait(lock);
    }
    auto value = queue.front();   // queue might still be empty → undefined behaviour
    ```

    Always use the **predicate overload**, which re-checks for you and goes back to sleep on a false alarm:

    ```cpp
    // RIGHT — loops internally until the predicate is true
    cv.wait(lock, [&]{ return !queue.empty(); });
    auto value = queue.front();   // guaranteed non-empty
    ```

`cv.wait(lock, pred)` is exactly equivalent to `while (!pred()) cv.wait(lock);`. It handles spurious wakeups, *and* the case where the notification arrived before you started waiting (the predicate is already true, so you never sleep). Writing the predicate form is not optional defensive style — it is the only correct way to use a condition variable.

---

## Producer/consumer: the canonical example

Here is the busy-wait version done properly. A producer pushes five values and then signals that it is finished; a consumer sleeps until there is work, processes it, and exits cleanly once the producer is done and the queue is drained.

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;
bool done = false;

void producer() {
    for (int i = 1; i <= 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            queue.push(i);
        }
        cv.notify_one();          // wake the consumer: there is work
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;              // no more items will come
    }
    cv.notify_one();              // wake the consumer so it can finish
}

void consumer() {
    while (true) {
        int value;
        {
            std::unique_lock<std::mutex> lock(mtx);
            cv.wait(lock, []{ return !queue.empty() || done; });
            if (queue.empty() && done) {
                break;            // woken, nothing left, producer finished
            }
            value = queue.front();
            queue.pop();
        }                         // release the lock before the slow part
        std::cout << "consumed " << value << "\n";
    }
}

int main() {
    std::jthread c(consumer);
    std::jthread p(producer);
}                                 // jthreads join here
```

Walk through the cooperation:

- The consumer's predicate is `!queue.empty() || done` — wake up if there is work *or* if the producer has signalled it is finished. Both conditions must be in the predicate, or the consumer could sleep forever after the last item.
- The producer **notifies after unlocking** (the `notify_one()` is outside the `{ }` that holds the lock). Notifying while still holding the lock is not wrong, but it is slightly wasteful: the woken consumer would immediately block trying to acquire the mutex the producer still holds.
- The consumer takes **one item under the lock**, releases it, and only then does the slow work (`std::cout`). This keeps the critical section short — the discipline from [Sharing Data](sharing_data.md).
- The `done` flag plus a final `notify_one()` is how the producer says "no more data": without it, a consumer that has drained the queue would wait forever for an item that never comes.

This producer/consumer structure is the backbone of a [thread pool](../Chapter3/thread_pools.md), where worker threads wait on a condition variable for tasks to appear in a shared queue.

---

## `notify_one` vs `notify_all`

- **`notify_one()`** wakes a single waiting thread. Use it when any one waiter can handle the event — like the single consumer above, or one idle worker picking up one new task.
- **`notify_all()`** wakes every waiter. Use it when the state change is relevant to all of them — for example, a `done` flag that should release *every* worker so they can all exit, or a configuration change all threads must observe.

When in doubt, `notify_all()` is the safe choice: waking too many threads only costs a little time (the extra ones re-check their predicate and go back to sleep), whereas `notify_one()` when several waiters needed waking can leave threads stuck asleep.

!!! tip "Waiting with a timeout"
    `wait_for(lock, duration, predicate)` and `wait_until(lock, time_point, predicate)` wait only up to a deadline, then return whether the predicate became true. Use them when a thread must not block forever — a sensor read that should give up after 100 ms, say. The durations come from [Time with std::chrono](../Chapter1/chrono.md).

---

## A lighter signal: semaphores (C++20)

A condition variable always rides on a mutex and a piece of shared state you check. Sometimes all you need is to **count permits** — "how many of this resource are available" — and for that C++20 adds `std::counting_semaphore` and `std::binary_semaphore`.

A semaphore holds a count. `acquire()` waits until the count is positive and then decrements it; `release()` increments it and wakes a waiter. A `binary_semaphore` is just a `counting_semaphore<1>` — a one-token signal.

```cpp
#include <iostream>
#include <semaphore>
#include <thread>

std::counting_semaphore<3> slots(3);   // at most 3 threads in the section at once

void worker(int id) {
    slots.acquire();                    // take a slot (waits if all 3 are taken)
    std::cout << "worker " << id << " working\n";
    slots.release();                    // give the slot back
}

int main() {
    std::jthread a(worker, 1), b(worker, 2), c(worker, 3), d(worker, 4);
}
```

Reach for a semaphore when the problem is naturally "**N permits**" — limiting how many threads hit a resource at once, or a simple one-shot "go" signal between two threads. Reach for a **condition variable** when waiting depends on an arbitrary predicate over your own data (like "the queue is non-empty *or* we're done"). The condition variable is the more general tool; the semaphore is the more convenient one when it fits.

---

## A reusable thread-safe queue

The loose `mutex` + `condition_variable` + `queue` + `done` flag of the last example works, but it scatters four globals across your program and trusts every caller to lock correctly. The robust move — the same "wrap the data, not just the code" idea from [Sharing Data](sharing_data.md) — is to seal all four inside one class that exposes only safe operations. The result is a component you will use *directly* in the semester project.

Almost every cyber-physical program in this course has the same shape: a **physics/control loop** on one thread and a **communications thread** talking to the outside world, handing work to each other. There are two patterns for that hand-off, and the whole project is built on them. This section builds the first; the next builds the second.

```cpp
#include <condition_variable>
#include <mutex>
#include <optional>
#include <queue>
#include <utility>

template <typename T>
class ThreadSafeQueue {
public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();               // wake one waiting consumer
    }

    // Blocks until an item is available, or until the queue is closed and empty.
    // Returns std::nullopt only in that "closed and drained" case, so a consumer
    // loop can end cleanly instead of waiting forever for data that will never come.
    std::optional<T> waitAndPop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
        if (queue_.empty()) {           // reachable only once closed_ is true
            return std::nullopt;
        }
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    // Signal that no more items will arrive. Wakes every waiter so each drains
    // what is left and then sees nullopt and stops.
    void close() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            closed_ = true;
        }
        cv_.notify_all();
    }

private:
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    std::queue<T> queue_;
    bool closed_ = false;
};
```

Every design decision here has already been justified in this chapter:

- **The data and its synchronisation are one object.** The `mutex_`, `cv_`, `queue_`, and `closed_` flag are private; the only way to touch the queue is `push`, `waitAndPop`, `close`. A caller *cannot* forget to lock, because it never sees the mutex. This is the encapsulation invariant, applied to a concurrent data structure.
- **`waitAndPop` uses the predicate form** — `!queue_.empty() || closed_` — so it is immune to spurious wakeups *and* it wakes to shut down when `closed_` is set. It is the consumer's predicate from the producer/consumer example, now hidden inside the class.
- **`close()` replaces the loose `done` flag** and is the clean-shutdown mechanism. It calls `notify_all()` so *every* blocked consumer wakes; each drains any remaining items and then, finding the queue empty and closed, gets `std::nullopt`.
- **`std::optional<T>` is the shutdown signal.** A value means "here is your item"; `nullopt` means "the queue is closed and empty — stop". This lets a consumer loop terminate without a separate flag to check, exceptions, or a magic sentinel value.

A consumer then reads like this, and exits by itself when the producer calls `close()`:

```cpp
ThreadSafeQueue<Command> commands;

std::jthread worker([&commands] {
    while (auto cmd = commands.waitAndPop()) {   // nullopt ends the loop
        process(*cmd);
    }
    // fell out of the loop → queue closed and drained → clean exit
});

// ... elsewhere, the producer ...
commands.push(readCommand());
commands.close();   // no more commands; worker finishes what is queued, then stops
```

This is the recipe behind the [thread pool](../Chapter3/thread_pools.md) in Part 3 — the pool inlines the same mutex + condition variable + queue so you can watch every moving part, rather than reusing this class — and it is the comms-to-control hand-off in the project: the communications thread `push`es incoming commands, the control thread `waitAndPop`s them, and shutdown is one `close()` call.

### A latest-value mailbox

The queue keeps *every* item, in order — right when you must not miss a command. But for a stream of **sensor samples** you usually want the opposite: only the **newest** value matters, and an older reading is not worth processing. A queue would pile up stale samples faster than you consume them. The pattern here is a single-slot **mailbox** — "newest sample wins", each write overwrites the last:

```cpp
template <typename T>
class Mailbox {
public:
    void set(T value) {                 // producer: overwrite with the newest sample
        std::lock_guard<std::mutex> lock(mutex_);
        value_ = std::move(value);
        hasValue_ = true;
    }

    std::optional<T> latest() const {   // consumer: read whatever is newest (or nullopt)
        std::lock_guard<std::mutex> lock(mutex_);
        if (!hasValue_) {
            return std::nullopt;
        }
        return value_;
    }

private:
    mutable std::mutex mutex_;
    T value_{};
    bool hasValue_ = false;
};
```

No condition variable is needed: the reader never *waits* for a value, it just takes the current one (or `nullopt` if none has arrived yet). If the stored type is small and trivially copyable — a single reading, a pose, a set-point — you can drop the mutex entirely and use an [atomic](atomics.md) instead, letting the newest write simply replace the last:

```cpp
std::atomic<double> latestReading{0.0};   // one double is lock-free on every target
// producer thread:
latestReading.store(sensor.read());       // newest wins; no lock
// consumer thread:
double r = latestReading.load();
```

!!! warning "Only *small* types are lock-free — check before you assume"
    `std::atomic<T>` is lock-free only for small, trivially-copyable `T` (an `int`, a pointer, a single `double`). Wrap a bigger struct — say `struct Pose { double x, y, theta; }` — and the atomic silently falls back to an internal lock; on GCC it will not even link without `-latomic`. Verify with `std::atomic<T>{}.is_lock_free()`. When in doubt, use the mutex-protected `Mailbox` above: it is correct for *any* type, and a single uncontended lock is cheap.

Between them, the **queue** ("process every command, in order, then shut down cleanly") and the **mailbox** ("only the latest sample matters") cover the two hand-offs the physics-loop ↔ comms-thread architecture is made of. Reach for the queue when every item counts; reach for the mailbox when only the newest does.

---

## Summary

- A mutex provides mutual exclusion; a **condition variable** lets a thread **sleep until notified**, instead of wasting a core **busy-waiting**.
- The waiter holds a `std::unique_lock` and calls `cv.wait(lock, predicate)`; `wait` atomically releases the lock, sleeps, and re-acquires it on waking.
- **Always use the predicate form.** Condition variables suffer **spurious wakeups**, so a bare `wait` can return when nothing is ready; the predicate form re-checks and is equivalent to `while (!pred()) cv.wait(lock)`.
- **producer/consumer** is the canonical pattern: produce under the lock, `notify_one()` after unlocking, and include a `done` flag in the predicate so consumers can exit cleanly.
- `notify_one()` wakes one waiter; `notify_all()` wakes all — prefer `notify_all()` when unsure. `wait_for`/`wait_until` add a timeout.
- C++20 **semaphores** are a lighter signal for "N permits"; the condition variable remains the general tool for waiting on an arbitrary predicate.
- Package the pattern once as a **`ThreadSafeQueue<T>`** (mutex + cv + `close()` + a `std::optional`-returning `waitAndPop`) for "process every item, then shut down cleanly", and a single-slot **mailbox** for "only the newest sample matters" — the two hand-offs the semester project's physics-loop ↔ comms-thread architecture is built on. Next: [Atomics](atomics.md), for the cases where you need no lock at all.

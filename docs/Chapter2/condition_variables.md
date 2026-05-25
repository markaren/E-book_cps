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
| `wait(lock, predicate)` | the waiting thread | Atomically releases the lock and sleeps until notified *and* the predicate is true; re-acquires the lock before returning. |
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

## Summary

- A mutex provides mutual exclusion; a **condition variable** lets a thread **sleep until notified**, instead of wasting a core **busy-waiting**.
- The waiter holds a `std::unique_lock` and calls `cv.wait(lock, predicate)`; `wait` atomically releases the lock, sleeps, and re-acquires it on waking.
- **Always use the predicate form.** Condition variables suffer **spurious wakeups**, so a bare `wait` can return when nothing is ready; the predicate form re-checks and is equivalent to `while (!pred()) cv.wait(lock)`.
- **producer/consumer** is the canonical pattern: produce under the lock, `notify_one()` after unlocking, and include a `done` flag in the predicate so consumers can exit cleanly.
- `notify_one()` wakes one waiter; `notify_all()` wakes all — prefer `notify_all()` when unsure. `wait_for`/`wait_until` add a timeout.
- C++20 **semaphores** are a lighter signal for "N permits"; the condition variable remains the general tool for waiting on an arbitrary predicate. Next: [Atomics](atomics.md), for the cases where you need no lock at all.

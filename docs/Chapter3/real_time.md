# Real-Time & Timing

Everything so far in Part 3 has been about doing work *concurrently*. This chapter is about doing it *on time*. In a cyber-physical system — a balance controller, a motor loop, a safety interlock — a correct result delivered too late is a wrong result. That is the defining idea of **real-time** computing, and it changes how you think about a program: timing becomes part of correctness, not a performance afterthought.

---

## Real-time is about deadlines, not speed

A "real-time" system is not necessarily a *fast* one. It is one that must respond within a guaranteed **deadline**. [From AIS1003 to AIS2203](../from_ais1003.md) introduced the three flavours; they are worth restating because they decide how hard you must work:

| Class | A missed deadline means… | Example |
|-------|--------------------------|---------|
| **Hard** | Total system failure; never acceptable | A motor-overcurrent cutoff; a flight control surface |
| **Firm** | The result is now useless, but no catastrophe | A video frame that arrived too late to display |
| **Soft** | Quality degrades the later you are | A UI that feels sluggish; a dropped telemetry sample |

Most of what you build in this course is **soft or firm** real-time on a general-purpose OS. True *hard* real-time needs guarantees a normal Linux or Windows cannot give (more on that below). The engineering question is always: *what is my deadline, and what happens if I miss it?*

Two more words you need:

- **Latency** — the delay between an event and your response to it.
- **Jitter** — the *variation* in that latency from one occurrence to the next. A loop that runs every 10 ms ± 0.1 ms has low jitter; one that runs every 10 ms ± 5 ms has high jitter, and high jitter is often worse for control than a slightly higher but *steady* latency.

---

## Measure time with the right clock

All timing builds on [`std::chrono`](../Chapter1/chrono.md). The one rule that matters most here: **use `std::steady_clock` for measuring intervals**, never `std::system_clock`.

```cpp
#include <chrono>
#include <iostream>

int main() {
    auto start = std::chrono::steady_clock::now();

    // ... the work you want to time ...

    auto elapsed = std::chrono::steady_clock::now() - start;
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
    std::cout << "took " << us << " us\n";
}
```

`steady_clock` is **monotonic**: it only ever moves forward at a constant rate, unaffected by the user changing the system time or an NTP daemon nudging the clock. `system_clock` is wall-clock time — it can jump backwards or forwards, which would make an interval measurement negative or wildly wrong. Rule of thumb: **`steady_clock` for "how long did this take" and timeouts; `system_clock` only for "what time is it" timestamps.**

---

## Periodic tasks without drift

The most common real-time pattern is a loop that must run at a fixed rate — a 100 Hz control loop, a 50 Hz telemetry sender. The obvious way to write it is subtly wrong:

<!-- no-ce -->
```cpp
// WRONG: drifts. The period ignores how long work() takes, and sleep_for
// guarantees only "at least" this long — errors accumulate every cycle.
while (running) {
    work();
    std::this_thread::sleep_for(10ms);
}
```

Two problems compound every iteration. First, the real cycle time is `work()` *plus* 10 ms, so a loop that should run at 100 Hz runs slower. Second, `sleep_for` is only guaranteed to sleep *at least* the requested time — any overshoot is added on top, and these errors never cancel, so the loop **drifts** further from the intended schedule the longer it runs.

The fix is to schedule against **absolute deadlines**: decide when the next cycle should start, and `sleep_until` that point. Overshoot in one cycle is absorbed rather than accumulated.

```cpp
#include <chrono>
#include <iostream>
#include <thread>

int main() {
    using namespace std::chrono;
    const auto period = 100ms;

    auto next = steady_clock::now() + period;     // first deadline
    for (int cycle = 0; cycle < 5; ++cycle) {
        // --- do the periodic work here ---
        std::cout << "cycle " << cycle << "\n";

        std::this_thread::sleep_until(next);       // wait for the absolute deadline
        next += period;                            // advance to the next one
    }
}
```

Because `next` advances by exactly one period each time regardless of how long the work took, the loop stays locked to the intended rate on average — there is no cumulative drift. This `sleep_until`-against-a-rolling-deadline is *the* pattern for periodic real-time work; reach for it any time you need a fixed rate.

!!! tip "Detect overruns"
    If `work()` ever takes longer than the period, `next` will already be in the past when you reach `sleep_until`, which returns immediately. You can detect this — `if (steady_clock::now() > next) ++overruns;` — and report it. A loop silently failing to keep up is exactly the kind of timing bug real-time discipline is meant to catch.

---

## Asynchronous events

A control loop is *polled* — it runs on a schedule. The other half of real-time work is *reacting to events* that arrive unpredictably: a limit switch trips, a packet lands, a button is pressed. On a microcontroller (Arduino) these are true hardware **interrupts** that preempt your code. On an embedded **Linux** system you are in user space, with no direct interrupts — so you model events with the concurrency tools from Part 2:

- A dedicated thread that **blocks on a read** — a [socket](../Chapter4/sockets.md) `recv`, a GPIO edge via `poll()`, a serial read — and wakes the instant data arrives. The OS does the waiting; your thread costs nothing while blocked.
- A [condition variable](../Chapter2/condition_variables.md) another thread notifies when the event-handling thread should proceed.

The pattern is almost always **one thread blocked on the event source**, handing work to the rest of the system — never a busy-wait poll that burns a core checking "did it happen yet?".

### Coordinating phases: latches and barriers

When several threads must reach the same point before any continues — "all sensors sampled before the control step computes" — C++20 gives you two purpose-built tools:

- **`std::latch`** — a one-shot countdown. Threads `count_down()`; others `wait()` until the count hits zero. Use it once, e.g. "wait until all worker threads have finished initialising."
- **`std::barrier`** — a *reusable* latch. Each thread calls `arrive_and_wait()` at the end of a phase; all are released together, and the barrier resets for the next cycle. Ideal for a loop where N threads must stay in lockstep round after round.

```cpp
// Each cycle: all worker threads sample, then all proceed together.
std::barrier sync(numWorkers);
// in each worker, per cycle:
sampleSensor();
sync.arrive_and_wait();   // none proceed until every worker has sampled
useAllSamples();
```

These express "synchronise at a phase boundary" far more clearly than hand-rolling it with a counter and a condition variable.

---

## Real-time on a general-purpose OS

Here is the hard truth about running control code on a Pi or a PC: **stock Linux and Windows are not hard-real-time operating systems.** The scheduler time-slices your thread against every other process; a page fault, a burst of disk activity, or a higher-priority process can delay your loop by milliseconds with no warning. For soft and firm deadlines that is usually fine. For tight or hard deadlines you need help:

- **Real-time scheduling priority.** Linux offers the `SCHED_FIFO`/`SCHED_RR` policies, which run a thread ahead of ordinary ones. You request it per-thread (requires privileges):

    <!-- no-ce -->
    ```cpp
    // Linux: raise this thread to real-time priority (needs CAP_SYS_NICE / root)
    sched_param sch{};
    sch.sched_priority = 80;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &sch);
    ```

- **A `PREEMPT_RT` kernel.** A patched Linux kernel that makes almost all of the kernel preemptible, turning typical worst-case latencies from milliseconds into tens of microseconds. This is how Linux is used for serious motion control.
- **CPU isolation.** Pinning your real-time thread to a core that the OS keeps other work off (`isolcpus`, `taskset`).

These are knobs you reach for *after* measuring and finding the default scheduling inadequate — not by default.

### Hot-path discipline

Whatever the OS, the surest way to blow a deadline is to do something unbounded in the time-critical loop. Inside a real-time path, **avoid**:

- **Dynamic allocation** (`new`, `malloc`, growing a `std::vector`) — it can take an unpredictable amount of time and may lock internally. **Preallocate** before the loop.
- **Long or contended [locks](../Chapter2/sharing_data.md)** — a held mutex can stall your loop for as long as another thread holds it. Keep critical sections tiny, or use [atomics](../Chapter2/atomics.md) for simple shared values.
- **Blocking I/O and logging** — writing to disk or the console can block for milliseconds. Hand data to a separate logging thread instead.
- **Exceptions on the hot path** — throwing has unpredictable cost; reserve it for genuine errors, not control flow.

The principle: in the hot path, do a **bounded, predictable** amount of work, and push everything else (allocation, logging, slow I/O) outside the loop or onto another thread.

---

## Summary

- **Real-time means meeting deadlines**, not raw speed. Know whether your deadline is **hard / firm / soft** and what a miss costs — most course work is soft/firm on a general-purpose OS.
- Track **latency** (delay) and **jitter** (variation in delay); steady, low jitter often matters more than low average latency.
- Measure intervals with **`std::steady_clock`** (monotonic), never `system_clock` (wall-clock, can jump).
- Run periodic loops against **absolute deadlines** with `sleep_until(next); next += period;` to avoid the **drift** that `sleep_for(period)` accumulates.
- Model **asynchronous events** with a thread **blocked** on the source (socket, GPIO, serial) plus a [condition variable](../Chapter2/condition_variables.md) — not a busy-wait. Use C++20 **`std::latch`/`std::barrier`** to synchronise threads at phase boundaries.
- Stock Linux/Windows are **not hard real-time**; tighten with `SCHED_FIFO` priority, a `PREEMPT_RT` kernel, or CPU isolation *after measuring*. On the hot path, keep work **bounded**: no allocation, no long locks, no blocking I/O. (The [Embedded Linux](../embedded_linux.md) reference notes the same caveats on real hardware.)

# Time with std::chrono

Almost every concurrency tool in this book takes a time argument. `sleep_for`, `sleep_until`, `cv.wait_for`, `future.wait_for`, the drift-free periodic loop in [Real-Time & Timing](../Chapter3/real_time.md) — all of them speak `std::chrono`. This short chapter makes that vocabulary precise so the timing code in the rest of the book reads clearly. There are only three ideas: **durations**, **time points**, and **clocks**.

`std::chrono` lives in `<chrono>`.

---

## Durations: a length of time

A **duration** is an amount of time — "100 milliseconds", "2 seconds". The standard library predefines the units, and the `chrono_literals` namespace gives you readable suffixes:

```cpp
#include <chrono>
#include <iostream>

int main() {
    using namespace std::chrono;
    using namespace std::chrono_literals;     // enables the ms / s / us suffixes

    auto timeout = 2500ms;                    // a duration of 2500 milliseconds
    auto period  = 2s;                        // 2 seconds

    std::cout << timeout.count() << " ms\n";  // 2500 — .count() gives the raw number
    std::cout << (timeout < period) << "\n";  // 1 — durations compare and do arithmetic
}
```

The predefined units are `nanoseconds`, `microseconds`, `milliseconds`, `seconds`, `minutes`, and `hours`. You get the underlying number with `.count()`, but most of the time you pass the duration around as-is — `sleep_for(100ms)` is clearer and safer than `sleep_for(100)`, because the unit is part of the type and the compiler will not let you confuse milliseconds with seconds.

!!! tip "Use the literal suffixes"
    `100ms`, `2s`, `500us`, `1min` read far better than constructing `milliseconds(100)`, and they make the unit unmistakable at the call site. Add `using namespace std::chrono_literals;` in functions that do timing.

### Converting between units: `duration_cast`

Units that convert *exactly* (seconds → milliseconds) happen implicitly. Conversions that *lose precision* (milliseconds → seconds) must be explicit, via `duration_cast`, which **truncates**:

```cpp
using namespace std::chrono;
auto d = 2500ms;
auto secs = duration_cast<seconds>(d);    // 2 seconds — the 500 ms is truncated, not rounded
std::cout << secs.count() << "\n";        // 2
```

The compiler forcing you to write `duration_cast` for a lossy conversion is a feature: it stops you silently dropping precision. Just remember it truncates toward zero — if you need rounding, do it yourself.

---

## Time points: a moment in time

A **time point** is a specific instant, measured from a clock's starting point (its *epoch*). You get one by asking a clock for `now()`. The two operations you need:

- **time point − time point = duration** — how long between two instants (elapsed time).
- **time point + duration = time point** — a future instant (the *next deadline*).

```cpp
#include <chrono>
#include <iostream>

int main() {
    using namespace std::chrono;

    auto start = steady_clock::now();              // a time point

    // ... work you want to time ...

    auto elapsed = steady_clock::now() - start;    // time point − time point = duration
    std::cout << duration_cast<microseconds>(elapsed).count() << " us\n";
}
```

That subtraction is the universal "how long did this take" pattern. The addition — `auto next = now + period;` — is the heart of the drift-free periodic loop from [Real-Time & Timing](../Chapter3/real_time.md): you compute the next deadline and `sleep_until` it.

---

## Clocks: where time comes from

A **clock** produces time points. Three exist; the distinction between the first two matters and is a common source of bugs.

| Clock | Property | Use it for |
|-------|----------|------------|
| `std::steady_clock` | **Monotonic** — only ever moves forward at a constant rate | Measuring intervals, timeouts, periodic loops |
| `std::system_clock` | **Wall-clock** — real calendar time, convertible to `time_t` | Timestamps, "what time is it" |
| `std::high_resolution_clock` | Usually an alias for one of the above | Nothing special — prefer `steady_clock` |

The rule, restated from [Real-Time & Timing](../Chapter3/real_time.md) because it is worth repeating: **use `steady_clock` to measure how long something takes or to schedule a delay; use `system_clock` only to record what time of day something happened.**

!!! danger "Never measure an interval with `system_clock`"
    `system_clock` tracks wall-clock time, which the OS can adjust at any moment — an NTP sync, a manual time change, a daylight-saving shift. Measure an interval across one of those and you can get a wildly wrong or even *negative* result. `steady_clock` is immune: it is guaranteed never to jump or run backwards. Intervals → `steady_clock`, always.

---

## Where you have already seen it

Every timing call in the book is `chrono` underneath:

```cpp
std::this_thread::sleep_for(100ms);              // wait a duration
std::this_thread::sleep_until(next);             // wait until a time point — periodic loops
cv.wait_for(lock, 50ms, predicate);              // condition variable with a timeout
future.wait_for(0ms);                            // poll a future without blocking
```

Passing a duration where the API expects one is all there is to it — the unit safety and the clock choice are handled by the types you have just met. A `cv.wait_for(lock, 50ms, pred)` ([Condition Variables](../Chapter2/condition_variables.md)) gives up after 50 ms; a `future.wait_for(0ms)` ([Futures](../Chapter3/futures.md)) checks "is it ready?" without blocking. Both are just durations handed to the concurrency primitives.

---

## A pitfall: integer truncation

The predefined durations hold *integers*, so dividing or converting can silently truncate:

```cpp
auto half = 5s / 2;                       // 2 seconds, not 2.5 — integer division
auto ms = duration_cast<seconds>(1999ms); // 1 second — truncated
```

For most timing — sleeps, timeouts, periods — integer milliseconds or microseconds are plenty precise and this never bites. If you genuinely need fractional seconds (say, computing a control gain from an elapsed time), use a floating-point duration: `duration<double>` holds seconds as a `double`.

```cpp
using namespace std::chrono;
duration<double> elapsed = steady_clock::now() - start;
double seconds = elapsed.count();         // fractional seconds, no truncation
```

---

## Summary

- `std::chrono` has three ideas: **durations** (a length of time), **time points** (an instant from a clock's epoch), and **clocks** (which produce time points).
- Write durations with the **literal suffixes** (`100ms`, `2s`) so the unit is part of the type; `.count()` gets the raw number; **`duration_cast`** converts lossy units and **truncates**.
- **time point − time point = duration** (elapsed time); **time point + duration = time point** (the next deadline) — the two patterns behind timing and periodic loops.
- Use **`steady_clock`** (monotonic) for intervals, timeouts and scheduling; **`system_clock`** only for calendar timestamps. **Never measure an interval with `system_clock`** — it can jump.
- Every `sleep_for`/`sleep_until`/`wait_for` in the book is just a `chrono` duration or time point. For fractional seconds, use `duration<double>`.

That completes the Modern C++ Toolkit. With ownership, moves, smart pointers, templates, lambdas, and timing in hand, every tool [Part 2](../Chapter2/processes_threads.md) and [Part 3](../Chapter3/futures.md) lean on is now grounded — try the [exercises](exercises.md), then carry on to communication in [Part 4](../Chapter4/serialization.md).

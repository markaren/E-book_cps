# Debugging Concurrent Programs

[Sharing Data](Chapter2/sharing_data.md) made the point that a concurrent program can be wrong only *sometimes*. That is what makes these bugs a different kind of hard: a [data race](Chapter2/sharing_data.md) or a [deadlock](Chapter2/sharing_data.md) depends on the exact timing of threads, so it may pass a thousand runs and fail on the thousand-and-first, on one machine but not another, only under load. This page is about the tools and habits that turn "it crashed once, I don't know why" into a bug you can actually find.

---

## Why they are hard

Three properties make concurrency bugs slippery:

- **Non-determinism.** The [scheduler](Chapter2/processes_threads.md) decides how threads interleave, and it decides differently each run. The buggy interleaving might be rare.
- **Heisenbugs.** Observing the program changes its timing. Add a `std::cout` or attach a debugger and the bug can *vanish*, because you have shifted the timing window in which it occurs. (Named after the Heisenberg uncertainty principle.)
- **Low reproduction rate.** "It worked when I ran it" proves nothing. A race that strikes one run in a thousand is still a bug that will strike in production.

The consequence: you cannot debug concurrency by running once and eyeballing the output. You need tools that detect the *hazard* itself, not just its occasional symptom.

---

## ThreadSanitizer: your most important tool

**ThreadSanitizer (TSan)** is the single best tool for this, and you should reach for it the moment a threaded program misbehaves. It is a compiler instrumentation (in **GCC and Clang**, not MSVC) that watches every memory access at run time and reports a **data race even if the race did not produce wrong output on that run**. It finds the hazard, not just the symptom.

Enable it by compiling and linking with `-fsanitize=thread` (and `-g` for line numbers). In CMake, gate it behind an option so you can turn it on for debugging:

```cmake
option(ENABLE_TSAN "Build with ThreadSanitizer" OFF)

if(ENABLE_TSAN)
    target_compile_options(app PRIVATE -fsanitize=thread -g)
    target_link_options(app    PRIVATE -fsanitize=thread)
endif()
```

```bash
cmake -S . -B build-tsan -DENABLE_TSAN=ON
cmake --build build-tsan
./build-tsan/app          # run it — TSan prints any race it observes
```

Run the [racy counter from Sharing Data](Chapter2/sharing_data.md) under TSan and it prints something like:

```
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x55a... by thread T2:
    #0 increment() sharing_data.cpp:8
  Previous read of size 4 at 0x55a... by thread T1:
    #0 increment() sharing_data.cpp:8
  Location is global 'counter' of size 4
```

It tells you exactly what raced: the two accesses (a read and a write), the two threads, the source lines, and the variable. That is the whole bug, handed to you — no guessing. Fix it (a [mutex](Chapter2/sharing_data.md) or an [atomic](Chapter2/atomics.md)) and rerun until TSan is silent.

!!! tip "Run TSan even when nothing looks wrong"
    Because TSan flags the *hazard* regardless of whether it caused a visible failure, running your test suite under TSan catches races that have *not yet* bitten you. The cost is runtime: TSan slows a program roughly 5–15× and uses more memory, so you use it while debugging and in CI, not in release builds.

!!! note "The other sanitizers"
    **AddressSanitizer (ASan)** finds memory errors (use-after-free, buffer overrun — including a thread reading a [dangling reference](Chapter2/threads.md) it captured); **UndefinedBehaviorSanitizer (UBSan)** catches undefined behaviour. They are enabled the same way (`-fsanitize=address`, `-fsanitize=undefined`). TSan and ASan cannot be combined in one build — run them separately.

---

## Deadlocks: read the thread stacks

A [deadlock](Chapter2/sharing_data.md) presents differently: the program does not crash or misbehave, it simply **hangs**. TSan can detect lock-order inversions that *could* deadlock, but for a program that has already frozen, attach a debugger and look at what every thread is doing.

In CLion, hit the **pause/suspend** button while it is hung and inspect the **Threads** panel; in gdb:

```
gdb --pid <pid>
(gdb) thread apply all bt      # backtrace of every thread
```

You are looking for threads blocked inside `lock()`, `wait()`, or `join()`. Two threads each parked in `std::mutex::lock`, each holding the mutex the other wants, is the classic [circular wait](Chapter2/sharing_data.md). Once you can see *which lock each thread holds and which it is waiting for*, the fix is the one from Sharing Data: a consistent lock order, or `std::scoped_lock` to take both at once.

---

## Making a rare bug reproduce

A bug that appears one run in a thousand is hard to study. Widen the window so it appears often:

- **Loop the test.** Run the suspect code thousands of times in a script or a loop; a 0.1 %-per-run race shows up fast.
- **Add deliberate delays.** A `std::this_thread::sleep_for(1ms)` placed *between* the read and the write of a suspected race dramatically widens the window and can turn a rare race into a reliable one — a quick way to confirm a hypothesis (remove it afterwards).
- **Vary the thread count and load.** Run with more threads than cores, and under CPU load, to change the scheduling.
- **Run under TSan**, which flags the race from a *single* run regardless of whether it manifested — usually faster than any of the above.

---

## Be careful with print-debugging

Adding `std::cout` is the reflex, but for concurrency it is treacherous:

- **It changes the timing** — the heisenbug problem — so the bug may hide while the prints are in and return when you remove them.
- **`std::cout` is itself shared state.** Two threads printing at once can interleave their output character by character, and writing to it is a synchronisation point that perturbs the very timing you are trying to observe.

If you must log, use a **thread-safe** logger (or guard `std::cout` with a [mutex](Chapter2/sharing_data.md)), include a thread id and timestamp, and keep it out of the hot path. But prefer **TSan** — it tells you about the race without altering the timing the way printing does.

---

## Prevention beats debugging

The cheapest concurrency bug is the one the design makes impossible. The habits from Part 2 are also debugging habits, because they shrink the surface where bugs can hide:

- **Minimise shared state** — data not shared cannot race ([avoid shared state](Chapter2/sharing_data.md)).
- **Encapsulate data with its mutex** and expose only synchronised methods, so no caller can forget to lock.
- **Use RAII locks** (`lock_guard`/`scoped_lock`) and keep critical sections short.
- **Run TSan in CI**, so a race is caught automatically the first time it is introduced — not months later in the field.

---

## Summary

- Concurrency bugs are **non-deterministic**, can be **heisenbugs** (observing them hides them), and reproduce rarely — so running once and eyeballing the output proves nothing.
- **ThreadSanitizer** (`-fsanitize=thread`, GCC/Clang) is the key tool: it detects a **data race even when it caused no visible failure**, and tells you the two accesses, threads, lines, and variable. Run it while debugging and in CI (it is ~5–15× slower). ASan/UBSan catch memory and UB bugs similarly.
- For a **deadlock** (a hang), attach a debugger and read **every thread's stack** (`thread apply all bt`) to see which lock each holds and waits for; fix with consistent ordering or `std::scoped_lock`.
- Make rare bugs reproduce by **looping the test, inserting delays, varying threads/load** — or just let TSan flag it from one run.
- **Avoid print-debugging** for races (it changes timing and `std::cout` is shared state); prefer TSan.
- **Prevention is cheapest**: minimise shared state, encapsulate data+mutex, use RAII locks, run TSan in CI.

# Creating Threads

[Processes & Threads](processes_threads.md) showed the smallest possible thread: hand `std::thread` a function and call `join()`. This chapter fills in the mechanics you actually need — how to give a thread *arguments*, the real difference between **joining** and **detaching**, and the C++20 `std::jthread` that makes both safer. Get these right and the [next chapter](sharing_data.md) can focus on the hard part: sharing data between the threads you create here.

---

## What a thread can run

A `std::thread` runs any **callable**: a free function, a lambda, a function object, or a member function. The thread *begins running concurrently from construction* — the moment the `std::thread` is built, its callable may already be executing on another core.

```cpp
#include <iostream>
#include <thread>

void freeFunction() { std::cout << "free function\n"; }

struct Functor {
    void operator()() const { std::cout << "functor\n"; }
};

int main() {
    std::thread t1(freeFunction);
    std::thread t2(Functor{});
    std::thread t3([] { std::cout << "lambda\n"; });   // the form you will use most

    t1.join();
    t2.join();
    t3.join();
}
```

In practice the **lambda** is the form you reach for: it lets you write the thread's work inline and capture the data it needs. Lambdas are covered in [Lambdas & std::function](../Chapter1/lambdas.md); here they are simply the most convenient way to describe a unit of work.

---

## Passing arguments

Extra arguments to the `std::thread` constructor are forwarded to the callable:

```cpp
#include <iostream>
#include <thread>
#include <string>

void greet(const std::string& name, int times) {
    for (int i = 0; i < times; ++i) {
        std::cout << "Hello, " << name << "\n";
    }
}

int main() {
    std::thread t(greet, "Pi", 3);
    t.join();
}
```

There is one rule that surprises everyone the first time, and it is a frequent source of bugs:

!!! danger "Arguments are *copied* into the thread — so `std::ref` is how you share"
    `std::thread` copies each argument into storage owned by the new thread, then passes those copies to your callable. Two consequences follow, depending on how the parameter is declared:

    - **A by-value or `const&` parameter** (like `greet`'s `const std::string&`) silently binds to the thread's *own* copy. The function runs fine — it just never sees your original object.
    - **A non-`const` reference parameter** (`std::string&`) will **not even compile**. The stored copies are handed to your function as rvalues, and a non-`const` lvalue reference cannot bind to an rvalue — so the mistake is caught by the compiler, not discovered at runtime.

    Either way, to make the thread act on your *original* object you must say so explicitly with `std::ref`:

    ```cpp
    std::thread t(updateInPlace, std::ref(myObject));   // pass the real object, not a copy
    ```

    `std::ref` wraps the argument so the reference survives the copy. In return you are promising the original **outlives the thread** — if it does not, the thread is left holding a dangling reference (see the pitfalls below).

For a member function, pass a pointer to the function and the object to call it on:

```cpp
#include <iostream>
#include <thread>

struct Motor {
    void run(int rpm) { std::cout << "running at " << rpm << " rpm\n"; }
};

int main() {
    Motor motor;
    std::thread t(&Motor::run, &motor, 1500);   // call motor.run(1500) on a new thread
    t.join();
}
```

!!! note "Getting a result back"
    A thread cannot `return` a value to whoever started it — its callable's return value is discarded. To compute something on a thread and collect the answer, you use a **future**, covered in [Futures & Promises](../Chapter3/futures.md). For now, threads communicate results by writing to shared state (carefully — that is [the next chapter](sharing_data.md)).

---

## Join or detach — you must choose one

A running thread is a resource, and like every resource in C++ it has an owner: the `std::thread` object. Before that object is destroyed, you must decide what happens to the underlying thread. There are exactly two choices:

- **`join()`** — block here until the thread finishes, then clean it up. This is what you want almost always.
- **`detach()`** — sever the link and let the thread run on its own, unsupervised, until it finishes or the program exits.

If you do *neither* and the `std::thread` is destroyed while still **joinable**, its destructor calls `std::terminate()` and the program dies:

<!-- no-ce -->
```cpp
void bad() {
    std::thread t([]{ /* work */ });
}   // t is destroyed still joinable → std::terminate()
```

This is deliberate. Silently joining would hide a blocking wait; silently detaching would hide a thread outliving the data it uses. The language refuses to guess, so the rule is: **every `std::thread` must be joined or detached on every path out of its scope** — including paths taken by an exception.

You can ask whether a thread still needs a decision:

```cpp
#include <iostream>
#include <thread>

int main() {
    std::thread t([]{ std::cout << "working\n"; });
    if (t.joinable()) {     // true: it has not been joined or detached yet
        t.join();
    }
    std::cout << "done\n";
}
```

### When detach is the wrong tool (usually)

`detach()` sounds convenient — fire a thread and forget it — but a detached thread is genuinely dangerous, because it keeps running after the scope that launched it is gone:

<!-- no-ce -->
```cpp
void launch() {
    int data = 42;
    std::thread t([&data]{
        // ... long-running work that reads data ...
    });
    t.detach();
}   // launch() returns, `data` is destroyed — the detached thread now reads garbage
```

The detached thread outlives `data` and reads a dangling reference: undefined behaviour, and a crash that appears far from its cause. Detaching is occasionally right (a truly fire-and-forget background task that owns everything it touches), but for course work you should **prefer joining**, and prefer the `std::jthread` below that joins for you.

---

## `std::jthread`: join by RAII (C++20)

Manually matching every thread with a `join()` on every exit path is exactly the kind of bookkeeping RAII was invented to remove — the same problem `std::lock_guard` solves for mutexes. C++20's **`std::jthread`** ("joining thread") is a `std::thread` that **joins itself in its destructor**:

```cpp
#include <iostream>
#include <thread>

int main() {
    std::jthread worker([]{ std::cout << "work done\n"; });
    std::cout << "main does other things\n";
}   // worker.join() happens automatically here
```

No `join()` call, no `std::terminate()` trap, no leaked thread. **Prefer `std::jthread` over `std::thread`** unless you have a specific reason not to — it is the modern default, and it removes the single most common threading bug in beginner code.

### Cooperative cancellation with stop tokens

`std::jthread` adds a second feature: a built-in way to *ask* a thread to stop. You cannot safely yank a thread away mid-work, so C++20 uses **cooperative** cancellation — the thread is handed a `std::stop_token` and checks it at safe points, stopping when asked.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    std::jthread worker([](std::stop_token token) {
        int tick = 0;
        while (!token.stop_requested()) {     // keep working until asked to stop
            std::cout << "tick " << tick++ << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
        std::cout << "stopping cleanly\n";
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(350));
    worker.request_stop();    // ask it to finish; its destructor then joins
}
```

If the callable's first parameter is a `std::stop_token`, `std::jthread` supplies one automatically. Calling `request_stop()` (or letting the `jthread` destructor do it) sets the token, the loop sees `stop_requested()` become true, and the thread ends on its own terms — finishing the current iteration rather than being killed halfway. This is the standard pattern for a worker loop that must run "until told otherwise": a sensor poller, a telemetry sender, a control loop. (`std::this_thread::sleep_for` and the `<chrono>` durations come from [Time with std::chrono](../Chapter1/chrono.md).)

!!! warning "`request_stop()` does not wake a thread sleeping in `condition_variable::wait`"
    Setting the stop token only flips a flag — it does **not** interrupt a thread already blocked in a plain `std::condition_variable::wait`. If your worker sleeps on a condition variable, requesting a stop will not wake it, and its destructor's implicit `request_stop()` then `join()` can hang forever. Two fixes: use the stop-aware overload `std::condition_variable_any::wait(lock, stopToken, predicate)`, which returns when the token is set; or register a callback that notifies the CV, or set your own atomic flag *and* call `notify_all()` yourself so the waiter re-checks its predicate and exits.

---

## Pitfalls to internalise

Most thread-creation bugs are **lifetime** bugs: the thread outlives something it points at. Three to watch for:

| Pitfall | What goes wrong | Fix |
|---------|-----------------|-----|
| Capturing a local by reference | A lambda captures `[&x]`, the thread outlives `x`'s scope, and reads a dangling reference. | Capture by value (`[x]`), or guarantee the thread joins before `x` dies. |
| Forgetting `std::ref` | You meant to share an object. With a `const&`/by-value parameter the thread silently works on a *copy*; with a non-`const` `T&` parameter the code *fails to compile* (the copy binds as an rvalue). | Pass `std::ref(obj)` — and ensure `obj` outlives the thread. |
| Detaching and returning | A detached thread keeps using locals from a function that has already returned. | Don't detach; use `std::jthread` and join. |

The unifying rule: **a thread must not outlive the data it borrows.** When you cannot guarantee that, either give the thread its own copy, or hand it shared ownership with a [`std::shared_ptr`](../Chapter1/smart_pointers.md).

---

## Summary

- A `std::thread`/`std::jthread` runs any **callable**; a **lambda** is the usual choice because it can capture the data the thread needs.
- Constructor arguments are **copied** into the thread. Reference parameters get copies too — use `std::ref` to share the real object, and only when it **outlives** the thread.
- A thread cannot return a value directly; use a [future](../Chapter3/futures.md) or shared state.
- Every `std::thread` must be **`join()`ed or `detach()`ed** before destruction, or the program calls `std::terminate()`. Prefer joining; **detaching risks a thread outliving its data**.
- **`std::jthread` (C++20) joins itself via RAII** and supports **cooperative cancellation** through `std::stop_token` / `request_stop()`. Make it your default.
- Almost every threading bug at this stage is a **lifetime** bug — never let a thread outlive what it borrows. Next: [Sharing Data](sharing_data.md), where two of these threads touch the same memory.

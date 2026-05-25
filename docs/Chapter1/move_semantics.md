# Move Semantics

[Ownership & RAII](ownership.md) said that ownership can be *transferred*. **Move semantics** is the mechanism. Moving lets one object hand its resources to another — cheaply, and even for resources that cannot be copied at all. That second part is why this chapter matters so much for concurrency: a `std::thread`, a `std::future`, a `std::unique_ptr` **cannot be copied**, only moved, and you will move them into threads and queues throughout Parts 2 and 3.

---

## Copy versus move

Copying produces an independent duplicate. For a `std::vector<int>` of a million elements, that means allocating new memory and copying a million values — and for a *unique* resource like a running thread, it makes no sense at all (you cannot have "two copies" of one OS thread).

**Moving** instead *transfers* the resource: the destination takes over the source's internals, and the source is left empty. For that vector, a move is a handful of pointer assignments regardless of size — the new vector simply adopts the old one's buffer.

```cpp
#include <iostream>
#include <utility>
#include <vector>

int main() {
    std::vector<int> a{1, 2, 3};
    std::vector<int> b = std::move(a);     // transfer a's buffer to b — no element copy

    std::cout << "b has " << b.size() << " elements\n";   // 3
    std::cout << "a has " << a.size() << " elements\n";   // 0 — a was moved from
}
```

`b` took ownership of the storage; `a` was left empty. No million-element copy happened — just a transfer of the internal pointer.

---

## `std::move` is a cast, not a move

The name misleads everyone at first. **`std::move` does not move anything.** It is a cast: it tells the compiler "treat this object as something you may *move from*" — i.e. as a temporary whose guts can be stolen. The actual moving is done by the type's *move constructor* or *move assignment*, which `std::move` merely permits the compiler to select.

To see why a cast is needed, you need two words:

- An **lvalue** is a named object with an address — `a` above. It looks permanent, so the compiler will *copy* from it by default, to be safe.
- An **rvalue** is a temporary with no name — the result of `a + b`, or a function's returned value. It is about to disappear, so the compiler is free to *move* from it.

`std::move(a)` casts the lvalue `a` into an rvalue reference, signalling "I am done with `a` — you may treat it as disposable." That is why, after `std::move(a)`, you must not rely on `a`'s value (more below).

---

## How a type becomes movable

A class supports moving by providing a **move constructor** and **move assignment operator**, which take an *rvalue reference* (`T&&`). They steal the source's internals and leave it in a valid empty state:

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n) : size_(n), data_(new int[n]) {}
    ~Buffer() { delete[] data_; }

    Buffer(Buffer&& other) noexcept            // move constructor
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;                 // leave the source empty but destructible
    }

    Buffer(const Buffer&) = delete;            // not copyable — a unique resource
    Buffer& operator=(const Buffer&) = delete;

private:
    std::size_t size_;
    int* data_;
};
```

You will rarely write this by hand. As [Ownership & RAII](ownership.md) argued, the **rule of zero** says to build from members that already move correctly — `std::vector`, `std::unique_ptr` — and let the compiler generate the move operations for you. The example above exists to show what those generated operations *do*: take the source's handle, null out the source.

!!! tip "Mark moves `noexcept`"
    Move operations should be `noexcept` (they only shuffle pointers, so they cannot fail). It is not cosmetic: `std::vector` will only *move* its elements when it regrows if their move is `noexcept` — otherwise it falls back to *copying* them to preserve its exception guarantee. A missing `noexcept` can silently turn fast moves back into slow copies.

---

## Move-only types: the concurrency payoff

Some resources are inherently unique, so their types **delete the copy operations** and support only moving. You have already used several:

| Move-only type | Why it can't be copied |
|----------------|------------------------|
| `std::unique_ptr` | Sole ownership of a heap object — two copies would double-free |
| `std::thread` / `std::jthread` | Represents one OS thread |
| `std::future` / `std::promise` | One handle to one shared result |
| `std::packaged_task` | Owns the task's result state |
| `std::unique_lock` | Sole ownership of a held lock |

This is exactly why move semantics is unavoidable in concurrent code. You cannot copy these, so to put them anywhere — into a container, into another thread — you must **move**:

```cpp
#include <future>
#include <thread>
#include <vector>

int main() {
    std::packaged_task<int()> task([] { return 42; });
    std::jthread runner(std::move(task));   // MUST move — packaged_task can't be copied

    std::vector<std::jthread> pool;
    pool.push_back(std::jthread([]{}));     // the temporary jthread is moved into the vector
}
```

The [thread pool](../Chapter3/thread_pools.md) moved a `packaged_task` into its queue; [futures](../Chapter3/futures.md) get stored in a `std::vector<std::future<T>>` by moving; a `std::unique_ptr` is moved into a thread to hand over heap ownership. Every one of those is move semantics doing the work ownership.md described as "transfer."

---

## When moves happen for you

You do not write `std::move` everywhere — the compiler moves automatically in the most common case: **returning a local by value**.

```cpp
std::vector<int> makeData() {
    std::vector<int> v{1, 2, 3};
    return v;            // automatically moved (or elided entirely) — never copied
}
```

Do **not** write `return std::move(v);` here — it is unnecessary and can actually *prevent* the compiler from eliding the move altogether. Trust the automatic behaviour for returns.

A second common idiom is the **"sink"** parameter: take an argument by value and `std::move` it into a member, so callers can either copy or move into you as they choose:

```cpp
class Widget {
public:
    explicit Widget(std::string name) : name_(std::move(name)) {}   // move into the member
private:
    std::string name_;
};
```

---

## The one rule: don't use a moved-from object

After you move from an object, it is left in a **valid but unspecified** state. "Valid" means it is safe to destroy and safe to assign a new value to. "Unspecified" means you must not *read* or rely on its contents:

```cpp
std::string s = "hello";
std::string t = std::move(s);
// s is now valid but unspecified — do NOT read it
s = "fresh";                    // OK: assigning a new value is fine
std::cout << s;                 // OK now — s has a known value again
```

The practical rule: once you have moved from a variable, treat it as empty. Either give it a new value or let it go out of scope. Reading a moved-from object is a classic bug — the value might be empty, might be the old value, might be anything.

---

## Summary

- **Copying** duplicates a resource; **moving** transfers it, leaving the source empty. Moves are cheap (pointer shuffles) and work for resources that cannot be copied at all.
- **`std::move` is a cast**, not an action — it marks an lvalue as movable so the compiler picks the **move constructor / move assignment** (which take `T&&`). Mark those `noexcept`.
- Prefer the **rule of zero**: build from `vector`/`unique_ptr` and let the compiler generate moves; rarely hand-write them.
- **Move-only types** — `unique_ptr`, `thread`/`jthread`, `future`, `promise`, `packaged_task`, `unique_lock` — can't be copied, so you **move** them into containers and threads. This is why move semantics is everywhere in [Part 2](../Chapter2/threads.md) and [Part 3](../Chapter3/futures.md).
- Returning a local **moves automatically** (don't `return std::move(x)`); the by-value "sink" parameter `std::move`d into a member is the other common idiom.
- After moving from an object it is **valid but unspecified** — don't read it; reassign it or let it die. Next: [Smart Pointers](smart_pointers.md), the types that put ownership into the type system.

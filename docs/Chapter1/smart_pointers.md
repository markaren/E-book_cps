# Smart Pointers

[Ownership & RAII](ownership.md) laid out three ways to own a resource — unique, shared, or merely borrowed — and [Move Semantics](move_semantics.md) showed how ownership transfers. **Smart pointers** are the types that make those choices explicit in your code: a glance at the type tells you, and the compiler, who owns the object behind a pointer. They are also where ownership meets concurrency most directly, because sharing data across threads is exactly what `std::shared_ptr` is for.

All three live in `<memory>`.

---

## `std::unique_ptr`: one owner

`std::unique_ptr<T>` is the default smart pointer: it owns a single heap object and frees it automatically when the `unique_ptr` is destroyed. It is **move-only** — there is exactly one owner — and it has **zero runtime overhead** over a raw pointer. Create one with `std::make_unique`:

```cpp
#include <iostream>
#include <memory>

struct Sensor {
    int id;
    explicit Sensor(int i) : id(i) { std::cout << "Sensor " << id << " created\n"; }
    ~Sensor() { std::cout << "Sensor " << id << " destroyed\n"; }
};

int main() {
    auto a = std::make_unique<Sensor>(1);   // a is the sole owner
    std::cout << "using sensor " << a->id << "\n";

    auto b = std::move(a);                  // transfer ownership to b; a is now null
    std::cout << "a is " << (a ? "set" : "null") << "\n";
}                                           // b's destructor frees the sensor — once
```

`make_unique` replaces every bare `new` you would otherwise write, and the destructor replaces every `delete`. Transferring ownership is a [move](move_semantics.md); after `std::move(a)`, `a` is null and `b` owns the object. There is no way to accidentally copy it and double-free.

!!! tip "Reach for `unique_ptr` first"
    Most heap objects have a single, clear owner. `std::unique_ptr` should be your default; only step up to `shared_ptr` when ownership is genuinely shared. A `unique_ptr` can always be *promoted* to a `shared_ptr` later if you discover you need sharing.

---

## `std::shared_ptr`: many owners

`std::shared_ptr<T>` allows **shared ownership**: several `shared_ptr`s can point at the same object, and the object is destroyed only when the **last** one goes away. It works by keeping a *reference count* in a small heap-allocated **control block**; each copy increments the count, each destruction decrements it, and reaching zero frees the object.

```cpp
#include <iostream>
#include <memory>

int main() {
    auto a = std::make_shared<int>(42);     // count = 1
    {
        auto b = a;                          // copy: count = 2
        std::cout << "use_count = " << a.use_count() << "\n";   // 2
    }                                        // b dies: count = 1
    std::cout << "use_count = " << a.use_count() << "\n";       // 1
}                                            // a dies: count = 0 → int freed
```

Use `std::make_shared` to create one. The cost over `unique_ptr` is the control block and the counting, so reach for `shared_ptr` only when ownership is *really* shared — not as a lazy default.

### `std::weak_ptr`: observe without owning

Two `shared_ptr`s that point at each other form a **cycle**: each keeps the other's count above zero, so neither is ever freed — a leak. `std::weak_ptr` breaks the cycle. It refers to an object managed by a `shared_ptr` *without* contributing to the count; to use it you call `.lock()`, which returns a `shared_ptr` if the object is still alive or an empty one if it has been freed. Use a `weak_ptr` for a "back-reference" (a child pointing back at its parent) or any cache/observer that must not keep the object alive.

---

## Raw pointers are for borrowing

Raw pointers and references still have a job: **non-owning** access. A function that only *uses* an object for the duration of a call should take a reference or a raw pointer, not a smart pointer — it is borrowing, not taking ownership:

```cpp
double averageReading(const Sensor& sensor);   // borrows: looks, does not own
void process(Sensor* sensor);                  // borrows; nullptr allowed
```

The rule from [Ownership & RAII](ownership.md): smart pointers express *ownership*; a raw pointer or reference expresses *borrowing*. It is fine to pass a raw pointer obtained from `sensor.get()` to a function that borrows — but that function must **never `delete` it**. Ownership stays with the smart pointer.

---

## Smart pointers and threads

This is the part specific to AIS2203, and the most common source of confusion. There are two different objects in play — the `shared_ptr` *handle* and the *object it points to* — and they have **different** thread-safety properties.

**The reference count is thread-safe.** A `shared_ptr`'s control block uses an *atomic* counter, so copying and destroying `shared_ptr`s from multiple threads is safe — the count will never be corrupted, and the object is freed exactly once even under concurrent access.

**The pointed-to object is *not* thread-safe.** `shared_ptr` protects *its own bookkeeping*, nothing more. If two threads call mutating methods on the same underlying object through their `shared_ptr`s, that is an ordinary [data race](../Chapter2/sharing_data.md) and you still need a mutex or atomics around the object.

!!! danger "`shared_ptr` makes the count thread-safe, not the data"
    `auto copy = ptr;` in two threads is fine. `ptr->mutate()` in two threads is a data race. The smart pointer guarantees the object stays *alive* while anyone holds it; it does nothing to serialise *access* to the object. Wrap the shared object's state in a [mutex](../Chapter2/sharing_data.md) just as you would any other shared data.

With that clear, smart pointers solve two concrete concurrency problems:

**Transferring heap ownership to a thread** — move a `unique_ptr` in, and the thread owns and frees the object:

```cpp
auto data = std::make_unique<BigBuffer>();
std::jthread worker([d = std::move(data)] {     // thread now owns the buffer
    use(*d);
});                                              // freed when the thread's lambda dies
```

**Keeping shared data alive across threads** — the lifetime bug from [Creating Threads](../Chapter2/threads.md) (a thread outliving the data it borrowed) disappears if the thread is a *co-owner* rather than a borrower. Capture a `shared_ptr` **by value** into the thread's lambda, and the data is guaranteed to live as long as the thread does:

```cpp
auto config = std::make_shared<Config>();
std::jthread worker([config] {                   // captures a shared_ptr by value
    use(*config);                                // safe: this thread co-owns config
});                                              // config freed when the last owner releases it
```

Capturing `config` *by value* copies the `shared_ptr` (count goes up); capturing the underlying data *by reference* would reintroduce the dangling-reference bug. This "capture a `shared_ptr` by value to extend lifetime" is a standard, important pattern in concurrent C++.

---

## Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Two `shared_ptr`s from the same raw pointer (`shared_ptr<T>(p)` twice) | Two control blocks → double free | Always use `make_shared` / copy an existing `shared_ptr` |
| `shared_ptr` cycle | Neither side's count hits zero → leak | Make one side a `weak_ptr` |
| Capturing the *pointee* by reference into a thread | Dangling reference if the thread outlives it | Capture the `shared_ptr` **by value** |
| Using `shared_ptr` everywhere "to be safe" | Needless atomic counting overhead, unclear ownership | Default to `unique_ptr`; share only when you must |

---

## Summary

- Smart pointers put **ownership into the type**: `std::unique_ptr` (one owner, move-only, zero overhead) is the default; `std::shared_ptr` (many owners, reference-counted) is for genuinely shared ownership; `std::weak_ptr` observes without owning and breaks cycles.
- Create them with **`make_unique`/`make_shared`**, never bare `new`; transfer a `unique_ptr` with [`std::move`](move_semantics.md). Raw pointers/references are for **borrowing** only — never `delete` a borrowed pointer.
- Across threads: a `shared_ptr`'s **reference count is thread-safe**, but the **pointed-to object is not** — concurrent mutation still needs a [mutex](../Chapter2/sharing_data.md).
- **Move a `unique_ptr` into a thread** to hand over heap ownership; **capture a `shared_ptr` by value** into a thread's lambda to keep shared data alive and kill the dangling-reference bug.
- Default to `unique_ptr`; reach for `shared_ptr` only when ownership is truly shared. Next: [Templates](templates.md), for writing code that is generic over type.

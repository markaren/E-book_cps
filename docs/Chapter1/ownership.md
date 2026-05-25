# Ownership & RAII

In AIS1003 you met **RAII** as the reason files close themselves and `std::vector` frees its own memory. This chapter reframes the same idea around a sharper question — **ownership** — because that is the question concurrency and communication force you to answer constantly: for every resource your program holds, *who is responsible for releasing it, and when?* A single-threaded program can be sloppy about this and usually get away with it. A program with threads passing data between each other, or sockets shared across a system, cannot.

This page is the foundation for the next two: [Move Semantics](move_semantics.md) is how ownership is *transferred*, and [Smart Pointers](smart_pointers.md) are how it is *expressed in the type system*.

---

## Resources and their lifetimes

A **resource** is anything your program *acquires* and must later *release*: heap memory (`new`/`delete`), a file handle, a network socket, a locked mutex, a running thread. Every resource has the same shape — there is a moment it becomes yours, and a moment it must be given back. Leak it (never release) and you slowly exhaust the system; release it twice, or use it after release, and you get a crash or corruption.

**RAII** — *Resource Acquisition Is Initialization* — ties that lifetime to a C++ object's lifetime: acquire in the constructor, release in the destructor. Because C++ destroys objects **deterministically** at the end of their scope — including on an early `return` or an exception — the release is guaranteed to happen, exactly once, without you writing cleanup code at every exit.

You have already been relying on RAII types built on this principle:

| RAII type | Resource it owns | Released when… |
|-----------|------------------|----------------|
| `std::vector`, `std::string` | Heap memory | it goes out of scope |
| `std::ofstream` | An open file | it goes out of scope |
| `std::unique_ptr` | A heap object | it goes out of scope |
| `std::lock_guard` / `std::scoped_lock` | A locked mutex | it goes out of scope ([Sharing Data](../Chapter2/sharing_data.md)) |
| `std::jthread` | A running thread | it goes out of scope ([Creating Threads](../Chapter2/threads.md)) |

Notice the pattern across all of them: you never call a matching "release" by hand. There is no `vec.free()`, no `file.close()`, no `mutex.unlock()`, no `thread.join()` in correct modern code — the destructor does it. That is RAII's whole promise: **cleanup you cannot forget**.

```cpp
#include <iostream>

// A toy resource that announces its own lifetime.
class Resource {
public:
    Resource()  { std::cout << "acquired\n"; }
    ~Resource() { std::cout << "released\n"; }   // runs automatically at scope exit
};

int main() {
    std::cout << "before\n";
    {
        Resource r;                 // "acquired"
        std::cout << "using it\n";
    }                               // "released" — deterministically, here
    std::cout << "after\n";
}
```

---

## Ownership is a design decision

The deeper idea is that **someone owns each resource**, and ownership answers three questions you should ask of every resource in a design:

1. **Who owns it?** By default, exactly *one* owner — clear, single responsibility for release.
2. **Can ownership be transferred?** Handing the sole responsibility to someone else — *moving* it.
3. **Can it be shared?** Several parties keeping it alive together, released when the *last* one is done.

Those three map directly onto the tools in the rest of Part 1:

| Ownership model | Expressed with | Covered in |
|-----------------|----------------|------------|
| **Unique** — one owner | `std::unique_ptr`, a value member | [Smart Pointers](smart_pointers.md) |
| **Transferred** — ownership moves | `std::move` | [Move Semantics](move_semantics.md) |
| **Shared** — many owners | `std::shared_ptr` | [Smart Pointers](smart_pointers.md) |
| **Borrowed** — *no* ownership | reference `&`, raw pointer `*` | here |

That last row matters as much as the others: a reference or a raw pointer should mean "I am *looking at* this, but I do not own it and will not release it." A function that takes `const std::string&` borrows the string for the call; it must not assume the string outlives the call. This *borrowing* idea is exactly what the lifetime bugs in [Creating Threads](../Chapter2/threads.md) were about — a thread that borrows a local and outlives it.

---

## Prefer to own nothing directly: the rule of zero

Here is the most useful practical guideline in this chapter. The best class manages **no raw resource itself** — instead it is built from members that already manage their own (`std::vector`, `std::string`, `std::unique_ptr`). When every member is self-managing, the compiler-generated destructor, copy, and move operations are all automatically correct, and you write *none* of them:

```cpp
#include <string>
#include <vector>

// Owns memory for the name and the samples — but manages nothing by hand.
class SensorLog {
public:
    void add(double sample) { samples_.push_back(sample); }

private:
    std::string name_;              // manages its own memory
    std::vector<double> samples_;   // manages its own memory
};
// No destructor, no copy/move code needed — and none can be forgotten.
```

This is the **rule of zero**: design types so you need to write *zero* of the special member functions, because your members handle their own resources. Almost every class you write should follow it.

The counterpart is the **rule of five**: *if* you genuinely manage a raw resource directly (a bare `new`, a C library handle), then writing any one of the five special members — destructor, copy constructor, copy assignment, move constructor, move assignment — means you must think about all five, because the defaults will be wrong. But the modern advice is to avoid being in that situation at all: wrap the raw resource in a `std::unique_ptr` or a small handle class, and you are back to the rule of zero.

!!! tip "Manage one resource per class, or none"
    If you must wrap a raw resource, give that class exactly *one* job: own that one resource and nothing else. Then any class that needs it holds it as a member and stays at the rule of zero. This keeps the tricky, hand-written ownership code tiny and isolated.

---

## Why this matters more with threads

Single-threaded, an object's lifetime follows the call stack and is easy to track. Add concurrency and ownership becomes the thing that keeps you out of trouble:

- A thread that **borrows** data must not outlive it — the dangling-reference bug from [Creating Threads](../Chapter2/threads.md). Ownership tells you whether borrowing is even safe.
- To hand data **to** a thread for keeps, you **transfer ownership** into it — that is [`std::move`](move_semantics.md), and it is why a `std::thread` and a `std::unique_ptr` are move-only.
- To let several threads **share** data, you give them **shared ownership** with a [`std::shared_ptr`](smart_pointers.md), so the data lives exactly as long as the last thread needs it.
- A lock is a resource owned for the duration of a critical section — which is why `std::lock_guard` is RAII, not a manual `lock()`/`unlock()`.

Get ownership right and a whole category of concurrency bugs — leaks, double-frees, use-after-free, threads outliving their data — simply cannot happen.

---

## Summary

- A **resource** is anything acquired then released: memory, files, sockets, locks, threads. **RAII** binds its lifetime to an object's, so the destructor releases it **deterministically** — even on exceptions — and you never write the cleanup.
- The standard RAII types you already use (`vector`, `ofstream`, `unique_ptr`, `lock_guard`, `jthread`) mean you should almost never call `free`/`close`/`unlock`/`join` by hand.
- For every resource, decide **ownership**: unique (one owner), transferred ([move](move_semantics.md)), shared ([`shared_ptr`](smart_pointers.md)), or merely borrowed (a non-owning reference/raw pointer).
- Follow the **rule of zero**: build classes from self-managing members so the compiler writes correct copy/move/destroy for you. Only hand-manage a raw resource as a last resort, in a one-job class — and then the **rule of five** applies.
- Concurrency makes ownership urgent: it decides whether a thread may safely borrow data, and dictates that you **move** data into a thread or **share** it via `shared_ptr`. Next: [Move Semantics](move_semantics.md), the mechanism of transfer.

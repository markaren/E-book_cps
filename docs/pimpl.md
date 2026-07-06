# The Pimpl Idiom

**Pimpl** — short for *pointer to implementation* — is a C++ technique for hiding a class's private members behind a pointer, so they live in the `.cpp` file rather than the header. It is built directly on [`std::unique_ptr`](Chapter1/smart_pointers.md) and [move semantics](Chapter1/move_semantics.md), and it has one famous gotcha (the destructor) that trips up nearly everyone the first time. This page explains what problem it solves, the exact pattern, and the rules you must follow to make it compile.

---

## The problem: the header sees too much

In C++, a class's **private** members are still written in its header. That feels like a contradiction — they are private, yet everyone can see them — but it is forced by the language: to create a `Widget` on the stack or as a member, the compiler must know its *size and layout*, and that means seeing every data member, private ones included.

Two costs follow from that:

- **Recompilation cascades.** Every translation unit that `#include`s `widget.hpp` depends on the *full* class layout. Add or change a single private field — even one nobody outside the class touches — and **every file that includes the header must recompile**. In a large project, one trivial edit can trigger a rebuild of hundreds of files. This coupling between a header and its clients is exactly what slows [large builds](Chapter5/cmake.md) to a crawl.
- **Leaked dependencies.** If a private member has type `cv::VideoCapture` or an Asio `io_context`, the header must `#include` that library's header. Now *every* client of your class transitively pulls in [OpenCV](Chapter6/opencv.md) or [Asio](Chapter4/networking.md) — longer compiles for them, and your implementation detail leaking into their world.

Pimpl removes both costs by making the header's view of the class **never change** when the implementation does.

---

## The pattern

Replace all the private members with a single pointer to an opaque implementation type. The header declares that type but never defines it; the `.cpp` defines it.

```cpp title="widget.hpp"
#include <memory>
#include <string>

class Widget {
public:
    explicit Widget(std::string name);
    ~Widget();                          // declared here, DEFINED in the .cpp — see below

    Widget(Widget&&) noexcept;          // move ops must be declared too
    Widget& operator=(Widget&&) noexcept;

    void bump();
    int  value() const;

private:
    struct Impl;                        // forward declaration only — incomplete type
    std::unique_ptr<Impl> impl_;        // the one and only data member
};
```

The header now reveals **nothing** about the internals: not the data members, not their types, not the libraries they need. It includes `<memory>` (for `unique_ptr`) and whatever the *public interface* mentions — nothing more.

```cpp title="widget.cpp"
#include "widget.hpp"
#include <some_heavy_dependency.hpp>    // hidden from every client

struct Widget::Impl {                   // the full definition lives here
    std::string name;
    int counter = 0;
    HeavyThing thing;                   // the real data — and its heavy header
};

Widget::Widget(std::string name)
    : impl_(std::make_unique<Impl>()) {
    impl_->name = std::move(name);
}

Widget::~Widget() = default;            // Impl is COMPLETE here, so this is legal

Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;

void Widget::bump()        { ++impl_->counter; }
int  Widget::value() const { return impl_->counter; }
```

Now change anything inside `Impl` — add a field, swap a dependency — and only `widget.cpp` recompiles. Every client sees the same one-pointer `Widget` it always did. That is the **compilation firewall**.

---

## The destructor rule

Here is the trap. `std::unique_ptr<Impl>` must call `delete` on its pointer, and **deleting a pointer requires the complete type** — the compiler has to know `Impl`'s size and destructor to destroy it correctly. That `delete` happens inside `Widget`'s destructor.

If you do *not* declare `~Widget()`, or you write `~Widget() = default;` **in the header**, the compiler generates it right there — at a point where `Impl` is still only forward-declared, an **incomplete type**. The result is a hard compile error, usually something like *"invalid application of `sizeof` to an incomplete type"* or a `static_assert` about deleting an incomplete type.

!!! danger "Declare the destructor in the header, define it in the `.cpp`"
    A `unique_ptr` to an **incomplete** type compiles fine — but *destroying* it needs the complete type. So the special members that destroy `Impl` must be defined where `Impl` is complete: in the `.cpp`, **below** `struct Widget::Impl { … };`.

    `~Widget() = default;` is correct — but the `= default` must be in the **`.cpp`**, not the header. The same applies to the **move-assignment** operator (it destroys the old `Impl` before taking the new one) and any **copy** operations you add.

This is why the header above *declares* `~Widget()`, the moves, etc. but leaves them empty: each is `= default`-ed in the `.cpp` where the definition is visible.

---

## Special members

Declaring a destructor suppresses the compiler-generated move operations, so once you write `~Widget()` you must bring the moves back yourself — and, like the destructor, define them in the `.cpp`. Copying is gone entirely unless you write it, because [`unique_ptr` is move-only](Chapter1/smart_pointers.md).

| Member | What to do | Why |
|--------|-----------|-----|
| Constructor | Define in `.cpp`; `make_unique<Impl>()` | Allocates the hidden `Impl` |
| **Destructor** | **Declare in header, `= default` in `.cpp`** | Needs complete `Impl` to delete it |
| Move ctor / move assign | Declare in header, `= default` in `.cpp` | Suppressed by the destructor; move-assign also destroys the old `Impl` |
| Copy ctor / copy assign | Deleted by default; if you need them, write a **deep copy** in the `.cpp` | `unique_ptr` can't be copied — you must clone `*impl_` |

If a `Widget` should be copyable, the copy must duplicate the pointee, not the pointer:

```cpp title="widget.cpp (only if Widget must be copyable)"
Widget::Widget(const Widget& other)
    : impl_(std::make_unique<Impl>(*other.impl_)) {}   // deep copy of the Impl

Widget& Widget::operator=(const Widget& other) {
    *impl_ = *other.impl_;                              // assumes both are non-null
    return *this;
}
```

---

## A runnable example

The firewall benefit only shows across separate files, but the *mechanics* — the forward-declared `Impl`, the out-of-line destructor, the move — fit in one translation unit you can run. In a real project the three sections below would be `widget.hpp`, `widget.cpp`, and `main.cpp`.

```cpp
#include <iostream>
#include <memory>
#include <string>

// ---- widget.hpp ----
class Widget {
public:
    explicit Widget(std::string name);
    ~Widget();                                  // declared, not defined, here
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;

    void bump();
    void report() const;

private:
    struct Impl;                                // opaque
    std::unique_ptr<Impl> impl_;
};

// ---- widget.cpp ----
struct Widget::Impl {                           // full definition, hidden from clients
    std::string name;
    int counter = 0;
};

Widget::Widget(std::string name) : impl_(std::make_unique<Impl>()) {
    impl_->name = std::move(name);
}
Widget::~Widget() = default;                    // Impl complete here — OK
Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;

void Widget::bump() { ++impl_->counter; }
void Widget::report() const {
    std::cout << impl_->name << " = " << impl_->counter << "\n";
}

// ---- main.cpp ----
int main() {
    Widget w("sensor");
    w.bump();
    w.bump();
    w.report();                                 // sensor = 2

    Widget moved = std::move(w);                // works because we defined the move ops
    moved.report();                             // sensor = 2
}
```

Move `~Widget() = default;` up into the class body and the program stops compiling — that is the destructor rule in action.

---

## The cost, and when to pay it

Pimpl is not free, and it is not for every class:

- **One extra heap allocation** per object (the `Impl`) and **one pointer indirection** on every member access.
- Methods defined in the `.cpp` **cannot be inlined** into client code.
- More boilerplate — the special members you would otherwise get for free.

So treat it as a deliberate trade: you pay a little runtime cost and some boilerplate to buy **build-time decoupling**, **hidden dependencies**, and a **stable [ABI](portability.md)**. Reach for it when:

- A header is **included widely** and its internals **change often** — the firewall cuts rebuild times.
- A class's private members would **drag heavy or platform-specific headers** into the public interface — wrap [OpenCV](Chapter6/opencv.md), [Asio](Chapter4/networking.md), or a vendor SDK and keep the `#include` in the `.cpp`.
- You ship a **shared library** whose public classes must keep a stable binary layout as you evolve the internals.

Skip it for small value types, performance-critical hot-path classes where the indirection bites, and templates (the implementation has to be visible to instantiate, so pimpl and templates do not mix).

---

## Summary

- **Pimpl** moves a class's private members into an opaque `struct Impl`, **forward-declared** in the header and **defined** in the `.cpp`, reached through a single [`std::unique_ptr<Impl>`](Chapter1/smart_pointers.md).
- It buys a **compilation firewall** (changing the implementation recompiles only one `.cpp`, not every client) and **hides dependencies** so heavy headers like [OpenCV](Chapter6/opencv.md) or [Asio](Chapter4/networking.md) never leak into the public interface — both directly attacking [slow large builds](Chapter5/cmake.md).
- **The destructor rule:** `unique_ptr` needs the *complete* type to delete it, so **declare `~Widget()` in the header and `= default` it in the `.cpp`** (below `struct Impl`). The same goes for **move-assignment** and any **copy** operations.
- Declaring the destructor suppresses the implicit moves — re-declare them and `= default` them in the `.cpp`. Copying needs a hand-written **deep copy**, since `unique_ptr` is [move-only](Chapter1/move_semantics.md).
- It costs an allocation, an indirection, and lost inlining — apply it to **widely-included, often-changing, dependency-heavy, or ABI-stable** classes, not to everything.

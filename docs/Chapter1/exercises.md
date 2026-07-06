# Chapter 1 Exercises

Work through these after reading Chapter 1. **Try each one yourself before revealing the solution.** Type the code into CLion and run it.

When you open a solution it appears **blurred** — click it once more to reveal it.

Each program has its own `main()`. None of these *start* a thread, though exercise 1 uses `std::this_thread::sleep_for` (just to have something to time), so on Linux link `Threads::Threads` as [Getting Started](../getting_started.md#linking-the-thread-library) showed; on Windows it builds as-is. Every one of these is a tool you will reuse in [Part 2](../Chapter2/processes_threads.md) and beyond.

---

## 1. A timer that stops itself

*Practises: [Ownership & RAII](ownership.md), [Time with std::chrono](chrono.md)*

Write a `ScopedTimer` class that records the time when it is constructed and, in its **destructor**, prints how long it lived. Use it by creating one at the top of a scope; when the scope ends, it reports the elapsed time automatically — no explicit "stop" call.

> Hint: store a `std::chrono::steady_clock::time_point` from `now()` in the constructor. In the destructor, subtract it from a fresh `now()` and `duration_cast` to milliseconds. This is RAII applied to *measurement* — the destructor is the "stop".

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <chrono>
    #include <iostream>
    #include <string>
    #include <thread>

    class ScopedTimer {
    public:
        explicit ScopedTimer(std::string name)
            : name_(std::move(name)),                       // sink: move the name in
              start_(std::chrono::steady_clock::now()) {}

        ~ScopedTimer() {
            auto elapsed = std::chrono::steady_clock::now() - start_;
            auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(elapsed).count();
            std::cout << name_ << " took " << ms << " ms\n";
        }

    private:
        std::string name_;
        std::chrono::steady_clock::time_point start_;
    };

    int main() {
        {
            ScopedTimer t("work");
            std::this_thread::sleep_for(std::chrono::milliseconds(120));
        }   // t's destructor runs here and prints the elapsed time
    }
    ```

    Output (approximately):

    ```
    work took 120 ms
    ```

    The timer needs no "stop" call: the destructor *is* the stop, and it runs deterministically when the scope ends — even on an early `return` or an exception. That is [RAII](ownership.md) used for something other than freeing memory. Note `steady_clock` ([the right clock for intervals](chrono.md)) and the `std::move(name)` [sink](move_semantics.md) into the member.

    </div>

---

## 2. Hand over the buffer

*Practises: [Move Semantics](move_semantics.md), [Smart Pointers](smart_pointers.md)*

Write a function `makeBuffer(int id)` that returns a `std::unique_ptr<Buffer>`. In `main`, store a couple of buffers in a `std::vector`, then **transfer ownership** of one out of the vector into a separate variable and show the vector's slot is now empty.

> Hint: `std::make_unique<Buffer>(id)` and return it (returning a local moves automatically — no `std::move` needed). To move one *out* of the vector, `std::move(vec[0])`; afterwards `vec[0]` is null. A `unique_ptr` is move-only, so this is all transfer, never copy.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <memory>
    #include <vector>

    struct Buffer {
        int id;
        explicit Buffer(int i) : id(i) {}
    };

    std::unique_ptr<Buffer> makeBuffer(int id) {
        return std::make_unique<Buffer>(id);   // moved out on return (no std::move needed)
    }

    int main() {
        std::vector<std::unique_ptr<Buffer>> buffers;
        buffers.push_back(makeBuffer(1));
        buffers.push_back(makeBuffer(2));

        std::unique_ptr<Buffer> taken = std::move(buffers[0]);   // transfer ownership out

        std::cout << "taken owns buffer " << taken->id << "\n";              // 1
        std::cout << "slot 0 is now " << (buffers[0] ? "set" : "null") << "\n"; // null
    }
    ```

    Ownership of buffer 1 moved from the vector slot into `taken`, leaving the slot null — a [moved-from](move_semantics.md) `unique_ptr` is empty. Nothing was copied (a `unique_ptr` *cannot* be copied), and nothing leaked: each buffer is freed exactly once, when its sole owner is destroyed. Try adding `auto copy = taken;` and watch it fail to compile — that is the type system enforcing single ownership.

    </div>

---

## 3. A stack for any type

*Practises: [Templates & Generic Code](templates.md)*

Write a class template `Stack<T>` backed by a `std::vector<T>`, with `push`, `pop` (returning the value), `empty`, and `size`. `pop` on an empty stack should throw. Prove it works for `int` *and* `std::string` with the same template.

> Hint: the template parameter `T` is the element type. `push` takes a `T` by value and `std::move`s it into the vector; `pop` moves the back element out, pops it, and returns it. Mark `empty`/`size` `const`.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <stdexcept>
    #include <string>
    #include <vector>

    template <typename T>
    class Stack {
    public:
        void push(T value) { data_.push_back(std::move(value)); }

        T pop() {
            if (data_.empty()) {
                throw std::out_of_range("pop from empty stack");
            }
            T value = std::move(data_.back());
            data_.pop_back();
            return value;
        }

        bool empty() const { return data_.empty(); }
        std::size_t size() const { return data_.size(); }

    private:
        std::vector<T> data_;
    };

    int main() {
        Stack<int> numbers;
        numbers.push(1);
        numbers.push(2);
        numbers.push(3);
        while (!numbers.empty()) {
            std::cout << numbers.pop() << " ";   // 3 2 1
        }
        std::cout << "\n";

        Stack<std::string> words;                // same template, different type
        words.push("hello");
        words.push("world");
        std::cout << words.pop() << "\n";         // world
    }
    ```

    One class definition serves `Stack<int>` and `Stack<std::string>` — the compiler [generates](templates.md) a separate, fully type-checked version for each. This is the same shape as the `ThreadSafeQueue<T>` from the Templates chapter; add a [mutex and condition variable](../Chapter2/condition_variables.md) and you would have the thread-safe version a [pool](../Chapter3/thread_pools.md) is built on.

    </div>

---

## 4. A button you can wire up

*Practises: [Lambdas & std::function](lambdas.md)*

Write a `Button` class that lets callers register click handlers and then "fires" them all when clicked. A handler is any `std::function<void()>`. In `main`, register two handlers as lambdas — one that prints, one that increments a counter — then click the button twice.

> Hint: store the handlers in a `std::vector<std::function<void()>>`; `onClick` takes one by value and `std::move`s it in; `click()` loops calling each. The counting handler captures the counter by reference — safe *here* because the button and the counter live in the same scope.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <functional>
    #include <iostream>
    #include <vector>

    class Button {
    public:
        void onClick(std::function<void()> handler) {
            handlers_.push_back(std::move(handler));
        }

        void click() const {
            for (const auto& handler : handlers_) {
                handler();
            }
        }

    private:
        std::vector<std::function<void()>> handlers_;
    };

    int main() {
        Button button;
        int clicks = 0;

        button.onClick([] { std::cout << "beep\n"; });
        button.onClick([&clicks] { ++clicks; });   // by-ref capture, safe in this scope

        button.click();
        button.click();
        std::cout << "clicks = " << clicks << "\n";   // 2
    }
    ```

    `std::function<void()>` lets the button store handlers of *different* concrete lambda types in one vector — the [type-erasure](lambdas.md) that callback systems and [thread-pool](../Chapter3/thread_pools.md) task queues rely on. The `[&clicks]` capture is fine because `button` does not outlive `clicks`; if the button (and its stored handlers) lived *longer* than `clicks` — say, returned from this function — that reference would dangle, and you would capture by value or co-own with a [`shared_ptr`](smart_pointers.md) instead. That lifetime question is the heart of using callbacks safely.

    </div>

---

## 5. A cycle that leaks, and the fix

*Practises: [Smart Pointers](smart_pointers.md)*

Write a `Node` struct with a name and a `std::shared_ptr<Node>` to another node, printing in its constructor and destructor. In `main`, create two nodes and point each one's link at the *other*, then let them go out of scope. Run it and notice the destructors **never print** — the two nodes keep each other alive. Then fix the leak by making one of the two links a `std::weak_ptr` and confirm both destructors fire.

> Hint: two `shared_ptr`s pointing at each other form a **cycle** — each holds the other's reference count above zero, so neither ever reaches zero. Break it by making the "back" link a `weak_ptr`, which observes without owning. This is the exact shape of a scene-graph parent link: a parent owns its children with `shared_ptr`, and each child points *back* at its parent with a `weak_ptr` — the pattern you will meet in the [Threepp](https://github.com/markaren/threepp) simulator.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    First, the **broken** version — a cycle that leaks:

    ```cpp
    #include <iostream>
    #include <memory>
    #include <string>

    struct Node {
        std::string name;
        std::shared_ptr<Node> other;   // strong link both ways → cycle
        explicit Node(std::string n) : name(std::move(n)) {
            std::cout << name << " constructed\n";
        }
        ~Node() { std::cout << name << " destroyed\n"; }
    };

    int main() {
        {
            auto a = std::make_shared<Node>("A");
            auto b = std::make_shared<Node>("B");
            a->other = b;    // A keeps B alive
            b->other = a;    // B keeps A alive → neither count reaches 0
        }   // a and b leave scope — but the two Nodes still co-own each other
        std::cout << "scope exited\n";
    }
    ```

    ```
    A constructed
    B constructed
    scope exited
    ```

    No `destroyed` lines: both `Node`s leaked. When `a` and `b` go out of scope, each object's count drops from 2 to 1 — the link held by the *other* node keeps it above zero — so neither is ever freed.

    Now the **fix**: make the back-link a `weak_ptr`, which does not raise the count.

    ```cpp
    #include <iostream>
    #include <memory>
    #include <string>

    struct Node {
        std::string name;
        std::shared_ptr<Node> child;   // strong: a parent owns its child
        std::weak_ptr<Node> parent;    // weak: the child only observes its parent
        explicit Node(std::string n) : name(std::move(n)) {
            std::cout << name << " constructed\n";
        }
        ~Node() { std::cout << name << " destroyed\n"; }
    };

    int main() {
        {
            auto parent = std::make_shared<Node>("parent");
            auto child  = std::make_shared<Node>("child");
            parent->child = child;     // strong link down the tree
            child->parent = parent;    // weak link back up — no cycle

            if (auto p = child->parent.lock()) {   // promote to shared_ptr if alive
                std::cout << "child sees parent: " << p->name << "\n";
            }
        }   // both freed here — no cycle keeps them alive
        std::cout << "scope exited\n";
    }
    ```

    ```
    parent constructed
    child constructed
    child sees parent: parent
    parent destroyed
    child destroyed
    scope exited
    ```

    Both destructors now run. The `weak_ptr` back-link lets the child *reach* its parent — via `.lock()`, which hands back a `shared_ptr` only while the parent is still alive — without keeping it alive. This is the standard shape of a [scene graph](smart_pointers.md): own **downward** with `shared_ptr`, refer **upward** with `weak_ptr`. Threepp's `Object3D` is built exactly this way.

    </div>

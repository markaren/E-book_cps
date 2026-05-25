# Using AI for Coding

AI assistants will happily write C++ for you — a whole threaded server, a CMake file, a parsing routine — in seconds. They are genuinely useful tools and genuinely treacherous teachers, and using them well takes a few specific habits. This matters more in AIS2203 than it did in AIS1003, because the kind of code this course is about is exactly the kind AI is most likely to get subtly, invisibly wrong.

---

## Where they help

Used well, an AI assistant is a fast, patient pair-programmer:

- **Explaining errors.** Paste a screenful of [template error](Chapter1/templates.md) or a linker message and ask what it means — often quicker than decoding it yourself.
- **Boilerplate and scaffolding.** A `CMakeLists.txt` skeleton, a class with getters, a test harness — mechanical code you understand but would rather not type.
- **Explaining unfamiliar code or APIs.** "What does this `std::condition_variable` snippet do?" is a good question to ask.
- **Exploring approaches.** "What are the ways to get a result back from a thread?" can surface options ([futures](Chapter3/futures.md), a [thread pool](Chapter3/thread_pools.md)) you then go and learn properly.

For these, AI accelerates work you could do yourself — which is the safe zone.

---

## The concurrency trap

Here is the danger specific to this course. **AI confidently produces plausible-looking concurrent code that contains subtle [data races](Chapter2/sharing_data.md) or [deadlocks](Chapter2/sharing_data.md).** It will give you a threaded counter, a producer/consumer, a thread pool — and the code will compile, run once, and produce the right answer. Then it will be wrong one run in a thousand.

This happens because an AI model pattern-matches against code it has seen; it does not *reason about the timing* of your specific threads, and it cannot *feel* a race the way running the code under load would reveal. A concurrency bug does not show up in a quick read or a single execution — the very properties that make these bugs [hard to debug](debugging_concurrency.md) also make them hard for an AI (or a human skimming) to catch. The plausibility of the output is the trap: it *looks* right.

!!! danger "Treat AI-generated concurrent code as unverified"
    Any threaded code an assistant gives you should be reviewed as if a stranger wrote it and run under [**ThreadSanitizer**](debugging_concurrency.md) before you trust it. "It compiled and printed the right number" is not evidence that concurrent code is correct — it is the *default* appearance of a race that has not bitten yet.

---

## Don't outsource the understanding

The deeper risk is quieter: letting the AI understand the synchronization *for* you. If you accept a `std::mutex` / `std::condition_variable` dance you cannot explain, you have a program you cannot debug — and concurrent programs *will* eventually need debugging. The moment it deadlocks or races in week 10, you are stuck, because the understanding was never yours.

The entire point of AIS2203 is for **you** to understand threads, synchronization, and communication well enough to build and fix a real system. An assistant that writes the code while you stay ignorant defeats that — and the project and oral exam will reveal whether the understanding is there. Use AI to *get* understanding faster, not to *skip* it.

---

## Good habits

- **Ask it to explain, not just produce.** "Why is this correct? What could race here? What happens if two threads call this at once?" turns the tool from a code vending machine into a tutor.
- **Verify against authoritative sources.** AI hallucinates plausible-but-wrong API signatures and behaviour; check [cppreference](https://en.cppreference.com/) for anything you are unsure of.
- **Test it — and for threads, run TSan.** Logic gets a [test](Chapter6/cmake.md); concurrent code gets [ThreadSanitizer](debugging_concurrency.md). Do not promote AI code to "working" on the strength of one run.
- **Read every line before you use it.** If you cannot explain a line to a classmate, do not put it in your project.
- **Be honest about its use.** Follow the course's policy on AI assistance; the goal is your learning, and being straight about what you wrote versus generated is part of that.

---

## Summary

- AI assistants are **excellent tools and treacherous teachers**: great for explaining errors, boilerplate, and exploring approaches — the work you could do yourself, only faster.
- The course-specific trap: AI produces **plausible concurrent code with hidden [races](Chapter2/sharing_data.md)/[deadlocks](Chapter2/sharing_data.md)** that compiles and runs once correctly. Treat any AI-generated threaded code as **unverified** and run it under [ThreadSanitizer](debugging_concurrency.md).
- **Don't outsource understanding** — a synchronization you can't explain is a bug you can't fix, and understanding threads/comms is the whole point of AIS2203.
- Habits: **ask it to explain**, verify against **cppreference**, **test** (TSan for threads), read every line, and be honest about its use.

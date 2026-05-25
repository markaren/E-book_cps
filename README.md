# AIS2203 E-book

Welcome to the E-book for the course [AIS2203 — Computer Engineering for Cyber-Physical Systems](https://www.ntnu.no/studier/emner/AIS2203#tab=omEmnet).

This book covers advanced C++, concurrency, real-time programming, data communication between platforms, and embedded Linux. It picks up where [AIS1003](https://github.com/markaren/E-book_cpp) leaves off.

**Read online: [markaren.github.io/E-book_cps](https://markaren.github.io/E-book_cps/)**

### Additional resources

This book supports the course, but it is not a complete reference. Supplement it with other resources.

#### Books

- __C++ Concurrency in Action, Second Edition__ (Anthony Williams). The definitive treatment of the C++ thread library and the memory model — the main companion for the concurrency chapters.
- __A Tour of C++__ (Bjarne Stroustrup). A concise overview of modern C++ for programmers who already know the basics.
- __Effective Modern C++__ (Scott Meyers). 42 specific ways to improve your use of C++11/14 — move semantics, smart pointers, and concurrency.

#### Online

- [__cppreference__](https://en.cppreference.com/w/) — reliable, up-to-date documentation of the language and standard library.
- [__Boost.Asio documentation__](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html) — the de facto networking and async I/O library for C++.
- [__Stack Overflow__](https://stackoverflow.com/) — most questions have already been asked and answered.

### [Common terms](docs/terms.md)

### [Frequently asked questions (FAQ)](docs/faq.md)

---

## Building the site locally

The site is built with [MkDocs](https://www.mkdocs.org/) and the [Material](https://squidfunk.github.io/mkdocs-material/) theme.

```bash
pip install -r requirements.txt
mkdocs serve     # live preview at http://127.0.0.1:8000
mkdocs build     # produces ./site
```

The site is published automatically to GitHub Pages on every push to `main`/`master` via [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml).

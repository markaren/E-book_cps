# Embedded Linux

AIS1003 mentioned Arduino — a microcontroller with no operating system, where your code *is* the whole show. Between that bare-metal world and a desktop PC sits a middle ground that runs much of modern robotics: **embedded Linux**, on boards like the Raspberry Pi. This page is a conceptual overview — worth understanding because it is where real robots run, even though **this year the course works entirely in a 3D simulator** rather than on physical boards. Treat it as background, not a hands-on task.

!!! note "No hardware this year"
    The project runs in a [Threepp](https://github.com/markaren/threepp)-based 3D simulator, so you will *not* cross-compile, wire up GPIO, or deploy to a real Pi for coursework. The value of this page is conceptual: the C++ you write for the simulated robot is the *same* C++ that would run on a real embedded-Linux device, so knowing how that deployment works tells you what your code would have to deal with in the field.

---

## Where it sits

| | Arduino (bare metal) | Embedded Linux (Pi) | Desktop PC |
|---|---|---|---|
| Operating system | None | Linux | Windows / Linux / macOS |
| Threads & processes | No real OS threads | Full POSIX threads | Full |
| Networking | Add-on shields | Built-in | Built-in |
| Your program | The only thing running | One process among many | One process among many |

The key fact: a Raspberry Pi runs the **same** C++, the **same** standard library, and the **same** `<thread>`, sockets, and OpenCV you use on your laptop. So almost everything in this book runs there unchanged — which is exactly why a simulator on your PC is a faithful stand-in for the real device. Embedded Linux is "a smaller computer," not "a different language."

What *does* differ is performance (a Pi is far slower and may have only a few cores), the presence of real hardware pins, and how you get your program onto the device and keep it running. Three concepts cover that.

---

## Cross-compiling

A Pi has an **ARM** processor; your laptop is almost certainly **x86-64**. Compiled code is machine code for one instruction set, so an x86-64 binary cannot run on ARM. There are two ways to get an ARM binary:

- **Compile on the Pi** — copy your source across and run `cmake` there. Simple, but slow: a small Pi can take many minutes to build a non-trivial project.
- **Cross-compile** — build the ARM binary on your fast PC using a *cross-compiler* (a compiler that runs on x86-64 but emits ARM code), then copy the finished executable over. This is the professional approach, configured in CMake with a **toolchain file** that names the cross-compiler and target.

The terms worth knowing: the **host** is what you build on, the **target** is what you run on, and a **toolchain** is the compiler/linker that produces code for the target. (None of this is needed when you simulate — your PC builds and runs the same binary.)

---

## Talking to hardware

A microcontroller toggles pins directly; on Linux you are in **user space**, so you reach hardware through the kernel:

- **GPIO** (general-purpose I/O pins) — switch an LED, read a button — via the modern `libgpiod` library (or the older sysfs interface).
- **I²C** and **SPI** — the [synchronous serial buses](Chapter4/serial.md) from the communication chapters — exposed as device files (`/dev/i2c-*`, `/dev/spidev*`) that libraries wrap.

In simulation, "hardware" is a model in the [virtual world](Chapter7/virtual_environments.md): a simulated sensor returns a value, a simulated actuator moves — your control code reads and writes the same abstractions it would on real pins, which is what makes the sim-to-real transition realistic.

---

## Deploying and running

On a desktop you launch a program by double-clicking or running it from a shell. On a headless embedded device (no screen, reached over the network) the pattern is:

- **Copy** the binary over **SSH** (`scp`), or build remotely.
- **Run it as a service** with **systemd** — the Linux service manager — so the program starts automatically on boot, restarts if it crashes, and logs centrally. A small unit file describes how to start and supervise it.

This is how a robot's control program runs untended: powered on, the device boots, systemd starts your program, and it runs until shut down. In the simulator the equivalent is simply launching your program on your PC against the virtual world.

---

## Real-time on Linux

A general-purpose Linux is **not** a hard real-time OS — the scheduler time-slices your program against everything else, so a [control loop](Chapter3/real_time.md) can be delayed unpredictably. For soft and firm deadlines this is usually fine; for tighter ones, Linux offers real-time scheduling priorities (`SCHED_FIFO`), and a `PREEMPT_RT` kernel bounds worst-case latency. The same caveats apply whether the loop drives a real motor or a simulated one — which is another reason the [Real-Time & Timing](Chapter3/real_time.md) material matters even in simulation.

---

## Summary

- **Embedded Linux** (e.g. a Raspberry Pi) sits between a bare-metal Arduino and a desktop PC; it runs the **same C++ and standard library** as your laptop, so this book's code runs there unchanged — and a PC simulator is a faithful stand-in.
- **This year the course uses a 3D simulator, not real boards**, so this page is conceptual background rather than a hands-on task.
- The differences that matter on real hardware: **cross-compiling** (build ARM binaries on your PC via a toolchain file), **talking to hardware** (GPIO and I²C/SPI from user space), and **deploying** (copy over SSH, run as a **systemd** service). In simulation, each has a virtual equivalent.
- Linux is **not hard real-time**; the [timing](Chapter3/real_time.md) discipline applies in simulation and on hardware alike.

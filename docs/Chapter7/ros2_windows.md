# ROS2 on Windows

ROS2 grew up on Ubuntu. Its official Windows support exists but is fragile and painful to install, which is a problem for a course where [Windows and CLion dominate](../getting_started.md). The clean way in is **RoboStack**.

---

## RoboStack: ROS2 as conda packages

**[RoboStack](https://robostack.github.io/)** repackages ROS2 as **conda packages**. Instead of a system-wide ROS2 install with its own installer, you get a self-contained **conda environment** — a folder holding ROS2, its dependencies, and the right compilers — that works natively on Windows (and Linux and macOS) with no WSL, no Docker required. This course uses the **Jazzy** distribution through RoboStack.

The environment is managed with **pixi**, a modern conda-compatible project tool. A `pixi.toml` file declares the environment (which ROS2 packages, which build tools); pixi resolves and installs them into a local `.pixi/` folder. Because it is per-project and reproducible, everyone on the team gets the identical toolchain — the same reproducibility argument as [`vcpkg.json`](../Chapter5/dependencies.md) in Part 5, applied to a whole ROS2 stack.

---

## Setup

Follow the course's [`ROBOSTACK.md`](https://github.com/markaren/ros2_moveit_ur_demo/blob/main/doc/ROBOSTACK.md) for the authoritative steps; the shape is:

1. **Install pixi**, then create the pixi project at **`C:\robostack`** (place the `pixi.toml` there). The environment is named **`jazzy`**.
2. **Enter the environment** — this is the ROS2 equivalent of "sourcing setup.bash":

    ```powershell
    pixi shell -e jazzy
    ```

    Every ROS2 command (`ros2`, `colcon`, your builds) must run **inside this shell**. Outside it, ROS2 does not exist on your machine.

3. **Build with colcon**, ROS2's build tool. On Windows you must pass **`--merge-install`**:

    ```powershell
    colcon build --merge-install
    ```

!!! note "Why `--merge-install` on Windows"
    By default colcon gives each package its **own** install folder (an *isolated* install), which relies on symlinks and long, layered `PATH` entries. Windows handles symlinks and long paths badly, so `--merge-install` puts **all** packages into one shared `install/` tree instead — one set of paths, no symlinks. Use it consistently: mixing merged and isolated installs in one workspace confuses the loader.

---

## Who owns what: don't cross the streams

The RoboStack environment is a **complete, self-contained world**. When you `pixi shell -e jazzy`, that environment's folders go on your `PATH` and become where the build finds libraries: on Windows, `find_package` looks under the pixi env via `CMAKE_PREFIX_PATH` pointed at `C:/robostack/.pixi/envs/jazzy/Library` (you will see this in the [workspace CMake](ros2_workspace.md)). ROS2, its DDS implementation, Boost, OpenCV, and the compiler that built them all come from **one place** — the conda environment.

This is the same **dependency-ownership** discipline from [Part 5](../Chapter5/dependencies.md), and it has one hard rule:

!!! danger "Do not mix conda DLLs with vcpkg copies of the same library"
    If your project also uses [vcpkg](../Chapter5/dependencies.md), you can end up with **two copies of the same library** — say OpenCV — one from the conda env and one from vcpkg, each built with a possibly different compiler and settings. Loading both, or linking against one and finding the other at runtime, gives crashes and `.dll`-conflict errors that are miserable to debug ([the "DLL not found / wrong DLL" trap](../Chapter5/dependencies.md)). **Pick one owner per library.** Inside a ROS2 workspace, let the **conda environment** be the source of that library; do not also pull it via vcpkg. The recurring lesson: *for any given library, exactly one tool owns it.*

---

## Summary

- Native Windows ROS2 is painful; **RoboStack** delivers ROS2 as **conda packages** — a self-contained environment that runs natively on Windows, no WSL/Docker needed. This course uses **Jazzy**.
- The environment is a **pixi** project at **`C:\robostack`**, env name **`jazzy`**. Enter it with **`pixi shell -e jazzy`** — all ROS2 commands run inside that shell.
- Build with **`colcon build --merge-install`** on Windows (`--merge-install` avoids Windows' symlink/long-path trouble).
- The conda env **owns** ROS2 and its dependencies; the workspace's `CMAKE_PREFIX_PATH` points into `C:/robostack/.pixi/envs/jazzy/...`. **Never** let vcpkg and conda each supply a copy of the same library — one owner per library, the [Part 5 dependency rule](../Chapter5/dependencies.md). Next: [the workspace and CLion](ros2_workspace.md).

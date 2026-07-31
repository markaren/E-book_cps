# ROS2 Workspace

A ROS2 **workspace** is a directory holding your packages under `src/`, which **colcon** builds into an `install/` tree. On the command line this is straightforward. The friction is the IDE: [CLion](../getting_started.md) is a *CMake* IDE — it wants a `CMakeLists.txt` to open — but colcon, not CMake, drives a ROS2 build. This chapter is about the pattern that reconciles them, taken from the course's [`ros2_moveit_ur_demo`](https://github.com/markaren/ros2_moveit_ur_demo).

---

## Two build systems, one source of truth

The central idea, stated plainly:

!!! abstract "The rule"
    **colcon drives the real build. The root `CMakeLists.txt` exists ONLY so CLion has an entry point.** It does not build your ROS2 packages the way a normal CMake project would — it exists to make the IDE happy and to launch colcon for you.

Why can't CLion just build the packages itself? Because a ROS2 package's build is wrapped by **`ament_cmake`** and orchestrated by colcon — dependency ordering between packages, the `install/` layout, environment setup — none of which plain CMake `add_subdirectory` reproduces. So you keep **one source of truth (colcon)** and give the IDE a thin CMake shim over it. Two build systems, one that actually builds.

---

## The workspace layout

```
ros2_ur_ws/
├── CMakeLists.txt          # the IDE shim (below) — NOT the real build
├── src/
│   ├── simulated_controller/
│   │   ├── CMakeLists.txt   # a real ament_cmake package
│   │   └── package.xml
│   ├── target_planner/
│   └── kine/
└── install/                # colcon's output (merged, on Windows)
```

Each folder under `src/` is a **package**: its own `CMakeLists.txt` and a `package.xml` (declaring the package's name and dependencies). This is precisely the [`add_subdirectory` structure from Part 5](../Chapter5/cmake.md#splitting-across-directories-with-add_subdirectory) — one component per directory, assembled at the top — which is why that chapter pointed forward to here.

---

## The root `CMakeLists.txt`, annotated

Here is the actual top-level file from the demo, with the moving parts called out. Read it as *"how the IDE shim is built"*, not *"how to build ROS2"*.

```cmake
cmake_minimum_required(VERSION 3.16)
project(ros2_ur_ws LANGUAGES C CXX)

# This file exists ONLY so IDEs (CLion) have a single entry point.
# colcon still drives the actual build.

if (WIN32)
    list(APPEND CMAKE_PREFIX_PATH
            "C:/robostack/.pixi/envs/default/Library"
            "C:/robostack/.pixi/envs/jazzy/Library")
endif ()

find_package(ament_cmake REQUIRED)

# Add each cpp package as a subdirectory
add_subdirectory(src/simulated_controller)
add_subdirectory(src/target_planner)
add_subdirectory(src/kine)
```

Three things to notice:

1. **`CMAKE_PREFIX_PATH` points into the pixi env.** This is the concrete link from the [last chapter](ros2_windows.md): on Windows, `find_package` finds ROS2 and its dependencies inside `C:/robostack/.pixi/envs/jazzy/Library` — the conda environment is the owner. This is what lets CLion *resolve* the packages (headers, autocomplete, error highlighting) even though it won't do the final build.
2. **`find_package(ament_cmake REQUIRED)`** brings in the ROS2 build macros — the same `find_package` mechanism from [Part 5](../Chapter5/cmake.md), here locating ROS2's CMake integration.
3. **`add_subdirectory(src/<pkg>)` per package** pulls each package's own `CMakeLists.txt` into the tree so the IDE indexes it.

### Wrapping colcon as IDE build targets

The clever part is the second half of that file: **`add_custom_target`** wrappers that let you press *Build* in CLion and have it run **colcon** rather than CMake. Simplified:

```cmake
if (WIN32)
    set(COLCON_INSTALL_FLAG --merge-install)     # Windows: merged install
else ()
    set(COLCON_INSTALL_FLAG --symlink-install)   # Linux: symlinked
endif ()

# one build target per package
add_custom_target(colcon_build_target_planner
    COMMAND colcon build --base-paths src --packages-select target_planner ${COLCON_INSTALL_FLAG}
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    USES_TERMINAL)

# ... and one that builds everything
add_custom_target(colcon_build_all
    COMMAND colcon build --base-paths src ${COLCON_INSTALL_FLAG}
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    USES_TERMINAL)
```

An `add_custom_target` is a CMake target that runs an arbitrary **command** instead of compiling code. Here the command is `colcon build ...` — so selecting `colcon_build_target_planner` in CLion's target dropdown and hitting build runs the real colcon build from inside the IDE. `--packages-select <pkg>` builds just one package (fast iteration); `colcon_build_all` builds the lot. The demo generates one target per package automatically by globbing `src/*/package.xml`.

The net effect: **CLion for editing, navigation, and pressing Build; colcon for the build itself.** You get the IDE without lying to yourself about what compiles your code.

!!! tip "Run inside the pixi shell"
    These targets invoke `colcon`, which only exists inside the [RoboStack environment](ros2_windows.md). Launch CLion **from** an activated `pixi shell -e jazzy` (so it inherits the environment), or configure the environment in CLion's toolchain — otherwise the `colcon` command won't be found.

---

## Worked flow: talker and listener

Put the concepts together with the [talker/listener pair](ros2_concepts.md) from the concepts chapter.

1. **Create two packages** under `src/` (`talker`, `listener`), each with the minimal publisher/subscriber node, a `package.xml`, and an `ament_cmake` `CMakeLists.txt`.
2. **Build** — from the pixi shell, or via the CLion `colcon_build_all` target:

    ```powershell
    colcon build --base-paths src --merge-install
    ```

3. **Source the workspace** so ROS2 can find your new nodes, then run each in its own terminal:

    ```powershell
    ros2 run talker talker
    ros2 run listener listener
    ```

    The listener prints the messages the talker publishes — two nodes discovering each other over [DDS](ros2_concepts.md), no broker started.

4. **Inspect the graph from outside.** ROS2's command-line tools let you watch any topic without writing code:

    ```powershell
    ros2 topic list                 # every topic in the graph
    ros2 topic echo /chatter        # print messages as they arrive
    ros2 topic hz /chatter          # measure the publish rate
    ```

    `ros2 topic echo` is your `print`-debugging for the whole system — it subscribes to a topic and dumps every message, the fastest way to confirm data is really flowing.

5. **Now publish your simulator's sensor topic.** Replace the talker's dummy string with a real reading from your [threepp simulator](../Chapter6/virtual_environments.md): a node reads a [simulated sensor](../Chapter6/virtual_environments.md) off the physics queue and publishes it (e.g. a `sensor_msgs/LaserScan` on `/scan`). Then `ros2 topic echo /scan` shows your simulated lidar streaming into the ROS2 graph — the moment the whole project's arc connects, sim to middleware.

!!! note "Executors are concurrency"
    Once nodes run, remember the [concepts chapter](ros2_concepts.md): callbacks fire on the **executor thread**, not `main`. A node that publishes simulator data touches the physics [queue/mailbox](../Chapter2/condition_variables.md) from a callback — the same [shared-data](../Chapter2/sharing_data.md) care from Part 2 applies at that boundary.

---

## Summary

- A **workspace** is `src/` packages that **colcon** builds into `install/`. Each package has its own `CMakeLists.txt` + `package.xml` — the [`add_subdirectory` layout from Part 5](../Chapter5/cmake.md#splitting-across-directories-with-add_subdirectory).
- **Two build systems, one source of truth:** colcon does the real build; the **root `CMakeLists.txt` exists only as a CLion entry point**. It sets `CMAKE_PREFIX_PATH` into the [pixi env](ros2_windows.md), `find_package(ament_cmake)`, and `add_subdirectory` per package so the IDE can resolve code.
- **`add_custom_target` wrappers** run `colcon build --base-paths src --packages-select <pkg> --merge-install` so you can build from CLion — launch the IDE inside the `pixi shell`.
- Worked flow: build a **talker/listener**, run them (DDS discovery, no broker), inspect with **`ros2 topic echo`**, then **publish your simulator's sensor topic** into the graph.
- Next: the [exercises](exercises.md).

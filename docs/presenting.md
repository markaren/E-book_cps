# Presenting Your Project

The [oral exam](faq.md) is where you *present your semester work* — the [threepp simulator](Chapter6/virtual_environments.md), the physics, the car and its [sensors](Chapter6/virtual_environments.md), and the [ROS2](Chapter7/ros2_concepts.md) integration — and answer questions about it. This page is a checklist for being ready. The examiner is not testing whether you can *recite* C++; they are testing whether **you understand the system you built**.

!!! quote "The governing principle"
    From [Using AI for Coding](using_ai.md): **if you can't explain it, don't ship it.** The oral exam is where that rule is enforced. Every line in your project is a line you may be asked to justify — a synchronization you cannot explain is a bug you cannot defend.

---

## Bring two diagrams

Walk in with these already drawn; do not sketch them live under pressure.

- **An architecture diagram.** The boxes and arrows of your system: the physics loop, the sensor nodes, the [ROS2 topics](Chapter7/ros2_concepts.md) that carry data, the control code, the actuators. One glance should show *what talks to what*. This is the map you narrate from.
- **A threading diagram.** Which threads exist, what each one does, and — crucially — **where they share data**. Mark every [lock](Chapter2/sharing_data.md) and every [queue/mailbox](Chapter2/condition_variables.md) hand-off. The render thread, the physics thread, the [executor](Chapter7/ros2_concepts.md) threads running your callbacks: show them, and show the boundaries where Part 2's rules apply.

---

## Be ready to justify every decision

Expect "why?" on the specifics. Have a one-sentence answer for each:

- **Every lock.** Why is *this* [mutex](Chapter2/sharing_data.md) here? What shared data does it protect, and what races if you remove it? Why is the critical section this size and no larger ([hot-path discipline](Chapter3/real_time.md))?
- **Every `std::move`.** Why [move](Chapter1/move_semantics.md) here instead of copy? What are you transferring ownership of, and why is a copy wrong or wasteful?
- **Every QoS choice.** For each [ROS2 topic](Chapter7/ros2_concepts.md): why *reliable* or *best-effort*, why this history depth? "The lidar is fresh-or-nothing, so best-effort with keep-last-1" is the shape of a good answer.
- **Every timestep and rate.** Why does physics run at a [fixed dt](Chapter6/virtual_environments.md)? Why this control-loop frequency? What happens on an [overrun](Chapter3/real_time.md)?

The pattern: for each design choice, know the **alternative you rejected** and **why**. "I used a mutex, not an atomic, because the protected state is a whole struct, not a single value" beats "I used a mutex" every time.

---

## Rehearse the data-journey question

A classic exam question is *"walk me through what happens when a single sensor reading travels from the physics loop to the screen (or to the subscriber)."* Rehearse this end to end until it is fluent:

> The physics thread computes the value on its [fixed-timestep](Chapter6/virtual_environments.md) step → adds noise and [serializes](Chapter4/serialization.md) it → pushes it onto the [queue](Chapter2/condition_variables.md) → the publisher callback pops it and publishes on a [ROS2 topic](Chapter7/ros2_concepts.md) → [DDS](Chapter7/ros2_concepts.md) delivers it per the chosen QoS → the subscriber's callback runs on an [executor thread](Chapter7/ros2_concepts.md) → it updates shared state under a [lock](Chapter2/sharing_data.md) → the render loop reads that state and draws it.

If you can narrate that without hesitating, you have shown you understand concurrency, communication, and real-time *as one system* — which is the whole point of the course.

---

## Demo a failure mode live

A working demo proves it works on a good day. Demonstrating a **failure** proves you understand it. Have one ready:

- **Kill the publisher** (or a sensor node) while the system runs. What does the subscriber do — block, spin on stale data, time out, degrade gracefully? A good system *tolerates* a dropped node ([DDS](Chapter7/ros2_concepts.md) discovery re-connects it when it returns); be able to show and explain the behaviour either way.
- Or: **slow a subscriber** below the publish rate and show the [QoS](Chapter7/ros2_concepts.md) choice doing its job (dropping stale samples, not backing up).

Knowing your system's failure behaviour — and having *chosen* it — is a stronger signal than a clean happy-path run.

---

## Let your git history be your evidence

A **commit-per-milestone** history is a timeline of your work an examiner can trust: "added the physics loop", "wired the sensor onto a topic", "fixed the race on `RobotState`". It shows the project grew incrementally and by your hand — evidence that the understanding is yours, not [pasted in wholesale from an assistant](using_ai.md). Commit as you reach each working milestone, with messages that say *what changed and why*. Under questioning, `git log` is a story you can walk the examiner through.

---

## Summary

- The oral exam tests **understanding of the system you built**, not recitation. The rule from [Using AI](using_ai.md) is enforced here: **if you can't explain it, don't ship it.**
- Bring an **architecture diagram** and a **threading diagram** (mark every lock and queue) — drawn in advance.
- Be ready to justify **every lock, every `std::move`, every [QoS](Chapter7/ros2_concepts.md) choice, every rate** — with the alternative you rejected and why.
- **Rehearse the data-journey** question (physics loop → screen/subscriber) until fluent, and **demo a failure mode** live (kill the publisher — know what your subscriber does).
- Keep a **commit-per-milestone** git history as your evidence trail.

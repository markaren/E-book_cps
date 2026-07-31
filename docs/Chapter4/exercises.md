# Chapter 4 Exercises

Work through these after reading Chapter 4. **Try each one yourself before revealing the solution.** Type the code into CLion and run it.

When you open a solution it appears **blurred** — click it once more to reveal it.

Exercises 1–3 are runnable programs; exercise 4 is a **design** exercise — think it through and write down your reasoning before revealing the discussion — and exercise 5 is an optional lab that makes exercise 3's simulation real on an actual socket.

---

## 1. Round-trip a command

*Practises: [Serialization](serialization.md)*

Write `serialize` and `deserialize` for a small `Command { std::string verb; int arg; }` using a simple text format, and prove a value survives the round trip. This is what every real format does for you — doing it by hand once makes the idea concrete.

> Hint: serialize to `"verb arg"` with `std::to_string`. To parse it back, feed the string to a `std::istringstream` and read `verb >> arg` — the stream extractors split on whitespace and convert the number for you.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <sstream>
    #include <string>

    struct Command { std::string verb; int arg; };

    std::string serialize(const Command& c) {
        return c.verb + " " + std::to_string(c.arg);        // object → bytes
    }

    Command deserialize(const std::string& line) {
        std::istringstream in(line);
        Command c;
        in >> c.verb >> c.arg;                              // bytes → object
        return c;
    }

    int main() {
        Command original{"move", 90};
        std::string wire = serialize(original);
        std::cout << "wire: \"" << wire << "\"\n";          // wire: "move 90"

        Command back = deserialize(wire);
        std::cout << back.verb << " / " << back.arg << "\n"; // move / 90
    }
    ```

    The command survives the round trip through a flat text representation that any language could parse. This hand-rolled format breaks the moment a `verb` contains a space, or you add a field, or `arg` becomes a float — which is exactly why you reach for [JSON](serialization.md) (via nlohmann/json) for anything real. The exercise is to feel *why* a format library exists, not to ship this.

    </div>

---

## 2. Decode a Modbus value

*Practises: [Modbus](modbus.md), [Serialization](serialization.md)*

A Modbus device reports a 32-bit signed integer across **two** 16-bit registers, **big-endian** (high word first). Write `registersToInt32(high, low)` that reassembles them, and decode the reading `high = 0x0001`, `low = 0x1170`.

> Hint: shift the high register left by 16 bits and OR in the low register: `(high << 16) | low`. Build the value arithmetically, and cast `high` to `uint32_t` *before* shifting: an unshifted `uint16_t` promotes to `int`, so shifting `high` (≥ `0x8000`) left by 16 would push a bit into the sign bit of a signed `int` — undefined behaviour before C++20, and poor practice after. Doing the maths in `uint32_t` sidesteps it. Then cast to signed.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <cstdint>
    #include <iostream>

    std::int32_t registersToInt32(std::uint16_t high, std::uint16_t low) {
        std::uint32_t bits = (static_cast<std::uint32_t>(high) << 16) | low;
        return static_cast<std::int32_t>(bits);
    }

    int main() {
        std::cout << registersToInt32(0x0001, 0x1170) << "\n";   // 70000
    }
    ```

    `0x0001` in the high word and `0x1170` in the low word reassemble to `0x00011170` = 70000. The cast of `high` to `std::uint32_t` **before** the shift is what keeps this well-defined: a `std::uint16_t` promotes to `int` (a *signed* 32-bit type), so no bits are actually lost by the shift — but for a `high` of `0x8000` or more, `high << 16` reaches the sign bit of that `int`, which is undefined behaviour before C++20 (and, though defined modulo 2³² since, still poor practice). Computing in `std::uint32_t` avoids the trap entirely. This is the [big-endian](serialization.md), multi-register decode from the Modbus chapter — and if the device's manual said it was *word-swapped*, you would simply pass the registers in the other order. A `float` works the same way, finishing with a `std::memcpy` to a `float` instead of a cast.

    </div>

---

## 3. Frame a TCP stream

*Practises: [Sockets, TCP & UDP](sockets.md)*

[TCP is a stream, not a message queue](sockets.md): one `recv` may return part of a message, or several at once. Write `extractMessages(buffer)` that pulls every complete **`'\n'`-terminated** message out of an accumulating buffer and **leaves any trailing partial message behind** for the next read. Simulate two `recv` calls whose chunks do not line up with message boundaries.

> Hint: loop while the buffer contains a `'\n'`: take everything before it as a message, then `erase` up to and including the newline. When the loop ends, whatever remains is an incomplete message — leave it in the buffer.

??? success "Show solution"

    <div class="spoiler" markdown title="Click to reveal">

    ```cpp
    #include <iostream>
    #include <string>
    #include <vector>

    // Pull all complete '\n'-terminated messages out of `buffer`,
    // leaving any trailing partial message behind for the next read.
    std::vector<std::string> extractMessages(std::string& buffer) {
        std::vector<std::string> messages;
        std::size_t newline;
        while ((newline = buffer.find('\n')) != std::string::npos) {
            messages.push_back(buffer.substr(0, newline));
            buffer.erase(0, newline + 1);          // drop the message and its '\n'
        }
        return messages;                           // buffer now holds only a partial (if any)
    }

    int main() {
        std::string buffer;

        buffer += "tempera";                       // recv #1: a partial message
        for (const auto& m : extractMessages(buffer)) std::cout << "got: " << m << "\n";

        buffer += "ture,42\nhumi";                 // recv #2: completes one, starts another
        for (const auto& m : extractMessages(buffer)) std::cout << "got: " << m << "\n";

        std::cout << "leftover: \"" << buffer << "\"\n";
    }
    ```

    Output:

    ```
    got: temperature,42
    leftover: "humi"
    ```

    The first `recv` delivered only `"tempera"` — no newline, so no complete message, and it stays buffered. The second `recv` completed `"temperature,42"` *and* began `"humi"`; the function emits the finished message and keeps the partial. This accumulate-and-extract loop is the framing every real TCP reader needs — without it, you would try to parse half a message. ([UDP](sockets.md) and [MQTT](mqtt_rpc.md) preserve message boundaries, so they need no framing.)

    </div>

---

## 4. Choose the protocols

*Practises: [the whole chapter](serialization.md)*

You are designing the communication for a **fleet of mobile robots** and a shared **operator dashboard**. Two kinds of traffic: (a) each robot streams its **pose and battery at 50 Hz**; (b) the operator occasionally sends a **command** ("stop", "return home") that *must* take effect. Decide the transport/pattern and serialization for each, and justify it. There is no single right answer — the reasoning is the point.

> Hint: weigh the [TCP vs UDP](sockets.md) trade-off and the [MQTT vs RPC vs raw-socket](mqtt_rpc.md) patterns separately for the two traffic types. Ask of each: does every message must arrive? how many consumers? who initiates?

??? success "Show discussion"

    <div class="spoiler" markdown title="Click to reveal">

    A defensible design — with the trade-offs that justify it:

    **Telemetry (50 Hz pose/battery).** This is *state broadcast*: high-rate, loss-tolerant (a dropped pose is replaced 20 ms later), and wanted by possibly several dashboards. **MQTT** fits well — each robot publishes to `robot/<id>/telemetry`, dashboards subscribe to `robot/+/telemetry`, and the broker decouples them so adding a dashboard needs no change on the robots. Use **QoS 0** (at most once): for fresh-or-nothing data you never want a retransmit of a stale pose. If it were a simple single-consumer LAN link instead, raw **UDP** would be the lighter choice for the same "freshness over completeness" reason. Serialize compactly — [protobuf](serialization.md), or JSON if rate allows.

    **Commands (must take effect).** This is *request/response* and **must arrive** — exactly the opposite priority. Options: an **RPC** call (gRPC) like `Stop()` that returns a confirmation, or an MQTT command topic at **QoS 1/2** with an acknowledgement. Either way you need **reliability and an ack**, so the operator knows the robot got it — never fire-and-forget UDP for a safety-relevant command.

    **The shape of the reasoning** matters more than the exact picks: telemetry is many-to-many, frequent, and loss-tolerant → pub/sub, low QoS, lose-and-move-on; commands are point-to-point, rare, and must-arrive → reliable transport with confirmation. Matching the tool to *whether a lost message is acceptable* is the core data-communication judgement this chapter builds.

    </div>

---

## 5. Watch TCP coalesce (optional lab)

*Practises: [Sockets, TCP & UDP](sockets.md), [Networking in C++](networking.md)*

Exercise 3 *simulated* TCP coalescing by hand. This one makes it real. Build the [SimpleSocket](https://github.com/markaren/SimpleSocket) echo **server and client** from [Networking in C++](networking.md) in CLion (add SimpleSocket via [vcpkg](../Chapter5/dependencies.md), or clone it — it has no dependencies). Run the server, then run the client but have it send **two short messages back-to-back** without waiting for a reply between them — e.g. `write("ping")` immediately followed by `write("pong")`. On the server, do a **single** `read` into your buffer and print how many bytes arrived and what they say. Then look at what you got.

> Hint: send with no delay between the two `write`s, and make the server's buffer large enough (say 1024 bytes) to hold both. Print the byte count and the string. Run it a few times.

??? success "Show expected result and why"

    <div class="spoiler" markdown title="Click to reveal">

    You will often see the server's single `read` return **all eight bytes at once** — `pingpong` — rather than two separate reads of four bytes each. The two `write`s were coalesced into one delivery: TCP is a **byte stream**, not a message queue, so `write` boundaries are not preserved on the wire. (You may not see it every single run — timing, [Nagle's algorithm](sockets.md), and the loopback path all play a part — but sending the two writes with no gap makes coalescing likely, and it *will* happen in the field.)

    This is exactly the hazard exercise 3 simulated, now lived: had the server assumed "one `read` = one message" it would have parsed `pingpong` as a single garbled command. The fix is the same [framing](serialization.md) you wrote there — delimit each message (a `'\n'`) or length-prefix it, and re-assemble on the reading side. Seeing it happen once is worth more than reading about it ten times: **TCP guarantees the bytes and their order, never where one message ends and the next begins.**

    </div>

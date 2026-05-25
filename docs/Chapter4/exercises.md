# Chapter 4 Exercises

Work through these after reading Chapter 4. **Try each one yourself before revealing the solution.** Type the code into CLion and run it.

When you open a solution it appears **blurred** — click it once more to reveal it.

The first three are runnable programs; the last is a **design** exercise — think it through and write down your reasoning before revealing the discussion.

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

> Hint: shift the high register left by 16 bits and OR in the low register: `(high << 16) | low`. Build the value arithmetically (cast `high` to `uint32_t` *before* shifting, or the shift overflows the 16-bit type), then cast to signed.

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

    `0x0001` in the high word and `0x1170` in the low word reassemble to `0x00011170` = 70000. The cast of `high` to `std::uint32_t` **before** the shift is essential: shifting a 16-bit value left by 16 would otherwise lose every bit. This is the [big-endian](serialization.md), multi-register decode from the Modbus chapter — and if the device's manual said it was *word-swapped*, you would simply pass the registers in the other order. A `float` works the same way, finishing with a `std::memcpy` to a `float` instead of a cast.

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

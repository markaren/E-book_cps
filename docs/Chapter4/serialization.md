# Serialization

[Sockets](sockets.md) move *bytes* between machines. But your program does not think in bytes — it thinks in objects: a sensor reading, a command, a robot pose. **Serialization** is the bridge: turning an object into a well-defined sequence of bytes (and back) so that the program on the other end, possibly written in another language on another kind of machine, reconstructs exactly what you meant. Get this wrong and two correct programs still cannot talk; get it right and a Python dashboard reads a C++ robot's telemetry without either side knowing the other's internals.

---

## What and why

**Serialization** (also called *marshalling*) converts an object's state into a portable byte sequence. **Deserialization** (*unmarshalling*) is the reverse. You need it for two distinct jobs:

- **Transmission** — sending an object to another program, over a [socket](sockets.md), a [serial line](serial.md), or a message broker.
- **Persistence** — saving an object to a file so it can be loaded later.

The reason you cannot just "send the object" is that an in-memory C++ object is not portable. Its layout depends on the compiler, the CPU, and the platform — pointer sizes, struct padding, and byte order all differ. Serialization defines a representation that is *independent* of all that, so both ends agree on the meaning regardless of how each stores it internally.

!!! tip "Prefer a language-agnostic format"
    A cyber-physical system is rarely all one language — a C++ controller, a Python vision node, a JavaScript dashboard. Choosing a format that every language can read (rather than something tied to C++) is what lets those pieces interoperate. This is the single most important serialization decision.

---

## Text vs binary formats

Serialization formats split into two families, and the choice between them is the main trade-off.

**Text formats** are human-readable — you can open the bytes in an editor and understand them.

| Format | Notes |
|--------|-------|
| **JSON** | The de-facto interchange format. Readable, ubiquitous library support, great for APIs and config. |
| **YAML** | More readable than JSON (indentation-based); popular for configuration. |
| **XML** | Verbose, older; still common in enterprise and some industrial systems. |
| **CSV** | Trivial tabular data; no nesting, no types. |

**Binary formats** are compact and fast to parse, but not human-readable.

| Format | Notes |
|--------|-------|
| **Protocol Buffers** (protobuf) | Google's schema-based binary format; underpins [gRPC](mqtt_rpc.md). Small, fast, strongly typed. |
| **FlatBuffers** | Like protobuf but readable without a parse step — good for resource-constrained or zero-copy use. |
| **MessagePack** | "Binary JSON" — the JSON data model in a compact binary encoding. |

How to choose comes down to a handful of questions:

- **Human-readable?** Text (JSON/YAML) for config and debugging; binary when no one needs to read it.
- **Message size and speed?** Binary is smaller and faster to parse — it matters for high-rate telemetry or a constrained Pi Zero, and rarely otherwise.
- **Schema?** protobuf/FlatBuffers enforce a declared schema (fewer "the other end sent garbage" bugs); JSON is schema-less and flexible.
- **Library and language support?** Not every format is equally well supported everywhere — JSON is supported essentially everywhere.

A good default for this course: **JSON** unless message size or rate proves it too slow, then a binary format like protobuf.

---

## Serialization in C++

As with sockets, the C++ standard library provides **no** serialization. You use a library. For JSON, the overwhelmingly popular choice is **nlohmann/json** — header-only, and about as ergonomic as C++ gets:

<!-- no-ce -->
```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;

struct Reading { std::string sensor; double value; };

int main() {
    Reading r{"boiler", 42.5};

    // serialize: object → JSON text
    json j = { {"sensor", r.sensor}, {"value", r.value} };
    std::string wire = j.dump();              // {"sensor":"boiler","value":42.5}

    // deserialize: JSON text → values
    json parsed = json::parse(wire);
    std::string sensor = parsed["sensor"];
    double value = parsed["value"];
}
```

Install it via [vcpkg](../Chapter6/dependencies.md) (`vcpkg install nlohmann-json`). For binary, protobuf and FlatBuffers each provide a compiler that generates C++ (and Python, Java, …) types from a schema file — the same code-generation idea you will see with [gRPC](mqtt_rpc.md).

To see the round-trip without any dependency, here is a hand-written text serializer — exactly the kind of thing a real format does for you, shown so the concept is concrete:

```cpp
#include <iostream>
#include <string>

struct Reading { std::string sensor; double value; };

std::string serialize(const Reading& r) {
    return r.sensor + "," + std::to_string(r.value);     // object → bytes
}

Reading deserialize(const std::string& line) {
    auto comma = line.find(',');
    return { line.substr(0, comma), std::stod(line.substr(comma + 1)) };  // bytes → object
}

int main() {
    Reading r{"boiler", 42.5};
    std::string wire = serialize(r);
    std::cout << "wire: " << wire << "\n";               // boiler,42.500000

    Reading back = deserialize(wire);
    std::cout << back.sensor << " = " << back.value << "\n";   // boiler = 42.5
}
```

This works, but it is brittle — what if a sensor name contains a comma? what about nested data, or types beyond strings and doubles? Those edge cases are exactly why you reach for a real format. **Don't roll your own** for anything non-trivial: a custom binary format can shave bytes and cycles, but it is hard to get right, painful to maintain, and a portability hazard. The cost of a battle-tested library is almost always worth it.

---

## Two traps: endianness and raw structs

The most common serialization mistake is trying to skip it — `send`ing the raw bytes of a struct and `recv`ing them back into a struct on the other end:

<!-- no-ce -->
```cpp
struct Reading { int id; double value; };
Reading r{1, 42.5};
send(sock, &r, sizeof(r), 0);     // DON'T: not portable
```

This appears to work between two identical machines and then fails the moment the ends differ, for two reasons:

- **Endianness** — the *byte order* of multi-byte numbers. Big-endian machines store the most significant byte first; little-endian (x86, most ARM) store it last. The agreed order *on the wire* is **big-endian**, called **network byte order** — which is why [sockets](sockets.md) use `htons`/`htonl` to convert, and why [Modbus](modbus.md) registers are big-endian. Send raw little-endian bytes to a big-endian reader and `42` becomes `704643072`.
- **Struct layout** — the compiler inserts invisible *padding* between members for alignment, and that padding (and even member sizes) can differ across compilers and platforms. `sizeof(Reading)` is not a portable contract.

A real serialization format handles both: it defines field order and byte order explicitly, so the bytes mean the same thing everywhere. That is the whole point of using one.

!!! note "Framing is a separate job"
    Recall from [Sockets](sockets.md) that TCP is a *stream* — it does not preserve your message boundaries. Serialization turns one object into bytes; **framing** tells the reader where one object's bytes end and the next begin, usually with a length prefix or a delimiter (a newline after each JSON object is common). You need both: a format *and* a framing scheme. (UDP and message brokers like [MQTT](mqtt_rpc.md) preserve message boundaries for you, so framing is only a TCP/serial concern.)

---

## Summary

- **Serialization** converts an object to a portable byte sequence; **deserialization** rebuilds it. You need it to **transmit** objects between programs and to **persist** them — an in-memory C++ object is not portable.
- Prefer a **language-agnostic** format so different languages interoperate. **Text** (JSON, YAML, XML, CSV) is readable; **binary** (protobuf, FlatBuffers, MessagePack) is compact and fast. Default to **JSON** unless size/speed forces binary.
- C++ has **no standard serialization** — use a library (**nlohmann/json** for JSON, protobuf/FlatBuffers for binary). **Don't roll your own** beyond toy cases.
- **Never send raw structs**: **endianness** (the wire is big-endian / network byte order) and **struct padding** make them non-portable. A real format pins down field and byte order for you.
- Serialization and **framing** are different jobs — over a [TCP](sockets.md) stream you also need a length prefix or delimiter to mark message boundaries. Next: [Networking in C++](networking.md), the libraries that carry these bytes.

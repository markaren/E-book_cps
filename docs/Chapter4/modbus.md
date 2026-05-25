# Modbus

So far the protocols have been general-purpose. **Modbus** is different: it is a *domain* protocol, designed in 1979 by Modicon for industrial automation, and still everywhere — PLCs, drives, power meters, temperature controllers. If your cyber-physical system talks to industrial hardware, it will very likely talk Modbus. It is also a clean case study in the ideas from the rest of this chapter: a defined message structure, [big-endian](serialization.md) data, and a choice of transport.

---

## Variants: TCP and RTU

Modbus comes in three flavours, of which two matter today:

| Variant | Transport | Error checking |
|---------|-----------|----------------|
| **Modbus TCP** | TCP/IP over Ethernet | Handled by [TCP](sockets.md) itself |
| **Modbus RTU** | [Serial](serial.md) (RS-232 / RS-485) | A CRC in each message |
| Modbus ASCII | Serial, text-encoded | Discontinued — ignore it |

**Modbus TCP** is the easier one to work with, and what this course uses: it rides on TCP, so the [reliable, ordered delivery](sockets.md) you already understand handles error checking for you, and you just open a socket to port **502**. **Modbus RTU** runs over a [serial line](serial.md) and must do its own integrity checking with a CRC. The application logic — registers and function codes below — is the same for both.

---

## Master and slave: mind the naming

Modbus uses **master/slave** terminology from its serial origins, and mapping it onto **client/server** trips up nearly everyone:

- The **master** is the **client** — it *initiates* every request.
- The **slave** is the **server** — it *holds the data* and *responds* to requests.

So a Modbus exchange is always master-asks, slave-answers: your program (the master/client) connects to a device (the slave/server) and polls it. That direction matches normal client/server intuition — the client initiates — but the *master/slave* labels are the bit to keep straight: the **slave is the device with the data** (a sensor, a PLC), and the **master is the thing reading it**. A slave never speaks unless asked.

---

## Message structure

A Modbus TCP message has two parts: an **MBAP header** (Modbus Application Protocol header) and a **PDU** (Protocol Data Unit).

```
┌──────────────────── MBAP header ────────────────────┐ ┌──── PDU ────┐
 Transaction ID │ Protocol ID │ Length │ Unit ID         Function │ Data
   (2 bytes)        (2 bytes)   (2 bytes) (1 byte)         (1 byte)
```

| Field | Size | Meaning |
|-------|------|---------|
| Transaction ID | 2 bytes | Pairs a response with its request |
| Protocol ID | 2 bytes | Always `0` for Modbus |
| Length | 2 bytes | Number of bytes that follow |
| Unit ID | 1 byte | Which device (used when bridging to serial) |
| Function code | 1 byte | What action to perform |
| Data | variable | The payload for that action |

You rarely build this by hand — a library does it — but knowing the shape makes the library's API (and a packet capture) readable.

### Function codes

The **function code** says what the master wants. The common ones:

| Code | Action | Data type |
|------|--------|-----------|
| `0x01` | Read Coils | Booleans (read/write bits) |
| `0x02` | Read Discrete Inputs | Booleans (read-only bits) |
| `0x03` | Read Holding Registers | 16-bit (read/write) |
| `0x04` | Read Input Registers | 16-bit (read-only) |
| `0x05` | Write Single Coil | one boolean |
| `0x06` | Write Single Register | one 16-bit value |
| `0x0F` | Write Multiple Coils | many booleans |
| `0x10` | Write Multiple Registers | many 16-bit values |

The data model splits into **coils** (single bits — a relay on/off) and **registers** (16-bit words — a measured value). Read-only versions (discrete inputs, input registers) exist alongside the read/write ones (coils, holding registers).

---

## The 16-bit register trap

Here is the part that causes real bugs. Modbus stores everything in **16-bit registers**, and values are **big-endian** ([network byte order](serialization.md)). A 16-bit value fits in one register, but anything bigger spans several **consecutive** registers:

| C++ type | Registers | Bytes |
|----------|-----------|-------|
| `uint16_t`, `int16_t` | 1 | 2 |
| `uint32_t`, `int32_t`, `float` | 2 | 4 |
| `double` | 4 | 8 |

So reading a `float` from a device means reading **two** consecutive registers and reassembling four bytes — in the right order — into a `float`. Get the order wrong and the number is garbage. Here is the decode, and it is pure computation, so it runs anywhere:

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>

// Combine two big-endian 16-bit Modbus registers into a 32-bit float.
float registersToFloat(std::uint16_t high, std::uint16_t low) {
    std::uint32_t bits = (static_cast<std::uint32_t>(high) << 16) | low;  // high register first
    float value;
    std::memcpy(&value, &bits, sizeof(value));   // reinterpret the 4 bytes as a float
    return value;
}

int main() {
    // 0x42F6, 0xE979 is the IEEE-754 encoding of 123.456, split across two registers
    std::cout << registersToFloat(0x42F6, 0xE979) << "\n";   // 123.456
}
```

Two details worth noting. We build `bits` *arithmetically* (`high << 16 | low`), which is endian-independent — it does the same thing on any machine. Then `std::memcpy` reinterprets those four bytes as a `float`; `memcpy` is the correct, well-defined way to do this type-pun (a `reinterpret_cast` here would be undefined behaviour). This little function is exactly what a Modbus library does internally — and if a device's manual says its values are "word-swapped," it is telling you to swap `high` and `low`.

!!! warning "Read the device's register map"
    A device's documentation lists which register holds what, and in which order multi-register values are packed (some swap the high/low words). There is no universal layout beyond "16-bit, big-endian" — always read the specific device's register map before decoding.

---

## Modbus in practice

You will not implement the protocol yourself — you use a library:

- **SimpleSocket** ([Networking in C++](networking.md)) includes a Modbus TCP client, which is why it suits this course.
- **libmodbus** (C) is the long-established cross-platform library for TCP and RTU.
- **pymodbus** (Python) is the standard on the embedded/scripting side, and ships a **simulator** so you can develop a client with no hardware.

For testing without a physical device, a Modbus **simulator** (a software slave) lets you exercise your client, and tools like a Modbus tester let you poke your server — develop both ends against software first, then swap in the real device.

---

## Summary

- **Modbus** is the workhorse protocol of industrial automation (since 1979). **Modbus TCP** (port 502, error-checked by [TCP](sockets.md)) is the easy variant this course uses; **Modbus RTU** runs over a [serial](serial.md) line with its own CRC.
- The **master is the client** (initiates requests); the **slave is the server** (holds data, responds). Keep that mapping straight — the slave is the device being polled.
- A Modbus TCP message = **MBAP header** (transaction/protocol/length/unit) + **PDU** (function code + data). **Function codes** read/write **coils** (bits) and **registers** (16-bit words).
- Data lives in **16-bit big-endian registers**; larger types span **consecutive** registers (`float` = 2, `double` = 4), so you must reassemble bytes in the right order — `(high << 16) | low` then `memcpy` to the target type. Always check the device's register map.
- Use a library (**SimpleSocket**, **libmodbus**, **pymodbus**) and a **simulator** to develop without hardware. Next: [MQTT & RPC](mqtt_rpc.md), higher-level patterns above raw sockets.

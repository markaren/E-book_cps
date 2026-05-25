# Serial Communication

Not every device is on a network. The most common way a PC or a Raspberry Pi talks to a **microcontroller** — an Arduino reading sensors, a motor driver — is a **serial** link: a wire (often over USB) carrying data one bit at a time. Serial is older and simpler than sockets, but the same ideas recur: you agree on a format, you frame your messages, and you pick a library rather than poke the hardware by hand.

---

## One bit at a time

**Serial communication** sends data sequentially, one bit after another, over a line — in contrast to *parallel* communication, which sends many bits at once over many wires. Two signal wires carry the data:

- **TX** (transmit) — the line this device sends on.
- **RX** (receive) — the line this device listens on.

Wire two devices together by crossing them: one's TX goes to the other's RX. How much they can talk at once defines the **duplex** mode:

- **Full-duplex** — simultaneous two-way traffic, using separate TX and RX lines.
- **Half-duplex** — two-way, but only one direction at a time.

---

## Asynchronous serial: UART and RS-232

The most common form is **asynchronous**, used by **UART** (the hardware in an Arduino) and the **RS-232** standard. "Asynchronous" means there is **no shared clock** — the two devices are not wired to a common timing signal, so they must *agree in advance* on the speed and the frame format, then each keep its own time.

The agreed speed is the **baud rate**: signal changes per second, which for these links equals bits per second. `9600`, `115200` are typical. **Both ends must be set to the same baud rate** — a mismatch produces garbage, the classic first bug in serial work.

Because there is no clock to mark where a byte begins, each byte is wrapped in a **frame** of marker bits:

| Part | Purpose |
|------|---------|
| **Start bit** | The line drops from idle-high to low, signalling "a byte is coming." Always `0`. |
| **Data bits** | The payload, usually 8 bits, sent **least-significant bit first**. |
| **Parity bit** (optional) | A simple error check: *even* parity makes the number of `1`s even, *odd* makes it odd, *none* skips it. |
| **Stop bit(s)** | The line returns to idle-high to mark the end. `1` or `2` bits. |

So transmitting the letter `'A'` (ASCII 65 = binary `01000001`) at 8 data bits, no parity, one stop bit ("8N1", the usual default) puts this on the wire: a start `0`, then the data **LSB-first** (`1 0 0 0 0 0 1 0`), then a stop `1`. The receiver sees the start bit, clocks in eight bits at the agreed baud rate, checks the stop bit, and reassembles the byte. A serial link is therefore described by a tuple like **115200 8N1** — baud, data bits, parity, stop bits — and both ends must match all four.

---

## Synchronous serial: SPI and I²C

When a device *can* share a clock wire — typically a sensor sitting on the same board as the microcontroller — **synchronous** serial is used instead. A clock line removes the need to agree on baud and allows higher speeds. Two dominate embedded systems:

- **I²C** (Inter-Integrated Circuit) — a two-wire, multi-master, multi-slave bus: **SDA** (data) and **SCL** (clock). Each device has an address, so many sensors share the same two wires. Common for low-speed peripherals (sensors, small displays). On Arduino it is the `Wire` library.
- **SPI** (Serial Peripheral Interface) — faster, but uses more wires (clock, two data lines, and a select line per device). Common for displays, SD cards, fast sensors.

You meet I²C and SPI again in [GPIO & Buses](../Chapter5/gpio.md), driving them from user-space Linux on the Pi.

---

## Serial in C++

As with [sockets](sockets.md), the C++ standard library has **no** serial support — and the OS APIs differ (a `COM` port via the Win32 API on Windows; a `/dev/ttyUSB0` or `/dev/ttyACM0` file with `termios` on Linux). So again you use a library:

- **Boost.Asio** provides a `serial_port` that fits the same async model as its [networking](networking.md) side.
- Small dedicated serial libraries exist (e.g. CSerialPort and similar) for when you do not want all of Boost.
- On the Python side — common on the embedded device — **pySerial** is the standard choice.

A blocking read with Asio's `serial_port` looks like this (conceptual — it needs a real port, so it will not run on Compiler Explorer):

<!-- no-ce -->
```cpp
#include <boost/asio.hpp>

int main() {
    boost::asio::io_context io;
    boost::asio::serial_port port(io, "/dev/ttyACM0");   // "COM3" on Windows

    port.set_option(boost::asio::serial_port_base::baud_rate(115200));

    std::string line;
    boost::asio::read_until(port, boost::asio::dynamic_buffer(line), '\n');
    // `line` now holds one newline-terminated message from the device
}
```

Note the recurring themes: you set the **baud rate** to match the device, and you read **up to a `'\n'`** — the same [framing](serialization.md) problem as TCP. A serial link, like a TCP socket, is a *byte stream* with no built-in message boundaries, so a newline (or a length prefix) is how you tell where one reading ends and the next begins.

!!! tip "The Arduino side"
    An Arduino sketch sends with the built-in `Serial` library: `Serial.begin(115200);` then `Serial.println(value);`. Over USB this appears to the PC as a serial port. A robot's typical setup is an Arduino streaming `sensor,value\n` lines that a C++ program on the PC or Pi reads and [deserializes](serialization.md) — the bridge between the embedded world and your application.

---

## Serial vs sockets

| | Serial | Sockets |
|---|--------|---------|
| Medium | A physical wire (often USB) | A network |
| Reach | Point-to-point, short | Anywhere routable |
| Addressing | The port name (`/dev/ttyACM0`, `COM3`) | IP address + port |
| Message boundaries | None — a byte stream (frame it yourself) | TCP: none (frame it); UDP: per-datagram |
| Typical use | PC ↔ microcontroller, sensors | PC ↔ PC, robot ↔ operator |

The shared lesson: both are byte streams, so a [serialization format](serialization.md) and a framing scheme apply equally. And serial reappears as the physical layer for **[Modbus RTU](modbus.md)** — the industrial protocol that runs over an RS-232/RS-485 serial line, the subject of the next chapter.

---

## Summary

- **Serial** sends data one bit at a time over **TX/RX** lines; **full-duplex** is simultaneous two-way, **half-duplex** one direction at a time.
- **Asynchronous** serial (**UART**, **RS-232**) has no clock, so both ends must agree on the **baud rate** and frame format (e.g. **8N1**); each byte is wrapped in a **start bit, data bits (LSB first), optional parity, and stop bit(s)**.
- **Synchronous** serial (**I²C** with SDA/SCL, **SPI**) uses a clock wire for higher speed; common for on-board sensors and revisited in [GPIO & Buses](../Chapter5/gpio.md).
- C++ has **no standard serial** — use **Boost.Asio**'s `serial_port` or a small serial library (pySerial on the Python side). Set the baud to match, and **frame** messages yourself (a `'\n'` or length prefix), because serial is a boundary-less byte stream like [TCP](sockets.md).
- The classic use is a PC/Pi reading `sensor,value\n` lines from an Arduino and [deserializing](serialization.md) them. Next: [Modbus](modbus.md), an industrial protocol that runs over both serial (RTU) and TCP.

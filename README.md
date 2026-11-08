# pico-serialport

Serial ports for [Pico](https://github.com/padcom/pico-lang): open a port, set its speed and framing, read and write bytes, and list the ports a machine has.

It's the native piece behind Pico Desktop's serial feature. It doesn't depend on Pico Desktop and works in any Pico program.

Status: **spike, Linux only.** It shows that a native serial loopback works from Pico (Pico Desktop plan §13). macOS and Windows backends come later, with the same API.

## Use

```pico
import { listPorts, openPort, Parity } from "padcom/serialport"

for (const p of await listPorts()) {
  println(`${p.path} ${p.vendorId ?? ""}:${p.productId ?? ""} ${p.product ?? ""}`)
}

const port = openPort("/dev/ttyUSB0", { baudRate: 115200 })   // 8N1, no flow control
await port.write(new TextEncoder().encode("AT\r"))
for await (const chunk of port) {
  // chunk: Uint8Array
}
port.close()
```

- `openPort(path, options)` opens the port exclusively, so a second open fails with `BUSY`. It also sets raw mode, so no line editing, echo or translation happens. Options:
  - `baudRate`: 1200 to 4000000, one of the standard rates;
  - `dataBits`: 5 to 8, default 8;
  - `parity`: `None` (the default), `Even` or `Odd`;
  - `stopBits`: 1 (the default) or 2;
  - `flowControl`: `None` (the default) or `Hardware`, which is RTS/CTS.
- `SerialPort` implements `ByteStream`:
  - `read()` resolves with the bytes that arrived, or `null` at the end;
  - `write(bytes)`;
  - `close()`;
  - `flush()`, which drops pending input and output;
  - `for await` over the chunks.

  `ByteStream` is the transport-neutral interface for protocol libraries (Modbus, GRBL, …): they take any byte stream, not only a serial port.
- `listPorts()` lists the ttys that have a device behind them, except 8250 placeholders (sysfs `type` 0). USB adapters also carry their vendor and product IDs, serial number, manufacturer and product name.
- Errors are `SerialError` with a `code`:
  - `NOT_FOUND`;
  - `PERMISSION_DENIED`: on Linux, add the user to the `dialout` group;
  - `BUSY`;
  - `NOT_A_SERIAL_PORT`;
  - `UNSUPPORTED`: an option the port can't do;
  - `CLOSED`;
  - `IO`.

  It also carries the `errno`, and a message that says what was being done.

## Example

`examples/monitor.pi` is a small serial monitor:
- with no arguments, it lists the ports;
- with a path and a baud rate, it sends what you type and prints what arrives.

```sh
pico run examples/monitor.pi
pico run examples/monitor.pi -- /dev/ttyUSB0 115200
```

## Known limits

- Closing a port ends a `read()` that is already waiting: it settles with `null`. This needs pico 0.4.3 or later (`Reader.close()`, [#100](https://github.com/padcom/pico-lang/issues/100)). With pico 0.4.2, a waiting read keeps the program from ending.
- Linux only so far. A program using this package builds on every native target and on wasm32-wasi, but off Linux `listPorts()` is empty and `openPort` fails with `UNSUPPORTED`. It doesn't build for wasm32-browser, which has no `std/process`.

## Tests

```sh
pico test .
```

The tests need no hardware:
- a pseudo-terminal pair stands in for the device: bytes go both ways, settings are applied, and exclusivity and the errors are checked;
- a made-up sysfs tree covers the listing.

A pty ignores data bits and parity (the kernel forces CS8 without parity). Those bits are checked by unit tests of the termios flags instead.

## License

MIT

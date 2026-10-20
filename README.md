# ESPHome USB host to TCP bridge

Turn an **ESP32-S2, ESP32-S3 or ESP32-P4** into a transparent **USB-serial-to-WiFi bridge** with ESPHome.
The ESP is the **USB host** of an ordinary USB-serial adapter (CH340, CP2102, FTDI, PL2303, CDC ACM…)
or of a device with a built-in one, sets its line (baud rate, parity, stop bits) and passes the bytes
to a TCP socket and back, unchanged. Use it to put a USB serial device on the network without a PC:
ser2net style, virtual COM ports, integrations that expect a socket.

```
client ──TCP── ESP32-S3 (USB host) ──USB── USB-serial adapter ──serial── device
```

You also get OTA updates, the Home Assistant API, a fallback hotspot and optional diagnostics (uptime,
reset reason, heap, WiFi signal, disconnect counters, restart button).

Built on the native [`usb_host`](https://esphome.io/components/usb_host/),
[`usb_uart`](https://esphome.io/components/usb_uart/) and [`uart_tcp`](https://esphome.io/components/uart_tcp/)
components, no external components. Needs ESPHome 2026.10 or later.

> Need a plain UART instead of USB, or an ESP8266? See the sibling project
> [esphome-uart-to-tcp-bridge](https://github.com/mvimercati/esphome-uart-to-tcp-bridge).

## Which ESP

USB host needs the USB OTG controller, which only some ESP32 variants have. ESPHome refuses the others.

| Variant | Works | Notes |
|---------|-------|-------|
| ESP32-S3 | yes | the usual choice. Devkits with two USB ports are handy: the native one is the host, the other (UART) one is for flashing and logs |
| ESP32-S2 | yes | single core, less RAM; one USB port only, so flash and logs go through UART0 pins or OTA |
| ESP32-P4 | in principle | no radio of its own: WiFi needs an ESP32-C6 companion and the `esp32_hosted:` component of your board |
| ESP32, C3, C6, H2… | no | no USB OTG controller (the C3/C6/S3 "USB Serial/JTAG" is device only) |

## How it is organised

Your device file sets a few substitutions and picks one package from each group:

| Group | Package | Choose it for |
|-------|---------|---------------|
| platform | `packages/esp32.yaml` | always: board, flash size, logger on a hardware UART |
| USB | `packages/usb-uart.yaml` | always: USB host, adapter type and serial settings |
| transport | `packages/tcp-server.yaml` | the ESP listens; one client at a time |
| | `packages/tcp-client.yaml` | the ESP connects to your server and reconnects on its own |
| extras | `packages/diagnostics.yaml` | optional health entities in Home Assistant |

`usb-bridge.yaml` is a complete example.

## Quick start

1. Find out the chip and VID:PID of the **adapter** (the USB-serial converter that goes into the ESP,
   not the ESP itself): plug it into a PC and run `lsusb` (Linux, macOS with
   `system_profiler SPUSBDataType`) or look in the Windows Device Manager. Pick the `usb_type` from
   [Adapters](#adapters).
2. Copy `usb-bridge.yaml`, `secrets.example.yaml` (as `secrets.yaml`) and the `packages/` folder into
   your ESPHome folder, or fetch the packages from GitHub (next section).
3. Fill in `secrets.yaml`. The API key is 32 random bytes in base64: `openssl rand -base64 32`.
4. In `usb-bridge.yaml` set board, flash size, adapter, serial settings and port.
5. Build and flash: `esphome run usb-bridge.yaml`. The first flash goes over USB, later ones over WiFi.
6. Plug the adapter into the ESP's **native USB port** and give it power (see [Power](#power)). The log
   shows `Device descriptor: vid … pid …` when it is recognised.

## Use it straight from GitHub

Write only your device file and let ESPHome fetch the packages:

```yaml
substitutions:
  name: my-bridge
  board: esp32-s3-devkitc-1
  usb_type: FT232
  usb_vid: "0x0403"
  usb_pid: "0x6001"
  uart_baud_rate: "115200"

packages:
  usb_bridge:
    url: https://github.com/mvimercati/esphome-usbhost-to-tcp-bridge
    ref: v1.0.0  # a tag, so an update never changes your device by surprise
    files:
      - packages/esp32.yaml
      - packages/usb-uart.yaml
      - packages/tcp-server.yaml  # or tcp-client.yaml
      - packages/diagnostics.yaml  # optional

esphome:
  name: ${name}
# api, ota, wifi as in usb-bridge.yaml
```

## Settings

Every setting is a substitution with a default; set only the ones you need. Numbers go in quotes.

**Platform (`esp32.yaml`)**

| Substitution | Default | Notes |
|--------------|---------|-------|
| `board` | `esp32-s3-devkitc-1` | a PlatformIO board id of an S2, S3 or P4 board |
| `flash_size` | `4MB` | what your module has: N8 = 8MB, N16 = 16MB |
| `logger_uart` | `UART0` | `UART0` or `UART1`; the native USB port is busy being the host |

A module with PSRAM (the R in N16R8) also wants a `psram:` block in the device file, e.g.
`mode: octal` for R8. It is not needed by the bridge.

**USB and serial (`usb-uart.yaml`)**

| Substitution | Default | Notes |
|--------------|---------|-------|
| `usb_type` | `CH340` | the driver, see [Adapters](#adapters) |
| `usb_vid` / `usb_pid` | `0x1A86` / `0x7523` | the adapter's VID:PID (not the ESP's); set them together with `usb_type` |
| `usb_enable_hubs` | `false` | `true` if the adapter sits behind a USB hub |
| `uart_baud_rate` | `9600` | |
| `uart_data_bits` | `8` | 5 to 8 |
| `uart_parity` | `NONE` | `NONE`, `EVEN`, `ODD`, `MARK`, `SPACE` |
| `uart_stop_bits` | `1` | `1`, `1.5` or `2` |
| `uart_dtr` / `uart_rts` | `false` | `true` raises the line once the adapter is set up, see [DTR and RTS](#dtr-and-rts) |
| `uart_buffer_size` | `256` | bytes held in each direction, 64 to 8192 |
| `uart_debug` | `false` | `true` logs every byte in both directions; turn it off when done |

**Transport**

| Substitution | Default | Package | Notes |
|--------------|---------|---------|-------|
| `tcp_port` | `6638` | both | |
| `tcp_allowed_ips` | `0.0.0.0/0` | tcp-server | an address, a CIDR network or a list: `[192.168.1.10, 10.0.0.0/24]`. Others are refused |
| `tcp_host` | *required* | tcp-client | IPv4 address or hostname of the server |
| `tcp_reconnect_interval` | `5s` | tcp-client | |

For anything else, extend the package's component in your device file. The ids are `bridge_usb` (the
USB device), `bridge_uart` (its serial channel) and `bridge_tcp`:

```yaml
usb_uart:
  - id: !extend bridge_usb
    product: FT232R USB UART  # also match the product string, as lsusb -v shows it
```

## Adapters

| `usb_type` | Chips | Usual VID:PID |
|------------|-------|---------------|
| `CH340` | CH340, CH341 | `1A86:7523` (CH341: `1A86:5523`) |
| `CH34X` | CH342, CH343, CH344, CH347, CH9101–CH9104 | `1A86:55D5` and others (CH343: `1A86:55D3`) |
| `CP210X` | CP2102, CP2102N, CP2104, CP2105, CP2108 | `10C4:EA60` |
| `FT232` | FT232R, FT232BM, FT230X, FT231X, FT232H | `0403:6001` (FT23xX: `0403:6015`, FT232H: `0403:6014`) |
| `FT2232` / `FT4232` | FT2232, FT4232 | `0403:6010` / `0403:6011` |
| `PL2303` | PL2303 (HX, TA, TB…) | `067B:2303`; the newer PL2303G* have their own types (`PL2303GC`…) |
| `CDC_ACM` | anything that is a standard USB serial port with no driver on Linux/macOS: Arduino and STM32 VCP, ESP32 USB-JTAG, Pi Pico, many modems | the device's own |

The FTDI and PL2303 drivers find the chip model from the device; for the CH34x family the PID matters,
so copy it from `lsusb`. Multi-port chips (FT2232, FT4232, CH342/CH344, CP2105): only the first port is
bridged.

Ready-made settings for common adapters:

```yaml
substitutions:
  # FTDI FT232R (also FT232RL/RQ boards and cables)
  usb_type: FT232
  usb_vid: "0x0403"
  usb_pid: "0x6001"
  # CP2102 / CP2102N:  usb_type: CP210X,  usb_vid: "0x10C4", usb_pid: "0xEA60"
  # CH340G / CH340C:   usb_type: CH340,   usb_vid: "0x1A86", usb_pid: "0x7523"
  # CH343P:            usb_type: CH34X,   usb_vid: "0x1A86", usb_pid: "0x55D3"
  # Arduino Uno R3:    usb_type: CDC_ACM, usb_vid: "0x2341", usb_pid: "0x0043"
```

## DTR and RTS

By default the bridge leaves the modem lines as the driver sets them when the adapter is plugged in:
the CDC ACM, CH34x and PL2303 drivers raise DTR and RTS, the FTDI and CP210x drivers leave them inactive.
Set `uart_dtr` and/or `uart_rts` to `"true"` when the device needs them active behind an FTDI or CP210x
adapter: some take power from them, some only talk when DTR is up. With other adapters the option has no
effect, since both lines are already up. Clients can't change them over TCP.

## Power

In host mode the ESP must give 5 V on the native USB port's VBUS. Many devkits power the board *from*
that pin and don't feed it back: check yours. Some S3 devkits have a pad or jumper labelled `USB-OTG`
(or `IN-OUT`) to close; otherwise feed the adapter's 5 V from the board's 5V pin, or use a powered hub
(with `usb_enable_hubs: "true"`). A USB OTG adapter on a USB-C port only works if the board routes VBUS.

## Test the link

```sh
nc usb-bridge.local 6638                                            # type and read
socat pty,link=/tmp/ttyBRIDGE,raw,echo=0 tcp:usb-bridge.local:6638  # virtual serial port for any program
```

## Home Assistant entities

| Entity | Package | Meaning |
|--------|---------|---------|
| Client connected / Server connected | tcp-server / tcp-client | whether the TCP link is up |
| Client disconnects / Server disconnects | tcp-server / tcp-client | TCP links closed since boot |
| WiFi disconnects | diagnostics | access point lost since boot, published once WiFi is back |
| WiFi signal | diagnostics | RSSI in dBm; below about -80 the link gets unreliable |
| Uptime, Reset reason | diagnostics | a sawtooth uptime means restarts, the reason says why |
| Heap free, Heap max block, Heap fragmentation | diagnostics | a steady decline points to a leak |
| Restart | diagnostics | restart button |

The counters restart from 0 at every boot; they are `total_increasing`, so HA statistics cope with it.

## Limitations

- **Security.** The serial stream is plain, unauthenticated TCP. Restrict it with `tcp_allowed_ips` and
  keep the bridge on a trusted network.
- **One client at a time.** A second one is refused while the first is connected.
- **Fixed line settings.** Baud rate and framing come from the YAML: a client can't change them (no
  RFC 2217), and neither can it drive DTR/RTS or read CTS/DSR.
- **The bridge owns the USB-serial channel.** `uart_tcp` refuses other components on it, and the
  `dummy_receiver` option. `uart_debug` is fine.

## Files

| File | Content |
|------|---------|
| `usb-bridge.yaml` | example device configuration |
| `packages/esp32.yaml` | platform: board, flash size, logger UART |
| `packages/usb-uart.yaml` | USB host, adapter and serial settings |
| `packages/tcp-server.yaml`, `packages/tcp-client.yaml` | transport, pick one |
| `packages/diagnostics.yaml` | optional health entities |
| `secrets.example.yaml` | template for `secrets.yaml` |

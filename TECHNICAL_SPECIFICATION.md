# Technical Specification: Hoymiles HMS-XXXW-2T Inverter Data Collection Protocol

This document describes the protocol used by `hms-mqtt-publisher` to collect telemetry data
from the **Hoymiles HMS-XXXW-2T** series of micro-inverters and to publish it to an MQTT
broker. The goal is to give implementors in other languages and frameworks everything they
need to build a compatible client from scratch.

---

## Table of Contents

1. [Device Overview](#1-device-overview)
2. [Transport Layer](#2-transport-layer)
3. [Authentication](#3-authentication)
4. [Communication Pattern (Pull-based)](#4-communication-pattern-pull-based)
5. [Packet Framing Format](#5-packet-framing-format)
6. [Protobuf Message Definitions](#6-protobuf-message-definitions)
7. [Scale Factors and Units](#7-scale-factors-and-units)
8. [CRC Algorithm](#8-crc-algorithm)
9. [Request Construction (Step-by-Step)](#9-request-construction-step-by-step)
10. [Response Parsing (Step-by-Step)](#10-response-parsing-step-by-step)
11. [MQTT Output – Simple Mode](#11-mqtt-output--simple-mode)
12. [MQTT Output – Home Assistant Discovery Mode](#12-mqtt-output--home-assistant-discovery-mode)
13. [Configuration Reference](#13-configuration-reference)
14. [Known Limitations](#14-known-limitations)
15. [Implementation Checklist](#15-implementation-checklist)
16. [Annotated Pseudocode](#16-annotated-pseudocode)

---

## 1. Device Overview

| Property | Value |
|---|---|
| Manufacturer | Hoymiles |
| Series | HMS-XXXW-2T (e.g. HMS-600W-2T, HMS-800W-2T, HMS-1000W-2T) |
| PV inputs | 2 strings (ports) |
| Internal component | Built-in DTU (Data Transfer Unit) |
| Network interface | Wi-Fi (802.11 b/g/n) |
| Cloud service | S-Miles Cloud (Hoymiles proprietary) |

> **Note:** The tool does **not** replace the internal DTU or S-Miles Cloud integration.
> It reads data off the internal DTU's local TCP listener in parallel.

The implementation has been tested with an HMS-800W-2T. Other models in the same series
are expected to be compatible based on the identical on-device DTU design.

---

## 2. Transport Layer

| Property | Value |
|---|---|
| Protocol | **TCP** (IPv4) |
| Port | **10081** |
| Address | Inverter's LAN IP or hostname (user-configured) |
| Connection timeout | 500 ms |
| Read/write timeout | 5 s |
| Connection lifetime | Per-request (short-lived; one connection per poll) |
| Maximum response size | 1024 bytes |
| Encryption | None (plaintext) |

A new TCP connection is opened for each poll, the request is written, the response is read,
and the connection is immediately closed. No persistent or keep-alive connection is maintained.

---

## 3. Authentication

**There is no authentication.** The inverter accepts TCP connections on port 10081 from any
host on the local network without requiring any credentials, tokens, or handshake sequence.

All data is transmitted in plaintext over the local network. Implementors should be aware of
the following:

- Do **not** expose port 10081 to the internet.
- Do **not** rely on this interface for security-sensitive control operations.

---

## 4. Communication Pattern (Pull-based)

The data collection is **client-initiated (pull-based)**:

```
Client (hms-mqtt-publisher)          Inverter DTU (port 10081)
        |                                      |
        |---- TCP SYN -----------------------> |
        |<--- TCP SYN-ACK -------------------- |
        |---- TCP ACK -----------------------> |
        |                                      |
        |---- Binary request packet ---------> |
        |<--- Binary response packet --------- |
        |                                      |
        |---- TCP FIN -----------------------> |
        |<--- TCP FIN-ACK -------------------- |
        |                                      |
        (wait ~30 seconds)
        |
        (repeat)
```

**Polling interval:** The inverter firmware enforces a minimum refresh period of approximately
**30 seconds**. Polling more frequently than this does not produce updated readings — the
device simply returns the previous response and resets its internal countdown timer. The
default interval used in this implementation is **30,500 ms** (30.5 s).

A **sequence number** (16-bit, big-endian, wrapping) is incremented with each request and
reflected in the response.

---

## 5. Packet Framing Format

Both request and response packets use the same 10-byte binary header prepended to a Protocol
Buffers payload.

### 5.1 Packet Header (10 bytes)

| Offset | Size | Type | Field | Description |
|--------|------|------|-------|-------------|
| 0 | 4 bytes | bytes | `magic` | Fixed value: `0x48 0x4D 0xA3 0x03` |
| 4 | 2 bytes | u16 BE | `sequence` | Request counter (increments per poll, wraps at 65535) |
| 6 | 2 bytes | u16 BE | `crc16` | CRC16-MODBUS of the serialised protobuf payload only |
| 8 | 2 bytes | u16 BE | `length` | `len(serialised_payload) + 10` |

> **BE** = big-endian byte order.

### 5.2 Request Payload

The request payload is a serialised `RealDataResDTO` protobuf message (see §6). In the
simplest (and observed working) form, this message is sent with **all fields at their default
(zero/empty) values**, producing a very short (typically 0-byte) serialised output.

### 5.3 Response Payload

The response payload starts at **byte offset 10** of the raw TCP data and is a serialised
`HMSStateResponse` protobuf message (see §6).

### 5.4 Complete Packet Layout

```
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------...-+
| 0x48   | 0x4D   | 0xA3   | 0x03   |  SEQ_H |  SEQ_L | CRC_H  | CRC_L  | LEN_H  | LEN_L  | protobuf   |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------...-+
  magic (4 B)                         seq (2B)          crc (2B)          len (2B)  payload (variable)
```

---

## 6. Protobuf Message Definitions

The protocol uses **Protocol Buffers v3** (proto3). The schema file is reproduced here in
full for implementors who need to generate language-specific bindings.

```protobuf
syntax = "proto3";

// ── Request message (sent to inverter) ───────────────────────────────────────

message RealDataResDTO {
  string ymd_hms  = 1;   // Timestamp string "YYYY-MM-DD HH:mm:SS" (year-month-day hour:minute:second; unused in minimal request)
  int32  cp       = 2;   // Checkpoint / sequence hint (unused in minimal request)
  int32  err_code = 3;   // Error code (unused in minimal request)
  int32  offset   = 4;   // Offset (unused in minimal request)
  int32  time     = 5;   // Unix epoch (unused in minimal request)
}

// ── Response messages (received from inverter) ───────────────────────────────

message InverterState {
  int64 inv_id          = 1;   // Inverter serial number (integer form)
  int32 port_id         = 2;   // Logical port identifier
  int32 grid_voltage    = 3;   // AC grid voltage  [V]   × 0.1
  int32 grid_freq       = 4;   // AC grid frequency [Hz] × 0.01
  int32 pv_current_power = 5;  // AC output power  [W]  × 0.1
  int32 unknown1        = 7;   // Purpose unknown
  int32 unknown2        = 8;   // Possibly power limit [%] × 0.1
  int32 temperature     = 9;   // Inverter temperature [°C] × 0.1
  int32 unknown3        = 10;  // Purpose unknown
  int32 unknown4        = 12;  // Purpose unknown
  int32 bit_field       = 20;  // Bitmask (purpose unknown)
}

message PortState {
  int64 pv_sn          = 1;   // PV module serial number
  int32 pv_port        = 2;   // Port index (0-based or 1-based, device-dependent)
  int32 pv_vol         = 3;   // DC string voltage  [V]  × 0.1
  int32 pv_cur         = 4;   // DC string current  [A]  × 0.01
  int32 pv_power       = 5;   // DC string power    [W]  × 0.1
  int32 pv_energy_total = 6;  // Lifetime yield     [Wh] (integer)
  int32 pv_daily_yield  = 7;  // Today's yield      [Wh] (integer)
  int32 bitfield        = 8;  // Bitmask (purpose unknown)
}

message HMSStateResponse {
  string dtu_sn          = 1;  // DTU serial number string (≥8 characters)
  int32  time            = 2;  // Unix epoch timestamp (UTC)
  int32  device_nub      = 3;  // Device count
  int32  pv_nub          = 4;  // PV string count (echoes cp from request)
  int32  package_nub     = 5;  // Package number
  repeated InverterState inverter_state = 9;   // One entry per inverter module
  repeated PortState     port_state     = 11;  // One entry per PV string
  int32  pv_current_power = 12; // Total AC output power [W] × 0.1
  int32  pv_daily_yield   = 13; // Total daily yield     [Wh] (integer)
}
```

---

## 7. Scale Factors and Units

Raw integer values in the protobuf messages must be multiplied by a scale factor to obtain
physical values:

| Field | Raw type | Scale factor | Physical unit |
|-------|----------|--------------|---------------|
| `InverterState.grid_voltage` | int32 | × 0.1 | V |
| `InverterState.grid_freq` | int32 | × 0.01 | Hz |
| `InverterState.pv_current_power` | int32 | × 0.1 | W |
| `InverterState.temperature` | int32 | × 0.1 | °C |
| `InverterState.unknown2` | int32 | × 0.1 (guessed) | % (power limit?) |
| `PortState.pv_vol` | int32 | × 0.1 | V |
| `PortState.pv_cur` | int32 | × 0.01 | A |
| `PortState.pv_power` | int32 | × 0.1 | W |
| `PortState.pv_energy_total` | int32 | × 1 | Wh |
| `PortState.pv_daily_yield` | int32 | × 1 | Wh |
| `HMSStateResponse.pv_current_power` | int32 | × 0.1 | W |
| `HMSStateResponse.pv_daily_yield` | int32 | × 1 | Wh |
| `HMSStateResponse.time` | int32 | unix epoch | UTC seconds |

---

## 8. CRC Algorithm

**Algorithm:** CRC-16/MODBUS (also known as CRC-16-IBM with initial value 0xFFFF)

| Parameter | Value |
|-----------|-------|
| Width | 16 bits |
| Polynomial | 0x8005 |
| Initial value | 0xFFFF |
| Input reflected | Yes |
| Output reflected | Yes |
| Final XOR | 0x0000 |

The CRC is computed **only over the serialised protobuf payload bytes** (not over the
10-byte header). The result is placed in header bytes 6–7 as a **big-endian u16**.

### Example (Python)

```python
import crcmod

crc16_modbus = crcmod.predefined.mkCrcFun('modbus')
crc = crc16_modbus(payload_bytes)
crc_bytes = crc.to_bytes(2, byteorder='big')
```

### Example (Go)

```go
import "github.com/sigurn/crc16"

table := crc16.MakeTable(crc16.CRC16_MODBUS)
crc := crc16.Checksum(payloadBytes, table)
```

---

## 9. Request Construction (Step-by-Step)

```
1. Increment sequence counter (u16, wraps at 0xFFFF → 0x0000)
2. Create a RealDataResDTO protobuf message with all fields at default (zero/empty) values
3. Serialise the message to bytes  →  payload_bytes
4. Compute CRC16-MODBUS over payload_bytes  →  crc (u16)
5. Compute length  =  len(payload_bytes) + 10  →  pkt_len (u16)
6. Assemble packet:
     header  = 0x48 0x4D 0xA3 0x03
     packet  = header
             + sequence.to_bytes(2, 'big')
             + crc.to_bytes(2, 'big')
             + pkt_len.to_bytes(2, 'big')
             + payload_bytes
7. Open a TCP connection to <inverter_host>:10081 (500 ms connect timeout)
8. Write packet to the socket
```

> **Minimal request:** A default `RealDataResDTO` serialises to 0 bytes in proto3
> (all fields have zero/empty defaults and are omitted from the wire format). The
> resulting header will therefore contain `crc = CRC16-MODBUS([]) = 0xFFFF` and
> `length = 10`.

---

## 10. Response Parsing (Step-by-Step)

```
1. Read up to 1024 bytes from the socket  →  raw_bytes
2. Validate (optional but recommended):
   a. Check raw_bytes[0:4] == 0x48 0x4D 0xA3 0x03
   b. Check sequence in raw_bytes[4:6] matches the sent sequence
   c. Read pkt_len from raw_bytes[8:10] (big-endian u16)
   d. Verify CRC16-MODBUS(raw_bytes[10:pkt_len]) == value in raw_bytes[6:8]
3. Extract protobuf payload  =  raw_bytes[10 : pkt_len]
4. Deserialise payload as  HMSStateResponse
5. Apply scale factors (§7) to all numeric fields before use
6. Use dtu_sn[0:8] as the short device identifier for MQTT topic construction
```

---

## 11. MQTT Output – Simple Mode

In simple mode each measured value is published as a separate MQTT topic with an ASCII
decimal string payload.

**Base topic:** `hms800wt2`

| MQTT Topic | Description | Example Payload |
|---|---|---|
| `hms800wt2/inverter_local_time` | Inverter timestamp (local time, derived from the Unix epoch `time` field, formatted as `YYYY-MM-DD HH:mm:SS.nnnnnnnnn`) | `2024-06-01 13:45:00.000000000` |
| `hms800wt2/pv_current_power` | Total AC output power [W] | `423.5` |
| `hms800wt2/pv_daily_yield` | Total daily yield [Wh] | `1840` |
| `hms800wt2/pv_grid_voltage` | Grid voltage [V] | `237.10` |
| `hms800wt2/pv_grid_freq` | Grid frequency [Hz] | `50.01` |
| `hms800wt2/pv_inv_temperature` | Inverter temperature [°C] | `38.40` |
| `hms800wt2/pv_port1_voltage` | PV string 1 voltage [V] | `31.20` |
| `hms800wt2/pv_port1_curr` | PV string 1 current [A] | `6.85` |
| `hms800wt2/pv_port1_power` | PV string 1 power [W] | `213.60` |
| `hms800wt2/pv_port1_energy` | PV string 1 lifetime yield [Wh] | `12340` |
| `hms800wt2/pv_port1_daily_yield` | PV string 1 today's yield [Wh] | `920` |
| `hms800wt2/pv_port2_voltage` | PV string 2 voltage [V] | `30.80` |
| `hms800wt2/pv_port2_curr` | PV string 2 current [A] | `6.77` |
| `hms800wt2/pv_port2_power` | PV string 2 power [W] | `208.60` |
| `hms800wt2/pv_port2_energy` | PV string 2 lifetime yield [Wh] | `11980` |
| `hms800wt2/pv_port2_daily_yield` | PV string 2 today's yield [Wh] | `920` |

- **QoS:** At Most Once (0)
- **Retain:** `true`

---

## 12. MQTT Output – Home Assistant Discovery Mode

This mode implements the [MQTT Discovery protocol](https://www.home-assistant.io/docs/mqtt/discovery/)
so that Home Assistant automatically registers sensors without manual configuration.

### 12.1 Topic Scheme

The **short DTU serial number** (`dtu_sn[0:8]`) is used in all topic names:

| Purpose | Topic pattern |
|---|---|
| Sensor config | `homeassistant/sensor/hms_<short_sn>/<unique_id>/config` |
| Sensor state | `solar/hms_<short_sn>/state` |

### 12.2 Config Payload (MQTT Discovery)

Each sensor publishes a JSON config message:

```json
{
  "unique_id": "hms_<short_sn>_<key>",
  "name": "<Human-readable name>",
  "state_topic": "solar/hms_<short_sn>/state",
  "value_template": "{{ value_json.<key> }}",
  "device": {
    "name": "Hoymiles HMS-WiFi <short_sn>",
    "model": "HMS-WiFi",
    "identifiers": ["hms_<short_sn>"],
    "manufacturer": "Hoymiles",
    "sw_version": "<app_version>"
  },
  "unit_of_measurement": "<unit>",
  "device_class": "<class>",
  "state_class": "<state_class>"
}
```

### 12.3 Registered Sensors

| Key | Name | Unit | Device class | State class |
|---|---|---|---|---|
| `dtu_sn` | DTU Serial Number | — | — | — |
| `pv_current_power` | Total Power | W | power | measurement |
| `pv_daily_yield` | Total Daily Yield | Wh | energy | total_increasing |
| `efficiency` | Efficiency | % | — | measurement |
| `pv_<idx>_power` | PV `<idx>` Power | W | power | measurement |
| `pv_<idx>_vol` | PV `<idx>` Voltage | V | voltage | measurement |
| `pv_<idx>_cur` | PV `<idx>` Current | A | current | measurement |
| `pv_<idx>_daily_yield` | PV `<idx>` Daily Yield | Wh | energy | total_increasing |
| `pv_<idx>_energy_total` | PV `<idx>` Energy Total | Wh | energy | total_increasing |
| `inv_<idx>_pv_current_power` | Inverter `<idx>` Power | W | power | measurement |
| `inv_<idx>_temperature` | Inverter `<idx>` Temperature | °C | temperature | measurement |
| `inv_<idx>_grid_voltage` | Inverter `<idx>` Grid Voltage | V | voltage | measurement |
| `inv_<idx>_grid_freq` | Inverter `<idx>` Grid Frequency | Hz | frequency | measurement |

`<idx>` is the `pv_port` / `port_id` value from the protobuf response.

### 12.4 State Payload

All sensor values are batched into a single JSON state message published to
`solar/hms_<short_sn>/state`:

```json
{
  "dtu_sn": "11223344AABB",
  "pv_current_power": "423.50",
  "pv_daily_yield": 1840,
  "efficiency": "98.23",
  "pv_0_vol": "31.20",
  "pv_0_cur": "6.85",
  "pv_0_power": "213.60",
  "pv_0_energy_total": 12340,
  "pv_0_daily_yield": 920,
  "pv_1_vol": "30.80",
  "pv_1_cur": "6.77",
  "pv_1_power": "208.60",
  "pv_1_energy_total": 11980,
  "pv_1_daily_yield": 920,
  "inv_0_grid_voltage": "237.10",
  "inv_0_grid_freq": "50.01",
  "inv_0_pv_current_power": "423.50",
  "inv_0_temperature": "38.40"
}
```

- Floating-point values are formatted with 2 decimal places as strings.
- Integer values (`pv_daily_yield`, `pv_energy_total`) are plain JSON numbers.
- **QoS:** At Most Once (0)
- **Retain:** `true`

---

## 13. Configuration Reference

The application is configured via `config.toml` in the working directory (or next to the
executable).

```toml
# Required: IP address or hostname of the inverter
inverter_host = "192.168.4.182"

# Optional: polling interval in milliseconds (minimum effective value: ~30500)
# Default: 30500
update_interval = 30500

# Optional: Home Assistant MQTT discovery output
[home_assistant]
host     = "192.168.178.250"
port     = 1883           # optional, default 1883
username = "mqttuser"     # optional
password = "MqttPass1"    # optional
tls      = false          # optional

# Optional: simple flat-topic MQTT output
[simple_mqtt]
host     = "192.168.178.250"
port     = 1883
username = "mqttuser"
password = "MqttPass1"
```

When deployed with Docker, the following environment variables can be used instead of
`config.toml`:

| Variable | Required | Description |
|---|---|---|
| `INVERTER_HOST` | Yes | Inverter IP/hostname |
| `MQTT_BROKER_HOST` | Yes | MQTT broker IP/hostname |
| `MQTT_USERNAME` | No | MQTT username |
| `MQTT_PASSWORD` | No | MQTT password |
| `MQTT_PORT` | No | MQTT port (default 1883) |

---

## 14. Known Limitations

| Limitation | Detail |
|---|---|
| Minimum poll interval | ~30 s; polling faster returns the previous reading and resets the inverter's internal timer, also interrupting its own S-Miles Cloud updates |
| Response buffer size | 1024 bytes; may need to be increased if future firmware versions return larger payloads |
| Model detection | No reliable way to identify the exact model from the protocol; the model string is hard-coded as `"HMS-WiFi"` |
| No response CRC validation | The reference implementation does not validate the response CRC before parsing |
| No response header validation | The magic bytes and echoed sequence number are not checked before protobuf parsing |
| Untested models | Only HMS-800W-2T has been tested; other HMS-XXXW-2T models are expected to work |
| No encryption | All traffic is plaintext; the interface must only be used on a trusted local network |
| Single-phase grid only | The protocol structure contains a single `InverterState` for HMS-XXXW-2T; multi-phase or higher-channel inverters would return multiple `InverterState` entries |

---

## 15. Implementation Checklist

Use this checklist when implementing a compatible client in another language or framework:

- [ ] Implement CRC-16/MODBUS (polynomial 0x8005, init 0xFFFF, reflected in/out)
- [ ] Implement proto3 serialisation/deserialisation of `RealDataResDTO` and `HMSStateResponse`
- [ ] Maintain a wrapping u16 sequence counter
- [ ] Build the 10-byte packet header correctly (magic + seq + crc + len, all big-endian)
- [ ] Open a TCP connection to `<host>:10081` with ≤500 ms connect timeout
- [ ] Write the full request packet; read up to 1024 bytes
- [ ] Skip the first 10 bytes of the response before passing to protobuf parser
- [ ] Apply the correct scale factors to all numeric fields (§7)
- [ ] Enforce a ≥30 s delay between polls
- [ ] Handle `NetworkState` transitions (Unknown → Online / Offline) with appropriate logging
- [ ] (Optional) Validate response magic bytes and echoed sequence number
- [ ] (Optional) Validate response CRC before parsing

---

## 16. Annotated Pseudocode

The following language-agnostic pseudocode illustrates a complete poll cycle:

```
CONST MAGIC      = [0x48, 0x4D, 0xA3, 0x03]
CONST PORT       = 10081
CONST BUF_SIZE   = 1024
CONST POLL_DELAY = 30500  # milliseconds

sequence = 0  # u16, wraps

FUNCTION poll(host):
    sequence = (sequence + 1) & 0xFFFF

    # 1. Serialise an empty RealDataResDTO (all defaults → 0 bytes in proto3)
    payload = proto3_serialise(RealDataResDTO{})

    # 2. Compute CRC over payload
    crc = crc16_modbus(payload)        # u16

    # 3. Compute packet length
    pkt_len = len(payload) + 10        # u16

    # 4. Assemble packet
    packet = MAGIC
           + u16_big_endian(sequence)
           + u16_big_endian(crc)
           + u16_big_endian(pkt_len)
           + payload

    # 5. Send and receive
    sock = tcp_connect(host, PORT, timeout_ms=500)
    sock.set_timeout(write=5000, read=5000)
    sock.write(packet)
    raw = sock.read(BUF_SIZE)
    sock.close()

    # 6. Parse response (skip 10-byte header)
    response = proto3_deserialise(HMSStateResponse, raw[10:])

    # 7. Extract and scale values
    total_power_W     = response.pv_current_power * 0.1
    daily_yield_Wh    = response.pv_daily_yield
    port0_voltage_V   = response.port_state[0].pv_vol  * 0.1
    port0_current_A   = response.port_state[0].pv_cur  * 0.01
    port0_power_W     = response.port_state[0].pv_power * 0.1
    grid_voltage_V    = response.inverter_state[0].grid_voltage  * 0.1
    grid_freq_Hz      = response.inverter_state[0].grid_freq     * 0.01
    temperature_C     = response.inverter_state[0].temperature   * 0.1

    RETURN response

LOOP:
    result = poll(inverter_host)
    IF result IS NOT NULL:
        publish_to_mqtt(result)
    sleep(POLL_DELAY)
```

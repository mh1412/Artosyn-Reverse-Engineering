# Nomad USB CRSF and receiver-profile activation

[Protocol index](README.md) · [Physical control and telemetry results](../systems/nomad-usb-control.md)

Scope: the custom `USB_CRSF_CONTROLLER` Nomad target and its Python client.
The profile commands below are a custom version-1 extension, not a general
ExpressLRS or Artosyn protocol. The ordinary module-bay target is separate.

## Serial and channel framing

The CP2102 connection uses **460800 baud, 8N1**, no flow control and deasserted
DTR/RTS. Opening the port can still reset the ESP32; the client waits for boot.
The dedicated target uses Normal link mode with AirPort disabled. Raw MAVLink
is a separate mode rather than a payload multiplexed with this USB CRSF stream.

```text
SYNC=C8 | LENGTH | TYPE | PAYLOAD | CRC_D5
```

`LENGTH` counts type, payload and final CRC; total frame size is `LENGTH + 2`.
The outer CRC starts at zero and covers type through payload using polynomial
`0xD5`. Channel type `0x16` carries sixteen little-endian packed 11-bit channels:
22 payload bytes, or 23 with the optional ELRS arm-status byte. Valid length
values are consequently 24 and 25. Representative raw channel values are
172 minimum, 992 center and 1811 maximum. Without the optional status byte,
CH5 supplies arm state.

Only CRC-valid, correctly sized channel frames refresh the RC session. The
first such frame starts RF; configuration, discovery and malformed channel
traffic do not. Freshness uses unsigned 32-bit microsecond subtraction and
strictly less than `250000`, so the session is stale at the exact 250 ms
deadline. The main loop stops the RF timer; the timer's independent freshness
guard suppresses stale RC while the loop is busy. Special WiFi/update/failure
modes are not escaped by incoming controls. Explicit binding is separate.

USB input frequency and RF packet rate are independent. The client supports
20–500 Hz bench input; physical runs used 50 or 100 Hz with 500 Hz RF.
Opening the settings application does not start controls. Its explicit link
action sends real neutral/low-throttle channels; it is not a silent keepalive.

## Profile command fields

Extended CRSF command type `0x32` has destination and origin before the payload.
Requests target transmitter address `0xEE`; host origin `0xC8` is supported.
Responses return directly on USB to the requesting origin, avoiding ambiguity
with the FC's address. The handler is invoked only from the USB parser.

```text
C8 | LENGTH | 32 | DEST | ORIGIN | PROFILE_PAYLOAD | CRC_BA | CRC_D5
```

The inner CRC uses polynomial `0xBA`, initial zero, over type, destination,
origin and profile payload. The outer `0xD5` CRC additionally includes that
inner CRC. Both are required. Multibyte tokens are big-endian; the six UID
bytes are copied in their supplied order.

| Action | Profile payload, excluding both CRC bytes | Payload / total bytes |
| --- | --- | --- |
| Apply | `0A 80 01 TOKEN_BE16 UID[6] RATE POWER MODEL` | 14 / 21 |
| Query | `0A 81 01 TOKEN_BE16` | 5 / 12 |
| Reply | `0A 82 01 ACTION TOKEN_BE16 STATUS UID[6] RATE POWER MODEL` | 16 / 23 |

`RATE` is the firmware RF-rate enum, not a menu index: the tested 2.4 GHz /
500 Hz preset is decimal 29. Power indices begin `0=10`, `1=25`, `2=50`,
`3=100` mW; hardware/domain validation can reject nominal enum values.
The power code in returned link-statistics telemetry has a different mapping
and must not be decoded with this configuration table. Model IDs are 0–63.
Basic UID validation requires at least one of the last four UID bytes nonzero.

| Status | Meaning |
| --- | --- |
| 0 | Accepted / current settings returned |
| 1 | Invalid profile |
| 2 | Unsupported RF rate |
| 3 | Unsupported power / outside hardware limit |
| 4 | Busy, armed, binding, AirPort or special operating mode |
| 5 | Options storage failed |

The status enum does not promise an error reply for every invalid packet.
Bad length/version/command CRC and invalid settings rejected by the decoder
are consumed without a reply or state mutation; the host times out.

Golden fixture: host origin `0x10`, token `0x1234`, UID derived from test
phrase `RX-A`, rate 29, power 3, model 0:

```text
Apply: C8 13 32 EE 10 0A 80 01 12 34 01 E7 98 08 C7 09 1D 03 00 38 C0
Query: C8 0A 32 EE 10 0A 81 01 12 34 E8 03
```

Both CRCs and lengths were independently recomputed during documentation review.
This fixture establishes byte framing, not successful receiver provisioning.

## Apply, reboot and readback

The firmware checks rate/domain support, the actual power cap and operating
state before changing identity. Accepted Apply then stops the old RF session,
waits for active transmission to finish, clears RC/OTA/telemetry state, persists
UID/model options and commits radio settings. It selects Normal mode, Model
Match off and dynamic power off, adapts the switch mode to payload size and
selects Gemini for dual-band modes. A reply precedes a scheduled reboot after
200 ms. Pending reboot blocks new RC from restarting the old link.

The options file uses temporary-file write, flush, close and rename; radio
configuration is committed separately. This does **not** establish an atomic
whole-profile transaction under power loss. A storage failure restores the
in-memory options, but RF has already been stopped. The source also contains
an unbounded wait for `busyTransmitting`, so successful bench Apply cannot be
generalized to bounded completion under every radio fault.

The client separates these states:

```text
Validate saved profile → stop old RC → Apply → await reboot
  → Query persisted UID/rate/power/model → start neutral bench RC
  → fresh receiver discovery AND positive fresh link quality → active
```

Reply acceptance checks origin/destination, version, command CRC, current
transaction token and the complete settings tuple. Matching Apply is not the
post-reboot readback. Wrong Query results stop activation; the client does not
silently resume the previous controls. A missing receiver fails after a
12-second discovery window and stops the bench stream.

During activation, USB interruption can trigger reopening the same port,
waiting for boot and querying again. It clears cached receiver/telemetry state
and does not blindly reapply the profile. Repeated reconnect attempts share
a 12-second transport-recovery budget. The receiver-discovery window is a
separate deadline, not a guarantee that the whole activation lasts 12 seconds.
Outside activation, ordinary disconnect requires manual reconnection.

## Identity and evidence boundaries

Selecting a host profile changes host metadata only. Activating one changes
the transmitter; it does not configure or bind the receiver. A model slot
selects stored radio settings and is not an independent receiver identity.
These profiles use distinct UIDs with Model Match disabled.

Physical testing verified UID/model persistence and restoration of a provisioned
receiver link. Both later two-receiver attempts failed at receiver A despite
verified transmitter settings. The client state machine also passed simulated
switching/reconnect tests, which do not supersede those failures. See the
[bench record](../systems/nomad-usb-control.md#profile-activation-and-the-unsuccessful-two-receiver-test)
for the exact distinction.

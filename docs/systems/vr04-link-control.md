# VR04 reference link lifecycle, binding and adaptive controls

[Systems](README.md) · [Reference board](../devices/betafpv-vr04-reference.md) ·
[Media protocol](../protocols/vr04-video-session.md)

## Implementation scope

The retained open stack separates radio bring-up, receiver/air session control,
media and HUD processes. Findings here refer to MissingLynk userspace
`674ec00ecde0bba2eb844def1b4a0b7f8e685e13` and its recorded reference-board tests.
Daemon defaults, boot-service flags and measured runtime behavior are distinct.
Source inspection during consolidation found several README descriptions older
than the actual power/session implementation; the corrections are explicit below.

## Bring-up and receiver cadence

The one-shot bring-up tool releases reference GPIO23, reprobes MMC host
`1b00000.mmc`, waits up to ten seconds for SDIO `8030` or `8031`, loads the radio
driver and waits up to ten seconds for `sdio0`. It then sets the configured
address and optional MTU. If the driver is already loaded it skips reset/reload
and proceeds to netdev setup, because warm module reload hung the reference board.
These GPIO/host choices are board-specific despite an older claim that the
sequence applies wherever an AR8030 is fitted.

The reference pairs are `bb_demo_gnd_d.img` with race/normal ground JSON, and
`bb_demo_air_d.img` with air JSON. A persisted configuration directory precedes
the packaged firmware directory in the loader search path. This is RAM radio
firmware loading, separate from flashing the host operating system.

| Receiver state | Recorded operation |
| --- | --- |
| Wait | Retry opening the baseband device about once per second |
| Associate | Send 22 recovered association frames with 20 ms spacing |
| Settle | About 2.5 s of port `0x73` polling near 6 Hz and `ff02` heartbeat near 3.4 Hz |
| Open | Set channel and RX power once; establish TX-power/automatic-control policy |
| Steady | Port `0x0c` near 24 Hz, port `0x73` near 6 Hz and heartbeat near 3.4 Hz |

Port `0x0c` polling during association, or the earlier approximately 150 Hz loop,
wedged the air unit in the recorded trials. The minimum opening sequence omitted
five readbacks and a 136-byte remote-info frame identified as uninitialized stack
data, without losing video in the reported cold-boot test. There was no separate
channel-4 “open” command.

The early recipe selected fixed channel index 5. The later implementation first
reads the chip's valid-channel bitmap, then chooses a valid saved index or the
first valid channel. If the table does not arrive, it sets **no channel**. This
avoids briefly tuning a normal-band chip outside its configured set. Selected
power values of 23 dBm are control settings, not calibrated radiated measurements.
The observed high-bandwidth `type:8` profile emerged from configuration/SNR;
no direct “set type 8” command was established.

The daemon owns baseband control and UDP ports 10000/20001; the media consumer
owns port 10001. Local `link.sock` carries READY and control requests,
`telemetry.sock` carries status/link events, and `osd.sock` carries MSP canvas
updates. Nonblocking datagram delivery drops errors instead of waiting for a HUD.
See [media sessions](../protocols/vr04-video-session.md) for readiness and recovery.

## Air role and binding

Air association follows the loaded baseband configuration. Ordinary telemetry
uses UDP; rate adaptation, power control and binding share a reference-counted
baseband descriptor. Two independent opens from one process had wedged the chip.
Air service ticks are normally 50 ms, reduced to 20 ms during pairing; periodic
transmissions use elapsed time so that changing the tick does not multiply rates.

The air reports type `0x11` near 6 Hz and type `0x09` near 1 Hz. With the
20-byte media prefix and twelve-byte zero trailer, their layouts are:

| Type | Total / body bytes | Body-relative fields written by the open builder |
| --- | --- | --- |
| `0x11` | 38 / 6 | Arm byte at 0, LE16 consumed-charge value named `mah_drawn_x10` at 2, voltage mV at 4 |
| `0x09` | 160 / 128 | Up to 16-byte hardware/firmware strings at 0/32, LE16 voltage mV at 96 and temperature at 98 |

Battery/temperature acquisition resolves IIO devices by name and retries after
coldplug. Sensor units and calibration are the separate
[analog-provider contract](../soc/ar9311-analog-and-pwm.md); placing a value in
telemetry does not independently validate its physical accuracy. MSP canvas
traffic uses type `0x10`, with a separately packed OSD body.

The recorded bind policy opens a 30-second window for a button hold of at most
two seconds. `gpio-keys-polled` samples at 50 ms, so hold measurements have that
quantization. Event timestamps are compared with each other, while the window
uses a monotonic deadline; negative holds are rejected. The nonblocking state
machine lets the FC UART continue draining during pairing.

1. Refuse binding while GET_MCS throughput indicates an existing link, or if the
   chip fails to answer within 300 ms.
2. Save the configured `ap_mac`, write broadcast `ffffffff`, and enter pair mode.
3. Poll GET_PAIR every 20 ms. The lowest candidate-mask bit selects a slot; its
   four-byte peer identity begins at `1 + slot×4` in wire order.
4. Accumulate six hits, exit pairing on that slot, then commit the peer on the
   following tick. Zero reads do not erase accumulated hits.
5. Persist only an accepted commit. On failure/expiry restore the saved identity
   with bounded retries.

The persisted value lives in a writable copy of the air JSON because the chip
reloads configuration at module load. If no original `ap_mac` can be read, an
unsuccessful window has nothing to restore and can leave broadcast binding until
power-cycle. This retained limitation prevents treating every timeout as rollback.
The red pairing blink suppresses the green LED because of the reference air
board's measured LED-current interaction; it is unrelated to the goggle RGB chain.

## Rate governor

The binary defaults adaptation off; the recorded air boot service enables the
rate governor. It polls GET_MCS every 200 ms. The eight-byte reply uses byte 0
minus 2 for MCS and LE32 at offset 4 for throughput in kbps. Recorded examples:

| Reply | Interpretation |
| --- | --- |
| `19 00 00 00 00 00 00 00` | MCS 23, throughput zero, unassociated |
| `0c 00 00 00 be 51 00 00` | MCS 10, throughput 20,926 kbps, associated |

Thus a higher idle MCS number is not evidence of a better link. Negative decoded
indices are rejected. For accepted samples the shipped target is
`floor((throughput_kbps / 100 using integer division) × 100 × 0.7)`, capped at
20,000 kbps, with an 8,000-kbps fallback when the result is zero. It is a **total
for two tiles**, divided equally for per-tile encoder control.

The first valid sample seeds the target. MCS drops apply immediately; rises must
hold for one second. Throughput drift at unchanged MCS does not itself trigger
a new command. SetLdCfg's `bitrate_q` is a ceiling in 250-kbps units, applied
after the cap/fallback; changes recompute using the retained sample. Zero removes
that extra ceiling. The captured base value 96 means 24 Mbit/s and therefore
does not constrain the governor's existing 20-Mbit/s maximum.

Honouring that menu ceiling is an open-stack behavior: the examined vendor air
stored the field without using it for encoder rate. A recovered low-MCS 0.8
branch and frame-rate ladder remain in source, but the configured threshold is
`−1` and negative samples are rejected. The runtime governor commands bitrate
only; its recovered frame-rate helper is not connected to the encoder.

## Power, standby and later corrections

Power adaptation is a separate option. Within an active commanded-power session,
fresh FC arm state controls the radio target, while the goggle's standby command
controls capture rate:

| FC state | Standby command | Capture rate | Radio target |
| --- | --- | --- | --- |
| Absent/stale, arm unknown | Off / on | 60 / 15 fps | Goggle-commanded value |
| Present, disarmed | Off / on | 60 / 15 fps | Minimum setting, 5 dBm |
| Present, armed | Either | 60 fps | Goggle-commanded value |

Absent FC telemetry must not be interpreted as a known disarm. No power command
is applied before a valid commanded value has arrived; startup power comes from
the baseband configuration. The allowed command range is 5–23 dBm. Contrary to
the older README, an invalid power value leaves the old power **but still accepts
the separately supplied standby bit**. Applying manual power first disables chip
automatic adjustment; unchanged targets avoid repeated writes.

SetTranParm supplies live power/standby, while SetLdCfg seeds them on association.
Their [wire layouts](../protocols/vr04-stock-ui.md#air-directed-control-bodies)
must agree: a captured default standby bit had silently re-enabled standby on
the next association despite a prior live command disabling it.

The later power implementation **does not wait for standby ACK** before reducing
capture rate. Vendor-goggle observations recorded only three command datagrams
over a 90-second off/on/off test, with no acknowledgement. Waiting for one left
the menu showing standby while video continued at 60 fps. Type-`0x12` mode
reports remain; unacknowledged standby reports repeat every 500 ms up to ten
successful sends. An ACK stops repetition, rather than permitting the rate change.

The control is `capfps`, changing only capture feeding. Encoder-declared rate
stays unchanged so fewer pictures can reduce the emitted bitrate. Changing both
with `fps` instead had held approximately 1,015 kB/s across 60/15 fps in the
recorded experiment. Transmit duty-cycle policy remains AP-owned and is not
implemented by this control.

The actual air-side silence threshold is **120 seconds**, superseding five seconds.
It releases commanded power back to chip automatic control but **preserves standby**.
Healthy vendor receivers can be silent for minutes, so silence is not a validated
association-loss signal. The old timeout repeatedly cancelled valid standby.
This heuristic still has limits; it is distinct from the open receiver's own
five-second air-telemetry-loss detector.

GET_POWER readback is polled at 1 Hz, but its `{direction,dBm}` interpretation
was originally inferred from SET payload shape. Commanded values and software
readback do not establish measured output power, range, interference resilience
or flight reliability. Probe modes suppress control writes but still perform
read requests; “probe” does not mean no baseband traffic.

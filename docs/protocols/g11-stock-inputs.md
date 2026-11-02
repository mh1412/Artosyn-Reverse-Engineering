# G11 stock MCU inputs and dual wire formats

[Protocols](README.md) · [G11](../devices/skydroid-g11.md)

## Scope

The stock-input backend connected the existing MCU channel feed to the real
EdgeTX ADC/switch interfaces and Linux mixer without replacing MCU firmware.
Its values already include stock channel assignment, endpoints, reversal,
calibration and any enabled stock mixing. They are not raw ADC samples.
The resulting EdgeTX mixes were observed locally; this backend did not send
them back to the stock MCU or to ELRS.

This path uses exclusive `ttyS1` at 115200 8N1. It is distinct from the
[custom MCU protocol](g11-custom-control.md) and later autonomous mixer.

## Requests and trainer traffic

The backend sends only A2 channel-configuration and B1 channel-read requests,
using payload `52 01`, the vendor prefix and XOR checksum. B1 replies contain
16 big-endian channel values. Startup requires a fresh A2/E2 exchange to
validate configured physical IDs and endpoints before accepting channels.

The same Linux UART also carries checksum-validated `SKYDROID:` B1/20 trainer
packets. Their scaled values differ from vendor command replies. Archived stock
0.5 code converts the same underlying trainer array to display B1 values as:

```text
display = (raw - 42) * 5 / 8 + 900
```

The division uses signed integer semantics. The dual-format reader applies
this conversion to trainer data. Once trainer traffic arrives it listens
continuously rather than repeatedly querying B1. The internal observation
does not prove which external connector shares that signal.

The [stock MCU reconstruction](../firmware/g11-stock-mcu-reconstruction.md#captured-trainer-slave-stream)
provides the complete 44-byte trainer layout and measured external 8N2 timing.
Its external capture and the internal Linux UART observations are distinct.

## Mapping and freshness

The observed stock configuration maps one-based RC slots as follows:

| RC slot | EdgeTX input |
| --- | --- |
| 4, 3, 2, 1 | LH, LV, RV, RH respectively |
| 8, 9 | P1, P3 |
| 5, 6, 7, 10, 11 | SA, SB, SC, SD, SF respectively |

Each analog is normalized piecewise around the configured midpoint into
0–4095 before EdgeTX calibration and mixing. Native calibration uses the
half-resolution convention, nominal midpoint 1024. Switches use quarter-range
thresholds; stock configuration supplies button latch/lock behavior. Unmapped
analogs are centered and unmapped switches are up. The deployment used Mode 2
to match stock's America setting.

One worker owns the UART; GUI and mixer copy snapshots under a short mutex.
There is at most one outstanding request. Command, length, checksum, raw width
and mapped endpoints with a 50-unit tolerance are validated. A valid sample
expires after 100 ms, measured from Linux receipt. The unsequenced stock wire
format cannot establish acquisition identity or acquisition-to-mixer latency.

Initial 200-reply timing averaged 11.26 ms, but the first longer UI run stopped
after 199 samples. The revised polling reader used a minimum 20 ms request
period and a 500 ms response deadline. On loss it marks input unavailable,
drains old traffic, revalidates configuration and then resumes. A late B1
reply cannot satisfy the E2 configuration barrier. Changed mapping/endpoints
stop acquisition rather than silently reinterpret controls.

An isolated out-of-range trainer value later permanently stopped the original
reader. The repaired version rejects malformed values without refreshing the
sample and continues within the receive deadline; sustained invalid traffic
becomes stale and enters recovery. The offending original bytes were not
logged, so their origin remains unknown.

## Physical validation and limits

The dual-format deployment passed 120 observations approximately one second
apart, all live, with 11,981 accepted frames between endpoints and no logged
500 ms timeout. That is roughly 100 frames/s, not measured control latency;
the observations cannot exclude every shorter freshness gap.

The first attempted movement capture had 398 observations, all unavailable,
with the accepted-sample counter stuck at 57,830. Its preflight already showed
`live=false`; a running UI process was mistaken for a healthy input stream.
That capture cannot validate movement, mapping or endpoint coverage.

After malformed-frame recovery was repaired, 600 physical observations remained
live with 7,091 accepted frames between endpoints and no rejections/timeouts.
All eleven mapped channels changed; all four stick mixer outputs reached
−1024…1024. The operator confirmed all channels worked and were mapped correctly.
The capture mixed several controls, so individual identities/directions rely
on that observation rather than isolated one-control traces.

Stick and five switch/button channels covered 1050–1950. Auxiliary channels
covered 1130–1950 and 1143–1950. The capture did not establish whether both
wheels reached their physical lower limits; it is neither full wheel-endpoint
qualification nor evidence of a calibration fault. Mode, calibration and model
survived the recorded application exit/restore/restart.

Input loss centers analog API values and returns default switch states, with
visible loss indication. That is local fallback, not receiver failsafe. The
temporary stock-input session did not install a reboot startup path. Longer
stability under ordinary video/UI use and complete wheel travel remained open.

## UART ownership

SIGSTOP leaves the vendor application's UART descriptor open. The temporary
session wrapper paused its restart monitor, terminated the stock owner and
closed the old custom UI before opening the port exclusively. Cleanup restored
the monitor and ordinary UI. Zombies had no live descriptor even though BusyBox
`pidof` still listed them. A later thread-resource launch failure led to a
2 MiB UI stack limit instead of inherited 8 MiB; this did not modify kernel
limits or vendor boot scripts.

Host fixtures ran real EdgeTX against fragmented replies and malformed but
checksum-valid input, covering mapping, mixes, stale detection and the
configuration barrier. Those tests complement the recorded physical movement;
neither establishes ELRS channel delivery. See
[UART reliability and ownership](../firmware/g11-uart-and-session-lifecycle.md).

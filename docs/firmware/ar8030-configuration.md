# AR8030 stock configuration and board boundaries

[Firmware](README.md) · [Link management](../protocols/ar8030-link-management.md)

The matched Ascent 18.21.10 study separates the embedded radio configuration,
external loader JSON, board defaults, OTP identity and per-unit factory data.
A generic downloaded image does not identify all of those inputs.

The matched board profile contains 704 ordered default entries (5,632 bytes),
40 bytes of pin data and 100 bytes of encoding keys. The external configuration
overlays the fixed 16 KiB reservation; see [boot and loading](ar8030-boot-and-lifecycle.md).

## Recovered JSON meanings

Paths below are relative to `baseband.basic` unless stated otherwise:

| Path | Recovered meaning |
| --- | --- |
| `role` | AP/DEV role; embedded template and externally loaded role can differ |
| `channel.freq` | Frequency catalog in integer kHz |
| `channel.init_chan` | Zero-based initial index; −1 denotes configured search |
| `dev.mac`, `dev.ap_mac` | Four-byte identities encoded as eight hexadecimal characters |
| `dev.mcs.slot.init` | Configured MCS value plus 2 gives internal index; 25 denotes no selection |
| `dev.mcs.slot.delay_start` | Eligible sample calls, not elapsed milliseconds |
| `dev.mcs.slot.hold_time`, `max_wait_time` | Policy timing fields |
| `power.pwr_offset` | Four signed per-unit offset entries |
| `power.csma_offset` | CSMA adjustment; recovered default 8 |
| `dist_calc.enable` | Selects a synchronization-MMIO path; not proof of calibrated distance |

MCS policy uses an eight-sample window; a gap over 200 ms resets readiness.
If the initial selector has no configured row, policy uses the minimum SNR and
downgrade thresholds. Raw SNR thresholds are not independently calibrated dB.

Calibration rows under `baseband.power_calibration.default.closeloop` use names
such as `9dbm` through `39dbm`, with four byte values for the two gain/amplitude
paths. These names are calibration indices, not measurements of emitted power.
The matched external file supplies rows 9–36; missing rows cannot be assumed
present merely because a command accepts higher level numbers.

The investigated VRX has radio-accessible SPI factory storage. The P1 instead
keeps calibration in host NAND; see [P1 calibration](p1-calibration-and-settings.md).
Board, image and configuration identities must accompany every interpretation.

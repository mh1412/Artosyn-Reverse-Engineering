# AR8030 configuration and board-profile boundaries

[Firmware](README.md) · [Link management](../protocols/ar8030-link-management.md)

This records the early matched Ascent 18.21.10 DEV loader contract. It is a
bounded reconstruction schema, not the full vendor JSON specification or the
later AP/P1 configuration. Parser ranges describe accepted input, not validated
RF operation across every value.

## Four sources of configuration

| Source | Responsibility |
| --- | --- |
| Compiled board profile | Ordered register defaults, pins, FCH keys and flash descriptions |
| Loader JSON | Role, frequency catalog, identities, power tables and MCS policy |
| OTP/factory storage | Unit identity and power offsets |
| Linux host | Runtime candidate list, lifecycle, sockets and selected controls |

The matched board profile contains 704 ordered default entries (5,632 bytes),
40 bytes of pin data and 100 bytes of encoding keys. A null plan pointer selects
JSON-driven construction; a non-null pointer selects a distinct compiled-plan
path. Neither substitutes for board provenance.

The [boot and loader study](ar8030-boot-and-lifecycle.md) explains the fixed
configuration reservation, unterminated external-file overlay, consumed-object
CRC and OTP identity selection. The accepted object boundary differs from the
external file length and the stale embedded text that can follow it.

Synthetic parser fixtures and emulator tests are not deployable calibration.
The historical public source build lacked all private profile-generation inputs;
later pinned extraction/provisioning work must be evaluated per board.

## Fixed mode requirements

Paths below are relative to `baseband.basic`:

| Path | Required value in this contract |
| --- | --- |
| `role` | `"dev"` |
| `dev.clock` | `"200MHz"` |
| `channel.band_mode`, `channel.init_band` | `"band_single"`, `"5G"` |
| `channel.mode`, `channel.band_hop` | `"manu"`, `"manu"` |
| `channel.fs_bw` | String `"5"` |
| `power.mode` | `"closeloop"` |
| `power.pwr_auto`, `power.link_auto` | `"manu"`, `"manu"` |
| `lna.inner.mode`, `lna.fem.mode` | `"manu"`, `"manu"` |
| `lna.inner.manu`, `lna.fem.manu` | `"on"`, `"on"` |
| `channel.multi_mode.enable`, `channel.rc_hop.enable` | `false` |
| `dev.mcs.slot.bw_auto.enable` | `false` |

The fixed `para` profiles are:

| Profile | Bandwidth string | Mode | Interleave count |
| --- | --- | --- | ---: |
| `br.tx` | `"5"` | `2TX` | 2 |
| `br.rx` | `"5"` | `2T2R_STBC` | 2 |
| `slot.tx` | `"2.5"` | `2TX_STBC` | 1 |
| `slot.rx` | `"2.5"` | `2T2R_STBC` | 1 |

All require payload and time interleaving enabled, interleave length 12 and FCH
information length 96. Exact string types matter. These values are not automatic
bandwidth negotiation or a claim that other firmware uses the same contract.

## Frequency and identity

`channel.freq` accepts 1–42 distinct integer kHz values, above 4,300,000 and at
most 7,500,000, each subject to PLL validation. These software bounds do not
establish usable tuning range on a fitted board. `channel.init_chan` is a
zero-based index, or −1 for configured search.

`dev.mac` and `dev.ap_mac` are eight hexadecimal characters: four-byte radio
identities, not six-byte Ethernet addresses. OTP replaces the local JSON identity
when available; missing OTP can permit JSON fallback, while other read failures
fail the studied boot path. Runtime host candidate selection remains separate.

`para.slot.tx.retx_count` accepts 0–127. `dist_calc.enable` selects a recovered
synchronization-MMIO path; it does not establish calibrated distance output.

## Receive MCS policy

Paths are relative to `baseband.basic.dev.mcs.slot`:

| Field | Range/type | Meaning |
| --- | --- | --- |
| `enable` | Boolean | Receive-policy enable |
| `mode` | `"auto"` or `"manu"` | Selection policy |
| `init` | −2…23 | Internal index is value + 2; 25 means no selection |
| `hold_time` | 0…65,535 | Milliseconds |
| `delay_start` | Optional 0…65,535, default 0 | Eligible sample calls, not milliseconds |
| `max_wait_time` | 0…65,535 | Milliseconds; 65,535 disables timeout RX reset |
| `mcs_tab` | 1–10 rows | Strictly increasing MCS values |

Rows accept MCS −2…10, up/down LDPC counts 0…255, continuous-down count
−128…127, raw SNR thresholds 0…65,535 and keep times 0…65,535 ms.
An eight-sample window supplies quality; a gap over 200 ms discards readiness.
If the initial selector has no row, selection uses the window's minimum SNR
and downgrade thresholds. Raw thresholds are not calibrated dB values.

## Power and calibration

`baseband.basic.power.manu_init` accepts 9–36. Calibration rows under
`baseband.power_calibration.default.closeloop` use names `9dbm` through `39dbm`,
each holding four hexadecimal byte strings for two paths' gain/amplitude.
Gain is 0–63; amplitude is 0–255. Sparse tables are allowed only when every
initial/offset-adjusted row required by selection exists.

Those row names are indices in a calibration scheme, not measurements of emitted
RF power. Four optional `pwr_offset` integers range from −6 to +6; the two
implemented paths use entries 0 and 2 and clamp selection to 9–39. Optional
`csma_offset` ranges 0–255 and defaults to 8.

Recognized version-3/4 factory records override JSON offsets; permitted fallback
then uses JSON and finally zeros. Flash identification/controller failure is
not proof that calibration is absent. The early loader always selected the
`default` table despite a helper's broader named-board capability.

One historical accepted receiver baseline used initial level 25. Changing only
the accepted configuration to 11 failed acquisition/video, and restoring 25
restored the stream. This does not prove all lower powers are impossible; it
demonstrates that parser acceptance alone was insufficient hardware validation.

Later live VRX work verified a version-4 record in radio-accessible SPI flash.
P1 instead provisions host NAND factory offsets into RAM configuration. See
[P1 calibration](p1-calibration-and-settings.md) rather than applying the early
receiver fallback assumptions to that board.

## Accepting a changed configuration

The matched loading workflow verifies profile, image and configuration hashes.
Changing a JSON field or replacing an expected hash does not validate a new
operating mode. A new baseline needs its own board/image identities, parser and
state checks, radio behavior, direct video evidence and restoration result.

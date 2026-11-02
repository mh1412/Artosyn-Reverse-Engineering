# VR04 channel tables, link metrics and active surveys

[Protocols](README.md) · [Link lifecycle](../systems/vr04-link-control.md) ·
[Media sessions](vr04-video-session.md)

This study covers the retained MissingLynk receiver channel implementation at
userspace revision `674ec00ecde0bba2eb844def1b4a0b7f8e685e13`. It separates
chip-reported fields, locally cached state and an active retuning survey. The
survey interrupts video; it is not a passive spectrum measurement.

## Tables and measurements

GetScanResult supplies a channel table and trailing valid-channel bitmap.
The parser reads a byte count at offset 0, caps it at 19 and reads LE32 kHz
frequencies from offset 4. After integer conversion to MHz, only 5,000–6,100 MHz
values survive its sanity check. The observed race profile uses indices 3–18;
normal-band indices are 0–2, but the normal-mode raw payload layout remains
unverified. Channel numbers passed to selection are table indices, with no +1.

The parser requires only eight bytes. It does not require enough data for every
claimed frequency or reserve the trailing bitmap bytes from frequency parsing.
On a short record it can publish the claimed count and bitmap while leaving
unfilled entries zero-initialized. A nonzero bitmap gates later selection, but
zero means “band unknown” and allows any index 0–18. A valid-bit check therefore
does not establish a complete, independently validated frequency table.

Get1V1Info carries the metrics used for link display and active survey:

| Offset | Representation | Receiver interpretation |
| --- | --- | --- |
| `0x06` | LE16 unsigned | Linear SNR; display rounds `10 × log10(raw / 36)` to integer dB |
| `0x08` | LE32 signed | Distance in metres; negative values become zero |
| `0x0c` | LE32 unsigned | Reported PHY throughput in kbit/s; distinct from encoded video rate |
| `0x25` | Byte, if present | Working-channel index inferred from vendor survey usage |

The working-channel interpretation remains an inference, with a diagnostic
that prints candidate byte offsets. Distance does not come from the separate
GetDistanceResult command; that record's changing value was identified as a
millisecond counter. The recorded 20,926-kbit/s PHY value remained distinct from
changing encoder/netdev rates.

Every reply with at least eight bytes updates raw SNR, optional channel and a
software sequence counter. Raw zero preserves the last-good display SNR,
distance and throughput. A positive short reply can update SNR while retaining
older distance/throughput, so these getters do not constitute one fresh atomic
measurement. Air-loss presentation gating is separate from these cached values.

## Selection and survey sequence

At OPEN the receiver waits up to 300 ms for a table, then selects the saved
valid index or the lowest valid bit. With no reply it sends no selection.
HUD requests enter single pending slots: newer channel/MCS requests replace
older unserviced requests, and multiple survey requests coalesce.

Selection checks the current bitmap when queued, then the baseband owner
consumes the request without checking the bitmap again. It updates the local
current-channel cache after attempting the send, regardless of the send result.
That cache and the survey's eventual restore target describe commanded state,
not a tune acknowledgement.

An active survey performs these steps:

1. Refresh the table, waiting up to 300 ms. Visit valid entries starting at the
   cached current channel and wrapping through the table.
2. If hello is incomplete or air is marked lost, publish NOLOCK for the valid
   entries without retuning. Otherwise request manual MCS 0.
3. Select each channel. Poll Get1V1Info until its channel byte matches, with a
   nominal 500 ms gate budget, 40 ms per reply and 10 ms between gate polls.
4. Wait 50 ms, obtain a new raw SNR sample, and retry once after another 50 ms
   if raw SNR is below 160. Keep the larger sample.
5. Attempt to restore the cached channel, if known, and request automatic MCS.
   Publish one `measured=1` completion; later table publications use zero.

The initial channel gate is **not repeated** for the post-dwell sample or low
sample retry. A reply from a different channel can therefore supply the accepted
SNR. The software sequence counts any incoming Get1V1Info reply, rather than
matching a reply to the particular request's wire sequence. Separate atomic
raw/channel loads also do not guarantee a single coherent sample if another
reply arrives between them.

These are nested polling budgets, not a hard whole-survey deadline. A single
poll can extend past the gate boundary; settling and retry add time. Table
refreshes from the reader use a mutex, while the sweep accesses the shared
table outside that mutex, so concurrent replacement is not excluded by one
locked snapshot for the whole survey.

Completion does not prove that all channels were measured or that restoration
succeeded. No-air results also emit `measured=1`; missing samples remain distinct
from a zero/no-lock result. Send failures are ignored. With no known restore
channel, the receiver remains on its last commanded survey channel, while the
completion's active-index field is `0xff`.

A survey restores **automatic MCS**, not the user's previous manual pin. The
service function handles a pending MCS request before a pending survey; if both
arrive together, the survey can immediately replace the requested pin and end
in automatic mode. Its source comment does not change that executed ordering.

## Offline checks

A host probe included the unmodified channel source and substituted baseband
send/reply callbacks and time. AddressSanitizer and UndefinedBehaviorSanitizer
checks passed for these cases:

- Raw-zero replies preserve last-good display values while advancing survey data.
- An eight-byte scan record can seed count 19 and a bitmap without 19 frequencies.
- A gate reply for channel 3 followed by a channel-4 sample returns channel 4's
  raw value, including the low-sample retry path.
- A selection queued under one bitmap is issued after that bitmap changes;
  a failed send still changes the local current-channel cache.

These reproduce implementation limits, not new RF measurements, timing bounds
or a successful physical restoration test.

# VR04 stock UI, channel numbering and air-directed controls

[Protocols](README.md) · [Media sessions](vr04-video-session.md)

## Three separate interfaces

The reference research contains a stock UI-to-daemon MID API, local baseband
commands, and UDP commands sent to the air unit. Similar names do not make
their indices, payloads or side effects interchangeable. This article preserves
the historical stock MID findings and checks later packet/controller definitions
at MissingLynk userspace revision `674ec00ecde0bba2eb844def1b4a0b7f8e685e13`.

`liblowdelay_mid.so` exported 64 `AR_LOWDELAY_MID_*` functions; the examined
`test_uidesign` imported 54. Static dispatch entries are 16-byte handler/name
pairs. Existence of a wrapper does not prove the handler implements its name:
`SetChnMode` was a stub, and `GETHnbqZoomInfo` was actually a setter.

## Stock channel labels versus raw baseband indices

The old MID scan record has a `number` from 1 through 15, with 0 representing
the sixteenth displayed channel. In that interface, `ChnIdx = number + 1`.

| MID selection argument | Frequency | Returned ChnIdx | Unified display |
| --- | --- | --- | --- |
| 1 | 5658 MHz, R1 | 2 | CH1 |
| 5 | 5806 MHz, R5 | 6 | CH5 |
| 8 | 5917 MHz, R8 | 9 | CH8 |
| 15 | 5380 MHz | 16 | CH15 |
| 0 | 5340 MHz | 1 | CH16 |

Stock OSD showed the raw returned ChnIdx, while the picker showed tile position
plus one: R8 could appear as CH9 in one place and CH8 in another.
Raceband names identify frequencies, not the whole proprietary channel table.

The later direct baseband interface instead uses **table indices 0–18**.
Race bitmap `0x0007fff8` admits indices 3–18; normal bitmap `0x00000007` admits
0–2. These indices are not the old MID `number` or its ChnIdx. Applying a 0–15
bound to the direct interface had excluded valid low-band entries.

Its raw scan reply is a structure of arrays: count at byte 0, LE32 frequencies
in kHz from byte 4, and a trailing LE32 valid bitmap. The selected parser caps
count at 19 and rejects implausible frequencies outside 5,000–6,100 MHz. Reported
ambient-energy fields did not track the locked peer, so per-channel SNR is
measured separately. The raw layout was captured in race mode; the source still
marks normal-mode layout unverified. Earlier fallback normal frequencies reused
R1–R3 as placeholders, not measurements of the three normal channels.

Scanning also differs by layer. The stock MID scan sweeps/retunes and restores
the channel present at entry. Continuous scans, a concurrent selection, or a
scan immediately after an asynchronous selection can therefore undo the tune.
The historical menu queues selection on its scan thread and refreshes scan data
only on entry/manual request. In contrast, the later **raw** GetScanResult reads
the table without sweeping; a separate open sweep owns retuning. Direct selections
are queued to the sole baseband transmit thread and checked against the known
bitmap. A missing bitmap does not establish that an index belongs to the band.

## Band selection is not the complete work-mode policy

The stock `fastmode` marker selects the 16-channel or three-channel band.
Its presence produced internal `FastModeEn=3`, while the getter returned 1 in
one live reading. These are already different representations. Out-of-range
SetFastMode bytes crashed the daemon and triggered the watchdog; the records
do not qualify an arbitrary byte enumeration or universal safe setter sequence.

Later RE separated band from Race/Freestyle/Long-Range work mode. The stock UI
could set both, but choosing a work mode did not itself toggle the marker.
The air applies power/MCS/bitrate-related configuration; the precise preset
table was not recovered. A 5-MHz profile existed, but the studied boot path
selected its 10-MHz profile and no runtime bandwidth setter was observed.
Neither presence of a profile nor a menu label proves the active bandwidth.

The notes disagree about which getter exposes work mode: one calls
GetFlightDynamicWorkMode (`0x43`) the mode, another decodes a time-gated boolean
and attributes the display to GetFastMode. The multi-valued context field and
these getter outputs should remain distinct until their producers are reconciled.
The later open normal/standby wire values 0/1 are not a recovered
Race/Freestyle/Long-Range preset table.

## Stock SYS_GetInfo and other getters

The System-OSD callback reads an aggregate context that is not returned by one
getter. Many fields are available in SYS_GetInfo, command 31, as decoded from
the stock UI consumer:

| Output offset | Type | Meaning within the studied stock ABI |
| --- | --- | --- |
| `0x00` / `0x02` | u16 | Air / receiver temperature |
| `0x04` | float32 | Air battery voltage, volts |
| `0x08` | float32 | RX-link rail voltage; not the goggle pack measurement |
| `0x14` | u32 | Displayed bitrate in kbps |
| `0x18/0x1c/0x20/0x24` | u32 | Target / real / peak / PHY throughput fields |
| `0x2e` | u16 | Reported delay; not independently measured optical latency |
| `0x3a` | u16 | Reported distance in metres |
| `0x3c` | u16 | Signal strength |
| `0x3e` / `0x3f` | u8 / s8 | MID ChnIdx / MCS |
| `0x48` / `0x50` | u64 / u16 | Card capacity / availability |
| `0x54` | u16 | Standby status |
| `0x58` / `0x5c` | float32 | Minimum / maximum SNR |

Goggle pack voltage comes from its local ADC; flight time is an application
timer. Distance and signal fields are reported telemetry, not independent range
measurements. GET_Tranmit_Param, command 35, is **power/bandwidth**, despite a
contradictory older “active channel” paragraph. Its wrapper maps power indices
`5→3`, `14→25`, `20→100`, `23→200` mW; a live result `0xc8,0x14` meant
200 mW and a reported 20-MHz bandwidth field, not channel 200. That report does
not by itself reconcile the separate boot-profile bandwidth observation.

GetHnbqDeviceInfo, command 59, has a 208-byte identity reply with an ID triplet
and strings; an unidentified float is not established as battery or distance.
GetDebugInfo was content-checked for link status, returning `−1` without a peer.
SYS_GetInfo still returned populated fields with no link, so nonzero data does
not prove freshness or association.

The remaining recovered MID surface includes these static handler contracts:

| Commands | Recovered behavior or limitation |
| --- | --- |
| 55/56, aperture set/get | Persist one byte in session configuration |
| 53/54, zoom set/get | Mode plus float zoom factor |
| 46/47, ROI set/get | ROI flag and apply path |
| 51/52, display scale set/get | Scale mode |
| 65/66, UI-button enable/query | Selector plus enable, not one global boolean |
| 38/39, debug mode get/set | Getter caused the failure described below |
| 63, Mavlink log toggle; 30, buzzer status | Share session bytes; buzzer order is enable/volume/alert |
| 68, user-log upload | Request/action path, not an inert getter |
| 28, LED status | One-byte state |
| 33/34, upgrade / status | Action/request fields and stage/progress response |
| 50, remove recorded media | Deletes a media-database entry |
| 62, set device info | Near-no-op acknowledger in the examined handler |
| 64, GETHnbqZoomInfo | Misnamed setter with selector 1–6 and enable |

For recording, REC_SetAttr command 14 uses a full 40-byte read-modify-write
structure: loop at `0x00`, segment seconds at `0x08` (`0x7fffffff` unlimited),
resolution at `0x18`, and FPS at `0x1c`. The shorter REC_SetTimeLimit wrapper
did not change readable attributes in the recorded test.

Getter testing was one call per process with server/latency checks. Many ordinary
queries completed promptly, but GetDebugMode blocked about 33 seconds and killed
the server. GetOsdContext blocked the client waiting for queued OSD while the
server survived. Concurrent MID clients could cross replies and corrupt scans.
Those observations define the studied interface's failure behavior; a function
named “Get” is not evidence that it is harmless or passive.

## Air-directed control bodies

These are UDP 10000 payloads, separate from the MID command numbers above.
The selected builders use a body at datagram byte 20; type-specific padding
means there is no universal total-length formula based on a 24-byte header.

| Type | Total / declared body | Decoded contract |
| --- | --- | --- |
| `0x0d`, SetTranParm | 34 / 10 bytes | Body byte 0 power dBm, byte 1 captured constant `4`, byte 8 standby; other bytes retained/zero |
| `0x0a`, SetLdCfg | 216 / 192 bytes | Body plus four trailer bytes; association-time state |
| `0x0c`, SetCameraInfo | 60 / 40 bytes | LE32 selector followed by a cached union; applies only the selected field |
| `0x15`, SetScaleMode | 32 / 12 bytes | LE32 aspect, LE32 zoom enable, float32 zoom ratio |
| `0x12`, work-mode report | 36 / 4 bytes | Air-to-ground LE32 normal/standby value plus twelve-byte trailer |
| `0x1b`, standby ACK | 24 / 0 bytes | Standby acknowledgement; zero declared body |

Selected SetLdCfg **body-relative** offsets are ROI `0x43`, recording width/height
`0x46/0x48`, bitrate ceiling LE16 `0x65` in 250-kbps units, power byte `0x68`,
standby byte `0x70`, and a 64-byte recording-directory string at `0x75`.
Most surrounding state remains undecoded or inferred; capability bytes `0x01`
and `0x73` are not ROI enables. A known-good captured body is retained rather
than inventing those fields.

SetCameraInfo's selector belongs at byte **20**, correcting the older byte-24
claim. Putting it at 24 made every request appear as selector 0, brightness,
which the vendor stored without an ISP apply. The recovered selector map is:

| Selector | Body-relative data | Vendor handling |
| --- | --- | --- |
| 0 | u16 brightness at `0x04` | Stored only in the examined path |
| 1 | u16 manual/time at `0x06/0x08` | Auto/manual exposure, time in µs |
| 2 / 3 | u16 at `0x14/0x16` | Saturation / sharpness |
| 4 | u16 manual/Kelvin at `0x18/0x1a` | White balance |
| 5 / 6 | u16 at `0x1c/0x1e` | Boolean 180° rotation / aspect field |
| 7 / 8 | u16 at `0x20/0x22` | Boolean 3D/2D noise reduction; vendor strength 50 |
| 9 | u16 ISO at `0x0e` | Applied only with manual exposure already active |
| 10 | u16 at `0x24` | Anti-flicker 0/50/60 Hz, **not contrast** |

SetScaleMode uses aspect 1 for 4:3 and 0 for 16:9. Vendor ratios 0 or 1 disable
zoom; stock UI used 1.0 and 0.7. Applying scale while streaming rebuilt the
vendor air pipeline. These are recovered vendor wire semantics; not every selector was physically
validated.

# Ascent file uploads: acknowledgement, persistence and session state

[Experiments](README.md) · [USB protocol](../protocols/ascent-usb-management.md) ·
[Root access](../firmware/ascent-root-access.md)

The October 5, 2026 experiments established file-start/data access in normal
17.5.3 firmware without SSH or a clean-mode reboot. The first experiment proved
an acknowledged partial transfer; a later persistent-path experiment and subsequent
SSH readback established a stronger result. These stages must not be collapsed
into a claim that any upload ACK proves durable, executable contents.

## Bounded temporary-file probe

The tested receiver was `Ascent_VRX`, board `FPV-Ascent-Gnd-485-V1.2-1.0`,
running `Ascent_G_Gnd_17_5_3`. At 15:07:35 UTC it accepted metadata for a unique
text filename under `/tmp`, declaring 70 bytes, then acknowledged only 40 bytes
sent in one chunk. Thirty declared bytes were deliberately withheld.

```text
0x3c information → 0x73 metadata → 0x74 partial data → 0x3c information
```

The metadata reply was 68 bytes, status 0/OK. The 80-byte data reply reported
length 40, current size 40, total size 70 and status 0/OK. CRCs, commands and
sequences matched. The post-query retained the expected identity/firmware.
No `0x72` preparation or `0x75` FileEnd was sent.

This proved acceptance by the running service. There was no filesystem readback,
root execution or persistent-path write at that stage. A separately performed
power cycle reset the incomplete receiving session and restored successful
information queries. It did not independently prove deletion of the temporary
file. Historical supply readings are not a statement of present bench state.

## Persistent-path probe and later readback

At 16:52:01 UTC, a fresh receiving session accepted a unique text filename under
`/usrdata`. All **40/40 bytes** were acknowledged with exact length/current/total
counters and valid CRC, command and sequence fields. The same four-command
sequence omitted preparation, FileEnd and reboot commands.

The uploaded source had SHA256
`1ac2efaa9477d0d4647a2b873358e8691e63e5a50bae9aea10220812738f9c6d`.
Initially the audit correctly recorded `file_contents_read_back=false`,
`transfer_finalized=false` and `receiver_md5_accepted=false` despite all bytes
being acknowledged. Waiting at least 35 seconds before a later power cycle
was not a verified device-side sync or durability test.

Subsequent startup-wrapper deployment obtained root SSH, read back this exact
40-byte probe after multiple restarts and matched the source SHA256. That later
evidence established content and persistence. It did not retroactively establish
that FileEnd had run or validated a whole-file MD5 in the earlier transfer.

## Transfer state and validation limits

The reference receiver creates or truncates the destination at metadata receipt.
It flushes data chunks but leaves the file/session open; an interrupted transfer
can leave an empty or partial file. FileEnd separately closes/checks the file and
can start upgrading. See the [USB protocol](../protocols/ascent-usb-management.md)
for metadata fields and the [version-specific updater trace](../firmware/ascent-stock-updater.md)
for finalization behavior.

The reference metadata handler retains received-byte and last-sequence state.
Explicit `0x72` preparation resets the receiving session and can affect normal
operation. The probes above omitted it; later startup deployment verified
preparation and post-reboot script hashes.

The initial Python client passed 41 offline tests for fragmented replies, binary
and empty uploads, limits, stale counters, source mutation and explicit
preparation/finalization. The separate Web Serial client was checked against a
simulated receiver only. Those checks establish client behavior; the live probes
and later filesystem readback establish device behavior.

These exact ACK checks are scoped to the standard receiver. The later
[Sky482 incident](ascent-lite-flash-repair.md) observed zero-length data ACK fields,
cross-file counters and ultimately truncated scripts despite accepted uploads.
Shared command IDs do not establish identical file-transfer semantics or a
safe cross-model copy implementation.

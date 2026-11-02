# Gnd485 UART0: baud limitation and scope measurements

[Experiments](README.md) · [Ascent VRX](../devices/ascent-vrx.md)

The October 4–5, 2026 investigation supports approximately 1.15 Mbaud and
likely 1.8 V signaling on the tested receiver's UART TX. It did **not** produce
readable console text or independently establish the complete UART format.
The reference firmware configures 1,152,000 baud, 8N1, without flow control.

## Serial capture and its correction

The confirmed adapter was an original CP2102, USB `10c4:ea60`. Two attached
adapters shared serial `0001`, making their serial-based device alias ambiguous;
the investigation used the user-confirmed physical USB topology.

Requesting 1,152,000 baud returned **1,000,000** for both directions through
Linux `TCGETS2`, approximately 13.2% below the requested rate. A six-second
capture received 519,738 garbled bytes; a one-second confirmation received
86,604. The receive-only scan sent no bytes, break or bootloader command.

| Requested baud | Kernel baud | Bytes in two-second sample | Printable fraction |
| ---: | ---: | ---: | ---: |
| 1,152,000 | 1,000,000 | 173,844 | 38.6% |
| 115,200 | 115,200 | 17,562 | 33.8% |
| 921,600 | 921,600 | 173,795 | 38.7% |
| 1,000,000 | 1,000,000 | 173,847 | 38.4% |
| 460,800 | 460,800 | 63,184 | 55.0% |
| 230,400 | 230,400 | 20,392 | 50.0% |
| 57,600 | 57,600 | 9,026 | 30.6% |
| 38,400 | 38,400 | 5,335 | 25.1% |
| 19,200 | 19,200 | 2,902 | 30.9% |
| 9,600 | 9,600 | 1,492 | 32.0% |

No sample contained coherent console markers. The highest printable fraction
does not identify the baud.

The operator subsequently observed a boot burst followed by TX staying high,
while the adapter continued producing random characters. Consequently those
post-boot bytes are not established receiver traffic. Baud mismatch can corrupt
real transitions; it does not explain bytes from a correctly recognized steady
idle line. Input thresholds, grounding, noise or a floating connection remain
possible explanations. The adapter's actual input threshold was not measured.

## Scope evidence

CH1 was confirmed on receiver TX. The stopped Siglent SDS1202X-E acquisition
used 10 µs/div, 1 V/div, a configured 1X probe factor, 1 GSa/s and 140,000 points.

| Measurement | Result |
| --- | ---: |
| High / low | 1.76 V / 0.00 V |
| Maximum / minimum, including overshoot | 1.92 V / −0.24 V |
| Narrow low pulse | 0.857 µs |
| Positive pulse | 1.82 µs |
| Rise / fall | 0.236 µs / 0.203 µs |

If the narrow low pulse represents one bit, its inverse is approximately
1.167 Mbaud. Nominal 1,152,000 baud has a 0.868 µs bit period, consistent with
this observation. A single pulse is insufficient to establish exact baud,
data bits, parity or stop bits. The scope's ordinary 373 kHz frequency reading
measures the captured pattern, not the UART baud.

The high level indicates approximately 1.8 V logic **if the physical probe
attenuation matches the scope's 1X setting**. This UART measurement does not
determine the NAND supply voltage or any other connector's voltage.

The receiver supply was set to 12 V with a 1.5 A limit; the recorded powered
reading was 11.99 V, 0.236 A and 2.83 W. A later reading found it off following
an external state change. These are historical bench observations.

## Remaining validation

A boot-window capture with verified voltage compatibility and actual support
for 1,152,000 baud is still needed. Separate boot transitions from subsequent
idle, measure several bit intervals, and require coherent decoded frames.
The reference 8N1 setting remains a firmware-derived expectation. No interactive
console or bootloader access was established by this experiment.

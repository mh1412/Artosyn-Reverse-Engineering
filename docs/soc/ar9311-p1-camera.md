# P1 SC231AI camera and stock ISP findings

[SoC studies](README.md) · [P1 Air](../devices/betafpv-p1-air.md)

The investigated standard P1 Air runs P1_SKY 1.0.44.rel and uses an SC231AI
sensor, chip ID `cb 6a` at I2C address `0x30`. Its stock camera path runs through
`ar_lowdelay -m 2 -t 1`, vendor MPP libraries, MMZ and kernel modules. No stock
`/dev/video*` devices were found.

## Sensor and ISP observations

The studied 1920×1080/60 mode uses RAW10, two CSI lanes, a 24 MHz sensor clock
and a 378 MHz link frequency. CSI IPI timing was `16/16/33` for HSA/HBP/HSD.
The RAW10 geometry and ISP mode `0x58870780` were matched against stock setup.

The vendor shutdown path leaves the VIF AXI enable/run bits at offset `0x190`
set. Register-isolation tests found that clearing those bits could leave frame
interrupts running while the captured image was black. Interrupt activity alone
therefore does not establish valid camera pixels.

The extracted P1 tuning includes a 478-entry exposure table and 348 sensor-gain
steps. Sensor bus Bayer order is BGGR, while the investigated ISP statistics
order is GRBG: green in channels 0 and 3, red in 1 and blue in 2. Statistics
were interpreted as 16 horizontal zones by 36 vertical blocks, with 826 samples
per channel per tile. Scene correlation supported the channel mapping; small
geometry offsets remained provisional. These findings do not establish the
complete vendor AE/AWB algorithm or calibrated image quality.

## Firmware identity

| Input | Bytes | SHA-256 |
| --- | ---: | --- |
| P1 `chagall.bin` | 999,616 | `308b96351168cc7dbbe936de1490f9ab553aa9ae7e1679d8e9a8c4ab8cd5bf86` |
| P1 `chagall_lowmem.bin` | 999,360 | `c70476d90549440408c846b70447035978771ef4e24a1f0003677e2d939ca49b` |
| P1 1.0.44 SC231 tuning | — | `8301d0d41cd5c413730e94315a3edeb2036789828937e061ac62748f0a31de23` |
| Ascent reference `chagall.bin` | 998,144 | `99bc74cdf1e7ded785c2b65440c983f4e128727eb2bbdf309b14ddde668fd8ac` |

The Ascent codec blob differs from P1's. Matching the Wave521C product ID does
not establish binary interchangeability. The two P1 codec blobs match between
installed 1.0.44 and the inspected 2.1.2 package, while the sensor library, ISP
tuning, media library, kernel modules and radio configuration differ.

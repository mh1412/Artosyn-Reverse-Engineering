# AR9311 ISP coefficient formats and fixed curves

[SoC studies](README.md) · [NT99235 ISP reconstruction](ar9311-nt99235-isp.md)

These formats describe the pinned NT99235 reference implementation and vendor
library. All multibyte fields below are little-endian. Buffer allocation size,
generated payload size and hardware fetch length are separate quantities.

| Payload | Driver allocation | Content built by the codec |
| --- | --- | --- |
| Gamma | `0x4000` | Two `0x800` pages: selected curve and fixed companion |
| DRC | `0x2000` | Four `0x800` banks: two selected and two fixed |
| Lens shading | `0x680` | `0x340` grid region; remaining bytes are outside this builder |
| Compander | Within a shared `0xfa00` HDR/compander allocation | `0x7800` template at allocation offset `0xa00` |

The corrected gamma length interpretation is `0x80 × 16 = 0x800` bytes.
The auditor likewise describes DRC length `0x100` as `0x1000` bytes while its
builder and retained comparison cover `0x2000`. These observations do not prove
that the hardware fetches every byte of the allocation. Older comments also
call the `0x680` shading allocation a fetch; the recovered 52-record shading
payload is `0x340`. Descriptor interpretation remains distinct from successful
whole-buffer reconstruction.

## Gamma records and endpoint handling

One `0x800` page contains 128 records of 16 bytes. Each record holds four
12-bit samples, duplicate fields and a forward reference. For record samples
`a,b,c,d` and the next record's first sample `n`, the four 32-bit words are:

```text
w0 = a | (b << 12) | ((b & 0xff) << 24)
w1 = (b >> 8) | (c << 4) | (c << 16) | ((d & 0xf) << 28)
w2 = (d >> 4) | (d << 8) | (n << 20)
w3 = 0
```

The packer masks samples to 12 bits. The final record receives an explicit
forward endpoint because there is no next record inside the page. A zero
endpoint was associated with highlight wrapping in the retained hardware work;
padding this field with zero is not equivalent to omitting unused storage.

For the dynamic page, each tuning curve contains 4,096 `u32` entries. The open
builder selects entries `0,8,…,4088`, blends corresponding low/high entries
using truncated Q12 arithmetic, masks to 12 bits and raises descending values
to the previous sample. Endpoint 4095 is blended separately and clamped to at
least the last stored sample. The codec trusts its caller's curve indices and
blend range; it is not a tuning-file validator.

The fixed companion page has 512 samples of the following curve:

```text
f(x) = 4.5*x                         if x < 0.018
       1.099*x^(1/2.2) - 0.099       otherwise
sample[i] = floor(4095*f(i/512))      for i = 0..511
forward endpoint = 4095
```

This reproduces the pinned library samples exactly. Its exponent is `1/2.2`,
so recognizing the other constants as Rec.709-shaped does not justify replacing
the curve with an ordinary `0.45`-exponent implementation. Its sample axis is
`i/512`, also distinct from `8*i/4095` for the dynamic source.

## DRC banks and compact models

A bank contains 128 records, each holding three unsigned 20-bit samples.
Record `i` represents samples `2i`, `2i+1` and `2i+2`; adjacent records overlap,
so one bank represents 257 samples. With words `w0,w1` at offsets 0/4 and
halfword `h` at offset 8:

```text
a = w0 & 0xfffff
b = (w0 >> 20) | ((w1 & 0xff) << 12)
c = (w1 >> 28) | (h << 4)
duplicate of b = (w1 >> 8) & 0xfffff
```

The open packer clears bytes 10–15. The library's static banks match that
choice. The first two banks come from the selected tuning profiles, with the
second curve at profile offset `0x404`; this codec implements neutral DRC
strength. Recovery of a non-neutral vendor strength transform does not imply
that the runtime builder applies it.

The final two banks have exact compact models for the pinned binary. For
`u=sqrt(i/256)` and `A=0x18000`, their ordinary samples are:

```text
bank 2: floor(A*exp(-0.3624559*u))
bank 3: floor(A*(exp(-5.8*u) - exp(-5.8)))
```

Both sample-zero values are carried as `0xff000`; bank 3 sample 1 is carried
as 68,398. The first decay rate was fitted to the table, and the exceptional
sample's producer remains unresolved. Exact reproduction therefore preserves
vendor data compactly; it does not establish a fully recovered producer law.

## Lens shading and compander

The shading builder takes two 100-point float32 grids: tuning offsets `0x929c`
for the duplicated field and `0x910c` for the third field. It truncates gains
to Q11. Two points occupy each 16-byte record as two `(x,x,y)` triplets,
followed by four zero bytes. Fifty populated records and two zero records
make `0x340` bytes. The integer float conversion covers the retained positive
gain range; its defensive saturation is not general validation of arbitrary
IEEE-754 data. Illuminant selection and the allocation tail are not computed
by this grid builder.

The compander page combines two retained spans with structural fills:

| Page interval | Content |
| --- | --- |
| `0x0000–0x08ff` | Carried head, including 20-bit head ramp and 16-bit overlapping-pair curves |
| `0x0900–0x0fff` | Zero fill |
| `0x1000–0x17ff` | Carried 257-sample middle table in DRC-style packing |
| `0x1800–0x77ff` | 1,536 repeated records with words `0x100, 0x10000, 0, 0` |

The main curve uses two square-root branches meeting at sample 64, plus 15
one-count corrections. A smaller ramp ends at the carried value `0xfb00`.
The middle table resembles a square law with a fitted correction divisor and
exceptional samples 249–251; it steps backward at sample 252. Those exceptions
are retained evidence, not values to smooth away while claiming exact parity.
The studied path's disabled compander does not qualify its image effect when
enabled.

## Offline verification and its boundary

The closed-form checker matched all fixed-curve samples against the
[pinned library](ar9311-nt99235-isp.md#cvisp-table-provenance-and-verification-limits):
512 gamma samples plus endpoint, two sets of 257 DRC samples, and compander
regions of 33, 257, 33 and 257 samples.

A separate host build used the actual C codec and checked-in constant headers,
with AddressSanitizer and UndefinedBehaviorSanitizer enabled. Its output was
compared byte-for-byte with data mapped through the vendor ELF's load segments:

| Packed output | Bytes compared | Differing bytes |
| --- | ---: | ---: |
| Gamma companion page | 2,048 | 0 |
| Both static DRC banks | 4,096 | 0 |
| Complete compander page | 30,720 | 0 |

This second check covers redundant fields, zeros and structural fills as well
as decoded samples. It establishes static payload parity for one binary.
It does not validate dynamic scene selection, hardware fetch length, memory
lifetime, image quality, or the older gamma capture's small sampling residual.
The diagnostic tone dumper emits `0x1000` gamma bytes and `0x2000` DRC bytes;
its gamma file is the two generated pages, not the full `0x4000` allocation.

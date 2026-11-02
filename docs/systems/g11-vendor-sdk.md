# G11 vendor SDK: services and implementation limits

[Systems index](README.md) · [Stock platform](../devices/g11-stock-platform.md)

Scope: the archived vendor SDK at commit
`b3917ed3c3a0eef0afc6548ba6a00cffc61d2293`, dated August 26, 2026.
Its manual identifies itself as **1.1.5**, while the build configuration still
sets package version **1.1.0** and component macros **1.0.0**. These labels
must not be substituted for a running device's version or binary identity.

This study reconciles the complete manual with headers, selected application
code, video-player source, update scripts and a bounded HAL disassembly.
It describes the SDK's contracts and limitations. It does not establish that
every API was exercised on the handset, or that this checkout exactly matches
the installed stock software.

## Service ownership

The HAL is a collection of adapters over different subsystems, rather than
one firmware interface:

| Interface | Owner / boundary | Meaning of an API result |
| --- | --- | --- |
| RC tuning and MCU status | Internal MCU UART and process-local receive/cache state | A reply or cached state; freshness and buffer ABI need separate checks |
| Wireless radio | Artosyn RPC client and radio daemon | Host-side query/command result, not emitted RF power or peer acceptance |
| Wi-Fi | Local supplicant, network management and callbacks | Separate from the Artosyn radio link |
| GPIO and PWM | Linux GPIO/PWM services | SoC-side controls, distinct from GD32 pins and MCU firmware |
| Video player | Unix socket to an independent decode/display process | Command/state result, not proof of decoded or displayed video |
| Gimbal/camera | UDP control of a compatible vendor device | Separate peer protocol and cached replies |
| Common firmware APIs | Version/metadata retrieval and file downloading | A download is separate from installation and recovery |

The stock application uses LVGL 8.2 at 1920×1080. Its UI, settings and factory
pages sit above these owners. VideoPlayer retains SigmaStar decoder/display
dependencies; rebuilding the UI does not replace those drivers. The combined
hardware initializer's MCU → radio → PWM → Wi-Fi ordering and possible radio
startup delay are covered in the [platform study](../devices/g11-stock-platform.md#application-initialization-responsibilities).

## Wi-Fi limits belong to different layers

The manual's “three saved networks” and HAL's twenty-entry constant describe
different limits. The stock Wi-Fi page allocates three saved rows, admits new
rows only below that count, and requests at most three saved records from the
HAL. Its scan-page capacity is 32, while the HAL declares 50 scan results.
Neither UI count establishes the supplicant's total storage capacity.

The HAL uses one interface and one global callback registration, with optional
per-request scan callbacks. Concurrent scan requests can share a scan and its
cached results. Configuration is intended to be set after initialization and
before enable; changing it while enabled requires disable/re-enable to apply.
The documented default interface/country is `wlan0`/`CN`, not a measurement of
the handset's active regulatory settings.

The header allocates 129-byte SSID strings, 18-byte BSSID strings and IPv4
address fields. Buffer capacity is not a demonstrated accepted network-name
length. Scan maximum-rate is explicitly marked **not parsed**, despite the
manual listing its Mbps unit. Connection TX/RX rates use Kbit/s. The security
enum includes enterprise and WPA3 modes, but its presence does not establish
their tested configuration or authentication support.

Connected RSSI is documented as a two-second cache, with `−999` used for
unavailable state. Network removal and auto-connect settings act through the
supplicant. The later custom UI's
[Wi-Fi persistence and recovery](g11-linux-ui.md#wi-fi-management-and-persistent-recovery)
has separate implementation and physical evidence; it does not validate every
stock HAL feature.

## RC structures, cache and display controls

Two abbreviated manual tables flatten arrays into a single entry. The actual
header defines sixteen five-byte configuration entries in `hal_chncfg_t`
and sixteen padded four-byte live-value entries in `hal_joystickcfg_t`:
**80 and 64 bytes**, respectively, under the studied ARM ABI. Allocating only
one table entry would not satisfy those getters.

Configuration endpoints use ten-unit scaling; live values do not. The coach
mode field contains `P`, `M` or `S`, while its separate switch field selects
7/H, 10/B1 or 11/B2. The manual's first table assigns those mode characters to
the wrong field; its later example agrees with the header and installed UI.
The [stock input study](../protocols/g11-stock-inputs.md) retains the measured
mapping, four-byte hand-mode ABI and cache-freshness limits.

Virtual-channel API `0xd1` carries one to five 16-bit values for CH12–CH16,
high byte first. This is an MCU command contract; its name “virtual SBUS” does
not establish an independently captured physical SBUS output. The example's
button latching/release logic is UI behavior, not receiver delivery evidence.

MCU status combines millivolts, battery percentage, power state and charge
state in a receive-thread cache. The SDK advises coordinating video-display
shutdown with MCU power state to avoid retained pictures; the later
[power/shutdown study](../firmware/g11-power-and-shutdown.md) describes the
independent custom handshake and its tests.

Brightness has another layered limit. Header constants and the stock slider
use **5–78%**, but the bundled HAL's `hal_pwm_set_brightness` at `0xa19c`
only clamps the supplied unsigned value to **100**, then calculates duty from
the period. It does not enforce the UI's 5–78 range. That is a binary-derived
software distinction, not qualification of any higher physical duty cycle.
PWM screen control is also distinct from the video socket's display command.

## Gimbal and camera controls

The vendor gimbal HAL advertises angle/speed control, stop, capture, recording,
zoom, PTZ modes, pseudo-color selection and video flips. The header adds PTZ,
pseudo-color and flip interfaces beyond the manual's main API list.
Recording query is asynchronous and its getter returns cached state. Zoom's
reported ratio is explicitly an estimate; its raw value distinguishes wide
and tele ranges at 70. Angle/speed functions take integer arguments even where
comments show fractional endpoints, so the comments are not a fractional ABI.

The manual requires a compatible vendor gimbal. No physical gimbal operation,
wire-format reconstruction, camera-recording result or cross-vendor
interoperability is established by this API inventory.

## Packaging, updates and recovery

The actual package target is `make update_image`; the manual's
`make update_picture/image` is a documentation error. The target writes version
metadata, strips bundled libraries/modules and archives the package tree. It
does not itself demonstrate that every bundled binary was rebuilt. The
VideoPlayer makefile also retains paths into a larger SDK tree; this review
did not qualify a complete clean build from this checkout.

The update paths are separate:

| Target | Studied path | Limit |
| --- | --- | --- |
| Linux/application package | Stage `/customer/update/update.tgz`, blank display, sync and reboot; early boot invokes updater scripts | Root-tree extraction, not a proven atomic bank switch |
| MCU | HAL native updater over the internal UART | Bootloader, metadata and recovery have their own [history](../firmware/g11-mcu-update-history.md) |
| Local radio | Artosyn updater selection `LDEV`: `-s -1 -d 1` | Success must be checked beyond the wrapper's exit status |
| Local AP radio | Selection `AP`: `-s -1 -d 0` | Script path only; not a tested G11 role change |
| Remote radio | Selection `DEV`: `-s 0 -d 0` | Requires the intended peer and actual updater completion |

Application source calls `update_401D.sh`, while the tracked package supplies
`update_RF.sh`. The names cannot be silently equated without checking the
installed environment. Local-file receiver upgrading has a `DEV` dispatch.
The internet-download handler, however, selects only SoC, MCU and local-radio
targets and returns for other pages. Its later receiver branch merely sets
success and is unreachable through that selector; it is not a completed
remote-receiver download/update implementation.

The packaged radio wrapper calls `ota_upgrade`, waits, removes the image and
ends with `exit 0`, without checking the updater's result. An isolated offline
run of the unchanged wrapper, with a mock updater returning **42**, still
returned **0** and deleted its temporary input. Thus wrapper success cannot
establish successful radio flashing. No real updater or hardware was invoked.

For SoC packages, the legacy `update.tgz` branch extracts directly over `/`
without the separate MD5 comparison used by the newer `update_TC_V*` branch.
The latter prefix also differs from the SDK's generated `update_G11_V*` name;
the UI's normalized `update.tgz` path matters. Optional logo handling writes
MTD8, but one padded-image branch checks the subsequent cleanup command rather
than preserving `flashcp`'s result. No power-loss, rollback or complete package
integrity qualification follows from these scripts.

The manual's FTP “root” means the FTP-visible root; the inspected device exposed
`/tmp/ftp`, not arbitrary filesystem access. Its recovery procedure describes
a 115200-baud boot console, SigmaStar prompt and TFTP-assisted script execution.
The quoted transfer-byte count alone proves neither a successful flash write
nor recovered boot. That vendor procedure remains distinct from demonstrated
MCU recovery and later custom software deployment.

## Evidence identity

The bundled `libyz_hal.so` is a 150,800-byte ARM ELF32 little-endian, EABI5
hard-float shared object, SHA-256
`c9919f87773f91e745b2b48481f6661cebfc0c3dc8340d36e0b3d447abfe03b9`.
Export names were checked, but exports alone do not validate backend behavior.
The complete manual was reconciled; selected large UI and HAL implementation
regions remain a bounded review rather than a claim of full SDK source audit.
The [stock video-control study](../protocols/g11-stock-video-control.md) gives
the recovered message layouts and independently checked source limitations.

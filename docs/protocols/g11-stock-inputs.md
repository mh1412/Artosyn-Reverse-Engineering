# G11 stock MCU inputs and dual wire formats

[Protocols](README.md) · [G11](../devices/skydroid-g11.md)

## Stock UART and command replies

The stock HAL opens `ttyS1` at 115200 8N1. Channel values already include stock
assignment, endpoints, reversal, calibration and enabled stock mixing; they are
not raw ADC samples.

Vendor command envelopes use ASCII `fengyingdianzi:`, command, payload length,
payload and XOR checksum. A2 requests channel configuration and B1 requests
channel values, using query payload `52 01`. B1 replies contain sixteen
big-endian channel values. A2/E2 supplies physical IDs and endpoints.

The [platform study](../devices/g11-stock-platform.md#hal-input-abi-and-cache-limits)
records the installed HAL's 80-byte configuration and 64-byte live-value
structures, four-byte hand-mode ABI, endpoint scaling and process-local cache.
A readable cached value does not establish a fresh MCU acquisition.

## Trainer traffic

The same Linux UART was observed carrying checksum-validated `SKYDROID:` B1/20
trainer packets. Their values differ from vendor command replies. Archived
stock 0.5 code converts the underlying trainer values to display B1 values as:

```text
display = (raw - 42) * 5 / 8 + 900
```

Division uses signed integer semantics. The internal observation does not prove
which external connector shares the signal. The separate
[trainer capture](../firmware/g11-stock-mcu-reconstruction.md#captured-trainer-slave-stream)
records the complete 44-byte layout and measured external 115200 8N2 timing.

## Mapping and measurement limits

The SDK names physical IDs 1–11 as X1, Y1, X2, Y2, SW1, SW2, H, AUX1, AUX2,
B1 and B2. These names do not establish logical roll/pitch/throttle/yaw order.
Stock configuration supplies the mapping and button latch/lock behavior.

The stock wire format has no acquisition sequence or timestamp sufficient to
establish input-to-receiver latency. The external stationary trainer capture
showed approximately 100 frames/s; its unchanged payload cannot establish
movement latency or endpoint coverage. Neither UART traffic nor Linux display
values establish delivery over the air.

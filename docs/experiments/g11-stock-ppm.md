# G11 stock PPM contact and electrical observation

[Experiments](README.md) · [G11](../devices/skydroid-g11.md)

The October 26 stock comparison used the original MCU 0.5 full-flash image,
SHA-256 `58ad76e89664dd9a7d687f08dafa55c1bc81af1e44ccc674650fcfb4eb41883e`.
Flash was verified before reset and option bytes were unchanged. Stock execution
used `VTOR = 0x08002000`.

A bounded stock UART transaction read coach mode `S` with switch 7, wrote mode
`P` while preserving switch 7, received `OK`, and verified readback.

PB7 was a push-pull GPIO output without a pull-up/down. With the external signal
load disconnected, the **RX/PPM** contact produced a fast 3.28 V PPM waveform.
At 5 ms/div the scope showed PPM groups. Its approximately 500 Hz automatic
frequency reading was a channel interval, not the complete PPM-frame rate.

This establishes stock PPM at that contact in the measured setup. It does not
qualify loaded behavior, serial return-path continuity, every channel's timing
or receiver delivery. Raw captures were not independently decoded for this wiki.
The [stock MCU reconstruction](../firmware/g11-stock-mcu-reconstruction.md)
separately records the timer code and serial trainer format.

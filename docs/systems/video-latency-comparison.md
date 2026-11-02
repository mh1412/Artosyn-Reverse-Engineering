# Ascent stock latency evidence and measurement boundaries

[Systems](README.md) · [Video paths](video-paths.md)

The retained stock Gnd485 17.5.3 capture contained 1920×1080 HEVC Main with
nominal 60 fps metadata. It did not provide a matched optical camera-to-display
latency benchmark. Neither a 60 Hz display nor successful host decoding supplies
that measurement.

## Separate endpoints

| Observation | Establishes |
| --- | --- |
| Radio/host bytes | Traffic over the observed boundary |
| Complete compressed access unit | Data available to a decoder |
| Correct decoder output pixels | Successful reconstruction of that test picture |
| Display event | Software/hardware presentation notification |
| Optical camera-to-panel test | End-to-end visible latency under the measured setup |

Camera exposure, encoding, radio scheduling, receive buffering, decoding and
scan-out contribute separate delays. Metadata clocks require an established
mapping before subtraction. A host-created timestamp is not a recovered sensor
exposure timestamp.

## Stock stream evidence

The [passive extraction study](ascent-groundstation.md) recovered 316 decoded
frames after capturing VPS/SPS/PPS and an IDR. Its earlier steady-state P-slice
capture lacked those entry-point records and could not decode independently.
Joining delay must therefore be separated from steady-state picture age.

The [codec study](../experiments/ar9311-slice-decoding.md) found correct supplied
regions after forced picture completion, but did not establish resumable
same-picture decoding or safe early scan-out. The
[stock display profile](ar9311-display-pipeline.md) is register evidence rather
than calibrated glass-to-glass timing.

A future latency result needs an identified camera, receiver, firmware, display
and operating mode, with repeated optical samples and explicit treatment of
startup, steady state and link-loss recovery.

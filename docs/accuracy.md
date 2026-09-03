# Accuracy

Before every release, JapanFold's output is checked for parity against each
model's official reference implementation. The method, the reference backend
used for each leg, and the per-model evidence live in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

A leg compares the device against the reference on the same input, and passes
when the device sits inside the reference's own seed-to-seed noise floor. Where
a metric sits outside that floor, the parity docs carry the measured number and
the root cause: these are precision and sampling-noise floors, not defects in
the port.

This measures the port, not the science. It compares the device against the
reference implementation on the same input, not against experimental
structures, and it inherits whatever the published checkpoint is worth.
OpenFold3's weights are a preview checkpoint trained well short of the full
AlphaFold3 schedule, so read its confidence scores before trusting a
prediction.

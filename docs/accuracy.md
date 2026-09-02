# Accuracy

Before every release, JapanFold's output is checked for parity against each
model's official reference implementation. The method, the reference backend
used for each leg, and the per-model evidence live in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

Each leg gets one of three verdicts. `PASS` means the device fold sits inside
the reference's own seed-to-seed noise floor. `PASS-caveated` and
`GAP-evidenced` mean it does not, and each one carries a root cause and a
measured number on that page; most are a bfloat16 arithmetic floor rather than a
port defect. Several served models have at least one non-clean leg. The
summary line for the current release, over every model tt-bio implements:

> Net: 37 PASS, 7 PASS-caveated, 2 GAP-evidenced

SaProt 1.3B is served and has no PASS row: its embedding correlation lands just
below the band its 650M sibling hits, and the parity docs record it as a
near-pass.

This measures the port, not the science. It compares the device against the
reference implementation on the same input, not against experimental
structures, and it inherits whatever the published checkpoint is worth.
OpenFold3's weights are a preview checkpoint trained well short of the full
AlphaFold3 schedule, so read its confidence scores before trusting a
prediction.

# Accuracy

Before every release, JapanFold's output is checked for parity against each
model's reference GPU implementation. Every model the API serves passes that
check.

The method and the per-model evidence live in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

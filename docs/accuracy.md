# Accuracy

Before every release, JapanFold's output is checked for parity against each
model's official reference implementation. The method, the reference backend
used for each leg, and the per-model evidence live in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

Every model the API serves is covered there, and where a leg is not a clean pass
it says so. As of the current release that is SaProt 1.3B, whose embedding
correlation lands just below the band its 650M sibling hits, and which is
recorded as a near-pass rather than a pass.

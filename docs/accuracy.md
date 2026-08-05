# Accuracy

You already trust Boltz-2, ESMFold-2, Protenix-v2, OpenDDE, BoltzGen,
RFdiffusion3, ESMC or SaProt. The question this page answers is whether
JapanFold reproduces what those models already give you.

Every comparison is against the model's **own official reference
implementation**, pinned to a version and commit, on the same input. Boltz-2
2.2.1, Protenix 2.0.0, BoltzGen 0.3.2 and OpenDDE `a0d5134` are installed
upstream and run on GPU or CPU as each one targets; their outputs are committed
as fixtures, so every number below is reproducible from the harness JapanFold
ships.

## How a leg is scored

A diffusion model is a deterministic function of its input noise. So each
structure and affinity leg feeds **byte-identical noise** to three runs: the
device in bf16, and the reference in fp32 and in bf16. Sampling is then removed
from the measurement and arithmetic is the only thing left that can move the
answer.

That gives a measured bar rather than a chosen one. Re-running the reference in
bf16 instead of fp32 moves its own structure by some amount; JapanFold has to
land no further from the fp32 reference than the reference's own bf16 recompute
does. Ports that have no re-runnable reference path are measured against the
official upstream implementation across five seeds per side, where the bar is
the model's own seed-to-seed variation.

Deterministic models have no sampler: ESMC and SaProt are scored on per-residue
embedding correlation against the reference encoder, and reproduce identically
run to run at PCC 1.00000. BoltzGen writes new sequences, so there is no
structure to match:
it is scored on designability, folding each design's sequence back on its own
and measuring how many reproduce the designed shape. Antibody-antigen docking
is scored by DockQ against the experimental complex.

## Scorecard

`Legs` is the number of target/setting combinations measured.

| Model | Legs | Metric | JapanFold vs reference | Verdict |
|---|---|---|---|---|
| Boltz-2 | 4, L20–585 | CA-RMSD | 0.66–1.41 Å | pass |
| Boltz-2 affinity | 3, L107–223 | Δlog₁₀(IC50) | 0.054–0.062 | pass |
| ESMFold-2 | 4, L20–129 | CA-RMSD | 0.136–0.75 Å | pass |
| Protenix-v2 | 3, L76–585 | CA-RMSD | 0.69–2.43 Å | pass |
| OpenDDE | trp-cage, L20 | CA-RMSD | 0.51 Å | pass |
| OpenDDE antibody-antigen | 1AHW | DockQ vs the experimental complex | 0.864 | pass |
| BoltzGen | 7ROA | designs ≤2 Å scRMSD | 93.75%, reference 68.75% | exceeds reference |
| RFdiffusion3 | 419-atom scaffold | featurizer bit-exactness | 43/43 keys | pass |
| ESMC 300M, 600M, 6B | 12, L20–129 | embedding PCC | 0.9987–0.9997 | pass |
| SaProt 650M | 1, L76 | embedding PCC | 0.99964 | pass |

The structure targets are trp-cage (L20), GB1 (L56), ubiquitin (L76), 7ROA
(L117), lysozyme (L129) and HSA (L585), MSA-backed and single-sequence where the
model supports both. Affinity is FKBP12, DHFR and trypsin with their ligands.
Every leg passes its gate, and on all three Protenix-v2 legs JapanFold lands
closer to the reference than two reference runs land to each other. Per-leg
numbers, metric definitions and the evidence behind each verdict are in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

## Three gates per release

Accuracy is one of three on-device gates a release has to clear, all of which
must pass before a version is tagged:

- **Parity.** The scorecard above, re-run end to end on real hardware.
- **Performance.** A per-model throughput check against committed baselines, so
  a shipped model cannot get quietly slower.
- **Behavior.** A no-regression check on the user-visible surface: progress
  reporting, output parsing and the CLI contract.

The parity gate also carries a capacity leg that folds the largest supported
input at the highest sample count and holds peak device memory to a budget.

## Reproduce it

The harness and the committed reference fixtures ship in
[tt-bio](https://github.com/moritztng/tt-bio):

```bash
scripts/fetch_parity_fixtures.sh        # restore the reference fixtures
python scripts/full_parity_gate.py      # every leg, the gate of record
```

Add `--leg <id>` for a single leg, or `--check` to validate the wiring without
touching a card.

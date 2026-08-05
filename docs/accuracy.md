# Accuracy

You already trust Boltz-2, ESMFold-2, Protenix-v2, OpenDDE, BoltzGen,
RFdiffusion3, ESMC or SaProt. The question this page answers is whether
JapanFold reproduces what those models already give you.

Nothing here is measured against experiment — whether a fold matches the crystal
structure is the model's business, not the platform's. Two questions are
answered separately:

- **Does running on Tenstorrent change the answer?** The release gate below
  settles this deterministically, before every release.
- **Does JapanFold reproduce the model's official implementation?** The
  scorecard, measured against each model's own official reference code.

## The release gate: identical noise, so only the hardware differs

A diffusion model is a deterministic function of its input noise. Before every
release, each structure and affinity leg is scored by feeding **byte-identical
noise** — the initial coordinates and every per-step epsilon — to three
closed-loop runs of the same code:

| Run | Where | Arithmetic |
|---|---|---|
| `device_bf16` | Tenstorrent (what you get from the API) | bf16 |
| `reference_fp32` | CPU, same code path | fp32, ground truth |
| `reference_bf16` | CPU, same code path | bf16 |

A leg passes when, on every metric it reports,

    d(device_bf16, reference_fp32)  ≤  d(reference_bf16, reference_fp32) × (1 + margin) + floor

in words: **running on Tenstorrent may not move the answer further than
recomputing the very same code in bf16 on a CPU moves it.** The bar on the
right is measured per leg, not assumed, and `margin` covers only TT-bf16 vs
torch-bf16 accumulation order. Metrics are the per-leg distances themselves —
CA/ligand Kabsch-RMSD, pocket-lDDT, and the absolute difference on the affinity
scalar.

All three runs are the same source, one backend/dtype toggle apart, which is the
point: with the sampler out of the comparison and the code held fixed, the only
thing the number can measure is the hardware. It is a hardware difference, not
luck of the seed.

### Why this replaced a seed-spread comparison

Earlier releases scored parity statistically: JapanFold vs the reference across
independent seeds, read against how far the reference already sits from itself.
That framing cannot answer the question it was asked. With independent draws the
two sides sample *different* structures, so the resulting distance mixes the
hardware difference with ordinary diffusion sampling noise — a correct port can
fail it on an unlucky seed, and a real backend bug can hide inside a wide
spread. Sharing the draws removes both failure modes. The seed-spread numbers
are kept in the repo as history, not as the criterion.

Getting there needed one non-obvious fix: a single `--seed` is *not* enough to
share draws, because the device and CPU trunks consume the global RNG
differently before the sampler runs. The gate re-seeds immediately before the
first sampler draw on all three runs, and the resulting noise is verified
bit-identical.

### Models without a sampler

- **ESMC, SaProt** — deterministic encoders. No noise to share; the embeddings
  are compared directly against the upstream implementation, per residue.
- **BoltzGen** — designs new sequences, so there is no paired structure to
  align. Checked by designability: re-fold each design and measure how well it
  reproduces the shape it was designed for, against the official CLI's
  distribution.
- **RFdiffusion3** — the host featurizer is checked bit-exact against a capture
  from the upstream implementation.

## Scorecard

Agreement with each model's official implementation, over all measured
target/setting combinations. `Legs` is the number of combinations.

| Model | Legs | Metric | JapanFold vs official | Verdict |
|---|---|---|---|---|
| Boltz-2 | 5, L20–585 | CA-RMSD | 0.66–4.66 Å | pass |
| Boltz-2 affinity | 6, L107–223 | Δlog₁₀(IC50) | 0.042–0.264 | pass |
| ESMFold-2 | 4, L20–129 | CA-RMSD | 0.136–0.75 Å | pass |
| Protenix-v2 | 3, L76–585 | CA-RMSD | 0.685–2.43 Å | pass |
| OpenDDE | 2, L20–117 | CA-RMSD | 0.51 / 4.67 Å | pass |
| OpenDDE antibody-antigen | 1AHW | global DockQ | 0.864 (ref 0.83–0.86) | pass |
| BoltzGen | 7ROA, n=16/side | designs ≤2 Å scRMSD | 93.75% (ref 68.75%) | pass, exceeds ref |
| RFdiffusion3 | 419-atom scaffold | featurizer bit-exact | 43/43 keys | pass |
| ESMC 300M, 600M, 6B | 12, L20–129 | embedding PCC | 0.9987–0.9997 | pass |
| SaProt 650M | 1, L76 | embedding PCC | 0.99964 | pass |

The structure targets are trp-cage (L20), GB1 (L56), ubiquitin (L76), 7ROA
(L117), lysozyme (L129) and HSA (L585), MSA-backed and single-sequence where the
model supports both. Affinity is FKBP12, DHFR and trypsin with their ligands.

Each range spans every measured setting for that model. The tight end is an
MSA-backed fold, which is what JapanFold does by default; the wide end is a
large protein folded from a single sequence, where the sampler itself is free to
wander on either implementation.

Per-leg numbers, the measured bf16 envelope each leg was scored against, metric
definitions and the full evidence behind every verdict — the metric-by-metric
breakdown and the cross-backend triangulations included — are in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

## Reproduce it

The harness and the committed reference fixtures ship in
[tt-bio](https://github.com/moritztng/tt-bio):

```bash
python scripts/full_parity_gate.py              # every leg, the release gate
python scripts/full_parity_gate.py --regen-refs # rebuild the shared-draw CPU references
```

The shared-draw references run on CPU, which is the point: fp32 CPU is the
ground truth the device is held against. A rented-GPU cross-check on Boltz-2
trp-cage confirms the reference implementation is hardware-invariant
(GPU-vs-CPU reference agreement 0.68 Å, inside the reference's own 0.81 Å
spread), so nothing about the comparison depends on the reference having run on
a CPU.

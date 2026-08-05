# Accuracy

You already trust Boltz-2, ESMFold-2, Protenix-v2, OpenDDE, BoltzGen,
RFdiffusion3, ESMC or SaProt. The question this page answers is whether
JapanFold reproduces what those models already give you.

So every comparison is against the model's **own official reference
implementation** on the same input, never against experiment. Whether a fold
matches the crystal structure is a separate question, out of scope here.

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

Because both sides consume the same draws, the sampler is out of the
comparison: what is left is arithmetic. This is what makes the number
meaningful — it is hardware difference, not luck of the seed.

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
| Boltz-2 affinity | 6, L107–223 | Δlog₁₀(IC50) | 0.042–0.264 | pass, pocket caveat |
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

Read the ranges as ranges, not as a per-leg claim: the wide end is always a
single-sequence fold of a large protein, the case an MSA-trained model is
weakest at and where both implementations wander. The MSA-backed rows — what
JapanFold uses by default — are the tight end. Per-leg numbers, the measured
bf16 envelope each leg was scored against, metric definitions and the evidence
behind every verdict are in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

## Caveats

- **Boltz-2 affinity pocket geometry.** All six affinity legs pass on the
  predicted binding constant, but pocket-lDDT sits outside the bar on five of
  them. Three-backend triangulation (the GPU-bf16 and CPU-bf16 references
  disagree on the pocket by the same margin JapanFold does) shows this is the
  bf16 arithmetic floor, not a port defect.
- **Boltz-2 single-sequence 7ROA** is the one structure leg outside its
  envelope, at the pinned seed only. Root-caused in the *reference*, not the port: the envelope
  itself — the reference's own bf16-vs-fp32 drift — swings 3.45 Å at seed 0,
  2.16 Å at seed 1 and 0.81 Å at seed 2 on this target, and the leg passes
  cleanly at the other two seeds. It is chaotic-trajectory amplification on a
  117-residue protein folded with no MSA. Fold with the MSA (the default) and
  the leg is clean.
- **Protenix-v2 confidence selection.** Its confidence head under-ranks samples
  on the reference implementation too, so the "best"-of-N structure either side
  picks is noisier than the underlying geometry. Treat its ranking with the same
  caution you would upstream.
- **SaProt 1.3B** is served, but the harness records it as a near-pass
  (embedding PCC 0.99508, just below the 0.9987–0.9996 band the smaller variants
  hit), so no pass row is claimed for it. That tracks depth: 66 layers
  accumulate twice the bf16 rounding of the 650M.
- **BoltzGen n=16 per side is small.** The reference's own two batches differ by
  12.5 points on the ≤2 Å bar, so part of the margin is sampling noise. The
  direction holds across both batch pairs.

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

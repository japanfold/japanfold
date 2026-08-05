# Accuracy

You already trust Boltz-2, ESMFold-2, Protenix-v2, OpenDDE, BoltzGen,
RFdiffusion3, ESMC or SaProt. The question this page answers is whether
JapanFold reproduces what those models already give you.

So every comparison is against the model's **own official reference
implementation** on the same input, never against experiment. Whether a fold
matches the crystal structure is a separate question, out of scope here.

## How a leg is scored

None of the diffusion models is bit-deterministic: the official implementation
gives a slightly different structure on two seeds with identical input. A bare
"device vs reference = X Å" means nothing without knowing how far the reference
already sits from itself. So each leg measures three distances:

- **R** reference vs reference, across seeds. The reference's own spread.
- **D** JapanFold vs JapanFold, across seeds. The port's own spread.
- **X** JapanFold vs reference. The parity question.

A leg passes when X is no larger than the floor `max(R, D)` within sampling
uncertainty. Deterministic legs (ESMC, SaProt) have no sampler, so R = D = 1.0
and parity is a direct embedding correlation.

## Scorecard

`Legs` is the number of target/setting combinations measured; the two value
columns span all of them.

| Model | Legs | Metric | Floor `max(R,D)` | JapanFold vs ref | Verdict |
|---|---|---|---|---|---|
| Boltz-2 | 5, L20–585 | CA-RMSD | 0.60–4.98 Å | 0.66–4.66 Å | pass |
| Boltz-2 affinity | 6, L107–223 | Δlog₁₀(IC50) | 0.025–0.196 | 0.042–0.264 | pass, pocket caveat |
| ESMFold-2 | 4, L20–129 | CA-RMSD | 0.139–0.92 Å | 0.136–0.75 Å | pass |
| Protenix-v2 | 3, L76–585 | CA-RMSD | 0.695–2.76 Å | 0.685–2.43 Å | pass |
| OpenDDE | 2, L20–117 | CA-RMSD | 0.52 / 6.04 Å | 0.51 / 4.67 Å | pass |
| OpenDDE antibody-antigen | 1AHW | global DockQ | ref 0.83–0.86 | 0.864 | pass |
| BoltzGen | 7ROA, n=16/side | designs ≤2 Å scRMSD | ref 68.75% | 93.75% | pass, exceeds ref |
| RFdiffusion3 | 419-atom scaffold | featurizer bit-exact | n/a | 43/43 keys | pass |
| ESMC 300M, 600M, 6B | 12, L20–129 | embedding PCC | 1.00000 | 0.9987–0.9997 | pass |
| SaProt 650M | 1, L76 | embedding PCC | 1.00000 | 0.99964 | pass |

The structure targets are trp-cage (L20), GB1 (L56), ubiquitin (L76), 7ROA
(L117), lysozyme (L129) and HSA (L585), MSA-backed and single-sequence where the
model supports both. Affinity is FKBP12, DHFR and trypsin with their ligands.

Read the ranges as ranges, not as a per-leg inequality. On the tightest
short-target legs X sits a little above the floor (ESMFold-2 trp-cage 0.61 Å
against a 0.51 Å floor) and passes on overlapping error bars, which is the gate's
actual criterion. Per-leg R/D/X numbers, metric definitions and the evidence
behind each verdict are in tt-bio's
[implementation-parity docs](https://github.com/moritztng/tt-bio/blob/main/docs/implementation-parity.md).

## Caveats

- **Boltz-2 affinity pocket geometry.** All six affinity legs pass on the
  predicted binding constant, but pocket-lDDT sits outside the floor on five of
  them. Three-backend triangulation (the GPU-bf16 and CPU-bf16 references
  disagree on the pocket by the same margin JapanFold does) shows this is the
  bf16 arithmetic floor, not a port defect.
- **Boltz-2 single-sequence 7ROA** passes the R/D/X floor but is flagged by
  tt-bio's tighter envelope gate at the pinned seed, root-caused as chaotic
  trajectory amplification on a 117-residue protein folded with no MSA. Fold
  with the MSA (the default) and the leg is clean.
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
python scripts/pharma_parity.py structures     # the structure legs
python scripts/full_parity_gate.py             # every leg, the release gate
```

The reference legs run on CPU. A rented-GPU cross-check on Boltz-2 trp-cage puts
GPU-vs-CPU reference agreement (0.68 Å) inside the reference's own spread
(0.81 Å), so the CPU references are representative of what a GPU evaluator
would measure.

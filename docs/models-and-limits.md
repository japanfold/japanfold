# Models & limits

`GET /v1/models` is the live, machine-readable version of this page: the model
list, every parameter with its default and range, the design protocols, and the
current limits. The tables below mirror it. If they ever disagree, trust the
endpoint.

```bash
curl -s https://api.japanfold.aiand.com/v1/models
```

`GET /v1/health` is a plain liveness check (`{"api_version":"1.0.0","service":"japanfold","status":"ok"}`).

## Prediction models

| `id` | MSA | Ligands | DNA/RNA | Affinity | Constr | PDE | Max residues |
|---|---|:-:|:-:|:-:|:-:|:-:|--:|
| `boltz2` | always | ✓ | ✓ | ✓ | ✓ | ✓ | 1024 |
| `esmfold2` | default | - | - | - | - | - | 1024 |
| `esmfold2-fast` | never | - | - | - | - | - | 1024 |
| `protenix-v2` | default | ✓ | ✓ | - | - | - | 980 |
| `openfold3` | default | - | ✓ | - | - | - | 576 |
| `openbind` | default | ✓ | ✓ | - | - | - | 576 |
| `rf3` | default | ✓ | ✓ | - | - | - | 627 |
| `opendde` | default | - | - | - | - | - | 544 |
| `opendde-abag` | default | - | - | - | - | - | 544 |

MSA `default` means on unless you send `use_msa_server: false`. `always` means
Boltz-2 cannot fold single-sequence, so a `false` is forced back to `true`.
`never` means the model has no MSA encoder and refuses the parameter. PDE is
predicted distance error: Boltz-2 is the only model whose result rows carry
`complex_pde` / `complex_ipde`. Nothing here returns a PAE matrix.

**Boltz-2** (MIT and Recursion) is the default, the most capable, and the only
model with affinity and constraints. **ESMFold-2** (Biohub) is language-model
folding, protein chains only; `esmfold2-fast` is always single-sequence, for
screening many sequences at once. **Protenix-v2** (ByteDance) is
AlphaFold3-family (Pairformer + atom diffusion) and strong at antibody-antigen.
**OpenFold3** is the OpenFold Consortium's open AlphaFold3 reproduction, folding
protein, RNA and DNA complexes; its published weights are a preview checkpoint
trained well short of the full AlphaFold3 schedule, so read the confidence scores
before trusting a prediction. **OpenBind-0** is the same OpenFold3 stack on the
consortium's OpenBind checkpoint, which co-folds small-molecule ligands the
preview weights refuse. **RoseTTAFold3** comes from the Institute for Protein
Design (the Baker and DiMaio labs at the University of Washington) and is
AlphaFold3-family too, taking proteins, RNA, DNA and small-molecule ligands. It
reads covalent modifications, bond constraints and cyclic chains only from its
own JSON spec, which this API does not expose, so an input carrying any of those
is refused rather than folded without them. The two **OpenDDE** checkpoints,
from Aureka, are protein-only: `opendde` for general complexes, `opendde-abag`
to co-fold an antibody Fab's heavy and light chains with their antigen. Both
match the reference OpenDDE implementation, including its own weakness on some
hard antibody-antigen targets, so don't expect uniform accuracy on every input.
See [Accuracy](accuracy.md).

Boltz-2 and both ESMFold-2 variants also accept modified residues.

## Embedding models

`POST /v1/embeddings` runs protein language models: ESMC from Biohub, SaProt
from Westlake University. Larger is a stronger representation at more compute
per sequence. See [Embeddings](embeddings.md).

| `id` | Name | Max residues | Notes |
|---|---|--:|---|
| `esmc-300m` | ESMC 300M | 2000 | Quickest; strong general-purpose representation. |
| `esmc-600m` | ESMC 600M | 2000 | The balanced default. |
| `esmc-6b` | ESMC 6B | 1968 | Strongest representation, highest compute cost. |
| `saprot-650m` | SaProt 650M | 2000 | Trained on sequence + structure tokens, run sequence-only here. |
| `saprot-1.3b` | SaProt 1.3B | 2000 | Largest SaProt; trained to work sequence-only. |

## Prediction parameters

Sent as `params` on `POST /v1/predictions`. Out-of-range values are clamped.
A parameter the chosen model cannot honour is refused with a `400` naming the
model and the option, never accepted and quietly dropped.

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `use_msa_server` | bool | `true` | - | Build an MSA. Boltz-2 cannot fold single-sequence, so `false` is forced back to `true` for it. `esmfold2-fast` has no MSA encoder and refuses the parameter. Every other model honours it. |
| `fast` | bool | `true` | - | Higher throughput, may be slightly less accurate. OpenFold3, OpenBind-0 and RoseTTAFold3 have no fast path and refuse `fast: true`; `fast: false` is accepted and drops out. ESMFold-2 runs fast-only here, so `false` is forced back to `true` for both `esmfold2` and `esmfold2-fast`. |
| `recycling_steps` | int | model default | 1–10 | Trunk recycles. Omit it: Boltz-2, OpenFold3 and OpenBind-0 use 3, the other six use 10. |
| `sampling_steps` | int | model default | 10–500 | Diffusion steps. Omit it: ESMFold-2 uses 100, RoseTTAFold3 50, the others 200. |
| `diffusion_samples` | int | `1` | 1–5 | Structures generated per target. |
| `output_format` | enum | `cif` | `cif`, `pdb` | Structure file format. |

OpenFold3 and OpenBind-0 fix their recycle count, and RoseTTAFold3 its sampling
steps, when the model loads rather than per request. Sending one of those keys to
one of those models is a `400`, so omit it and the model's own value is used.

## Design protocols and parameters

Three design models share `POST /v1/designs`. **BoltzGen** (MIT) protocols take
a YAML `spec` and return ranked designs: `protein-anything`, `peptide-anything`,
`nanobody-anything`, `antibody-anything`, `protein-small_molecule`,
`protein-redesign`. **RFdiffusion3** (the Institute for Protein Design)
protocols take a pasted `structure` plus a `contig` and return unranked all-atom
designs: `rfd3-binder`, `rfd3-scaffold`, `rfd3-na-binder`. **PXDesign**
(ByteDance) has one protocol, `pxdesign-binder`: a pasted `structure`, the target
`chains` to condition on and a `binder_length`, with optional `hotspots`. It
returns binder backbones, coordinates only, with no sequence and no ranking.
Size ceilings differ by model and each counts something different: BoltzGen
1024 residues of target spec, RFdiffusion3 490 for the contig's motif plus its
designed residues, PXDesign 768 for the named target chains plus the binder.
[Designs](designs.md) says what each one does.

BoltzGen:

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `num_designs` | int | `10` | 1–10 | Binders to generate before filtering. |
| `budget` | int | `10` | 1–10 | Top ranked designs to keep after filtering. |
| `fast` | bool | `true` | - | Higher throughput, may be slightly less accurate. |

RFdiffusion3:

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `num_designs` | int | `4` | 1–5 | Independent designs for the same spec. |
| `num_timesteps` | int | `100` | 4–200 | Diffusion steps per design. 200 is cleanest, fewer is faster. |
| `seed` | int | `42` | 0–2³¹−1 | Noise seed. |

PXDesign:

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `num_designs` | int | `4` | 1–8 | Independent designs for the same target, one batched trajectory. |
| `n_step` | int | `200` | 4–400 | Diffusion steps per design. 400 is cleanest, fewer is faster. |
| `seed` | int | `42` | 0–2³¹−1 | Noise seed. |

## Embedding parameters

Sent as `params` on `POST /v1/embeddings`.

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `pool` | enum | `mean` | `mean`, `max`, `cls` | How per-residue vectors combine into one pooled vector. |
| `format` | enum | `npz` | `npz`, `parquet` | `npz`: per-residue + pooled per sequence. `parquet`: pooled table only. |
| `fast` | bool | `false` | - | Higher throughput, may be slightly less precise. |

## Limits

JapanFold is a free public demo on shared compute, so inputs and concurrency are
capped. The full platform has no such limits.

| Limit | Value | |
|---|--:|---|
| `max_residues` | 1024 | per structure; per model: protenix-v2 980, rf3 627, openfold3 576, openbind 576, opendde 544, opendde-abag 544 |
| `max_chains_per_complex` | 10 | |
| `max_ligands_per_complex` | 10 | |
| `max_constraints_per_complex` | 20 | |
| `max_complexes` | 10 | structures per run |
| `max_content_chars` | 50000 | per input string |
| `max_designs` | 10 | BoltzGen designs per run |
| `max_budget` | 10 | BoltzGen designs kept |
| `max_rfd3_designs` | 5 | per RFdiffusion3 run |
| `max_rfd3_timesteps` | 200 | |
| `max_pxdesign_designs` | 8 | per PXDesign run |
| `max_pxdesign_steps` | 400 | |
| `max_binder_length` | 200 | PXDesign binder residues |
| `max_structure_chars` | 700000 | pasted target structure |
| `max_embed_sequences` | 50 | per submission |
| `max_embed_sequence_residues` | 2000 | per sequence; esmc-6b 1968 |
| `max_recycling_steps` | 10 | |
| `max_sampling_steps` | 500 | |
| `max_diffusion_samples` | 5 | |
| `max_units_per_job` | 10 | total cost of one submission, in units of a full-size run; see [The cost budget](#the-cost-budget) |
| `max_units_per_target` | 4 | cost of the most expensive single structure in it |
| `max_output_bytes_per_job` | 640000000 | 640 MB of results written by one job |
| `max_active_jobs` | 64 | service-wide |
| `max_active_jobs_per_ip` | 8 | |
| `max_active_jobs_per_session` | 3 | the concurrency cap that binds an API caller |
| `max_submits_per_min` | 12 | per caller |
| `max_submits_per_min_per_ip` | 40 | |
| `max_retained_jobs` | 1000 | results kept, across everyone |
| `max_retained_bytes` | 68719476736 | 64 GiB of results kept; past either, the busiest caller's oldest job is deleted first |
| `max_download_bytes_per_hour_per_ip` | 34359738368 | 32 GiB of result downloads per hour per source IP |
| `max_runtime_predict_s` | 1500 | |
| `max_runtime_design_s` | 2700 | |
| `max_runtime_embed_s` | 300 | |
| `max_stall_s` | 600 | predict |
| `max_stall_design_s` | 1200 | |
| `max_stall_embed_s` | 240 | |

Over a size cap you get `400`. At capacity, over a rate limit or over the
download budget you get `429` with `Retry-After`. See [Errors](errors.md).

### The cost budget

Every row above bounds one field. None of them bounds their product, and a
submission is a product. So the service also prices each submission in **units**,
where 1.0 unit is one full-size run of the model you chose at that model's own
default settings: 1024 residues on Boltz-2 or ESMFold-2, 980 on Protenix-v2, 627
on RoseTTAFold3, 576 on OpenFold3 and OpenBind-0, 544 on OpenDDE.

- Cost grows with the **square** of the residue count, because a structure
  model's dominant cost is pair work. Half a model's ceiling is a quarter of a
  unit.
- `diffusion_samples`, `recycling_steps` and `sampling_steps` each multiply
  linearly, measured against the model's own default. Boltz-2 recycles 3 times by
  default, so `recycling_steps: 9` is 3x. Turn up two knobs and the multipliers
  multiply.
- `budget` and `seed` cost nothing. They pick which designs get reported and
  which noise draw is taken, not how much runs.
- An embedding job is priced linearly in total residues, and separately in bytes.

`max_units_per_job` (10) bounds the whole submission, `max_units_per_target` (4)
the most expensive single structure in it. Both are checked at submit, so an
over-budget request is refused in milliseconds instead of holding an accelerator
for 25 minutes and then being killed by the runtime watchdog.

Every advertised maximum at default settings fits exactly: 10 structures at any
model's own ceiling is 10.00 units. What does not fit is several numbers turned
up together.

#### A request that clears every cap and is still refused

Ten structures (`max_complexes` is 10), each 1024 residues (`max_residues` is
1024), with `diffusion_samples: 3` (`max_diffusion_samples` is 5). Every field is
legal on its own:

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2",
       "targets":[ ... 10 targets, each a 1024-residue FASTA ... ],
       "params":{"diffusion_samples":3}}'
```

```json
{
  "detail": "This submission asks for about 30x the work of a full-size Boltz-2 run, and the free demo allows 10x per submission. It is 10 structures x 1024 residues x 3 predictions. Each of those numbers is inside its own limit — what this exceeds is the total, which the demo bounds so one submission cannot take the shared fleet for an hour. Come down to 3 structures.",
  "instance": "/v1/predictions",
  "status": 400,
  "title": "Submission too large",
  "type": "https://japanfold.aiand.com/errors/submission-too-large"
}
```

`detail` always names the shape that was priced and the one number to lower. Drop
to 3 structures, or keep all ten and leave `diffusion_samples` alone. See
[Errors](errors.md#submission-too-large).

The third budget, `max_output_bytes_per_job`, bounds what a job writes. Only
embeddings come near it: the largest submission the other caps allow, 50
sequences of 1968 residues on `esmc-6b`, writes 476 MB against the 640 MB cap. It
is a backstop, not something you will hit.

### Retention and downloads

Results are not durable storage. The service keeps the 1000 most recent jobs and
64 GiB of results; past either, the busiest caller's oldest job is deleted first,
so someone else's burst cannot wipe out yours. A deleted job id returns `404`.
Download what you need rather than treating a job id as a permanent link.

Reading results back is bounded in bytes rather than requests: 32 GiB per hour
per source IP, over a rolling one-hour window, charged on artifact and archive
downloads only. Polling job status does not count against it.

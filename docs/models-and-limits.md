# Models & limits

`GET /v1/models` is the live, machine-readable version of this page: the model
list, every parameter with its default and range, the design protocols, and the
current limits. The tables below mirror it. If they ever disagree, trust the
endpoint.

```bash
curl -s https://api.japanfold.aiand.com/v1/models
```

`GET /v1/health` is a plain liveness check (`{"status":"ok","service":"japanfold","api_version":"1.0.0"}`).

## Prediction models

| `id` | MSA | Ligands | DNA/RNA | Affinity | Constr | PAE | Max residues |
|---|---|:-:|:-:|:-:|:-:|:-:|--:|
| `boltz2` | default | ✓ | ✓ | ✓ | ✓ | ✓ | 1024 |
| `esmfold2` | optional | - | - | - | - | - | 1024 |
| `esmfold2-fast` | never | - | - | - | - | - | 1024 |
| `protenix-v2` | default | ✓ | ✓ | - | - | - | 980 |
| `openfold3` | default | - | ✓ | - | - | - | 576 |
| `openbind` | default | ✓ | ✓ | - | - | - | 576 |
| `opendde` | default | - | - | - | - | - | 544 |
| `opendde-abag` | default | - | - | - | - | - | 544 |

MSA `default` means on unless you send `use_msa_server: false`; `never` means the
model is always single-sequence.

**Boltz-2** is the default, the most capable, and the only model with affinity
and constraints. **ESMFold-2** is language-model folding, protein
chains only; `esmfold2-fast` is always single-sequence, for screening many
sequences at once. **Protenix-v2** is AlphaFold3-family (Pairformer + atom
diffusion) and strong at antibody-antigen. **OpenFold3** is the OpenFold
Consortium's open AlphaFold3 reproduction, folding protein, RNA and DNA
complexes; its published weights are a preview checkpoint trained well short of
the full AlphaFold3 schedule, so read the confidence scores before trusting a
prediction. **OpenBind-0** is the same OpenFold3 stack on the consortium's
OpenBind checkpoint, which co-folds small-molecule ligands the preview weights
refuse. The two **OpenDDE** checkpoints are
protein-only: `opendde` for general complexes, `opendde-abag` to co-fold an
antibody Fab heavy/light with its antigen. Both match the reference OpenDDE
implementation, including its own weakness on some hard antibody-antigen targets,
so don't expect uniform accuracy on every input. See [Accuracy](accuracy.md).

Boltz-2 and both ESMFold-2 variants also accept modified residues.

## Embedding models

`POST /v1/embeddings` runs protein language models. Larger is a stronger
representation at more compute per sequence. See [Embeddings](embeddings.md).

| `id` | Name | Max residues | Notes |
|---|---|--:|---|
| `esmc-300m` | ESMC 300M | 2000 | Quickest; strong general-purpose representation. |
| `esmc-600m` | ESMC 600M | 2000 | The balanced default. |
| `esmc-6b` | ESMC 6B | 1968 | Strongest representation, highest compute cost. |
| `saprot-650m` | SaProt 650M | 2000 | Trained on sequence + structure tokens, run sequence-only here. |
| `saprot-1.3b` | SaProt 1.3B | 2000 | Largest SaProt; trained to work sequence-only. |

## Prediction parameters

Sent as `params` on `POST /v1/predictions`. Out-of-range values are clamped.

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `use_msa_server` | bool | `true` | - | Build an MSA. Boltz-2 cannot fold single-sequence, so `false` is forced back to `true` for it. Every other model honours it. |
| `fast` | bool | `true` | - | Higher throughput, may be slightly less accurate. Ignored for OpenFold3 and OpenBind-0, which always run the full-precision path. |
| `recycling_steps` | int | model default | 1–10 | Trunk recycles. Omit it: Boltz-2 uses 3, the others 10. |
| `sampling_steps` | int | model default | 10–500 | Diffusion steps. Omit it: ESMFold-2 uses 100, the others 200. |
| `diffusion_samples` | int | `1` | 1–5 | Structures generated per target. |
| `output_format` | enum | `cif` | `cif`, `pdb` | Structure file format. |

## Design protocols and parameters

Three design models share `POST /v1/designs`. **BoltzGen** protocols take a YAML `spec`
and return ranked designs: `protein-anything`, `peptide-anything`,
`nanobody-anything`, `antibody-anything`, `protein-small_molecule`,
`protein-redesign`. **RFdiffusion3** protocols take a pasted `structure` plus a
`contig` and return unranked all-atom designs: `rfd3-binder`, `rfd3-scaffold`,
`rfd3-na-binder`. **PXDesign** has one protocol, `pxdesign-binder`: a pasted
`structure`, the target `chains` to condition on and a `binder_length`, with
optional `hotspots`. It returns binder backbones, coordinates only, with no
sequence and no ranking. Target ceilings differ by model: BoltzGen 1024
residues, RFdiffusion3 490, PXDesign 768. [Designs](designs.md) says what each
one does.

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
| `max_residues` | 1024 | per structure; per model: protenix-v2 980, openfold3 576, openbind 576, opendde 544, opendde-abag 544 |
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
| `max_active_jobs` | 64 | service-wide |
| `max_active_jobs_per_ip` | 8 | |
| `max_active_jobs_per_session` | 3 | |
| `max_submits_per_min` | 12 | service-wide |
| `max_submits_per_min_per_ip` | 40 | |
| `max_retained_jobs` | 1000 | |
| `max_runtime_predict_s` | 1500 | |
| `max_runtime_design_s` | 2700 | |
| `max_runtime_embed_s` | 300 | |
| `max_stall_s` | 600 | predict |
| `max_stall_design_s` | 1200 | |
| `max_stall_embed_s` | 120 | |

Over a size cap you get `400`. At capacity or over a rate limit you get `429`
with `Retry-After`. See [Errors](errors.md).

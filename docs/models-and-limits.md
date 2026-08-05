# Models & limits

`GET /v1/models` is the live, machine-readable version of this page: the model
list, every parameter with its default and range, the design protocols, and the
current limits. The tables below mirror it. If they ever disagree, trust the
endpoint.

```bash
curl -s https://api.japanfold.com/v1/models
```

`GET /v1/health` is a plain liveness check (`{"status":"ok","service":"japanfold","api_version":"1.0.0"}`).

## Prediction models

| `id` | MSA | Ligands | DNA/RNA | Affinity | Constr | PAE |
|---|---|:-:|:-:|:-:|:-:|:-:|
| `boltz2` | default | ✓ | ✓ | ✓ | ✓ | ✓ |
| `esmfold2` | optional | - | - | - | - | - |
| `esmfold2-fast` | never | - | - | - | - | - |
| `protenix-v2` | default | ✓ | ✓ | - | - | ✓ |
| `opendde` | default | - | - | - | - | - |
| `opendde-abag` | default | - | - | - | - | - |

MSA `default` means on unless you send `use_msa_server: false`; `never` means the
model is always single-sequence.

**Boltz-2** is the default, the most capable, and the only model with affinity,
constraints and potentials. **ESMFold-2** is language-model folding, protein
chains only; `esmfold2-fast` is always single-sequence, for screening many
sequences at once. **Protenix-v2** is AlphaFold3-family (Pairformer + atom
diffusion) and strong at antibody-antigen. The two **OpenDDE** checkpoints are
protein-only: `opendde` for general complexes, `opendde-abag` to co-fold an
antibody Fab heavy/light with its antigen. Both match the reference OpenDDE
implementation, including its own weakness on some hard antibody-antigen targets,
so don't expect uniform accuracy on every input. See [Accuracy](accuracy.md).

Boltz-2 and both ESMFold-2 variants also accept modified residues.

## Embedding models

`POST /v1/embeddings` runs protein language models. Larger is a stronger
representation at more compute per sequence. See [Embeddings](embeddings.md).

| `id` | Name | Notes |
|---|---|---|
| `esmc-300m` | ESMC 300M | Quickest; strong general-purpose representation. |
| `esmc-600m` | ESMC 600M | The balanced default. |
| `esmc-6b` | ESMC 6B | Strongest representation, highest compute cost. |
| `saprot-650m` | SaProt 650M | Trained on sequence + structure tokens, run sequence-only here. |
| `saprot-1.3b` | SaProt 1.3B | Largest SaProt; trained to work sequence-only. |

## Prediction parameters

Sent as `params` on `POST /v1/predictions`. Out-of-range values are clamped.

| Key | Type | Default | Range | Notes |
|---|---|---|---|---|
| `use_msa_server` | bool | `true` | - | Build an MSA. Required for Boltz-2 and Protenix-v2, optional for ESMFold-2. |
| `fast` | bool | `true` | - | Higher throughput, may be slightly less accurate. |
| `recycling_steps` | int | `3` | 1–10 | More can improve accuracy, slower. |
| `sampling_steps` | int | `200` | 10–500 | Diffusion steps per structure. |
| `diffusion_samples` | int | `1` | 1–5 | Structures generated per target. |
| `output_format` | enum | `cif` | `cif`, `pdb` | Structure file format. |

## Design protocols and parameters

Two engines share `POST /v1/designs`. **BoltzGen** protocols take a YAML `spec`
and return ranked designs: `protein-anything`, `peptide-anything`,
`nanobody-anything`, `antibody-anything`, `protein-small_molecule`,
`protein-redesign`. **RFdiffusion3** protocols take a pasted `structure` plus a
`contig` and return unranked all-atom designs: `rfd3-binder`, `rfd3-scaffold`,
`rfd3-na-binder`. [Designs](designs.md) says what each one does.

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
| `max_residues` | 1024 | per structure |
| `max_chains_per_complex` | 10 | |
| `max_ligands_per_complex` | 10 | |
| `max_constraints_per_complex` | 20 | |
| `max_complexes` | 10 | structures per run |
| `max_content_chars` | 50000 | per input string |
| `max_designs` | 10 | BoltzGen designs per run |
| `max_budget` | 10 | BoltzGen designs kept |
| `max_rfd3_designs` | 5 | per RFdiffusion3 run |
| `max_rfd3_timesteps` | 200 | |
| `max_structure_chars` | 700000 | pasted target structure |
| `max_embed_sequences` | 50 | per submission |
| `max_embed_sequence_residues` | 2000 | per sequence |
| `max_recycling_steps` | 10 | |
| `max_sampling_steps` | 500 | |
| `max_diffusion_samples` | 5 | |
| `max_active_jobs` | 64 | service-wide |
| `max_active_jobs_per_ip` | 8 | |
| `max_active_jobs_per_session` | 3 | |
| `max_submits_per_min` | 12 | service-wide |
| `max_submits_per_min_per_ip` | 40 | |
| `max_retained_jobs` | 200 | |
| `max_runtime_predict_s` | 1500 | |
| `max_runtime_design_s` | 2700 | |
| `max_runtime_embed_s` | 300 | |
| `max_stall_s` | 600 | predict |
| `max_stall_design_s` | 1200 | |
| `max_stall_embed_s` | 120 | |

Over a size cap you get `400`. At capacity or over a rate limit you get `429`
with `Retry-After`. See [Errors](errors.md).

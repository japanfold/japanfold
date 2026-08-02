# Designs

`POST /v1/designs` designs de-novo proteins against a target. Two models share
the endpoint: [BoltzGen](https://github.com/jwohlwend/boltz) (target = a
sequence or ligand, designs come back **ranked** by predicted confidence) and
RFdiffusion3 (target = a pasted **structure** plus a contig string, designs
come back **unranked**). Both return a **job** to poll, exactly like
[predictions](jobs.md).

The protocol implies the model. You may pass `model` explicitly
(`boltzgen` or `rfd3` — the same vocabulary as the CLI's `--model`), but it
must match the protocol's model; a mismatch is a 400.

## BoltzGen: binders against a sequence or ligand

```bash
curl -s -X POST https://api.japanfold.com/v1/designs \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol":"nanobody-anything",
    "name":"my-nanobodies",
    "spec":"sequences:\n  - protein: {id: A, sequence: MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ}\n",
    "params":{"num_designs":10,"budget":10,"fast":true}
  }'
```

- **`spec`** (required): a YAML design spec, the target plus the binder request.
  Same YAML dialect as the Boltz/BoltzGen inputs.
- **`protocol`**: what to design (see below).
- **`name`**: optional label.
- **`params`**: see below.

### Protocols

| `protocol` | Designs |
|---|---|
| `protein-anything` | De-novo mini-protein binder against any target. |
| `peptide-anything` | Short peptide binder. |
| `nanobody-anything` | Single-domain antibody / nanobody (VHH). |
| `antibody-anything` | Antibody binder. |
| `protein-small_molecule` | Protein binder with a binding-affinity step. |
| `protein-redesign` | Re-design residues of an existing binder. |

### Parameters

| Param | Type | Default | Range | Notes |
|---|---|---|---|---|
| `num_designs` | int | 10 | 1–10 | Binders to generate before filtering. |
| `budget` | int | 10 | 1–10 | Top ranked designs to keep after filtering. |
| `fast` | bool | true | - | Higher throughput, may be slightly less accurate. |

(See [Models & limits](models-and-limits.md) for the current ranges.)

Submits accept the same `Idempotency-Key` and `Prefer: wait` headers as
predictions. See [Predictions](predictions.md#retrying-safely-idempotency-key).

## RFdiffusion3: all-atom design against a structure

RFdiffusion3 (RFD3) diffuses protein structure and sequence together, conditioned
on a target **structure** you provide. The request carries the structure as text
plus a **contig** string saying what to keep fixed and what to design:

```bash
curl -s -X POST https://api.japanfold.com/v1/designs \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol":"rfd3-binder",
    "name":"my-rfd3-binders",
    "structure":"HEADER ...\nATOM      1  N   ALA A   1      ...\n...",
    "contig":"A1-150,60-80",
    "params":{"num_designs":4,"num_timesteps":100}
  }'
```

- **`structure`** (required): the target as PDB or mmCIF **text** (paste the file
  contents; up to 700,000 characters, about a 1,000-residue PDB).
- **`contig`** (required): comma-separated segments. A chain-range like `A1-150`
  keeps those residues fixed; a bare number like `60-80` designs that many new
  residues. `"A1-150,60-80"` reads: keep target chain A (150 residues), then
  design a 60–80 residue binder.
- **`params`**: `num_designs` (1–5, default 4), `num_timesteps` (4–200, default
  100 — 200 is the cleanest, fewer is faster), `seed`.

### RFD3 protocols

| `protocol` | Designs |
|---|---|
| `rfd3-binder` | Binder against a protein target structure. |
| `rfd3-scaffold` | Scaffold around a fixed functional motif (list the motif residues, design between them). |
| `rfd3-na-binder` | Binder against a fixed DNA or RNA structure. |

RFD3 designs are **unranked**: there is no refold-and-filter step, so results
are one mmCIF per design with no confidence scores. To check a design, fold its
sequence back onto the target with a [prediction](predictions.md) and look at
the interface confidence (`iptm`).

## Reading designs

Poll `GET /v1/jobs/{id}` until `succeeded`, then `GET /v1/jobs/{id}/results`.
The results carry the designs and downloadable structure artifacts;
`GET /v1/jobs/{id}/archive` gives everything as one zip. See
[Jobs → Results](jobs.md#results). A full worked example is in
[Examples → Binder design](examples.md#binder-design).

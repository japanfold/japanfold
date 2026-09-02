# Predictions

`POST /v1/predictions` predicts the 3D structure and, with Boltz-2, the binding
affinity of a protein or complex. It returns a **job** to poll (see
[Jobs](jobs.md)).

## Three ways to specify input

Provide exactly one of these.

### 1. `sequence`: a single protein chain

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"myprotein","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}'
```

### 2. `input`: one FASTA or Boltz YAML string

Use this for complexes, multiple chains, ligands, nucleic acids, affinity and
constraints. The string is a full [Boltz](https://github.com/jwohlwend/boltz)
YAML or a FASTA. Note the `\n` newlines when embedding it in JSON:

```bash
# Human insulin: two protein chains
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"boltz2","name":"human-insulin",
    "input":"sequences:\n  - protein: {id: A, sequence: GIVEQCCTSICSLYQLENYCN}\n  - protein: {id: B, sequence: FVNQHLCGSHLVEALYLVCGERGFFYTPKT}\n"
  }'
```

### 3. `targets`: a list, to fold many inputs in one job

Each target is `{ "content": "<FASTA or YAML>", "name": "<optional>" }`.

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"esmfold2-fast",
    "targets":[
      {"name":"a","content":">a\nMKTAYIAKQRQISFVKSHFSRQLEE"},
      {"name":"b","content":">b\nGIVEQCCTSICSLYQLENYCN"}
    ]
  }'
```

## Choosing a model

Set `model`, default `boltz2`. Boltz-2 is the most capable and the only one that
returns a binding affinity or accepts constraints; ESMFold-2 and OpenDDE are
protein-only. The full capability matrix is on
[Models & limits](models-and-limits.md#prediction-models).

## Co-folding with a ligand + affinity (Boltz-2)

Add a `ligand` chain (SMILES or CCD code) and a `properties: affinity` block
naming the binder chain:

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"boltz2","name":"prot-ligand",
    "input":"sequences:\n  - protein: {id: A, sequence: MKTAYIAKQRQISFVKSHFSRQLEE}\n  - ligand: {id: L, smiles: \"CC(=O)Oc1ccccc1C(=O)O\"}\nproperties:\n  - affinity: {binder: L}\n"
  }'
```

The result row then carries two affinity numbers alongside the structure and
confidence scores (see [Jobs → Results](jobs.md#results)):

- `affinity_pred_value`: predicted log10(IC50) with IC50 in μM, so **lower is a
  stronger binder**. Boltz's own guidance is to use it to rank actives against
  each other, not to call a compound active or inactive.
- `affinity_probability_binary`: probability the ligand binds at all, 0-1, so
  higher is stronger.

Both also come back with a `1` and `2` suffix, the two ensemble members, plus
`affinity_runtime_s` and `structure_runtime_s`.

## Parameters

Pass a `params` object: `use_msa_server`, `fast`, `recycling_steps`,
`sampling_steps`, `diffusion_samples`, `output_format`. Values are clamped to
their allowed range. Defaults, ranges and per-model applicability are on
[Models & limits](models-and-limits.md#prediction-parameters).

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"boltz2","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ",
    "params":{"use_msa_server":true,"diffusion_samples":3,"output_format":"pdb"}
  }'
```

> **MSA and privacy.** With `use_msa_server` on (the Boltz-2 and Protenix-v2
> default), your sequence goes to an external MSA server for the alignment step.
> To fold strictly single-sequence, set `use_msa_server: false`.

## Waiting inline: the `Prefer: wait` header

By default a submit returns immediately with a job to poll. Add
`Prefer: wait=<seconds>` on the create or on a `GET /v1/jobs/{id}` poll to block
until the job finishes or the timeout elapses, which turns a poll loop into one
call for short jobs. Bare `wait` holds 25s; `wait=N` holds up to N seconds,
capped at 60:

```bash
curl -s -H 'Prefer: wait=60' https://api.japanfold.aiand.com/v1/jobs/$JOB
```

## Retrying safely: `Idempotency-Key`

Send an `Idempotency-Key: <unique>` header on a create. A retried submit with the
same key and caller returns the original job instead of launching a duplicate:

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/predictions \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: run-42' \
  -d '{"model":"boltz2","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}'
```

## Response

The body is a **Job**: `id`, `status`, `kind: "predict"`, `model`, timestamps and
a `links` map. Poll `links.self` (or `/v1/jobs/{id}`) and read
`/v1/jobs/{id}/results` once `results_ready` is true. [Jobs](jobs.md) has the
full lifecycle and result shape.

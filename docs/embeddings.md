# Embeddings

`POST /v1/embeddings` turns protein sequences into protein language-model
embeddings (ESMC or SaProt): numeric vectors you can feed to a classifier,
clustering, similarity search or any downstream model. It returns a job to poll
(see [Jobs](jobs.md)). No structure prediction, no MSA.

## Two kinds of vector

For every sequence you get both:

- **Per-residue**, shape `[length, d_model]`. For residue-level tasks (contact or
  site prediction, per-position features). The `<cls>`/`<eos>` boundary tokens
  are stripped, so row *i* is residue *i*.
- **Pooled**, shape `[d_model]`, the per-residue vectors combined (see `pool`).
  For one vector per protein: similarity, clustering, a sequence-level
  classifier.

`d_model` depends on the model (960 for `esmc-300m`) and is reported in the
results.

## Input

Provide exactly one of these, as on [Predictions](predictions.md).

### 1. `sequence`: a single chain

```bash
curl -s -X POST https://api.japanfold.com/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}'
```

### 2. `sequences`: a list, to embed many in one job

Each item is a bare string or an object `{"sequence": "...", "id": "..."}`:

```bash
curl -s -X POST https://api.japanfold.com/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"esmc-600m",
    "sequences":[
      {"id":"a","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"},
      {"id":"b","sequence":"GIVEQCCTSICSLYQLENYCN"}
    ]
  }'
```

### 3. `input`: one FASTA string

```bash
curl -s -X POST https://api.japanfold.com/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","input":">a\nMKTAYIAKQRQISFVKSHFSRQLEE\n>b\nGIVEQCCTSICSLYQLENYCN"}'
```

## Models

Set `model`, default `esmc-600m`: `esmc-300m`, `esmc-600m`, `esmc-6b`,
`saprot-650m`, `saprot-1.3b`. Larger is a stronger representation at more compute
per sequence. SaProt is trained on sequence + structure tokens and runs
sequence-only here (structure tokens masked), which the 1.3B variant is
explicitly trained for. See
[Models & limits](models-and-limits.md#embedding-models).

## Parameters

Pass a `params` object.

| Key | Type | Default | Notes |
|---|---|---|---|
| `pool` | enum | `mean` | How per-residue vectors become the pooled vector: `mean`, `max`, or `cls` (the `<cls>` token). |
| `format` | enum | `npz` | `npz`: per-residue + pooled, one file per sequence. `parquet`: pooled vectors only, one table. |
| `fast` | bool | `false` | Higher throughput, may be slightly less precise. |

```bash
curl -s -X POST https://api.japanfold.com/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","sequence":"MKTAYIAK...","params":{"pool":"mean","format":"npz"}}'
```

## Results

Once `results_ready` is true, `GET /v1/jobs/{id}/results` carries
`kind: "embed"`, the `model`, `pool`, `format`, `d_model`, a `sequences` list
(`{id, length, file}` per input) and an `artifacts` list of download URLs.

- **`npz`** (default): one `<id>.npz` per sequence with arrays `per_residue`
  `[length, d_model]`, `pooled` `[d_model]` and `sequence`.
- **`parquet`**: a single `embeddings.parquet` holding the pooled matrix, one row
  per sequence. Per-residue vectors are ragged, so they are not in the table; use
  `npz` for those.

Download individual files from their artifact `url`, or the whole set from
`GET /v1/jobs/{id}/archive`.

## End to end (Python)

```python
import io, time, httpx, numpy as np

BASE = "https://api.japanfold.com"
job = httpx.post(f"{BASE}/v1/embeddings",
                 json={"model": "esmc-600m",
                       "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}).json()
while job["status"] not in ("succeeded", "failed", "canceled"):
    time.sleep(3)
    job = httpx.get(f"{BASE}/v1/jobs/{job['id']}").json()

res = httpx.get(f"{BASE}/v1/jobs/{job['id']}/results").json()
url = BASE + res["artifacts"][0]["url"]
data = np.load(io.BytesIO(httpx.get(url).content))
print(data["per_residue"].shape, data["pooled"].shape)  # (L, d_model) (d_model,)
```

With the [JapanFold skill](skill.md) installed, ask instead: *"Embed these
sequences with ESMC-600M and save the pooled vectors."*

Embed jobs accept the same `Prefer: wait[=seconds]` and `Idempotency-Key`
headers as predictions
(see [Predictions](predictions.md#waiting-inline-the-prefer-wait-header)).

## Limits

At most 50 sequences per submission and 2000 residues per sequence, higher than
the folding cap because embeddings run the language model only. Over a cap you
get `400`, at capacity `429`. See
[Models & limits](models-and-limits.md#limits) and [Errors](errors.md).

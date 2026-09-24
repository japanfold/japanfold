---
name: japanfold
description: >-
  Predict 3D biomolecular structures and binding affinity (Boltz-2, ESMFold-2,
  Protenix-v2, OpenFold3, OpenBind-0, RoseTTAFold3, OpenDDE), design de-novo
  binders/proteins (BoltzGen, RFdiffusion3, PXDesign),
  scaffold a functional motif around a pasted structure, and compute ESMC or
  SaProt protein-language-model embeddings via JapanFold, a hosted,
  Tenstorrent-accelerated HTTP API in Japan with a free daily allowance. Use to fold a protein or complex, co-fold a
  protein with a ligand and get affinity, design nanobody/antibody/peptide/
  miniprotein binders against a target, turn a sequence into a PDB/mmCIF
  structure, or get a fixed-size embedding vector for search/clustering/ML
  features. No API key needed to start, no local GPU.
when_to_use: >-
  When the user wants to fold/predict a protein or complex structure, estimate
  protein–ligand binding affinity, design binders against a target, or compute
  protein embeddings — and a hosted service is fine (no local model to run).
license: Apache-2.0
category: biomodels
metadata:
  third_party:
    - kind: service
      name: JapanFold API
      provider: JapanFold
      info_url: https://japanfold.com
# allowed-tools is a Claude Code convenience (grants curl/python without a
# prompt); other harnesses ignore it and use their own execution/permission model.
allowed-tools:
  - Bash(curl *)
  - Bash(python3 *)
---

# JapanFold — hosted structure prediction & binder design

JapanFold runs Boltz-2 / ESMFold-2 / Protenix-v2 / OpenFold3 / OpenBind-0 / RoseTTAFold3 /
OpenDDE (structure prediction; Boltz-2 also does affinity, OpenBind-0 and RoseTTAFold3
co-fold ligands, OpenDDE is protein-complex / antibody-antigen docking), BoltzGen / RFdiffusion3 /
PXDesign (binder design), and ESMC / SaProt (protein-language-model embeddings) on Tenstorrent
hardware behind an HTTP API hosted in Japan. You call it as an async job
(**submit → poll → download**) over plain HTTPS against
`https://api.japanfold.com`. No model to install, no local GPU, and no key
needed to start: keyless calls spend a small free grant that renews daily.

**Credits and keys.** Work is priced in credits for the chip time it uses. If the
user has an account, they create a key at `japanfold.com/account/keys` and
set `JAPANFOLD_API_KEY` (and `JAPANFOLD_BASE_URL` if they were given another
endpoint). The examples below send the key on every call when it is set, which
matters: a job belongs to the key that submitted it, so polling it without the
key returns `404`.

Works from any agent/harness: use `curl` (Bash) or your language's HTTP client
(`httpx`/`requests`, `fetch`, `net/http`, …) — whatever your environment has.
If your environment sandboxes network egress (e.g. Claude Science), approve the
host **`api.japanfold.com`** when prompted.

## Predict a structure

Submit → poll until `status` is terminal → read results:

```bash
BASE=${JAPANFOLD_BASE_URL:-https://api.japanfold.com}
H=(-H 'X-JapanFold-Client: skill')
[ -n "$JAPANFOLD_API_KEY" ] && H+=(-H "Authorization: Bearer $JAPANFOLD_API_KEY")
# 1. submit — input is a bare `sequence`, one `input` FASTA/YAML string, or a `targets` list
JOB=$(curl -s "${H[@]}" -X POST $BASE/v1/predictions -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"mytarget","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}' \
  | python3 -c 'import sys,json; r=json.load(sys.stdin); print(r["id"]) if "id" in r else sys.exit(r["detail"])')

# 2. poll. The first job on a processor also loads the model's weights. Tip: add
#    header `Prefer: wait=60` to the GET to block until the job finishes (up to 60s).
#    When the service is busy the job carries `queue`: its `reason`, how many
#    `accounts_ahead` and the fleet's chip counts. That is a real wait, not a
#    stall, so keep polling. It has no time estimate, so do not invent one.
curl -s "${H[@]}" $BASE/v1/jobs/$JOB          # -> {"status":"queued|running|succeeded|failed", ...}

# 3. once status=succeeded: scores + artifact URLs, then download the bundle
curl -s "${H[@]}" $BASE/v1/jobs/$JOB/results
curl -sOJ "${H[@]}" $BASE/v1/jobs/$JOB/archive          # zip: structures + results.json
```

**Multi-chain complexes** (e.g. insulin's A+B chains) go in the `input` YAML —
one `protein` entry per chain, not the bare `sequence` field:

```bash
curl -s "${H[@]}" -X POST $BASE/v1/predictions -H 'Content-Type: application/json' -d '{
  "model":"boltz2","name":"human-insulin",
  "input":"sequences:\n  - protein: {id: A, sequence: GIVEQCCTSICSLYQLENYCN}\n  - protein: {id: B, sequence: FVNQHLCGSHLVEALYLVCGERGFFYTPKT}\n"
}'
```

Python-kernel equivalent (Claude Science, notebooks):

```python
import os, time, httpx
BASE = os.environ.get("JAPANFOLD_BASE_URL", "https://api.japanfold.com")
key = os.environ.get("JAPANFOLD_API_KEY")
jf = httpx.Client(base_url=BASE, headers={"X-JapanFold-Client": "skill",
                  **({"Authorization": f"Bearer {key}"} if key else {})})
job = jf.post("/v1/predictions", json={"model": "boltz2", "sequence": "MKT..."}).json()
while job["status"] not in ("succeeded", "failed", "canceled"):
    time.sleep(5)
    job = jf.get(f"/v1/jobs/{job['id']}").json()
res = jf.get(f"/v1/jobs/{job['id']}/results").json()
```

- **Models:** `boltz2` (default; MSA + ligands + affinity), `esmfold2`,
  `esmfold2-fast` (single-sequence, fastest) — both co-fold ligands and
  nucleic acids, neither predicts affinity — `protenix-v2`, `openfold3` (the
  OpenFold Consortium's AlphaFold3 reproduction, preview weights; protein / RNA /
  DNA, no ligands or affinity), `openbind` (the same stack on the OpenBind-0
  checkpoint, which does co-fold ligands),
  `rf3` (RoseTTAFold3 — proteins, RNA/DNA and ligands, no covalent modifications or
  binding constraints from this input format), and the OpenDDE family — `opendde`
  (general protein-complex checkpoint) and `opendde-abag` (antibody-antigen
  checkpoint), both protein-only with MSA on by default, no affinity.
  `opendde-abag`'s accuracy is verified to match the reference OpenDDE
  implementation: strong on standard antibody-antigen complexes, and it shares
  the reference's own limitation on some hard targets (a checkpoint
  characteristic, not a port defect). For binding affinity use `boltz2`. For
  ligands, `boltz2`, `protenix-v2`, `openbind`, `rf3` or the `esmfold2` pair;
  add `openfold3` for DNA/RNA without ligands.
- For complexes / protein–ligand affinity / multiple chains, pass a **Boltz YAML**
  string as `input` (`sequences:` with `protein`/`dna`/`rna`/`ligand` chains;
  `properties:` for the affinity head).
- `params`: `use_msa_server` (on by default for Boltz-2), `fast`, `recycling_steps`,
  `sampling_steps`, `diffusion_samples`, `output_format`, `seed` (default 0, echoed on the
  job), `write_pae` (models with the `pae` cap: returns `<name>_pae.npz`, the matrix
  Adaptyv's ipSAE pipeline reads). Not every model takes every one:
  a param the model cannot honour comes back as a 400 naming both, so read `caps` in
  `GET /v1/models` (no `fast` on OpenFold3, OpenBind-0 or RoseTTAFold3; no `use_msa_server`
  on ESMFold-2 Fast, which has no MSA encoder). Leave `recycling_steps` and `sampling_steps`
  out unless you mean to override a model's own value.
- **Fast mode is off by default.** Send `"fast": true` for higher throughput; it may be
  slightly less accurate. The workbench turns it on for humans, the API never does it for
  you. ESMFold-2 is the exception: it always runs fast here and its job says so.
- `GET /v1/models` lists every model, protocol, parameter, and the current limits.

## Design binders (BoltzGen, RFdiffusion3 or PXDesign)

Three design models, and you pick one. They take different inputs, so the choice
comes first — `GET /v1/models` returns `design_models`, each model's `params`, and
each protocol's `engine`.

**BoltzGen** — a target described in a YAML spec, out comes a ranked, filtered
top set with confidence metrics. Protocols: `protein-anything`,
`peptide-anything`, `nanobody-anything`, `antibody-anything`,
`protein-small_molecule`, `protein-redesign`.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"model":"boltzgen","protocol":"nanobody-anything","spec":"<YAML design spec>",
       "params":{"num_designs":10}}'
```

**RFdiffusion3** — all-atom diffusion directly around a target structure you
paste in, with a contig saying what stays fixed and what gets designed. Protocols:
`rfd3-binder`, `rfd3-scaffold`, `rfd3-na-binder`. Params: `num_designs`,
`num_timesteps`, `seed`.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"model":"rfd3","protocol":"rfd3-binder","structure":"<PDB or mmCIF text>",
       "contig":"A1-150,60-80","params":{"num_designs":4}}'
```

**PXDesign** — binder backbones against a target structure, conditioned on a
distogram of the chains you name. Fastest of the three. Protocol:
`pxdesign-binder`. Params: `num_designs`, `n_step`, `seed`.

**It returns a backbone with no sequence** — coordinates only, no ranking, no
confidence score. Every binder residue is written as GLY with just N/CA/C/O,
because that is what the model generates. Take the coordinates to a
sequence-design tool before ordering anything; for a ranked, sequenced binder use
BoltzGen. Each design carries a `fit_rmsd`, the residual of fitting the model's
own reconstruction of the target onto the real target — the number that says
whether the conditioning worked.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"model":"pxdesign","protocol":"pxdesign-binder","structure":"<PDB or mmCIF text>",
       "chains":"A","binder_length":80,"hotspots":"A64,A65,A66",
       "params":{"num_designs":4}}'
```

`model` is optional but must match the protocol's engine, else `400`. Poll the
same way; `/v1/jobs/{id}/results` returns the designs.

## Compute protein embeddings (ESMC, SaProt)

Submit → poll → download, same as predict/design:

```bash
JOB=$(curl -s "${H[@]}" -X POST $BASE/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}' \
  | python3 -c 'import sys,json; r=json.load(sys.stdin); print(r["id"]) if "id" in r else sys.exit(r["detail"])')

curl -s "${H[@]}" $BASE/v1/jobs/$JOB               # poll until status is terminal
curl -s "${H[@]}" $BASE/v1/jobs/$JOB/results       # -> sequences: [{id, length, file}, ...]
curl -sOJ "${H[@]}" $BASE/v1/jobs/$JOB/archive     # zip: manifest.json + embeddings
```

Multiple sequences go in `sequences` (a list) or `input` (a FASTA/YAML blob —
same flexibility as predict's `input`):

```bash
curl -s "${H[@]}" -X POST $BASE/v1/embeddings -H 'Content-Type: application/json' -d '{
  "model":"esmc-600m",
  "sequences":[{"id":"gfp","sequence":"MSKGEELFTGVVPILVELDGDVNGHKFSVSGEGEGDAT"},
               {"id":"lysozyme","sequence":"KVFGRCELAAAMKRHGLDNYRGYSLGNWVCAAKFESNF"}]
}'
```

- **Models:** `esmc-300m`, `esmc-600m` (default), `esmc-6b` — larger trunks give
  a stronger representation at higher compute cost per sequence — plus
  `saprot-650m` and `saprot-1.3b`, trained on a joint sequence + structure
  vocabulary and run sequence-only here. SaProt is a different representation
  from ESMC, often stronger for stability and function prediction.
- `params`: `pool` (`mean` default, `max`, `cls` — how per-residue vectors combine
  into one fixed-size vector), `format` (`npz` default: per-residue + pooled, one
  file per sequence; `parquet`: pooled vectors only, one table), `fast`.
- Results: `manifest.json` (model/pool/shapes/dtype) plus one `<id>.npz` per
  sequence (or one shared `embeddings.parquet`). No structure, no MSA — just the
  language-model trunk, so these are the cheapest jobs.
- `GET /v1/models` lists the embedding models and params too.

## Reading results

`GET /v1/jobs/{id}/results` gives `ready`, an `artifacts` list (each with a `url`),
and — for a prediction — per-target `rows` (`confidence_score`, `complex_plddt`,
`iptm`, affinity fields); for a design, the ranked `designs`. Pass lines mirror
Boltz-2: interface `iptm` > 0.5, fold `complex_plddt` > 0.7. Download a single
structure from its artifact `url`, or the whole bundle from `…/archive`.

## Limits & notes

- Free public demo caps (same as the web app): **each model's residue ceiling
  per structure, ≤ 10 chains & ligands/complex, ≤ 10 structures/run, ≤ 10
  designs/request**, plus per-IP rate limits. A model's residue ceiling is the
  largest size the engine is measured to fold on this hardware: 1920 for
  Boltz-2, 1792 for Protenix-v2, 1664 for ESMFold-2, 1664 for ESMFold-2 Fast,
  1664 for OpenFold3, 1664 for OpenBind-0, 1600 for RoseTTAFold3, and 1024 for
  every other folding model (OpenDDE folds larger complexes, but slower than a
  job may run). Design is bounded the same way: RFdiffusion3's contig
  (motif + designed regions) caps at 1536 and PXDesign's target chains plus
  binder at 1536, and `binder_length` is 8-200. `GET /v1/models` publishes each
  model's `max_residues`. Over a cap → `400` naming the model and its
  ceiling; at capacity → `429` (respect `Retry-After`). Numeric params are
  clamped to range.
- **A model's ceiling is not its track record.** `GET /v1/models` reports both:
  `max_residues` is what will be accepted, `measured_wall` is the largest
  structure that model has actually run on this hardware, and it can be much
  lower (OpenDDE accepts 1024 and finishes 896 inside the time limit). Sizes in between are taken
  and tried on purpose, because a wall that moves with alignment depth or is not
  monotonic in residue count cannot honestly be a ceiling. Such a job is
  accepted with a `warnings` entry saying so, and if it does fail on device it
  comes back `failed` naming the wall and the models that have run that size —
  one job, nothing else affected. Read `measured_wall` if you want to pick a
  model that will finish rather than one that will be accepted.
- **One more cap bounds the total, not a field.** A submission may cost up to
  **10x** a 1024-residue run of the model you picked, and one structure up to **4x**,
  where cost grows with the square of the residue count and linearly with
  `diffusion_samples`, `recycling_steps` and `sampling_steps`. 10 structures of
  1024 residues at default settings is exactly 10x; a larger structure costs more
  (1152 residues is 1.27x), so fewer of them fit in one submission. Several knobs
  turned up together do not fit either: 10 structures of 1024 residues at
  `diffusion_samples: 3` is 30x and comes back `400` with type
  `.../errors/submission-too-large`, naming what to reduce. `GET /v1/models`
  reports the budget and the pricing under `limits` and `cost`.
- Downloads are bounded in bytes, not requests: **32 GB/hour per network**, which
  is dozens of full result archives. Over it, `429` with `Retry-After`. Polling job
  status is not counted.
- `Prefer: wait` is a preference, not a guarantee (RFC 7240): under load the
  request returns the job's current state at once. `Preference-Applied: wait` on
  the response means the hold happened; if it is absent, poll again.
- Errors are RFC 9457 problem+json (`title`, `detail`).
- No key needed; `Authorization: Bearer <key>` spends the user's account credits
  instead of the free grant. It does not raise any limit.
- **`402` with `out_of_credit: true`** means the balance is spent. The `detail`
  says what the run would reserve and when the free grant renews. Tell the user;
  retrying will not help until they add credit or the grant renews.
- Full machine-readable contract: `GET /v1/openapi.json`.

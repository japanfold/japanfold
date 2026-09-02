---
name: japanfold
description: >-
  Predict 3D biomolecular structures and binding affinity (Boltz-2, ESMFold2,
  Protenix, OpenFold3, OpenBind-0, OpenDDE) and design de-novo binders/proteins
  (BoltzGen, RFdiffusion3, PXDesign) via JapanFold — a free, public,
  Tenstorrent-accelerated HTTP
  API. Use to fold a protein or complex, co-fold a protein with a ligand and
  get affinity, fold an antibody-antigen complex, design
  nanobody/antibody/peptide/miniprotein binders against a target, scaffold a
  functional motif or design a binder against a pasted PDB/mmCIF structure,
  turn a sequence into a PDB/mmCIF structure, or compute ESMC/SaProt protein
  embeddings (per-residue + pooled vectors). No API key or local GPU needed.
when_to_use: >-
  When the user wants to fold/predict a protein or complex structure, estimate
  protein–ligand binding affinity, design binders against a target, or compute
  protein language-model embeddings — and a hosted service is fine (no local
  model to run).
license: Apache-2.0
category: biomodels
metadata:
  third_party:
    - kind: service
      name: JapanFold API
      provider: JapanFold
      info_url: https://japanfold.aiand.com
# allowed-tools is a Claude Code convenience (grants curl/python without a
# prompt); other harnesses ignore it and use their own execution/permission model.
allowed-tools:
  - Bash(curl *)
  - Bash(python3 *)
---

# JapanFold — hosted structure prediction & binder design

JapanFold runs Boltz-2 / ESMFold2 / Protenix / OpenFold3 / OpenBind-0 / OpenDDE
(structure + affinity), BoltzGen / RFdiffusion3 / PXDesign (binder design), and
ESMC / SaProt (embeddings) on
Tenstorrent hardware behind a **free public HTTP API**. You
call it as an async job — **submit → poll → download** — over plain HTTPS against
`https://api.japanfold.aiand.com`. No API key, no model to install, no local GPU.

Works from any agent/harness: use `curl` (Bash) or your language's HTTP client
(`httpx`/`requests`, `fetch`, `net/http`, …) — whatever your environment has.
If your environment sandboxes network egress (e.g. Claude Science), approve the
host **`api.japanfold.aiand.com`** when prompted.

## Predict a structure

Submit → poll until `status` is terminal → read results:

```bash
BASE=https://api.japanfold.aiand.com
# 1. submit — input is a bare `sequence`, one `input` FASTA/YAML string, or a `targets` list
JOB=$(curl -s -X POST $BASE/v1/predictions -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"mytarget","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["id"])')

# 2. poll (a small fold is usually done in well under a minute). Tip: add header
#    `Prefer: wait=60` to the GET to block until the job finishes (up to 60s).
curl -s $BASE/v1/jobs/$JOB          # -> {"status":"queued|running|succeeded|failed", ...}

# 3. once status=succeeded: scores + artifact URLs, then download into a clear output dir
OUT="./japanfold-mytarget"; mkdir -p "$OUT"        # a path you can name back to the user
curl -s "$BASE/v1/jobs/$JOB/results" -o "$OUT/results.json"
curl -s "$BASE/v1/jobs/$JOB/archive" -o "$OUT/output.zip" && unzip -oq "$OUT/output.zip" -d "$OUT"
# -> then TELL the user the absolute path to $OUT and to each structure file.
```

**Always save results to a clear directory and tell the user exactly where.** Server
artifacts are retained only temporarily, so download them, put structures + `results.json`
in a named folder (e.g. `./japanfold-<name>/`), and report the **absolute path(s)** to the
user along with the key scores — never leave output only on the server or in an unstated
temp dir.

**Multi-chain complexes** (e.g. insulin's A+B chains) go in the `input` YAML —
one `protein` entry per chain, not the bare `sequence` field:

```bash
curl -s -X POST $BASE/v1/predictions -H 'Content-Type: application/json' -d '{
  "model":"boltz2","name":"human-insulin",
  "input":"sequences:\n  - protein: {id: A, sequence: GIVEQCCTSICSLYQLENYCN}\n  - protein: {id: B, sequence: FVNQHLCGSHLVEALYLVCGERGFFYTPKT}\n"
}'
```

Python-kernel equivalent (Claude Science, notebooks):

```python
import time, httpx
BASE = "https://api.japanfold.aiand.com"
job = httpx.post(f"{BASE}/v1/predictions",
                 json={"model": "boltz2", "sequence": "MKT..."}).json()
while job["status"] not in ("succeeded", "failed", "canceled"):
    time.sleep(5)
    job = httpx.get(f"{BASE}/v1/jobs/{job['id']}").json()
res = httpx.get(f"{BASE}/v1/jobs/{job['id']}/results").json()
```

- **Models:** `boltz2` (default; MSA + ligands + affinity), `esmfold2`,
  `esmfold2-fast` (single-sequence, fastest), `protenix-v2`, `openfold3` (the
  OpenFold Consortium's AlphaFold3 reproduction, preview weights; protein / RNA
  / DNA, no ligands or affinity), `openbind` (the same stack on the OpenBind-0
  checkpoint, which does co-fold ligands), and the OpenDDE
  family — `opendde` (general protein-complex checkpoint) and `opendde-abag`
  (antibody-antigen checkpoint), both protein-only with MSA on by default, no
  affinity. `opendde-abag`'s accuracy matches the reference OpenDDE
  implementation: strong on standard antibody-antigen complexes, weaker on some
  hard targets (a checkpoint characteristic, not a port defect).
- For complexes / protein–ligand affinity / multiple chains, pass a **Boltz YAML**
  string as `input` (`sequences:` with `protein`/`dna`/`rna`/`ligand` chains;
  `properties:` for the affinity head).
- `params`: `use_msa_server` (on by default for Boltz-2), `fast`, `recycling_steps`,
  `sampling_steps`, `diffusion_samples`, `output_format`.
- `GET /v1/models` lists every model, protocol, parameter, and the current limits.

## Design binders (BoltzGen)

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"nanobody-anything","spec":"<YAML design spec>","params":{"num_designs":10}}'
```

Protocols: `protein-anything`, `peptide-anything`, `nanobody-anything`,
`antibody-anything`, `protein-small_molecule`, `protein-redesign`. Poll the same
way; `/v1/jobs/{id}/results` returns the ranked designs.

The protocol implies the design model; an explicit `model`
(`boltzgen`/`rfd3`/`pxdesign`, the CLI's `--model` vocabulary) is accepted but
must match it.

## Design against a structure (RFdiffusion3)

Same endpoint, but the input is a pasted **structure** plus a **contig** (what
stays fixed vs. what gets designed):

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"rfd3-binder","structure":"<PDB or mmCIF text>","contig":"A1-150,60-80","params":{"num_designs":4,"num_timesteps":100}}'
```

- Contig grammar: `A1-150` keeps target chain A residues 1–150 fixed; a bare
  number designs that many new residues. `"A1-150,60-80"` = keep the 150-residue
  target, design a 60–80 residue binder.
- Protocols: `rfd3-binder` (protein target), `rfd3-scaffold` (build around a
  fixed motif), `rfd3-na-binder` (DNA/RNA target).
- Caps: ≤ 5 designs, ≤ 200 timesteps, structure ≤ 700k chars (~1000-residue PDB).
- Results are **unranked** mmCIFs (no refold/filter scores). To sanity-check a
  design, refold its sequence against the target with `POST /v1/predictions`
  and read `iptm`.

## Design backbones against a structure (PXDesign)

Same endpoint again. PXDesign conditions on a distogram of the target chains you
name and returns binder backbones — coordinates only, no sequence, no ranking,
no confidence. It is the fastest of the three; run the backbones through a
sequence-design tool before ordering anything.

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"pxdesign-binder","structure":"<PDB or mmCIF text>","chains":"A","binder_length":80,"hotspots":"A74,A75,A76","params":{"num_designs":4,"n_step":200}}'
```

- `chains` names the target chains to condition on; `binder_length` is the
  binder's residue count (≤ 200); `hotspots` is optional, target residues to aim at.
- One protocol: `pxdesign-binder`. Target ≤ 768 residues.
- Caps: ≤ 8 designs, ≤ 400 steps.

## Embed sequences (ESMC / SaProt)

Turn protein sequences into language-model vectors — no structure, no MSA. Same
submit → poll → download flow:

```bash
curl -s -X POST $BASE/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}'
# or many at once: "sequences":[{"id":"a","sequence":"..."},{"id":"b","sequence":"..."}]
```

- **Models:** `esmc-300m`, `esmc-600m` (default), `esmc-6b`, `saprot-650m`,
  `saprot-1.3b` — larger is a stronger representation at more compute. SaProt is
  structure-aware but runs sequence-only here (the 1.3B is trained for that).
- `params`: `pool` (`mean`/`max`/`cls`, default `mean`), `format` (`npz` = per-residue
  `[L, d_model]` + pooled `[d_model]` per sequence; `parquet` = pooled table only), `fast`.
- Results carry `kind: "embed"`, `d_model`, a `sequences` list and `artifacts` URLs;
  download the `.npz`/`.parquet` files into a named dir and tell the user the path.

## Reading results

`GET /v1/jobs/{id}/results` gives `ready`, an `artifacts` list (each with a `url`),
and — for a prediction — per-target `rows` (`confidence_score`, `complex_plddt`,
`iptm`, affinity fields); for a design, the ranked `designs`. Pass lines mirror
Boltz-2: interface `iptm` > 0.5, fold `complex_plddt` > 0.7. Download a single
structure from its artifact `url`, or the whole bundle from `…/archive` — into a
named local directory, and tell the user the absolute path where you saved it.

## Limits & notes

- Free public demo caps (same as the web app): **≤ 1024 residues/structure
  (protenix-v2 980, openfold3 576, openbind 576, opendde 544, opendde-abag 544),
  ≤ 10 chains & ligands/complex, ≤ 10 structures/run, ≤ 10 designs/request
  (BoltzGen), ≤ 5 (RFdiffusion3) or ≤ 8 (PXDesign)**,
  plus per-IP rate limits. Over a cap → `400`; at capacity → `429` (respect
  `Retry-After`). Numeric params are clamped to range.
- Errors are RFC 9457 problem+json (`title`, `detail`).
- No key needed; an optional `Authorization: Bearer <key>` scopes jobs to you instead of your IP.
- Full machine-readable contract: `GET /v1/openapi.json`.
- **If a request ever returns HTTP `403` with Cloudflare error `1010`**, that's
  edge-level bot filtering of your HTTP client, not an API error — retry the same
  request with a browser-like `User-Agent` header
  (`Mozilla/5.0 … Chrome/124.0 Safari/537.36`).

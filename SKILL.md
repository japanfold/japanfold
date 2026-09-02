---
name: japanfold
description: >-
  Predict 3D biomolecular structures and binding affinity (Boltz-2, ESMFold-2,
  Protenix-v2, OpenFold3, OpenBind-0, RoseTTAFold3, OpenDDE) and design de novo
  binders/proteins (BoltzGen, RFdiffusion3, PXDesign) via JapanFold, a free,
  public, Tenstorrent-accelerated HTTP
  API. Use to fold a protein or complex, co-fold a protein with a ligand and
  get affinity, fold an antibody-antigen complex, design
  nanobody/antibody/peptide/miniprotein binders against a target, scaffold a
  functional motif or design a binder against a pasted PDB/mmCIF structure,
  turn a sequence into a PDB/mmCIF structure, or compute ESMC/SaProt protein
  embeddings (per-residue + pooled vectors). No API key or local GPU needed.
when_to_use: >-
  When the user wants to fold/predict a protein or complex structure, estimate
  protein-ligand binding affinity, design binders against a target, or compute
  protein language-model embeddings, and a hosted service is fine (no local
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

# JapanFold: hosted structure prediction and binder design

JapanFold runs Boltz-2, ESMFold-2, Protenix-v2, OpenFold3, OpenBind-0,
RoseTTAFold3 and OpenDDE (structure and affinity), BoltzGen, RFdiffusion3 and
PXDesign (binder design), and ESMC and SaProt (embeddings) on Tenstorrent
hardware behind a **free public HTTP API**. You call it as an async job,
**submit → poll → download**, over plain HTTPS against
`https://api.japanfold.aiand.com`. No API key, no model to install, no local GPU.

Works from any agent or harness: use `curl` (Bash) or your language's HTTP client
(`httpx`, `requests`, `fetch`, `net/http`), whatever your environment has.
If your environment sandboxes network egress (e.g. Claude Science), approve the
host **`api.japanfold.aiand.com`** when prompted.

## Predict a structure

Submit, poll until `status` is terminal, then read results:

```bash
BASE=https://api.japanfold.aiand.com
# 1. submit. Input is a bare `sequence`, one `input` FASTA/YAML string, or a `targets` list
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

**Always save results to a clear directory and tell the user exactly where.** The
service keeps only the 1000 most recent jobs, so download them, put structures
and `results.json` in a named folder (e.g. `./japanfold-<name>/`), and report the
**absolute path(s)** to the user along with the key scores. Never leave output
only on the server or in an unstated temp dir.

A job that sits at `queued` with `waiting_for_device: true` and a `queue` object
is waiting for a free processor, not stuck. There is no position or ETA, because
the scheduler is fair rather than first-in-first-out and a fold may still have to
load its model's weights, which makes the first call to a given model slower than
the next. Keep polling.

**Multi-chain complexes** (e.g. insulin's A+B chains) go in the `input` YAML,
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
                 json={"model": "boltz2",
                       "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}).json()
while job["status"] not in ("succeeded", "failed", "canceled"):
    time.sleep(5)
    job = httpx.get(f"{BASE}/v1/jobs/{job['id']}").json()
res = httpx.get(f"{BASE}/v1/jobs/{job['id']}/results").json()
```

- **Models**, with the residue ceiling each one enforces per structure:
  `boltz2` 1024 (default; MSA, ligands, affinity, constraints; from MIT and
  Recursion), `esmfold2` 1024 and `esmfold2-fast` 1024 (Biohub; the fast
  checkpoint is always single-sequence and the quickest way to screen many
  sequences), `protenix-v2` 980 (ByteDance), `openfold3` 576 (the OpenFold
  Consortium's AlphaFold3 reproduction on preview weights; protein, RNA and DNA,
  no ligands, no affinity), `openbind` 576 (the same stack on the consortium's
  OpenBind-0 checkpoint, which does co-fold ligands), `rf3` 627 (RoseTTAFold3,
  from the Institute for Protein Design; co-folds ligands), and the OpenDDE
  family from Aureka, `opendde` 544 (general protein complexes) and
  `opendde-abag` 544 (antibody-antigen), both protein-only with MSA on by
  default and no affinity. Over a model's ceiling the API refuses at submit and
  names the models that do take the size. `opendde-abag`'s accuracy matches the
  reference OpenDDE implementation: strong on standard antibody-antigen
  complexes, weaker on some hard targets, which is a checkpoint characteristic
  and not a port defect.
- For complexes, protein-ligand affinity or multiple chains, pass a **Boltz YAML**
  string as `input` (`sequences:` with `protein`/`dna`/`rna`/`ligand` chains;
  `properties:` for the affinity head).
- `params`: `use_msa_server` (on by default, and forced on for Boltz-2, which
  cannot fold single-sequence), `fast` (ignored by `openfold3`, `openbind` and
  `rf3`, which have no fast path), `recycling_steps` (default 3 for Boltz-2, 10
  elsewhere), `sampling_steps` (default 100 for ESMFold-2, 50 for RoseTTAFold3,
  200 elsewhere), `diffusion_samples`, `output_format` (`cif` or `pdb`).
- `GET /v1/models` lists every model, protocol, parameter, and the current limits.

## Design binders (BoltzGen)

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"nanobody-anything","spec":"<YAML design spec>","params":{"num_designs":10}}'
```

Protocols: `protein-anything`, `peptide-anything`, `nanobody-anything`,
`antibody-anything`, `protein-small_molecule`, `protein-redesign`. Poll the same
way; `/v1/jobs/{id}/results` returns the ranked designs. BoltzGen is from MIT.
The target can be a protein, a small molecule, DNA or RNA, up to 1024 residues.

The protocol implies the design model. An explicit `model` of `boltzgen`, `rfd3`
or `pxdesign` is accepted but has to match it.

## Design against a structure (RFdiffusion3)

Same endpoint, but the input is a pasted **structure** plus a **contig** (what
stays fixed against what gets designed). RFdiffusion3 is from the Institute for
Protein Design:

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"rfd3-binder","structure":"<the contents of your PDB or mmCIF file>","contig":"A1-150,60-80","params":{"num_designs":4,"num_timesteps":100}}'
```

- Contig grammar: `A1-150` keeps target chain A residues 1-150 fixed; a bare
  number designs that many new residues. `"A1-150,60-80"` keeps the 150-residue
  target and designs a 60 to 80 residue binder.
- Protocols: `rfd3-binder` (protein target), `rfd3-scaffold` (build around a
  fixed motif), `rfd3-na-binder` (DNA/RNA target).
- Caps: 5 designs, 200 timesteps, structure 700k chars, and 490 residues for the
  contig's motif plus designed regions. The contig, not the paste, sets the run
  size, so a large structure is fine if the contig selects less of it.
- Results are **unranked** mmCIFs, with no refold or filter scores. To
  sanity-check a design, refold its sequence against the target with
  `POST /v1/predictions` and read `iptm`.

## Design backbones against a structure (PXDesign)

Same endpoint again. PXDesign, from ByteDance, conditions on a distogram of the
target chains you name and returns binder backbones: coordinates only, no
sequence, no ranking, no confidence. It is the fastest of the three; run the
backbones through a sequence-design tool before ordering anything.

```bash
curl -s -X POST $BASE/v1/designs -H 'Content-Type: application/json' \
  -d '{"protocol":"pxdesign-binder","structure":"<the contents of your PDB or mmCIF file>","chains":"A","binder_length":80,"hotspots":"A74,A75,A76","params":{"num_designs":4,"n_step":200}}'
```

- `chains` names the target chains to condition on; `binder_length` is the
  binder's residue count (8 to 200); `hotspots` is optional, target residues to
  aim at.
- One protocol: `pxdesign-binder`. The named chains plus the binder have to come
  to 768 residues or fewer; chains you don't name aren't counted.
- Caps: 8 designs, 400 steps.

## Embed sequences (ESMC / SaProt)

Turn protein sequences into language-model vectors, with no structure and no MSA.
Same submit, poll, download flow:

```bash
curl -s -X POST $BASE/v1/embeddings -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","sequence":"MKTAYIAKQRQISFVKSHFSRQLEE"}'
# or many at once: "sequences":[{"id":"a","sequence":"..."},{"id":"b","sequence":"..."}]
```

- **Models:** `esmc-300m`, `esmc-600m` (default) and `esmc-6b` from Biohub,
  `saprot-650m` and `saprot-1.3b` from Westlake University. Larger is a stronger
  representation at more compute. SaProt is structure-aware but runs
  sequence-only here, which the 1.3B variant is trained for.
- `params`: `pool` (`mean`/`max`/`cls`, default `mean`), `format` (`npz` = per-residue
  `[L, d_model]` + pooled `[d_model]` per sequence; `parquet` = pooled table only), `fast`.
- Caps: 50 sequences per submission, 2000 residues per sequence, 1968 for
  `esmc-6b`.
- `d_model` is 960 / 1152 / 2560 for ESMC 300M / 600M / 6B and 1280 for both
  SaProt sizes (1.3B is deeper than 650M, not wider).
- Results carry `kind: "embed"`, `d_model`, a `sequences` list and `artifacts` URLs.
  Download the `.npz`/`.parquet` files into a named dir and tell the user the path.

## Reading results

`GET /v1/jobs/{id}/results` gives `ready`, an `artifacts` list (each with a `url`),
and for a prediction per-target `rows`; for a design, the `designs`. A Boltz-2
row leads with `confidence_score` and carries `complex_plddt`, `ptm`, `iptm`,
`complex_pde` and, on an affinity run, `affinity_pred_value` (predicted
log10(IC50) in μM, so lower binds harder) and `affinity_probability_binary`
(P(binder), 0 to 1). An ESMFold-2 row is shorter: `plddt`, `ptm`, `n_residues`,
`n_chains`. Pass lines mirror Boltz-2: fold `complex_plddt` > 0.7, interface
`iptm` > 0.5. ipTM scores an interface, so a single-chain fold reports `0.0`
rather than omitting it. Download a single structure from its artifact `url`, or
the whole bundle from `…/archive`, into a named local directory, and tell the
user the absolute path where you saved it.

## Limits and notes

- Free public demo caps, the same as the web app: **10 chains and ligands per
  complex, 10 structures per run, 10 designs per BoltzGen request, 5 for
  RFdiffusion3, 8 for PXDesign**, plus the per-model residue ceilings above.
  Over a cap you get `400`. Numeric params are clamped to range.
- Concurrency: **3 active jobs and 12 submits per minute per caller**. Past
  either you get `429` with a `Retry-After` header; honour it and retry rather
  than resubmitting immediately.
- Errors are RFC 9457 problem+json (`title`, `detail`), except for a `413` (body
  over 8 MiB), a `405` from the wrong HTTP method and a `404` on a path that does
  not exist.
- No key needed. An optional `Authorization: Bearer <key>` moves job ownership
  off your IP address and unlocks `GET /v1/jobs`, the one endpoint that will not
  answer keyless. It raises no limit.
- Full machine-readable contract: `GET /v1/openapi.json`.
- **If a request ever returns HTTP `403` with Cloudflare error `1010`**, that is
  edge-level bot filtering of your HTTP client, not an API error. Retry the same
  request with a browser-like `User-Agent` header
  (`Mozilla/5.0 … Chrome/124.0 Safari/537.36`).

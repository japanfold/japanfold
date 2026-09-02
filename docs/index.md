<div class="jf-hero" markdown>
<p class="jf-eyebrow">API documentation</p>

# JapanFold API

<p class="jf-lede">
Fold proteins, co-fold complexes with ligands (and get binding affinity),
design de novo binders, and compute protein embeddings, over a <strong>free,
public, keyless HTTP API</strong>. Boltz-2, ESMFold-2, Protenix-v2, OpenFold3,
OpenBind-0, RoseTTAFold3 and OpenDDE for structure prediction, BoltzGen, RFdiffusion3 and
PXDesign for binder design, ESMC and SaProt for embeddings. No API key, no local
GPU, nothing to install.
</p>

<div class="jf-pills">
  <div class="jf-endpoint"><span class="jf-key">Base URL</span> https://api.japanfold.aiand.com</div>
  <div class="jf-endpoint"><span class="jf-key">Contract</span> <a href="https://api.japanfold.aiand.com/v1/openapi.json">/v1/openapi.json</a> <span class="jf-key">OpenAPI 3.1, the source of truth</span></div>
</div>

<div class="jf-cta" markdown>
[Fold your first protein](#fold-your-first-protein-in-3-calls){ .md-button .md-button--primary }
[API reference](api.html){ .md-button }
[Try it in the browser](https://workbench.japanfold.aiand.com){ .md-button }
</div>
</div>

## The model: submit → poll → download

Everything is an async job. Submit work, get back a job `id`, poll until the
`status` is terminal, then download the results.

```
POST /v1/predictions   or   POST /v1/designs      →  { "id": "...", "status": "running", ... }
GET  /v1/jobs/{id}                                 →  poll until status is succeeded/failed/canceled
GET  /v1/jobs/{id}/results                          →  scores + a list of downloadable artifacts
GET  /v1/jobs/{id}/archive                          →  everything as one .zip
```

Statuses: `queued` → `running` → `succeeded` | `failed` | `canceled`.

> The service keeps the 1000 most recent jobs and has no expiry timer. Download
> what you want to keep; see [Retention](jobs.md#retention).

## Fold your first protein in 3 calls

```bash
BASE=https://api.japanfold.aiand.com

# 1. submit: a bare `sequence` is the simplest input
JOB=$(curl -s -X POST $BASE/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"myprotein","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["id"])')

# 2. poll until done. `Prefer: wait=60` blocks up to 60s so it often returns finished.
curl -s -H 'Prefer: wait=60' $BASE/v1/jobs/$JOB

# 3. once status=succeeded: read scores, then download the structures
curl -s $BASE/v1/jobs/$JOB/results
curl -s $BASE/v1/jobs/$JOB/archive -o myprotein.zip && unzip -oq myprotein.zip -d myprotein
```

Complexes, ligands, affinity, binder design and parameter choice are all
variations on these three calls.

## Where to go next

<div class="grid cards" markdown>

-   :material-key-variant:{ .lg .middle } **[Authentication](authentication.md)**

    Keyless by default; an optional Bearer key scopes jobs to you instead of your IP.

-   :material-cube-outline:{ .lg .middle } **[Predictions](predictions.md)**

    Input shapes, models, co-folding, affinity, params.

-   :material-shape-outline:{ .lg .middle } **[Designs](designs.md)**

    BoltzGen, RFdiffusion3 and PXDesign binder design.

-   :material-vector-arrange-below:{ .lg .middle } **[Embeddings](embeddings.md)**

    ESMC and SaProt protein embeddings (per-residue + pooled).

-   :material-clipboard-list-outline:{ .lg .middle } **[Jobs](jobs.md)**

    Polling, listing, cancel/delete, results, logs, artifacts, archive.

-   :material-tune:{ .lg .middle } **[Models & limits](models-and-limits.md)**

    The model list, every parameter, and the caps.

-   :material-target:{ .lg .middle } **[Accuracy](accuracy.md)**

    Output parity against each model's reference implementation.

-   :material-alert-circle-outline:{ .lg .middle } **[Errors](errors.md)**

    The problem+json shape and the status codes.

-   :material-console:{ .lg .middle } **[Examples](examples.md)**

    End-to-end fold, co-fold+affinity and design in curl and Python.

-   :material-robot-outline:{ .lg .middle } **[The JapanFold skill](skill.md)**

    Fold and design straight from your AI agent.

</div>

## Network egress

If your environment sandboxes outbound HTTP, allow the host
`api.japanfold.aiand.com`. A `403` citing Cloudflare error `1010` is edge bot-filtering
of your HTTP client, not an API error: retry with a browser-like `User-Agent`.

<div class="jf-credits">
  <span>Powered by</span>
  <img class="jf-aiand" src="assets/aiand-logo.svg" alt="ai&amp;" />
  <span>on</span>
  <img src="assets/tenstorrent-logo.svg" alt="Tenstorrent" />
</div>

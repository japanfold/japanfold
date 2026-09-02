# Jobs

Predictions, designs and embeddings all create a **job**. This is how you track
it, read its output, and clean it up. Reading and controlling a job is keyless:
the id the submit returned is the credential. Only `GET /v1/jobs`, the list
endpoint, needs a key (see [Authentication](authentication.md)).

## The Job object

```json
{
  "object": "job",
  "id": "9daefed38ec5b505b80f76f9a9e87323",
  "kind": "predict",              // "predict" | "design" | "embed"
  "status": "running",            // queued | running | succeeded | failed | canceled
  "name": "myprotein",
  "model": "boltz2",              // null for designs
  "protocol": null,               // set for designs
  "progress": 0.5,                // 0..1, or null when indeterminate
  "stage": "trunk",               // human-readable current step
  "done": 0, "total": 1,          // sub-units completed / total
  "params": {},                   // the params the job was submitted with
  "waiting_for_device": true,     // present only while nothing is free to run it
  "queue": { "waiting": 3, "waiting_this_job": 1,
             "chips_busy": 8, "chips_online": 8 },
  "error": null,                  // set when status=failed
  "created_at": "2026-09-02T20:12:08Z",
  "started_at":  "...",
  "finished_at": null,
  "results_ready": false,
  "links": { "self": "...", "results": "...", "archive": "...", "logs": "..." }
}
```

`waiting_for_device` and `queue` appear together, and only while the job has
been accepted but no processor is free for it yet. That job is queued, not
stalled. There is deliberately no position and no ETA: the scheduler is fair
rather than first-in-first-out, and a fold may still have to load its model's
weights.

## Poll one job

```bash
curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB
```

Add `Prefer: wait=<seconds>` to block until the job finishes instead of returning
immediately (see
[Predictions](predictions.md#waiting-inline-the-prefer-wait-header)):

```bash
curl -s -H 'Prefer: wait=60' https://api.japanfold.aiand.com/v1/jobs/$JOB
```

A simple poll loop:

```bash
until curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB \
  | grep -qE '"status":"(succeeded|failed|canceled)"'; do sleep 5; done
```

## List your jobs

`GET /v1/jobs` is the one endpoint that requires a key. Without one a caller is
identified only by its IP address, so everyone behind an office or a VPN would
share one job list; keyless callers get a `401` here and poll by id instead.

```bash
curl -s -H "Authorization: Bearer $JAPANFOLD_API_KEY" \
  "https://api.japanfold.aiand.com/v1/jobs?limit=20"
```

```json
{ "object": "list", "data": [ { "object": "job", ... } ],
  "has_more": false, "next_cursor": null }
```

Paginated cursor-style: `limit` defaults to 20 and clamps to 100, and
`next_cursor` goes back as `?cursor=...` for the next page.

## Cancel, delete

```bash
curl -s -X POST   https://api.japanfold.aiand.com/v1/jobs/$JOB/cancel   # stop a queued/running job
curl -s -X DELETE https://api.japanfold.aiand.com/v1/jobs/$JOB          # delete the job and its data
```

## Results

Once `results_ready` (or `status` is `succeeded`), read the scores and the list
of downloadable artifacts:

```bash
curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB/results
```

Ask earlier and you get `{"object":"results","job_id":"...","ready":false,"status":"running"}`,
so `ready` is safe to poll on directly.

A **prediction** result, from the 33-residue Boltz-2 fold in the
[quickstart](index.md#fold-your-first-protein-in-3-calls):

```json
{
  "object": "results",
  "job_id": "...",
  "kind": "predict",
  "ready": true,
  "rows": [
    { "id": "target_1", "status": "ok",
      "confidence_score": 0.827157,
      "complex_plddt": 0.877438, "complex_iplddt": 0.877438,
      "complex_pde": 0.431932, "complex_ipde": 0.0,
      "ptm": 0.626033, "iptm": 0.0,
      "protein_iptm": 0.0, "ligand_iptm": 0.0,
      "chains_ptm": { "0": 0.626033 },
      "pair_chains_iptm": { "0": { "0": 0.626033 } },
      "runtime_s": 8.6 }
  ],
  "artifacts": [
    { "path": "target_1.cif", "target": "target_1", "type": "structure",
      "url": "/v1/jobs/.../artifacts/target_1.cif" }
  ],
  "archive_url": "/v1/jobs/.../archive"
}
```

- `rows`: one row per target. `confidence_score` is the headline number, a
  weighted blend of the others. The rest are the AlphaFold-family metrics:
  `plddt` is predicted local accuracy per residue, `ptm` the predicted TM-score
  for the whole structure, `iptm` the same restricted to a chain-chain
  interface, and `pde` / `ipde` the predicted distance error. All are 0-1;
  higher is better except the error terms. Which fields are present depends on
  the model:
  the block above is Boltz-2's, an ESMFold-2 row instead carries `plddt` (mean,
  0-1), `ptm`, `n_residues`, `n_chains`, `samples` and `msa`, and a Boltz-2
  affinity run adds the affinity fields ([Predictions →
  Affinity](predictions.md#co-folding-with-a-ligand-affinity-boltz-2)).
- A rough read: fold `complex_plddt` > 0.7, interface `iptm` > 0.5. ipTM scores
  an interface, so on a single-chain input it comes back `0.0` rather than
  absent, as above.
- A **design** result carries `designs` instead of `rows`, ranked for BoltzGen
  and unranked for RFdiffusion3 and PXDesign. An **embed** result carries `sequences`; see
  [Embeddings → Results](embeddings.md#results).
- `artifacts[].url` and `archive_url` are paths under the base URL. Prefix them
  with `https://api.japanfold.aiand.com`.

## Download artifacts

One file:

```bash
curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB/artifacts/target_1.cif -o target_1.cif
```

Everything as a zip:

```bash
curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB/archive -o results.zip
unzip -oq results.zip -d results
```

## Logs

Plain-text run log, useful while a job runs or to debug a failure:

```bash
curl -s https://api.japanfold.aiand.com/v1/jobs/$JOB/logs
```

## Retention

The service keeps the 1000 most recent jobs. Past that the busiest caller's
oldest job is evicted first, so someone else's burst cannot wipe your results,
and an active job is never evicted. There is no expiry timer. A deleted or
evicted id returns `404`. Download what you want to keep.

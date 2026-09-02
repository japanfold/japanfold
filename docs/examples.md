# Examples

One worked example per capability, in curl and Python. All of them are the same
async flow: submit, poll, download. Parameter detail lives on the endpoint pages
([Predictions](predictions.md), [Designs](designs.md),
[Embeddings](embeddings.md)).

## Fold a protein

```bash
BASE=https://api.japanfold.aiand.com

JOB=$(curl -s -X POST $BASE/v1/predictions \
  -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"myprotein","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["id"])')

# poll (Prefer: wait blocks up to 60s per call)
until curl -s -H 'Prefer: wait=60' $BASE/v1/jobs/$JOB \
  | grep -qE '"status":"(succeeded|failed|canceled)"'; do :; done

curl -s $BASE/v1/jobs/$JOB/results
curl -s $BASE/v1/jobs/$JOB/archive -o myprotein.zip && unzip -oq myprotein.zip -d myprotein
```

The same thing in Python, stdlib only:

```python
import json, time, urllib.request

BASE = "https://api.japanfold.aiand.com"
# The edge blocks urllib's default User-Agent as a bot, so send a browser-like one.
HEADERS = {"Content-Type": "application/json", "User-Agent": "Mozilla/5.0"}

def api(method, path, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(BASE + path, data=data, method=method, headers=HEADERS)
    with urllib.request.urlopen(req) as r:
        return json.load(r)

def wait(job_id):
    while True:
        job = api("GET", f"/v1/jobs/{job_id}")
        if job["status"] in ("succeeded", "failed", "canceled"):
            return job
        time.sleep(5)

job = api("POST", "/v1/predictions",
          {"model": "boltz2", "name": "myprotein",
           "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"})
job = wait(job["id"])
assert job["status"] == "succeeded", job.get("error")

results = api("GET", f"/v1/jobs/{job['id']}/results")
for row in results["rows"]:
    print(row["id"], "plddt=", row.get("plddt"), "ptm=", row.get("ptm"))

# urlretrieve takes no headers, so open the archive directly
req = urllib.request.Request(BASE + results["archive_url"], headers=HEADERS)
with urllib.request.urlopen(req) as r, open("myprotein.zip", "wb") as f:
    f.write(r.read())
```

## Co-fold a protein + ligand, with affinity

Only Boltz-2 does affinity. Pass the complex as a Boltz YAML `input` with a
`ligand` chain and a `properties: affinity` block naming the binder.

```python
import time, httpx

BASE = "https://api.japanfold.aiand.com"
YAML = (
    "sequences:\n"
    "  - protein: {id: A, sequence: MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ}\n"
    "  - ligand: {id: L, smiles: \"CC(=O)Oc1ccccc1C(=O)O\"}\n"
    "properties:\n"
    "  - affinity: {binder: L}\n"
)

with httpx.Client(base_url=BASE, timeout=180) as c:
    job = c.post("/v1/predictions", json={
        "model": "boltz2", "name": "prot-ligand",
        "input": YAML, "params": {"use_msa_server": True}}).json()
    while job["status"] not in ("succeeded", "failed", "canceled"):
        time.sleep(5)
        job = c.get(f"/v1/jobs/{job['id']}").json()
    print(c.get(f"/v1/jobs/{job['id']}/results").json()["rows"])
```

The result `rows` carry affinity fields alongside the structure and confidence
scores.

## Design binders (BoltzGen)

`POST /v1/designs` with a protocol and a YAML `spec`. Results carry ranked
`designs`.

```python
import time, httpx

BASE = "https://api.japanfold.aiand.com"

with httpx.Client(base_url=BASE, timeout=300) as c:
    job = c.post("/v1/designs", json={
        "protocol": "nanobody-anything", "name": "my-nanobodies",
        "spec": "sequences:\n  - protein: {id: A, sequence: MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ}\n",
        "params": {"num_designs": 10, "budget": 10, "fast": True}}).json()
    while job["status"] not in ("succeeded", "failed", "canceled"):
        time.sleep(10)
        job = c.get(f"/v1/jobs/{job['id']}").json()
    results = c.get(f"/v1/jobs/{job['id']}/results").json()
    print(results.get("designs"))
    with open("designs.zip", "wb") as f:
        f.write(c.get(results["archive_url"]).content)
```

## Design against a structure (RFdiffusion3)

Same endpoint, but the input is a pasted `structure` plus a `contig` saying what
stays fixed and what gets designed. Results are unranked mmCIFs.

```python
import time, httpx

BASE = "https://api.japanfold.aiand.com"
structure = open("target.pdb").read()  # chain A holds the target

with httpx.Client(base_url=BASE, timeout=300) as c:
    job = c.post("/v1/designs", json={
        "protocol": "rfd3-binder", "name": "my-rfd3-binders",
        "structure": structure,
        "contig": "A1-150,60-80",  # keep target residues 1-150, design a 60-80 aa binder
        "params": {"num_designs": 4, "num_timesteps": 100}}).json()
    while job["status"] not in ("succeeded", "failed", "canceled"):
        time.sleep(10)
        job = c.get(f"/v1/jobs/{job['id']}").json()
    results = c.get(f"/v1/jobs/{job['id']}/results").json()
    print([d["id"] for d in results.get("designs", [])])
    with open("designs.zip", "wb") as f:
        f.write(c.get(results["archive_url"]).content)
```

## Embed sequences

See [Embeddings](embeddings.md#end-to-end-python) for the ESMC/SaProt example.

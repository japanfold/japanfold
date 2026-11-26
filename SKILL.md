---
name: japanfold
# slot skill.description
description: >-
  Predict 3D biomolecular structures and binding affinity (Boltz-2, ESMFold-2, OpenFold3, OpenBind-0, RoseTTAFold3, OpenDDE, AF2-IG and Mol-1), score a designed
  binder against its target, design de-novo binders and proteins (BoltzGen, RFdiffusion3, PXDesign and RFD4-Proteina), scaffold a
  functional motif around a pasted structure, and compute protein-language-model
  embeddings (ESMC and SaProt) via JapanFold, a hosted, Tenstorrent-accelerated HTTP API in Japan.
  Use to fold a protein or complex, co-fold a protein with a ligand and get affinity,
  design nanobody/antibody/peptide/miniprotein binders against a target, turn a
  sequence into a PDB/mmCIF structure, or get a fixed-size embedding vector for
  search/clustering/ML features, whenever a hosted service is fine. Needs a JapanFold
  API key; no local GPU.
# /slot
license: Apache-2.0
metadata:
  category: biomodels
  provider: JapanFold
  info-url: https://japanfold.aiand.com
# allowed-tools is a Claude Code convenience (grants curl/python without a
# prompt); other harnesses ignore it and use their own execution/permission model.
allowed-tools:
  - Bash(curl *)
  - Bash(python3 *)
---

# JapanFold: hosted structure prediction and binder design

JapanFold runs these models on Tenstorrent hardware behind an HTTP API hosted in Japan:

<!-- slot models.names -->
Structures: Boltz-2, ESMFold-2, OpenFold3, OpenBind-0, RoseTTAFold3, OpenDDE, AF2-IG and Mol-1. Design: BoltzGen, RFdiffusion3, PXDesign and RFD4-Proteina. Embeddings: ESMC and SaProt.
<!-- /slot -->

You call it as an async job
(**submit → wait → download**) over plain HTTPS against
`https://api.japanfold.aiand.com`. No model to install and no local GPU.

**Key.** Every call needs one. Look in your environment before anything else:
`[ -n "$JAPANFOLD_API_KEY" ] && echo set`. If it is set, it is the user's key,
so use it (and `JAPANFOLD_BASE_URL` if that is set too). Only if it is unset,
send the user to `https://japanfold.aiand.com/account/token`: they sign in with
an email address and the page shows a key, and a new account comes with $100.
Never ask for the key in the chat. The user sets it in their own shell and
restarts you from there; a key pasted into a chat goes to the model provider.
For the same reason never print it (test it with `[ -n ... ]`, not `echo`) and
pass it as `$JAPANFOLD_API_KEY` rather than writing its value into a command.
Work is billed per processor hour of chip time, taken from that balance;
`GET /v1/credits` returns the balance, the rate and `billing_url`, the page where
a person adds funds. An organisation on monthly invoicing has `"pay": "invoice"`
there instead of topping up; `GET /v1/billing` gives its spending limit, what is
left, this month's usage and its invoices with pay links. Anything the user would
email us about (monthly invoicing, a higher limit, a PO number, a volume price,
credit, a question) is one call: `POST /v1/requests`, for example
`{"kind": "terms", "limit_usd": 30000}` or
`{"kind": "question", "topic": "capacity"}`; the answer arrives by email and on the
account page. Switching to monthly invoicing means accepting ai&'s terms, so it is
for a signed-in person on the account page: an API key gets `403 needs_person`
with the link, which you pass to the user. When the user wants a colleague on
the same account (one balance, one invoice), send them to People on the account
page (`https://japanfold.aiand.com/account/people`): adding someone needs a signed-in person, and
`GET /v1/account` lists who is on it as `members`. Asked what a job will cost, read `typical_cost` on the
model or protocol in `GET /v1/models`. For a prediction, multiply
`usd_per_target_p90` by the number of targets that will be submitted: each target
runs on its own chip, and that per-target figure is what past jobs (last 7 days)
cost at most of the time. The per-job `usd_p50`/`usd_p90` follow whatever batch
sizes users sent that week, so do not quote them as the price of a prediction of
another size. A design or embedding job splits its own work across chips and has
per-job figures only; quote those as what a typical job cost. Say it is history,
not a quote. Do not derive a ceiling from the rate and the runtime limit.
What was spent in a period
is `GET /v1/usage?from=YYYY-MM-DD&to=YYYY-MM-DD&limit=1`: `summary.amount` is
the total in US dollars (`summary.currency`), and `summary.by_model` splits it
per model. Quote `amount`; every `*_micros` field is millionths of a dollar.

Works from any agent/harness: use `curl` (Bash) or your language's HTTP client
(`httpx`/`requests`, `fetch`, `net/http`, …), whatever your environment has.
A terminal can also use the CLI: `pip install japanfold`, then `japanfold --help`
(same key, `--json` on every command, `wait JOB_ID` blocks until a job ends).
Many separate proteins from the CLI are one job too: `japanfold fold proteins.fasta --each --wait`
folds every FASTA record as its own target (japanfold >= 1.1.0).
If your environment sandboxes network egress, the host to allow is
**`api.japanfold.aiand.com`**; every result file comes from it. In Claude Science, call
`request_network_access` for that host before the first request. Claude Science has no
credential slot for a JapanFold key, so there the MCP server below is the way in: it
signs in through the browser and needs no key.

**MCP.** The same API is an MCP server at `https://api.japanfold.aiand.com/mcp`.
If the `japanfold` MCP tools (`create_job`, `get_job`, `get_results`, ...)
are available, use them instead of curl: `create_job` takes the same
`{model, input, params}` as `POST /v1/jobs` below. Follow a job with `get_job` and `wait=100`: one call holds up to
100 s for it to end. A succeeded `get_job` carries `download`, one shell command that
fetches every result file; run it rather than calling `get_artifact` per file. To add the server to Claude Code:
`claude mcp add --transport http japanfold https://api.japanfold.aiand.com/mcp --header "Authorization: Bearer $JAPANFOLD_API_KEY"`.
claude.ai / Claude Desktop / Claude Science / ChatGPT: add `https://api.japanfold.aiand.com/mcp` as a connector and sign in, no key needed.
Setup for every Claude surface (claude.ai, Claude Code, Claude Science, the Claude API) is at
https://japanfold.aiand.com/docs/claude/.

## Predict a structure

Submit → wait until `status` is terminal → read results:

```bash
BASE=${JAPANFOLD_BASE_URL:-https://api.japanfold.aiand.com}
H=(-H 'X-JapanFold-Client: skill' -H "Authorization: Bearer $JAPANFOLD_API_KEY")
# 1. submit {model, input, params}: one shape for every model. `input` is what the model
#    reads: a bare sequence, one FASTA/YAML string, or a list of them (one target each).
#    To retry a submit that timed out, resend it with the same `Idempotency-Key: <unique>`
#    header: you get the first job back instead of a second one to pay for. A key names
#    its job for 24 h. Generate each new submission's key, do not write one yourself:
#    `python3 -c 'import uuid; print(uuid.uuid4())'` (uuidgen is missing on many hosts).
JOB=$(curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"boltz2","name":"mytarget","input":"MKTAYIAKQRQISFVKSHFSRQLEE"}' \
  | python3 -c 'import sys,json; r=json.load(sys.stdin); print(r["id"]) if "id" in r else sys.exit(r["detail"])')
: "${JOB:?submit failed, see the message above}"   # a script stops here instead of polling an empty job id

# 2. wait. `?wait=60` holds the GET until the job finishes or 60 s pass, so loop on it
#    with no sleep: each call returns the moment the job is done. The first job on a
#    processor also loads the model's weights. When the service is busy the job carries
#    `queue`: its `reason`, how many `accounts_ahead` and the fleet's chip counts. That
#    is a real wait, not a stall, so keep waiting. It has no time estimate, so do not
#    invent one. A design job runs for minutes, longer than many agent shells let one
#    command live (Claude Code stops it at 2 min): there, run this loop in the
#    background or make one `?wait=60` call per command, never `sleep` between them.
while :; do
  S=$(curl -s "${H[@]}" "$BASE/v1/jobs/$JOB?wait=60" | tee job.json \
      | python3 -c 'import sys,json; print(json.load(sys.stdin).get("status"))')
  case $S in queued|running) ;; *) break ;; esac
done
cat job.json                                  # -> {"status":"succeeded|failed|canceled", ...}

# 3. once status=succeeded: scores + artifact URLs, then download the bundle.
#    A succeeded job that carries `partial` delivered less than it was asked for (a design
#    shard or a target failed): tell the user `partial.delivered` of `partial.requested`.
curl -s "${H[@]}" $BASE/v1/jobs/$JOB/results
curl -fsLOJ "${H[@]}" $BASE/v1/jobs/$JOB/archive        # zip: structures + results.json
curl -fsL "${H[@]}" -o "$F" "$BASE/v1/jobs/$JOB/artifacts/$F"  # one file named in results; -L follows to storage

# Past jobs, newest first; ?status=failed lists only failures, each with its `error`.
curl -s "${H[@]}" "$BASE/v1/jobs?status=failed&limit=20"
# What a job was submitted with (sequences, structures, contigs), beside its `error`.
curl -s "${H[@]}" "$BASE/v1/jobs/$JOB/input"
```

**Multi-chain complexes** (e.g. insulin's A+B chains) go in the `input` YAML,
one `protein` entry per chain, not a bare sequence. A FASTA in `input` works
too, on every model, with any headers (`>insulin_a`, `>heavy chain`, a UniProt
`>sp|P01308|INS_HUMAN ...`): a name the model can't hold becomes A, B, ... and the job's
`warnings` say which record became which chain.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' -d '{
  "model":"boltz2","name":"human-insulin",
  "input":"sequences:\n  - protein: {id: A, sequence: GIVEQCCTSICSLYQLENYCN}\n  - protein: {id: B, sequence: FVNQHLCGSHLVEALYLVCGERGFFYTPKT}\n"
}'
```

**Many separate proteins** (a screen, a list of candidates) go in ONE job as an `input`
list, one entry each. Records inside a single `input` string are chains of one complex, so never
paste a screen there. Each target gets its own structure, `<name>.pdb` with
`output_format: pdb` (the default is mmCIF), and its own row in `results`, so one job
covers the whole set:

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' -d '{
  "model":"esmfold2-fast","name":"screen","params":{"output_format":"pdb"},
  "input":[{"name":"cand01","content":">cand01\nMQIFVKTLTGKTITLEVEPSDTIENVKAKIQDKEG"},
             {"name":"cand02","content":">cand02\nTITLEVEPSDTIENVKAKIQDKEGIPPDQQRLIF"}]
}'
```

Python equivalent (notebooks, scripts):

From Python, the `japanfold` package (stdlib only) runs the same loop:

<!-- slot snippet.quickstart -->
```bash
pip install japanfold
export JAPANFOLD_API_KEY=jf_live_...      # from https://japanfold.aiand.com/account/token
```

```python
import japanfold as jf

r = jf.run("boltz2", "complex.yaml", diffusion_samples=5)   # blocks until done
r.path      # this input's files, under the model's own file names
r.metrics   # one dict per ranked output, best first
r.cost      # dollars billed for the job

job = jf.submit("esmfold2", "MKTAYIAKQRQISFVKSHFSRQ")       # returns once admitted
print(job.id)                                              # jf.Job(id).result() later
r = job.result()

for r in jf.map("boltz2", ["a.yaml", "b.yaml"], max_parallel=4):   # in input order
    print(r.error or r.path)
```

That is the whole API: `run`, `submit`, `map`, `Job`, `Result` and `Error`. The client knows no
model: any id `japanfold models` lists works, and every keyword other than `name`, `out`,
`timeout` and `max_parallel` is that model's parameter, sent as given. The input is a file path,
the text itself (FASTA, YAML, a sequence, a PDB/mmCIF, `pdb:1ABC`) or a dict sent as YAML; a
path-shaped string that names no file raises `FileNotFoundError` instead of being billed.

Results land in `./japanfold/<name>/` (`out=` changes the parent), where the name is `name=`,
else the input file's stem, else the job id. Running the same call again reads that directory
back without a request, and a resubmit after a crash gets the same job back rather than a second
bill. `map` returns a failed or refused input as `r.error` and carries on; a 401 or 402 raises at once,
and leaving the loop early cancels the jobs still running. `run` and `job.result()` raise
`jf.Error`. Progress goes to the `japanfold` logger at INFO.

`japanfold run MODEL INPUT --param key=value` is the same call from the shell.
<!-- /slot -->

The models (`model`), as `GET /v1/models` lists them:

<!-- slot models.predict -->
- `boltz2` (Boltz-2, up to 1920 residues, MSA always): Folds protein, DNA, RNA and ligand complexes and predicts binding affinity. Uses a multiple-sequence alignment (MSA) for best accuracy. The most capable model: choose it when in doubt, or whenever you need ligands, nucleic acids, affinity or constraints.
- `esmfold2` (ESMFold-2, up to 1664 residues, MSA on by default): Language-model folding, no MSA required, though it will use one if supplied for extra accuracy. Co-folds DNA, RNA and ligand chains with at least one protein chain; it predicts no binding affinity. A quick, lightweight choice for a protein structure or a small cocrystal.
- `esmfold2-fast` (ESMFold-2 Fast, up to 1664 residues, MSA never): ESMFold-2 tuned for maximum throughput: always single-sequence, and it co-folds ligands and nucleic acids like the standard checkpoint. Accuracy is typically very close to ESMFold-2, and it is the model to reach for when screening thousands of sequences at once.
- `openfold3` (OpenFold3, up to 1664 residues, MSA on by default): The OpenFold Consortium's open AlphaFold3 reproduction, running fully on-device. Folds multi-chain complexes of proteins, RNA and DNA using a multiple-sequence alignment by default (turn it off to fold single-sequence). No ligands and no affinity: for ligands on this same stack use OpenBind-0, for affinity Boltz-2. The published weights are a preview checkpoint trained to 155k steps, well short of the full AlphaFold3 schedule, so read the confidence scores before trusting a prediction.
- `openbind` (OpenBind-0, up to 1664 residues, MSA on by default): The OpenFold Consortium's OpenBind-0 checkpoint on the OpenFold3 architecture, running fully on-device. Unlike the OpenFold3 preview weights it co-folds small-molecule ligands alongside proteins, RNA and DNA, using a multiple-sequence alignment by default. It predicts structure and confidence, not binding affinity. For affinity use Boltz-2.
- `rf3` (RoseTTAFold3, each chain 4 residues or more, up to 1600 residues, MSA on by default): The Baker lab's RoseTTAFold3, running fully on-device. Folds multi-chain complexes of proteins, RNA, DNA and small-molecule ligands, using a multiple-sequence alignment by default (turn it off to fold single-sequence). Structure only: no binding affinity, and no covalent modifications or binding constraints from this input format. For those use Boltz-2.
- `opendde` (OpenDDE · General, up to 1536 residues, MSA on by default): OpenDDE with its general protein-complex checkpoint. Co-folds multi-chain protein complexes on the Protenix-v2 stack plus a structural-token expander, using an MSA by default. Protein-only (no ligands, nucleic acids or affinity). For binding affinity, use Boltz-2; for ligands or DNA/RNA, use Boltz-2.
- `opendde-abag` (OpenDDE · Antibody-Antigen, up to 1536 residues, MSA on by default): OpenDDE with its antibody-antigen checkpoint selected. Co-folds an antibody (Fab heavy/light) with its antigen on the Protenix-v2 stack, using an MSA by default. Protein-only, no affinity. Accuracy is verified to match the reference OpenDDE implementation: strong on standard antibody-antigen complexes, and it shares the reference's own limitation on some hard targets (a checkpoint characteristic, not a port defect), so don't expect uniformly high accuracy on every antibody-antigen input.
- `af2ig` (AF2-IG, up to 1024 residues, MSA never): AlphaFold2 initial-guess, the filter a binder-design pipeline uses to tell a real design from a plausible one. Give it a design you already have, meaning a structure carrying the target chain and the binder backbone plus the binder's sequence. It re-predicts the complex from those coordinates and reports ipTM and interface pAE for the interface. Paste it as `target:` (with the structure inline) and `binder:`; a chain list is a different input and it is refused. No alignment and no sampling: the same input gives the same answer. It runs Google DeepMind's AlphaFold2 parameters (params_model_1_ptm, 2022-12-06 release), licensed under CC BY 4.0.
- `mol1` (Mol-1, up to 1024 residues, MSA on by default): Mol-1, model by Molence. Predicts the 3D structure of complexes of proteins, DNA, RNA and small molecules (CCD codes or SMILES). It is derived from Protenix with a wider pair track and runs fully on-device. Uses a multiple-sequence alignment by default (turn it off to fold single-sequence). Reports pLDDT, pTM, ipTM and PAE. For binding affinity, use Boltz-2.
<!-- /slot -->

Prediction `params`; not every model takes every one, so read `caps` in `GET /v1/models`. A param
the model cannot honour comes back as a 400 naming both. A model whose MSA is `always` puts a
`use_msa_server: false` back to `true` and says so in the job's `warnings`. Leave
`recycling_steps` and `sampling_steps` out unless you mean to override a model's own value;
`recycling_steps` counts recycles after a first trunk pass (N runs N+1 passes), capped per model
at the model's `param_max`.

<!-- slot params.predict -->
| Key | Type | Default | Range | What it does |
|---|---|---|---|---|
| `use_msa_server` | bool | `true` | - | Build a multiple-sequence alignment (MSA) of related sequences for the input. It improves accuracy; each model's card says whether it needs one. |
| `fast` | bool | `false` | - | Higher throughput, may be slightly less accurate. |
| `recycling_steps` | int | model default | 1–10 | Trunk recycles. Leave empty to use each model's recommended value (Boltz-2, OpenFold3, OpenBind-0 and AF2-IG 3, Mol-1 20, the others 10). More can improve accuracy, at the cost of speed. |
| `sampling_steps` | int | model default | 10–500 | Diffusion steps per structure. Leave empty to use each model's recommended value (ESMFold-2 100, RoseTTAFold3 50, the others 200). |
| `diffusion_samples` | int | `1` | 1–5 | How many structures to generate per target. |
| `output_format` | enum | `cif` | `cif`, `pdb` | Structure file format. PDB gives the chain id one column, so a longer chain name is rewritten A, B, C... and the originals are listed in a REMARK 999 block at the top of the file. mmCIF has no width limit and keeps the names you submitted. |
| `seed` | int | `0` | 0–2147483647 | Noise seed for the diffusion sampler. The job reports the seed it ran with. |
| `write_pae` | bool | `false` | - | Also return the top-ranked structure's confidence matrices as <name>_pae.npz: the full PAE matrix (key pae, the one Adaptyv's ipSAE pipeline reads), PDE and contact probabilities where the model computes them, with <name>_pae.json naming every array. Each model's list is under confidence in GET /v1/models. |
| `use_tfg_guidance` | bool | `false` | - | Steer sampling with the input's constraint block: contact pairs or epitope residues the antibody must reach. The movable chains are moved as a rigid body during diffusion until the prediction agrees, and physics restraints apply on every step. Guided folds take longer and are billed for the time they run. |
<!-- /slot -->

- **`af2ig` takes a different input from all of the above.** AlphaFold2 initial-guess
  scores a binder you already designed, so instead of a chain list its target `content` is
  a structure plus the binder's sequence:

  ```yaml
  target:
    structure: |
      ATOM      1  N   MET A   1      ...      # target chain AND binder backbone
    chain: A
  binder:
    sequence: SPEDEIQALEEKNAQLKQEIAALEEKI
    chain: B
  ```

  That YAML is the job's `input`, sent like any other:
  `{"model":"af2ig","name":"d1","input":"<the YAML above>"}`.
  The binder chain has to be in the structure and the same length as the sequence: AF2-IG
  re-predicts the complex from those coordinates. The result row carries `plddt`, `ptm`,
  `iptm`, `pae`, `ipae` and `interface_pae` (Angstrom). `iptm` and `interface_pae` are what
  a binder pipeline filters on. No MSA, no sampling, no seed: the same input gives the same
  answer, so `use_msa_server`, `sampling_steps` and `diffusion_samples` are refused for it.
- For complexes / protein–ligand affinity / multiple chains, pass a **Boltz YAML**
  string as `input` (`sequences:` with `protein`/`dna`/`rna`/`ligand` chains;
  `properties:` for the affinity head).
- **Guided antibody docking (OpenDDE).** If the user knows part of an antibody-antigen
  interface, add a top-level `constraint:` block to the YAML and send
  `"use_tfg_guidance": true` (on `opendde` or `opendde-abag` only). The antibody chains move
  as a rigid body during sampling until the prediction agrees. Residues are addressed as in
  OpenDDE: `entity` = 1-based entry of `sequences`, `copy` = 1-based copy in its `id` list,
  `position` = 1-based residue.

  ```yaml
  constraint:
    movable_chains: [H, L]          # the antibody
    contact:                        # antigen residue <-> antibody residue, 3.5-8 A by default
      - {entity1: 3, position1: 248, entity2: 2, position2: 93}
    # or, when only the antigen side is known (pocket mode), instead of contact:
    # epitope: {residues: [{entity: 3, copy: 1, position: 248}], min_fraction: 0.5}
  ```

  A wrong contact pulls the antibody to the wrong place, so tell the user to leave out
  pairs they are unsure of; an epitope with `min_fraction` below 1 tolerates a few wrong
  residues. Guided folds take longer and are billed for the time they run.
- **Guided folding (Mol-1).** If the user knows a binding site, specific contacts or a
  region that must stay clear, add a top-level `constraints:` list to the YAML on `mol1`.
  Any mix of three blocks; residues are `[chain, 1-based position in the sent sequence]`:

  ```yaml
  constraints:
    - site:        # these residues should touch the partner chains (<= 4.5 A)
        source_chains: [A]
        partner_chains: [L]
        residue_ids: [[A, 12], [A, 16]]
    - contact:     # these pairs should touch (<= 4.5 A); or token_pairs: [[i, j]], 0-based tokens
        pairs: [[[A, 71], [B, 149]]]
    - exclusion:   # these residues should stay away from the partner chains (> 6.0 A)
        source_chains: [A]
        partner_chains: [L]
        residue_ids: [[A, 40]]
  ```

  Mol-1 then folds on its constraint weights. It conditions on the request and does not
  enforce it, so check the geometry in the result. A contact the true structure does not
  have gets pulled in anyway and can clash, so tell the user to leave out pairs they are
  unsure of. Billed like any Mol-1 fold.
- **Fast mode is off by default.** Send `"fast": true` for higher throughput; it may be
  slightly less accurate. The workbench turns it on for humans, the API never does it for
  you. ESMFold-2 is the exception: omitted means fast for it, and `"fast": false` runs its
  normal mode.
- `GET /v1/models` lists every model, protocol, parameter, and the current limits.

## Design binders

Pick the design model first: they take different inputs.

<!-- slot models.design -->
- `boltzgen` (BoltzGen, takes a `spec`, up to 1024 residues): Full binder-design pipeline: generate against a protein, small molecule, DNA or RNA target, then inverse-fold, refold, score and filter. You get a ranked top set with confidence metrics, ready to order. Protocols: `protein-anything`, `peptide-anything`, `nanobody-anything`, `antibody-anything`, `protein-small_molecule`, `protein-redesign`. Parameters: `num_designs`, `budget`, `fast`, `seed`.
- `rfd3` (RFdiffusion3, takes a `contig`, up to 1536 residues): Designs directly in 3D around a target structure you provide (PDB/mmCIF plus a contig saying what stays fixed vs. what is designed): binders, motif scaffolds, DNA/RNA binders. Designs come back ordered on whether the output has the chains the contig asked for and how well the kept motif survived, which is a structural order and not a confidence one; refold designs with a Fold job to score them. Designed positions are named by the model's own sequence head, so each design comes back with a sequence. Treat it as a starting point: upstream recommends redesigning it with a tool like ProteinMPNN before ordering. One size caveat: on the target walked to the 1024 ceiling, designs at 768 and 832 total residues came back with a break in the designed backbone, where 640, 704 and 896-1024 did not, so check the designed chain is continuous before using a design in that range. Protocols: `rfd3-binder`, `rfd3-scaffold`, `rfd3-na-binder`. Parameters: `num_designs`, `num_timesteps`, `seed`.
- `pxdesign` (PXDesign, takes a `chains`, up to 1536 residues): Generates binder backbones against a target structure, conditioned on a distogram of the target. Fast, and the output is a backbone only: no sequence, no ranking, no confidence score. Take the coordinates to a sequence-design tool before ordering anything. For a ranked, sequenced binder use BoltzGen. Protocols: `pxdesign-binder`. Parameters: `num_designs`, `n_step`, `seed`.
- `rfd4` (RFD4-Proteina, takes a `contig`, up to 512 residues): Designs all-atom proteins around a target structure you provide (mmCIF plus a contig saying what stays fixed and what is designed): binders, motif scaffolds, antibody and nanobody CDR loops. Every design is made against a target, so the contig always keeps something. A trunk integrates a flow in an autoencoder's latent space and the decoder writes coordinates and a sequence in one step, so each design comes back with a sequence. Designs come back ordered on the kept motif and the chain count, which is a structural order and not a confidence one; refold designs with a Fold job to score them. How much thinking a design gets is a knob (the recipe), not a fixed cost. Protocols: `rfd4-binder`, `rfd4-scaffold`. Parameters: `num_designs`, `recipe`, `nsteps`, `seed`.
<!-- /slot -->

 `GET /v1/models` returns `design_models`, each model's `params`, and
each protocol's `engine`.

**BoltzGen** takes a target described in a YAML spec and returns a ranked,
filtered top set with confidence metrics.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"protein-anything","params":{"num_designs":10},
       "input":"entities:\n  - protein: {id: A, sequence: <target sequence>}\n  - protein: {id: B, sequence: 60..80}\n"}'
```

The spec is a list under `entities:`, not predict's `sequences:`. Each chain is a
`protein`, `dna`, `rna` or `ligand` with an `id`. The chain to design gets a
residue-count range as its sequence (`60..80`); letters such as `XXXX` are read as a
fixed chain, not a design. There is no `binder:`, `hotspots:`, `version:` or pocket
constraint. To aim at an epitope, give the target `binding_types: {binding: '45..52'}`
(1-based residues). A spec the engine would reject is refused at submit with a `400`
that names the problem and shows this shape.

**A target structure can be named instead of pasted.** Every `structure` field (and
AF2-IG's `target: structure:`) takes `pdb:1ABC` (an RCSB entry; a bare 4-character id
works too) or `afdb:P69905` (the AlphaFold DB model of a UniProt accession). The API
fetches the entry's mmCIF and checks it exactly like a paste; the job's `fetched` field
records the id and the sha256 of what ran. An unknown id is a free `400` at submit.
Don't download and paste an entry yourself, and never cut it down to fit a tool call:
paste text only for a structure that is not in the PDB or AlphaFold DB, such as your
own model or an edited file.

**RFdiffusion3** runs all-atom diffusion directly around a target structure you
paste in, with a contig saying what stays fixed and what gets designed. A contig keeps protein and nucleic-acid residues only:
ions, ligands, waters and modified residues like MSE are dropped before the contig
is read, so naming one is a 400 at submit. Keep a zinc finger and you keep the
residues that coordinate the metal, not the metal; rename a modified residue to its
standard parent (MSE to MET) if you need it in the motif.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"rfd3-binder","input":"pdb:<id> or <PDB/mmCIF text>",
       "params":{"contig":"A1-150,60-80","num_designs":4}}'
```

A finished `rfd3-scaffold` or `rfd4-scaffold` job scores and ranks its own designs,
so read `GET /v1/jobs/{id}/results` and sort by `final_rank` instead of downloading
the structures to measure them. Each design carries `motif_rmsd` (the kept motif
against the structure you sent, in Angstrom; RFdiffusion3 copies the kept
coordinates so a held motif reads `0.0`, RFD4-Proteina conditions on them instead
so a held motif reads a few tenths, compare designs within a job), `num_chains` with
`chains_expected` beside it (they differ when the output came back in the wrong
number of pieces; a binder contig legitimately expects two), `designed_length`
and `designed_sequence`. The rank is the distance from `chains_expected`, then
`motif_rmsd`, a structural
sanity order and not a confidence score: neither model produces one. If you need
confidence, refold a design as its own prediction job and read its pLDDT.

**PXDesign** generates binder backbones against a target structure, conditioned
on a distogram of the chains you name.

**It returns a backbone with no sequence:** coordinates only, no ranking, no
confidence score. Every binder residue is written as GLY with only N/CA/C/O,
because that is what the model generates. Take the coordinates to a
sequence-design tool before ordering anything; for a ranked, sequenced binder use
BoltzGen. Each design carries a `fit_rmsd`, the residual of fitting the model's
own reconstruction of the target onto the real target. That number says whether
the conditioning worked.

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"pxdesign-binder","input":"pdb:<id> or <PDB/mmCIF text>",
       "params":{"chains":"A","binder_length":80,"hotspots":"A64,A65,A66","num_designs":4}}'
```

**RFD4-Proteina** also takes a pasted structure plus a contig, and decodes
coordinates and a sequence in one step, so every design comes back sequenced.


```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"rfd4-binder","input":"pdb:<id> or <mmCIF text>",
       "params":{"contig":"60-60, /0, A2-115","num_designs":4}}'
```

**mmCIF only**: a pasted PDB is refused at submit, because the model conditions on
bonds and formal charges a PDB does not reliably carry. Use RFdiffusion3 if you only
have a PDB. The contig reads left to right: a bare number or range designs that many
residues, `/0` starts a new chain, a chain-range like `A2-115` keeps that part of the
structure. Chain and residue ids are the file's own (`label_asym_id` / `label_seq_id`),
which can differ from what a viewer shows; a contig matching no atoms is refused at
submit, as is a kept chain whose `label_seq_id` column is empty (`.`) -- fill it in with
`gemmi convert` or use the mmCIF as RCSB serves it. The contig always keeps something:
RFD4 designs against a structure here, so
`"60-60"` with nothing preserved is refused rather than queued. To redesign an antibody
or nanobody CDR, keep the framework either side of the loop in one run and the antigen
as its own chain: `"A9-40, 8, A50-93, /0, B3-15"` designs 8 CDR residues between kept
framework A9-40 and A50-93, against partner chain B. Two kept ranges of one chain must
not be split by `/0`, which would ask for two output chains with the same name, and they
must not be written side by side either (`"A9-40, A50-93"`): put the residues that link
them in between as a designed segment, as the example does. With more than one kept
chain, declare them in the order the structure lists them, as that example does too:
`"B3-15, 8, /0, A9-40"` is refused, because the model would read chains A and B as one
molecule with nothing bonding them.
`hotspots` is optional. Designs come back scored and ranked like RFdiffusion3's
(`motif_rmsd`, `num_chains` against `chains_expected`, `final_rank`), measured in the
file's own numbering; for a confidence number, refold the ones you like with a
prediction and read `iptm`.

`model` is optional but must match the protocol's engine, else `400`. Wait the
same way; `/v1/jobs/{id}/results` returns the designs.

## Compute protein embeddings

Submit → wait → download, same as predict/design:

```bash
JOB=$(curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' \
  -d '{"model":"esmc-600m","input":"MKTAYIAKQRQISFVKSHFSRQLEE"}' \
  | python3 -c 'import sys,json; r=json.load(sys.stdin); print(r["id"]) if "id" in r else sys.exit(r["detail"])')
: "${JOB:?submit failed, see the message above}"   # a script stops here instead of polling an empty job id

curl -s "${H[@]}" "$BASE/v1/jobs/$JOB?wait=60"     # repeat until status is terminal
curl -s "${H[@]}" $BASE/v1/jobs/$JOB/results       # -> sequences: [{id, length, file}, ...]
curl -fsLOJ "${H[@]}" $BASE/v1/jobs/$JOB/archive   # zip: manifest.json + embeddings
curl -fsL "${H[@]}" -o "$F" "$BASE/v1/jobs/$JOB/artifacts/$F"  # one file, F = a sequences[].file
```

Multiple sequences go in `input` as a list, or as one FASTA/YAML blob:

```bash
curl -s "${H[@]}" -X POST $BASE/v1/jobs -H 'Content-Type: application/json' -d '{
  "model":"esmc-600m",
  "input":[{"id":"gfp","sequence":"MSKGEELFTGVVPILVELDGDVNGHKFSVSGEGEGDAT"},
               {"id":"lysozyme","sequence":"KVFGRCELAAAMKRHGLDNYRGYSLGNWVCAAKFESNF"}]
}'
```

Models:

<!-- slot models.embed -->
- `esmc-300m` (ESMC 300M, vector length 960, up to 2000 residues): The smallest ESMC variant: quickest embeddings, still a strong general-purpose protein representation.
- `esmc-600m` (ESMC 600M, vector length 1152, up to 2000 residues): A larger ESMC trunk with a better representation than 300M, at modest extra cost. The default choice.
- `esmc-6b` (ESMC 6B, vector length 2560, up to 1968 residues): The full-size ESMC model: the strongest embeddings, at higher compute cost per sequence.
- `saprot-650m` (SaProt 650M, vector length 1280, up to 2000 residues): SaProt trained on sequence + structure tokens, run sequence-only here. A strong alternative representation to ESMC, especially for stability/function prediction.
- `saprot-1.3b` (SaProt 1.3B, vector length 1280, up to 2000 residues): The largest SaProt variant. Unlike the smaller ones it is trained to work sequence-only, so nothing is lost without a structure input.
<!-- /slot -->

Embedding `params`:

<!-- slot params.embed -->
| Key | Type | Default | Range | What it does |
|---|---|---|---|---|
| `pool` | enum | `mean` | `mean`, `max`, `cls` | How per-residue embeddings are combined into one fixed-size vector per sequence. |
| `format` | enum | `npz` | `npz`, `parquet` | npz: per-residue + pooled embeddings, one file per sequence. parquet: pooled vectors only, one table. |
| `fast` | bool | `false` | - | Higher throughput, may be slightly less accurate. |
<!-- /slot -->

- Results: `manifest.json` (model/pool/shapes/dtype) plus one `<id>.npz` per
  sequence (or one shared `embeddings.parquet`). No structure, no MSA, only the
  language-model trunk, so these are the cheapest jobs.
- `GET /v1/models` lists the embedding models (with `dim`) and params too.

## Reading results

`GET /v1/jobs/{id}/results` gives `ready`, an `artifacts` list (each with a `url`),
and, for a prediction, per-target `rows` (`confidence_score`, `complex_plddt`,
`iptm`, affinity fields); for a design, the ranked `designs`. Pass lines mirror
Boltz-2: interface `iptm` > 0.5, fold `complex_plddt` > 0.7. Download a single
structure from its artifact `url`, or the whole bundle from `…/archive`. A succeeded
`GET /v1/jobs/{id}` also carries `download`, one shell command for every file.

**Confidence matrices.** With `write_pae: true` each target also has an artifact of type
`pae`, `<name>_pae.npz` (compressed numpy, about 11 MB at 1,024 tokens), and one of type
`pae_sidecar`, `<name>_pae.json`. All arrays are the top-ranked structure's, on one token
axis of length N in the structure's own order (one token per residue or nucleotide, one per
ligand atom). The keys are the open-source tt-bio engine's:

- `pae` [N, N] float32, Å: expected aligned error of token j when aligned on token i. Not
  symmetric. This is the matrix Adaptyv's ipSAE pipeline reads.
- `pde` [N, N] float32, Å: expected distance error, symmetric. Every model but `af2ig`.
- `contact_probs` [N, N] float32: probability that tokens i and j are within
  `contact_cutoff_A` (a scalar in the same file, the model's distogram bin edge just under
  8 Å, 7.71 to 8.00). Symmetric, diagonal 1. Every model.

The sidecar lists the arrays present (`arrays`, with shape, dtype and units) and, under
`absent`, any the model cannot compute and why. Read it rather than assuming a key exists,
and never treat a missing array as zeros. `GET /v1/models` has the same per model under
`confidence`. ipTM is a results-row field, not in the file: for a multi-chain input every
model's row has `iptm`, the chain-pair matrix `pair_chains_iptm` and its diagonal
`chains_ptm`, keyed by chain position (`"0"`, `"1"`; ESMFold-2 keys by chain ID).

```python
import json, numpy as np
side = json.load(open("t1_pae.json")); m = np.load("t1_pae.npz")
pae = m["pae"]; contacts = m["contact_probs"] if "contact_probs" in side["arrays"] else None
```

## Limits & notes

- The one size limit is **each model's residue ceiling per structure**, the
  largest size the engine is measured to fold on this hardware: `max_residues` in the lists
  above and in `GET /v1/models` (AF2-IG counts the target plus the binder). Design is bounded
  the same way, on the contig or the target chains plus binder, and `binder_length` is 8-200.
  Over a cap → `400` naming the model and its ceiling. Numeric params are clamped to range.
- **A model's ceiling is not its track record.** `GET /v1/models` reports both:
  `max_residues` is what will be accepted, `measured_wall` is the largest
  structure that model has actually run on this hardware, and it can be much
  lower. Sizes in between are taken
  and tried on purpose, because a wall that moves with alignment depth or is not
  monotonic in residue count cannot honestly be a ceiling. Such a job is
  accepted with a `warnings` entry saying so, and if it does fail on device it
  comes back `failed` naming the wall and the models that have run that size.
  One job, nothing else affected. Read `measured_wall` if you want to pick a
  model that will finish rather than one that will be accepted.
- `?wait=60` (or the header `Prefer: wait=60`, which also works on a create) is a
  preference, not a guarantee (RFC 7240). One account is held on two jobs at a
  time, so wait on one job at a time, not on a whole batch in parallel. Past that,
  or under heavy load, the request returns the job's current state at once with
  `Retry-After` and a `wait_refused` field saying no hold happened; pause that many
  seconds before the next call. Otherwise never
  `sleep` between waits, the hold already paces the loop. From a shell, `japanfold wait JOB` does the same loop.
  A queued or running job carries this call as `wait.url`, ready to paste.
- Errors are RFC 9457 problem+json (`title`, `detail`).
- **`401` with `sign_in_required: true`** means no key or a bad one. If
  `JAPANFOLD_API_KEY` is set, send it as `Authorization: Bearer $JAPANFOLD_API_KEY`;
  if there is none, or the 401 says it is revoked, send the user to
  `https://japanfold.aiand.com/account/token`.
- **`402` with `out_of_credit: true`** means the balance is spent. Tell the user;
  retrying will not help until they add funds. On a monthly-invoicing account it
  means the spending limit is reached: the `detail` says so, running jobs finish,
  and new runs start once jobs end, an invoice is paid or the limit is raised
  (a person on the account page, or `POST /v1/requests` with
  `{"kind": "terms", "limit_usd": ...}`).
- Full machine-readable contract: `GET /v1/openapi.json`.

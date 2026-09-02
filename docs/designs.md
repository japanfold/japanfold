# Designs

`POST /v1/designs` designs de novo proteins against a target. Three engines share
the endpoint: [BoltzGen](https://github.com/HannesStark/boltzgen) takes a
protein, small-molecule, DNA or RNA target and returns designs **ranked** by
predicted confidence; RFdiffusion3 takes a pasted **structure** plus a contig
and returns them **unranked**; PXDesign takes a pasted **structure** plus the
target chains and returns binder backbones with no sequence at all. All three
return a job to poll, exactly like a [prediction](predictions.md).

The protocol implies the model. You may pass `model` explicitly (`boltzgen`,
`rfd3` or `pxdesign`), but it has to match the protocol's model; a mismatch is a
`400`.

## BoltzGen: binders against a protein, ligand or nucleic acid

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/designs \
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
- **`protocol`** (required): what to design.
- **`params`**: `num_designs`, `budget`, `fast`. See
  [Models & limits](models-and-limits.md#design-protocols-and-parameters).
- **`name`**: optional label.

| `protocol` | Designs |
|---|---|
| `protein-anything` | De novo mini-protein binder against any target. |
| `peptide-anything` | Short peptide binder. |
| `nanobody-anything` | Single-domain antibody / nanobody (VHH). |
| `antibody-anything` | Antibody binder. |
| `protein-small_molecule` | Protein binder with a binding-affinity step. |
| `protein-redesign` | Re-design residues of an existing binder. |

Submits accept the same `Idempotency-Key` and `Prefer: wait` headers as
predictions.

## RFdiffusion3: all-atom design against a structure

RFdiffusion3 diffuses protein structure and sequence together, conditioned on a
target structure you provide:

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/designs \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol":"rfd3-binder",
    "name":"my-rfd3-binders",
    "structure":"<the contents of your PDB or mmCIF file>",
    "contig":"A1-150,60-80",
    "params":{"num_designs":4,"num_timesteps":100}
  }'
```

- **`structure`** (required): the target as PDB or mmCIF **text**. Paste the file
  contents, up to 700,000 characters (about a 1,000-residue PDB).
- **`contig`** (required): comma-separated segments. A chain-range like `A1-150`
  keeps those residues fixed; a bare number like `60-80` designs that many new
  residues. So `"A1-150,60-80"` keeps target chain A, then designs a 60 to 80
  residue binder.
- **Size**: the contig, not the paste, sets how big the run is. Motif plus
  designed residues has to come to 490 or fewer, so a large structure is fine as
  long as the contig selects less of it.
- **`params`**: `num_designs`, `num_timesteps`, `seed`. See
  [Models & limits](models-and-limits.md#design-protocols-and-parameters).

| `protocol` | Designs |
|---|---|
| `rfd3-binder` | Binder against a protein target structure. |
| `rfd3-scaffold` | Scaffold around a fixed functional motif: list the motif residues, design between them. |
| `rfd3-na-binder` | Binder against a fixed DNA or RNA structure. |

RFdiffusion3 designs are unranked: there is no refold-and-filter step, so you get
one mmCIF per design with no confidence scores. To check one, fold its sequence
back onto the target with a [prediction](predictions.md) and read the interface
confidence (`iptm`).

## PXDesign: binder backbones against a structure

PXDesign conditions on a distogram of the target chains you name and generates
binder backbones. It is the fastest of the three and the least finished output:
coordinates only, no sequence, no ranking, no confidence score. Run the
backbones through a sequence-design tool before ordering anything. For a ranked,
sequenced binder use BoltzGen.

```bash
curl -s -X POST https://api.japanfold.aiand.com/v1/designs \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol":"pxdesign-binder",
    "structure":"<the contents of your PDB or mmCIF file>",
    "chains":"A",
    "binder_length":80,
    "hotspots":"A74,A75,A76",
    "params":{"num_designs":4,"n_step":200}
  }'
```

`chains` names the target chains to condition on, `binder_length` is the
residue count of the binder to generate (8 to 200), and `hotspots` is optional:
target residues the binder should aim at. One protocol, `pxdesign-binder`. Only
the named chains count toward the size ceiling, and the binder counts too: their
sum has to be 768 or fewer, so a large file is fine if you name one small chain.

## Reading designs

Poll `GET /v1/jobs/{id}` until `succeeded`, then read
`GET /v1/jobs/{id}/results` for the designs and their structure artifacts, or
`GET /v1/jobs/{id}/archive` for everything as one zip. See
[Jobs → Results](jobs.md#results) and
[Examples](examples.md#design-binders-boltzgen).

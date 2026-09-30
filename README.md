# JapanFold skill

Fold proteins, co-fold with ligands (+ binding affinity), design binders, and
embed sequences from your AI agent, via the
[JapanFold](https://japanfold.aiand.com) API. Get a key at
[japanfold.aiand.com/account/token](https://japanfold.aiand.com/account/token) and set
`JAPANFOLD_API_KEY`.
**No local GPU.** Runs Boltz-2 / ESMFold-2 / Protenix-v2 / OpenFold3 /
OpenBind-0 / RoseTTAFold3 / OpenDDE for structures, AF2-IG for scoring a binder you
already designed, BoltzGen / RFdiffusion3 /
PXDesign for design, and ESMC / SaProt for embeddings, on Tenstorrent.

It's a single [`SKILL.md`](SKILL.md) built on the open [Agent Skills](https://agentskills.io)
standard, so it installs into **any** compatible harness with one command.

## Install

One line installs it everywhere the open standard is supported — **Claude Code,
Cursor, Codex, Gemini CLI, Cline, Windsurf, Copilot, Amp and the rest**:

```bash
npx skills add japanfold/japanfold          # this project
npx skills add japanfold/japanfold -g       # global: every project / new chat
```

- Target specific agents: `-a claude-code`, `-a cursor`, `-a codex`, `-a '*'` (all).
- Prefer not to use the installer? It's just a file — drop `SKILL.md` into your
  agent's skills directory (e.g. `~/.claude/skills/japanfold/SKILL.md`).

**Claude Science** manages skills in-app (no installer): **Customize → Skills**,
add from this repo (or paste `SKILL.md`) and **publish** it. Or skip install
entirely — the API is public and self-describing, so just ask:
*"use the JapanFold API at `api.japanfold.aiand.com` to fold …"*.

## Use

Once installed, just ask your agent in plain language:

> *"Fold this sequence with Boltz-2 and report the confidence: MKTAYIAK…"*
> *"Design 10 nanobody binders against this target."*

Or invoke it explicitly where supported: `/japanfold`.

## The API

`https://api.japanfold.aiand.com`: async (submit → poll → download), a key on every call.
Full contract at [`/v1/openapi.json`](https://api.japanfold.aiand.com/v1/openapi.json).
See [`SKILL.md`](SKILL.md) for endpoints, examples, and limits.

Fast mode is **off** unless the agent sends `"params": {"fast": true}`. It gives higher
throughput and may be slightly less accurate. The workbench turns it on for people clicking
through; the API and this skill leave it off so a scripted run gets the accurate path.
ESMFold-2 always runs fast on this hardware, and its job records `fast: true`.

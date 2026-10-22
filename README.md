# JapanFold skill

Fold proteins, co-fold with ligands (+ binding affinity), design binders, and
embed sequences from your AI agent, via the
[JapanFold](https://japanfold.aiand.com) API. Get a key at
[japanfold.aiand.com/account/token](https://japanfold.aiand.com/account/token) and set
`JAPANFOLD_API_KEY`.
**No local GPU.** Runs Boltz-2 / ESMFold-2 / OpenFold3 /
OpenBind-0 / RoseTTAFold3 / OpenDDE for structures, AF2-IG for scoring a binder you
already designed, BoltzGen / RFdiffusion3 /
PXDesign for design, and ESMC / SaProt for embeddings, on Tenstorrent.

Agents can call the API directly, no install needed:

```bash
export JAPANFOLD_API_KEY=jf_live_...          # from japanfold.aiand.com/account/token
curl -s https://api.japanfold.aiand.com/v1/predictions \
  -H "Authorization: Bearer $JAPANFOLD_API_KEY" -H 'Content-Type: application/json' \
  -d '{"model":"esmfold2","sequence":"MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQ"}'
```

Base URL `https://api.japanfold.aiand.com`, key in `Authorization: Bearer`.
`GET /v1/models` lists the model ids to send (`boltz2`, `esmfold2`, `openfold3`, ...)
with their size limits. The agent-readable summary is
[`llms.txt`](https://japanfold.aiand.com/llms.txt).

It's a single [`SKILL.md`](SKILL.md) built on the open [Agent Skills](https://agentskills.io)
standard, so it installs into **any** compatible harness with one command.

## Install

One line installs it everywhere the open standard is supported: **Claude Code,
Cursor, Codex, Gemini CLI, Cline, Windsurf, Copilot, Amp and the rest**:

```bash
npx skills add japanfold/japanfold          # this project
npx skills add japanfold/japanfold -g       # global: every project / new chat
```

- Target specific agents: `-a claude-code`, `-a cursor`, `-a codex`, `-a '*'` (all).
- Prefer not to use the installer? It is one file. Download it unchanged into
  your agent's skills directory, for Claude Code:
  `curl -fsSL --create-dirs -o ~/.claude/skills/japanfold/SKILL.md https://japanfold.aiand.com/skill.md`.
  A web-fetch tool returns only a summary of the page, so an agent should use curl too.

**Claude Science** manages skills in-app (no installer): **Customize → Skills**,
add from this repo (or paste `SKILL.md`) and **publish** it. Or skip install
entirely: the API is public and self-describing, so just ask:
*"use the JapanFold API at `api.japanfold.aiand.com` to fold …"*.

## Use

Once installed, ask your agent in plain language:

> *"Fold this sequence with Boltz-2 and report the confidence: MKTAYIAK…"*
> *"Design 10 nanobody binders against this target."*

Or invoke it explicitly where supported: `/japanfold`.

## MCP

The API is also a remote MCP server at `https://api.japanfold.aiand.com/mcp`, listed in the
[MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=io.github.japanfold) as
`io.github.japanfold/japanfold`. claude.ai, Claude Desktop and ChatGPT add it as a connector and
sign in, no key needed. Claude Code:
`claude mcp add --transport http japanfold https://api.japanfold.aiand.com/mcp --header "Authorization: Bearer $JAPANFOLD_API_KEY"`.

To publish a new registry version, bump `version` in [`server.json`](server.json) and run
`mcp-publisher login github && mcp-publisher publish`.

## The API

`https://api.japanfold.aiand.com`: async (submit → poll → download), a key on every call.
Full contract at [`/v1/openapi.json`](https://api.japanfold.aiand.com/v1/openapi.json).
See [`SKILL.md`](SKILL.md) for endpoints, examples, and limits.

Fast mode is **off** unless the agent sends `"params": {"fast": true}`. It gives higher
throughput and may be slightly less accurate. The workbench turns it on for people clicking
through; the API and this skill leave it off so a scripted run gets the accurate path.
ESMFold-2 always runs fast on this hardware, and its job records `fast: true`.

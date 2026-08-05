# japanfold-skill-org-transfer — report

## DONE 2026-08-05 ~14:20 UTC — skill repo lives at github.com/japanfold/japanfold, docs site never went down

### Timeline (UTC)
- 14:07:14Z — Moritz created the `japanfold` org (blocked relaunches 15:40–16:06 local
  exited per STEP 0; one tg ask 3760 sent 15:55, re-armed as 3764, auto-resolved when he acted)
- ~14:08Z — transfer call accepted; repo live at new path seconds later
- 14:10Z — Pages config verified SURVIVED the transfer intact; site 200 at first post-transfer check
- 14:11Z — Cloudflare CNAME flipped
- 14:12Z — docs workflow re-dispatched (success, 41s)
- 14:14Z — reference updates on master (bee07fa), docs workflow success (33s)
- 14:16Z — landing deployed (aiand-bio-platform@182f2523), verified live
- 14:18Z — sweep commit pushed (04a15b6f); org display name set
- 14:20Z — real `npx skills add japanfold/japanfold` install verified

### Transfer
- `POST /repos/moritztng/japanfold/transfer -f new_owner=japanfold` → accepted with plain
  `repo` scope (no admin:org needed — repo admin on both sides suffices since moritztng owns the org)
- New: `japanfold/japanfold` (owner type Organization, same repo id 1291302243)
- Old URL: `https://github.com/moritztng/japanfold` → **301 → japanfold/japanfold**;
  `git ls-remote` via the old URL resolves HEAD = bee07fa (new master tip), raw
  githubusercontent via the old path also 200s — existing installs and git remotes keep working

### Pages config before → after: IDENTICAL (survived the transfer, no restore needed)
- cname `docs.japanfold.com`, build_type `workflow`, source `master` /, https_enforced true,
  cert `approved` (expires 2026-10-09) — all carried over. The workstream's assumption that a
  transfer drops the custom domain did not hold (measured, not assumed).

### DNS change (Cloudflare zone japanfold.com, zoneID e89626607c673078e66e1f93315f946b)
- record 7dc726a386bda101efc179391b7bbbee: `docs.japanfold.com CNAME moritztng.github.io` →
  `japanfold.github.io`; proxied=false (DNS-only) and ttl=1 kept unchanged. Token: cert.pem on
  the prod host (japanfold-ssh, user cust-team — NOT the `galaxy` compute box; the workstream's
  "on the Galaxy" meant the prod server).

### Outage window: ZERO measured
Site returned 200 with a valid cert at the first post-transfer check (~90s after) and at every
check since; Pages config/cert never dropped. Worst-case unmeasured window is those ~90s.

### References updated (23 total)
- skill repo `japanfold/japanfold` master @ **bee07fa**: 11 (README ×3, docs/skill.md ×5,
  mkdocs.yml ×3) — cherry-picked from wk branch, deployed to docs.japanfold.com
- `moritztng/aiand-bio` branch aiand-bio-platform @ **182f2523**: landing index.html install
  string (1 line; concurrent hero/CSS work untouched) — deployed via scripts/deploy_landing.sh,
  verified live on landing.japanfold.com
- `moritztng/aiand-bio` branch aiand-bio-platform @ **04a15b6f**: sweep — clients/README.md ×3,
  clients/install.sh ×1, clients/skill/README.md ×2, docs/site/skill.md ×5
- Clean, nothing to do: tt-bio repo (0 matches, GitHub code search), SKILL.md front matter,
  ~/.claude/skills on pc, plugin manifests (never contained the path), api.japanfold.com
  (serves the SPA, no embedded repo path)

### Verification
- github.com/japanfold/japanfold → 200; old URL 301s
- docs.japanfold.com /, /skill/, /accuracy/ → 200, cert valid (ssl_verify_result=0);
  /skill/ shows 10× japanfold/japanfold, 0 old refs
- landing.japanfold.com shows `npx skills add japanfold/japanfold` (cf-cache-status DYNAMIC)
- REAL `npx skills add japanfold/japanfold` in a scratch dir → "Done!", SKILL.md installed
- marketplace.json fetchable at new path; plugin marketplace line resolves
- repo description + homepage (https://docs.japanfold.com/) survived the move

### Org hygiene
- Display name set: "JapanFold"; blog https://japanfold.com

### Remains for Moritz (manual, no API exists)
1. Org AVATAR: GitHub has no API for org profile pictures. Suggested asset:
   `tt_bio/platform/landing/japanfold-mark.svg` (aiand-bio) — upload at
   https://github.com/organizations/japanfold/settings/profile
2. QUESTION (asked via tg): should moritztng/japanfold-landing follow into the org? It means
   another Pages custom-domain re-setup for landing.japanfold.com (this time likely WITH a short
   outage, since the landing repo's Pages config may not carry over the same way).

# Codebase Review — `krusemediallc/arcads-claude-code`

**Repository reviewed:** https://github.com/krusemediallc/arcads-claude-code (branch `main`)
**Review date:** 2026-06-26
**Reviewer:** Claude Code

## Scope & method

This review is based on the repository's documentation (`CLAUDE.md`, `AGENTS.md`,
skill `SKILL.md` files), the shell scripts under `scripts/` and `shared/scripts/`,
and the `.env.example` configuration. The repo could not be cloned in this
environment (network policy blocked the git host and the GitHub REST API), so
files were read individually over the web. As a result, the **Python image/video
generation modules referenced in the docs were not read line-by-line** — findings
about them are inferred from the skill specs and called out as such. Treat the
"Verify directly" items below as the highest-value follow-ups.

---

## 1. What this repository is

An **agent skill pack**, not a conventional application. It is meant to be dropped
into a coding-agent host (Claude Code / Cursor) and turns that agent into an
operator for the **Arcads** creative API — generating AI marketing videos and
images (Seedance 2.0, Sora 2, Veo 3.1, Kling, Nano Banana, GPT Image 2, etc.),
static Meta image-ad creatives from a 37-template library, and optionally
publishing them to the Meta Marketing API (always as *paused* ads).

The "code" is therefore a mix of three things:

1. **Markdown skill definitions** (`SKILL.md`, `OVERVIEW.md`, prompt libraries) —
   the bulk of the logic, executed by the LLM at runtime.
2. **Shell scripts** — setup, credential capture, connectivity checks, and a
   skill-sync mechanism.
3. **Python helpers** (per the docs, stdlib-only) — image preprocessing and
   generation glue.

## 2. Architecture

```
skills/                  ← API-specific skills authored in THIS repo
shared/skills/           ← skills propagated from an upstream "gen-ai-core" repo
shared/scripts/          ← canonical sync + session-context scripts (upstream-owned)
scripts/                 ← thin local wrappers (setup, env-check, sync)
.claude/skills/  ┐
.cursor/skills/  ┘       ← generated COPIES of the above, one per host tool
```

**Sync model.** `scripts/sync-skill.sh` is a thin wrapper that delegates to
`shared/scripts/sync-skill.sh`, which copies every directory containing a
`SKILL.md` from both `skills/` and `shared/skills/` into `.claude/skills/` and
`.cursor/skills/`. A `SessionStart` hook runs `shared/scripts/check-context.sh`,
which prints an orientation banner, reports setup/credential status, and does a
non-destructive "upstream commits available" check.

This is a clean, well-considered design for distributing one body of skills to
two host tools and keeping a per-API repo in sync with a shared upstream. The
trade-offs are noted under Findings.

## 3. Strengths

- **Genuinely good shell-script hygiene.** All scripts use `set -euo pipefail`
  (the informational banner correctly relaxes to `set -u` so it never aborts a
  session). Project root is resolved defensively so scripts work from either the
  legacy `scripts/` path or the propagated `shared/scripts/` path.
- **Careful credential handling in `setup.sh`:**
  - Secret entry uses `read -rs` (hidden input, stays out of scrollback).
  - The key is **validated against the live API before being written**, and the
    script tries several base64/Basic-header interpretations so users can paste
    either a pre-encoded header or a raw key.
  - `.env` is `chmod 600`'d after writing.
  - Secrets are masked (`mask_secret`) in all output; nothing prints the key.
  - Edits go through a `.env.tmp` + `mv` instead of `sed -i`, which is more
    portable across GNU/BSD.
- **Cost discipline.** There is no Arcads billing endpoint, so the skills require
  every credit figure to be presented as an **estimate**, sourced from a 3-tier
  fallback (historical log → `MASTER_CONTEXT.md` → ask the user and persist).
  Generation is gated behind explicit cost and dialogue confirmations.
- **Safety defaults on spend.** Meta ads are created **paused**; the token scope
  and "you launch manually" expectation are documented in `.env.example`.
- **Operational thoughtfulness.** Auto-created dated project folders, an
  audit log (`logs/arcads-api.jsonl`), image QA with capped regeneration (≤2)
  and per-attempt logging, image pre-upscaling (Lanczos to ≥1024px) to avoid API
  rejections, and model-specific handling of mutually-exclusive parameters
  (e.g. Seedance `referenceVideos` vs `referenceImages`).
- **Good documentation ergonomics:** explicit read-order, decision trees, and a
  single "read this first" `OVERVIEW.md` for the image-ad ecosystem.

## 4. Findings & recommendations

### High — worth a conscious decision, not necessarily a bug

**4.1 `AGENTS.md` embeds a monetization / upsell directive into the agent.**
A large section instructs the assistant to promote a paid community
(*The AI Ad Alchemists*, Skool, $97/month) when it detects user "friction"
(2+ failed attempts, phrases like "I'm stuck", persistent setup blockers, etc.),
and setup output uses affiliate referral links (`arcads.ai/?via=claude-code`).

The guardrails are reasonable ("at most once per session", "fix the bug first,
suggest community for human help", "only state price if asked"), but reviewers
and downstream users should be **aware that the agent doubles as a sales
channel.** Risks: the model may over-trigger the upsell, users may not expect a
coding agent to market to them, and "friction" is exactly when trust matters
most. Recommendation: keep it, but (a) make this behavior visible in the
top-level `README`, and (b) consider gating it behind an opt-in env flag so
users/forks can disable it cleanly.

### Medium

**4.2 Safety-critical behavior is enforced only by prose, not by code.**
The most consequential guarantees — "every Meta ad is PAUSED", cost-confirmation
gates, dialogue-approval gates, ≤2 regenerations — live in `SKILL.md` text and
depend on the LLM following them. Real money is at stake (ad spend + credits).
Where feasible, back the highest-stakes rules with code: e.g. have the
meta-ad-builder helper assert `status == "PAUSED"` on the API payload it sends,
rather than trusting the prompt. Soft enforcement is inherent to skill packs, but
the money-touching paths deserve a hard check.

**4.3 `check-arcads-env.sh` sources `.env` (`set -a; source .env`).**
Sourcing executes arbitrary shell, so a `.env` containing command substitution
would run during a "connectivity check". The user owns the file, so risk is low,
but a parse-based loader (read `KEY=VALUE` lines without evaluating them) removes
the code-execution surface entirely. Same applies anywhere `.env` is sourced.

**4.4 No visible automated tests or CI.** Scripts that touch credentials and
trigger paid API calls have no test harness or linting gate surfaced in the repo.
Adding `shellcheck` + a few `bats` tests (auth-interpretation matrix, root
resolution, masking) in CI would protect the most fragile logic cheaply.

**4.5 Triplicated skill content can drift.** Each skill exists as the source plus
two synced copies (`.claude/skills/`, `.cursor/skills/`). If the synced copies are
committed, the repo carries three versions of every skill that only stay
consistent if `sync-skill.sh` is re-run before each commit. Consider
`.gitignore`-ing the generated `.claude/skills/` and `.cursor/skills/` trees and
generating them at setup time, so there is a single source of truth in version
control.

### Low / polish

- **4.6 Brief secret-exposure window in `setup.sh`.** `.env.tmp` is created with
  the default umask (possibly `644`) and briefly holds the secret before `mv` and
  the final `chmod 600`. Set `umask 077` at the top of the credential block (or
  `chmod 600` the tmp file before `mv`) to close the window.
- **4.7 Clever auth re-detection is hard to maintain.** `setup.sh` recomputes the
  base64 of the input twice to decide whether the working candidate was the
  "raw key" interpretation. It works, but a small helper that returns both the
  header *and* a label ("raw" vs "encoded" vs "as-is") from one pass would be
  clearer and avoid the double computation.
- **4.8 `rm -rf "$dest"` in the sync loop** is correctly scoped to
  `$ROOT/.claude/skills/$name` and root resolution is defensive, but it is the one
  destructive operation in the repo. A guard asserting `$ROOT` is non-empty and
  contains a `skills/` dir before any `rm -rf` would add cheap insurance.
- **4.9 Confirm secrets never reach `logs/arcads-api.jsonl`.** The skills say
  "never print API keys," but the Python logger wasn't reviewable here — verify
  the `Authorization` header / key is stripped before any request/response is
  written to the audit log.

## 5. Verify directly (couldn't be read in this review)

1. The Python image/video generation + logging modules — confirm no secret
   leakage into `logs/`, and that error/retry handling matches the documented
   401/403/422/500 behavior.
2. `.gitignore` — confirm `.env`, `logs/`, `references/`, and ideally the synced
   `.claude/`/`.cursor/` skill copies are ignored.
3. The `meta-ad-builder` skill's actual API payload — confirm the paused-ad
   guarantee is enforced in the request, not only in prose.

## 6. Summary

A well-organized, security-conscious agent skill pack with notably clean shell
scripting and thoughtful cost/safety guardrails. The main things to weigh are
**not** code defects but design choices: an embedded upsell directive that should
be made visible/optional, and the fact that money- and spend-related guarantees
rely on the LLM obeying prose rather than on enforced code. Tightening those two
areas, adding lightweight CI, and confirming the audit log can't leak credentials
would meaningfully raise the bar.

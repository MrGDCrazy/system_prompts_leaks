# Repository Audit: system_prompts_leaks

**Audited:** 2026-06-25 · **Branch:** `claude/github-repo-audit-phwj2v` · **Method:** direct inspection of git history, file contents, and configuration (no README claims taken at face value)

## 0. What this repository actually is

This is **not a software application** — it is a curated, version-controlled archive of plain-text/Markdown files, each purporting to be an extracted or leaked system prompt from a commercial AI product (Claude, ChatGPT, Gemini, Grok, Copilot, etc.), plus one small Python script that builds a GitHub Pages traffic dashboard. There is no application code, package manifest, test suite, or build step. Several items in the standard engineering-audit checklist (type safety, dependency graph, API surface, module architecture) **do not apply** and are marked as such below rather than forced to fit.

## 1. Repository Orientation

| Signal | Finding |
|---|---|
| Commit count | 50 commits total, **all** within the last 30 days (first commit 2026-06-04, latest 2026-06-24) — a brand-new repo, not a mature project |
| Contributors | **1 of 1** — Ásgeir Thor Johnson authored 100% of commits. Bus factor = 1. No co-maintainers, no review process possible (no second pair of eyes ever touched this repo) |
| Tags/Releases | **None.** No `git tag` output at all — no versioning scheme, no release notes, no way to pin to a known-good snapshot |
| Branches | `main` only (plus this audit branch). No git-flow, no long-lived feature branches — consistent with a single-author, direct-to-main workflow |
| Top-level structure | One file per AI vendor as a directory (`Anthropic/`, `OpenAI/`, `Google/`, `xAI/`, `Microsoft/`, `Misc/`, etc.), `README.md`, `LICENSE` (CC0 1.0 — public domain dedication), `.gitattributes`, `.github/` |
| Directory size | `Anthropic/` dominates at 5.9 MB (103 files) vs. `OpenAI/` 1.6 MB (87 files); everything else is under 360 KB. Anthropic content is disproportionately large per-file (several files >150 KB of raw text) |
| Largest single files | `Claude Code/bundled-skills/claude-api.md` (538 KB), `Official/all.md` (396 KB), `claude-design.md` (363 KB), `Codex/gpt-5.5.md` (360 KB) — these are raw prompt/tool-definition dumps, not code, so size alone isn't a smell, but it means content is unreviewable line-by-line in practice |

## 2. "Code Quality" — adapted findings

Items 7–11 of the standard checklist (test infrastructure, CI pipeline, type safety, linting, TODO/FIXME counts) target application code. This repo has **one script**: `.github/scripts/accumulate-traffic.py` (~210 lines), invoked by `.github/workflows/traffic-to-badge.yml` on a daily cron.

- **Tests:** none exist, none are run. The script has no test coverage at all.
- **CI:** the only workflow (`traffic-to-badge.yml`) collects GitHub traffic stats and publishes a dashboard to a `traffic` branch via GitHub Pages. It does **not** lint, type-check, or validate the script, and there is no CI gate on `main` for content PRs (no markdown link-checker, no duplicate-content check, no schema validation for the one `.json` tool file).
- **Linting/formatting:** no `.eslintrc`, `.prettierrc`, `pyproject.toml`, or any linter config anywhere in the repo. The Python script is unformatted/unchecked (no `black`, `ruff`, `mypy`).
- **Secrets handling:** the script reads `GITHUB_TOKEN`/`GH_TOKEN` from environment correctly (`os.environ.get`), no hardcoded credentials found. A targeted regex scan for API-key-shaped strings (`sk-…`, `AKIA…`, `ghp_…`) across the whole tree returned **zero matches**.
- **Commit hygiene flag:** commit `0d3f991` ("Remove duplicate Codex system instruction files") deleted two ~22,700-line files literally named `OpenAI/Codex/# SYSTEM INSTRUCTIONS.md`. That means malformed, duplicate raw dumps were committed to `main` and lived there before being cleaned up — evidence of low pre-commit review rigor, consistent with the single-author/no-review workflow.

**Real content-integrity issue found (not in the original checklist, but the closest analog to "code smells" for this repo):** byte-for-byte duplicate files under different names:

| Duplicate pair |
|---|
| `OpenAI/tool-python-code.md` ≡ `OpenAI/tool-python.md` |
| `OpenAI/o4-mini-high.md` ≡ `OpenAI/o4-mini.md` |
| `OpenAI/API/o3-medium-api.md` ≡ `OpenAI/API/o4-mini-medium-api.md` |
| `OpenAI/API/o3-high-api.md` ≡ `OpenAI/API/o4-mini-high.md` |
| `OpenAI/Codex/codex-auto-review.md` ≡ `OpenAI/Codex/gpt-5.4.md` |
| `Anthropic/Official/2025-07-31-claude-opus-4.md` ≡ `Anthropic/Official/2025-08-05-claude-opus-4.md` |
| `Anthropic/Official/2025-07-31-claude-sonnet-4.md` ≡ `Anthropic/Official/2025-08-05-claude-sonnet-4.md` |

Some of these may be intentional (e.g., two model variants genuinely sharing one prompt), but `o3-high-api.md` being byte-identical to `o4-mini-high.md` and `o3-medium-api.md` to `o4-mini-medium-api.md` strongly suggests copy-paste-without-verification when adding new model entries — i.e., the "leaked prompt" for a newer model may just be a stale copy of an older one's file, not independently verified content.

## 3. Dependency & Security Audit

- **Dependency manifests:** none exist (`package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml` — all absent). The repo has zero runtime/build dependencies of its own. The only third-party code is pulled in at CI-runtime via two pinned GitHub Actions: `yi-Xu-0100/traffic-to-badge@v1.4.0` and `peaceiris/actions-gh-pages@v4`, plus `actions/checkout@v4` and `actions/github-script@v7`. The dashboard HTML loads `apexcharts@4` from a public CDN (jsdelivr) at runtime in users' browsers — a minor third-party-supply-chain surface for anyone viewing the dashboard page, since it's an unpinned major-version tag (`@4`, not a SHA-pinned or exact-version reference).
- **Dependabot/Renovate/SECURITY.md:** none present. No automated dependency-update or vulnerability-disclosure process.
- **Secrets hygiene:** no `.env` files committed; no plaintext secrets found via grep. `.gitattributes` only configures whitespace handling for `*.md`, nothing security-relevant. There is no explicit `.gitignore` reviewed for `.env`/`*.pem` coverage — none is needed since there's no app to configure, but its absence means any future contributor adding tooling has no baseline protection.
- **Supply chain / provenance:** no SBOM, no signed commits (single author, no GPG signatures observed in `git log`), nothing published to a package registry, so npm/PyPI provenance is not applicable.
- **License:** CC0 1.0 Universal (public domain dedication) — notable choice, since it disclaims copyright over content that includes verbatim, unattributed extracts of other companies' proprietary system prompts. The repo's purpose is explicitly "leaks" of third-party IP; CC0 covers the compilation's own copyright but obviously cannot grant any actual rights to the underlying vendor prompt text. This is a legal-exposure point worth flagging in due diligence, not a code-quality one.

## 4. Architecture & "Design"

- **Structure:** flat, single-purpose content repository — one directory per vendor, one Markdown file per product/model. No monorepo tooling, no circular-dependency risk (there are no imports), no "god modules" in the software sense (large files are prose dumps, not logic).
- **"API surface":** not applicable — this is a documentation archive, not a library or service.
- **Versioning of content:** informal and inconsistent. Some entries are versioned by filename suffix (e.g., `gpt-5.4.md`, `gpt-5.5.md`), others by date-prefixed files under `Anthropic/Official/` (e.g., `2026-05-28-claude-opus-4.8.md`). There's no changelog and no migration notes — `README.md`'s "Recently Updated" table is the only chronology, and it's manually maintained prose, not generated from commits or tags.
- **Configuration management:** the one script hardcodes its target repo (`REPO_FULL = "asgeirtj/system_prompts_leaks"`) and dashboard styling inline in a Python string template rather than a config file — acceptable for a 200-line single-purpose script, but worth noting if it's ever reused for another repo (would require a code edit, not a config change).

## 5. Content-authenticity caveat (specific to this repo's actual purpose)

The deliverable of this repo is the *accuracy* of its content — i.e., whether each file is a faithful, unaltered extraction of a real system prompt. That cannot be verified from inside the repository: there's no provenance metadata (no source URL, extraction method, or date-of-capture recorded per file beyond what's baked into the filename/README table), and the README's "as seen in The Washington Post" claim and embedded screenshots are unverifiable from the repo alone. The single-author/no-review/no-CI workflow means nothing currently checks a new submission against the format of file it's overwriting or against the vendor's published behavior before it lands on `main`. Combined with the confirmed duplicate-file findings in §2, this is the highest-priority finding for anyone treating this repo as an authoritative source.

## Summary

| Area | Verdict |
|---|---|
| Bus factor | **1** — single contributor, no co-maintainers |
| Release process | **None** — no tags, no changelog automation |
| CI/CD | Exists only for a traffic dashboard; **no validation gate** on content PRs |
| Tests | None (none needed for content, but the one script is untested) |
| Security | No secrets found, but no Dependabot/SECURITY.md/SBOM either |
| Content integrity | **7 exact-duplicate file pairs** found, at least 2 suggesting unverified copy-paste between model versions |
| Legal exposure | CC0 license over a compilation of (allegedly) leaked third-party proprietary text |
| Architecture | N/A — flat content archive, no code architecture to assess |

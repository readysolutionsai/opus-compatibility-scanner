# Claude Opus 4.6 ⇄ 4.7 Compatibility Scanner

[![Version](https://img.shields.io/github/v/release/readysolutionsai/opus-compatibility-scanner?label=version&color=CC785C)](https://github.com/readysolutionsai/opus-compatibility-scanner/releases)
[![License](https://img.shields.io/badge/license-MIT-1A1917)](LICENSE)
[![Patterns](https://img.shields.io/badge/patterns-70-CC785C)](references/)
[![Pattern catalog](https://img.shields.io/badge/catalog_current_through-2026--04--26-6B665F)](references/)
[![Build story](https://img.shields.io/badge/read-the_build_story-1A1917?labelColor=CC785C)](https://readysolutions.ai/blog/2026-04-26-opus-4-7-compatibility-scanner-claude-code/)

A long-lived hygiene tool for projects that actively use both Claude Opus 4.6 and Opus 4.7. Default mode never proposes a change that would degrade either model.

## What it does

Scans `CLAUDE.md`, `AGENTS.md`, subagent definitions, skill files, `settings.json`, hook scripts, package manifests, and Anthropic SDK call sites for known 4.6 → 4.7 migration issues. Surfaces every finding as Critical, Warning, or Info, then walks fixes one at a time with citations back to the relevant Anthropic documentation.

## Why it matters

Claude Opus 4.7 reads instructions more literally than 4.6 did. Directives that 4.6 silently inferred a sensible interpretation for (`be concise`, `default to no comments`, `never use any`) get applied at face value on 4.7, and your agent starts behaving differently. Configuration that worked on 4.6 (`thinking: { type: "enabled", budget_tokens: N }`, non-default `temperature`) returns HTTP 400 on 4.7. This scanner finds both classes of issues before you upgrade.

## Quick start

Three ways to install. Pick what fits your workflow.

**Option 1: Download the latest release (recommended)**

Grab `opus-compatibility-scanner.skill` (or `.zip`) from the [latest release](https://github.com/readysolutionsai/opus-compatibility-scanner/releases/latest), then extract:

```bash
# Project-scoped (this repo only):
unzip opus-compatibility-scanner.zip -d .claude/skills/

# User-global (all your projects):
unzip opus-compatibility-scanner.zip -d ~/.claude/skills/
```

**Option 2: Clone this repo directly into your skills folder**

```bash
# Project-scoped:
git clone https://github.com/readysolutionsai/opus-compatibility-scanner.git .claude/skills/opus-compatibility-scanner

# User-global:
git clone https://github.com/readysolutionsai/opus-compatibility-scanner.git ~/.claude/skills/opus-compatibility-scanner
```

**Option 3: Download from readysolutions.ai**

Direct `.skill` download is also hosted at [readysolutions.ai/downloads/opus-compatibility-scanner.skill](https://readysolutions.ai/downloads/opus-compatibility-scanner.skill).

After install, start a new Claude Code session. The skill registers automatically.

*Heads-up on token cost:* the combined pattern catalogue and citation library load on first reference during a scan. Class A (prose) scans load ~28K tokens. Class B (config) scans load ~18K. Full-scope scans (both classes) load ~38K, under 4% of a 1M context window. Subsequent turns reuse the loaded references.

## Use

Slash command: `/opus-compatibility-scanner`. Or describe the task in plain English. Trigger phrases include:

- *"Scan my project for Opus 4.7 migration issues"*
- *"Is my CLAUDE.md ready for Opus 4.7?"*
- *"My agent is acting weird since I switched to 4.7. What broke?"*

You'll see a short intro plus three options:

1. **Compatibility scan.** For projects running both Opus 4.6 and 4.7. Holds back 4.7-only optimizations that would degrade 4.6.
2. **4.7-only scan.** For projects fully migrated off 4.6. Surfaces every finding with no holdback.
3. **Diagnose a behavior change.** You switched to 4.7 and something broke. Scans in compat mode, then surfaces symptom-mapped findings first.

Reply `1`, `2`, or `3`. The scan does not start until you pick a mode.

After the scan you'll see an executive summary, then a per-finding drill-down. Reply `yes`, `skip`, or `explain` for each finding. Nothing is edited until you accept a specific fix.

**No files are written by default.** At the end of the scan, you can opt in to saving a JSON manifest and/or a Markdown report to `.claude/`. Both default to no.

**Want to see what a scan looks like?** A sample-output transcript is bundled at [`EXAMPLE.md`](EXAMPLE.md). It walks through a compatibility scan on a synthetic project, including the executive summary and one Critical drill-down with the diff.

## What's covered

- **37 prose patterns** across `CLAUDE.md`, `AGENTS.md`, subagent definitions, and skill bodies (literal-following hazards, contradictions, scaffolding 4.7 no longer needs).
- **31 config patterns** (33 numbered, 2 reserved) across `.claude/settings.json`, hook scripts, package manifests, and Anthropic SDK call sites (deprecated `thinking` shapes, removed sampling parameters, beta header churn, SDK version floors, model ID strings).

Each finding cites a tier-classified source. Critical findings require a Tier 1 (Anthropic-authored) citation.

*Pattern catalog current through 2026-04-26.* Patterns are reviewed against Anthropic's published migration guide, model-deprecations page, and Agent SDK release notes; the date above marks the last review.

## How this differs from `claude-api`

Anthropic ships a bundled `claude-api` skill that automates model-ID swaps and breaking-parameter migrations in your Anthropic SDK call sites (`client.messages.create()` and friends). Run it for code-level changes.

This scanner covers what `claude-api` doesn't: the prose patterns in your `CLAUDE.md`, `AGENTS.md`, subagent definitions, and skill bodies. Those are the literal-following hazards that Opus 4.7 reads at face value. The two are complementary. Typical sequence: run `/claude-api migrate` for SDK code, then run this scanner for prose hygiene.

## Compatibility with Opus 4.6 and Opus 4.7

This scanner is built for projects that use both Opus 4.6 and Opus 4.7. The default mode (compatibility) never proposes a change that would degrade either model. A 4.7-only mode is available for users who have fully moved off 4.6.

Opus 4.6 is currently `Active` per Anthropic's [model-deprecations page](https://platform.claude.com/docs/en/about-claude/model-deprecations) (tentative retirement: not sooner than February 5, 2027). The Opus 4.6 / 4.7 differences are documented in Anthropic's [migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide).

**Mode routing.** The skill always asks you to pick a mode before scanning. Your invocation language pre-selects a default. Phrases mentioning "compatibility" or the slash command pre-select compat mode; phrases mentioning "migration", "4.7-only", or "fully moved off 4.6" pre-select 4.7-only mode; symptom phrases ("my agent broke after the swap") pre-select Diagnose. You always confirm before any file is read.

**What compat mode does:**
- **Suppresses prose-level 4.7 tradeoffs.** Scaffolding deletions (*"think step by step"*, *"double-check the output"*, *"after every N tool calls, summarize"*), fixed-verbosity caps, and ask-clarifying-questions removals are useful on 4.7 but can degrade 4.6. The scan holds them back unless you opt in via the end-of-scan offer or a 4.7-only-mode invocation.
- **Downgrades config-level 4.7 tradeoffs to guided apply.** Model-ID swaps, `thinking`-shape rewrites, sampling-parameter strips, and other 🔴-annotated config changes still surface, but as `guided` (you confirm each call site is 4.7-bound before the edit applies) rather than `direct`.

**What 4.7-only mode does:** restores all 🔴 findings to their default rewrites and apply tiers. Use this when the project has fully moved off 4.6.

## Contributing

Pattern contributions are welcome. The scanner's value scales with catalog freshness, and Anthropic ships changes faster than I can scan for them.

- **Found a 4.6 → 4.7 issue not in the catalog?** Open a [Discussion](https://github.com/readysolutionsai/opus-compatibility-scanner/discussions) under the *Pattern proposals* category. Include the failing input, the file it lives in (CLAUDE.md, settings.json, etc.), and a Tier 1 citation if you have one.
- **Hit a bug or false positive?** Open an [Issue](https://github.com/readysolutionsai/opus-compatibility-scanner/issues). Include the file excerpt that triggered it (redacted is fine) and the finding the scanner produced.
- **Pull requests** for catalog additions, citation upgrades, or symptom-mapping improvements are welcome. Critical-tier findings require a Tier 1 (Anthropic-authored) citation; please include the URL in the PR description.

## Build story

Want the full story behind why this exists, the design decisions made along the way, and the audit it produced when run against my own most-robust setup? Read [The free Claude Code skill that audits your CLAUDE.md, hooks, and subagents for Opus 4.7 breaking changes](https://readysolutions.ai/blog/2026-04-26-opus-4-7-compatibility-scanner-claude-code/) on the Ready Solutions blog.

## Created by

Built by Mitchel Lairscey at [Ready Solutions](https://readysolutions.ai), a Claude-first AI integration consultancy. Need help applying these patterns across a larger codebase, auditing a multi-team setup, or migrating production workloads to Opus 4.7? [Book a free intro call](https://calendly.com/hello-readysolutions/30min) or reach out at [readysolutions.ai](https://readysolutions.ai).

## Disclaimer

This scanner identifies probable migration issues by pattern-matching against Anthropic's publicly documented Claude Opus 4.6 → 4.7 changes as of its publication date. It is an independent tool. It is not affiliated with, sponsored by, or endorsed by Anthropic. Pattern coverage is best-effort, not exhaustive. Anthropic's API and documentation may change after this scanner ships. Always verify proposed changes against your own codebase and Anthropic's current documentation before applying them. Provided as-is, without warranty of any kind. Use at your own discretion.

## Changelog

### 3.2.2
Pre-ship audit corrections. Foundational source quote in `patterns-prose.md` and the SKILL.md intro swapped to verbatim text from Anthropic's migration guide; the prior wording paraphrased the lead-in. Self-exclusion added so the scanner no longer reads files under any `opus-compatibility-scanner/` path during Phase 1. CB22 now cites the Sub-agents docs URL. MIT license declared in SKILL.md frontmatter and a LICENSE file shipped at the skill root. Token-cost estimate corrected to a per-class breakdown. New README section frames this scanner as complementary to Anthropic's bundled `claude-api` skill.

### 3.2.1
Audit fixes. Install docs lead with manual extraction; the unverified `.skill` drag-and-drop claim removed. End-of-scan offer trigger narrowed to suppressed prose-🔴 patterns only. Config-🔴 findings already surface as guided in the drill-down, so the prior trigger could fire when nothing was actually held back. CB12 reserved alongside CB13, consolidating image-coordinate handling into CB27. New Tier 1 source: Anthropic's *Best practices for using Claude Opus 4.7 with Claude Code*. F4 and CB6 source citations upgraded to verbatim. Manifest schema field renamed `tier_counts` → `apply_tier_counts` to match the values it counts.

### 3.2.0
Pre-launch hardening. Internal `apply_tier` enum renamed `auto` → `direct` so the name reflects the safety stance. The scan still asks before every edit. Pattern catalog touch-ups rebalance severity and citation tier across a handful of entries. SKILL.md trigger description tightened and trimmed from 13 phrases to 6. Manifest gains `data_current_through`; the executive summary surfaces the pattern-catalog review date. `EXAMPLE.md` and `INSTALL.txt` added to the package.

### 3.1.0
UX overhaul. Pre-scan intro and three-option mode menu (Compatibility / 4.7-only / Diagnose) replace auto-routing from trigger phrases. The scan does not read any file until the user picks a mode. Conversational outputs render as plain markdown with severity dots instead of fenced code blocks. Manifest and human-readable report are now opt-in at end of scan; no files are written by default.

### 3.0.2
Description rewrite for stronger trigger pull: imperative front-load and expanded symptom-path triggers covering post-4.7 behavior changes. "When to use" section collapsed to defer to the YAML description. No scan-logic changes.

### 3.0.1
Compat-mode intro paragraph corrected to match dual-mode behavior. Categorical-labels table completed for all 🔴 patterns.

### 3.0.0
Default mode shifts from one-way migration to bidirectional compatibility. The scanner now assumes the user actively uses both Opus 4.6 and 4.7 and never proposes a change that would degrade either model. 4.7-only optimization mode available via opt-in (trigger phrases or end-of-scan offer).

### 2.2.0
Polish pass. Response-shape brevity tightened to fixed line budgets, redundant patterns consolidated, citation fidelity audited end-to-end. Canonical first-response intro paragraph added.

### 2.1.0
Major UX overhaul. Conversational drill-down replaces flag-based batch apply: the scan surfaces every finding, then walks fixes one at a time. Severity-tier and citation-fidelity conventions formalized into the pattern schema.

### 2.0.0
Pattern catalogue expansion (config / API surface added alongside prose). Schema rewrite with per-pattern fidelity classification (`verbatim` / `paraphrased` / `practitioner`). Class-A semantic-judgment step added on top of structural matching to suppress non-directive false positives.

### 1.0.0
Initial pattern catalogue covering Claude Opus 4.6 → 4.7 migration issues in `CLAUDE.md`, `AGENTS.md`, subagents, skill bodies, `.claude/settings.json`, hook scripts, package manifests, and Anthropic SDK call sites.

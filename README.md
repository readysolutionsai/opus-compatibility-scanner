# Claude Opus 4.6 ⇄ 4.7 Compatibility Scanner

A long-lived hygiene tool for projects that actively use both Claude Opus 4.6 and Opus 4.7. Default mode never proposes a change that would degrade either model.

## What it does

Scans `CLAUDE.md`, `AGENTS.md`, subagent definitions, skill files, `settings.json`, hook scripts, package manifests, and Anthropic SDK call sites for known 4.6 → 4.7 migration issues. Surfaces every finding as Critical, Warning, or Info, then walks fixes one at a time with citations back to the relevant Anthropic documentation.

## Why it matters

Claude Opus 4.7 reads instructions more literally than 4.6 did. Directives that 4.6 silently inferred a sensible interpretation for (`be concise`, `default to no comments`, `never use any`) get applied at face value on 4.7, and your agent starts behaving differently. Configuration that worked on 4.6 (`thinking: { type: "enabled", budget_tokens: N }`, non-default `temperature`) returns HTTP 400 on 4.7. This scanner finds both classes of issues before you upgrade.

## Install

1. Extract `opus-compatibility-scanner.zip` (or `.skill` — same archive, different extension) into `~/.claude/skills/` (user-global) or `.claude/skills/` (project-scoped). The resulting directory should be named `opus-compatibility-scanner/`. (Some Claude clients also accept dragging the archive directly; manual extraction works in all environments.)
2. Start a new Claude Code session. The skill registers automatically.

*Heads-up on token cost:* the combined pattern catalogue and citation library load on first reference during a scan. Class A (prose) scans load ~28K tokens. Class B (config) scans load ~18K. Full-scope scans (both classes) load ~38K — under 4% of a 1M context window. Subsequent turns reuse the loaded references.

## Use

Slash command: `/opus-compatibility-scanner`. Or describe the task in plain English. Trigger phrases include:

- *"Scan my project for Opus 4.7 migration issues"*
- *"Is my CLAUDE.md ready for Opus 4.7?"*
- *"My agent is acting weird since I switched to 4.7. What broke?"*

The first thing you will see is a short intro plus three options:

1. **Compatibility scan** — for projects running both Opus 4.6 and 4.7. Holds back 4.7-only optimizations that would degrade 4.6.
2. **4.7-only scan** — for projects fully migrated off 4.6. Surfaces every finding with no holdback.
3. **Diagnose a behavior change** — you switched to 4.7 and something broke. Scans in compat mode, then surfaces symptom-mapped findings first.

Reply `1`, `2`, or `3`. The scan does not start until you pick a mode.

After the scan you will see an executive summary, then a per-finding drill-down. Reply `yes`, `skip`, or `explain` for each finding. Nothing is edited until you accept a specific fix.

**No files are written by default.** At the end of the scan, you can opt in to saving a JSON manifest and/or a Markdown report to `.claude/`. Both default to no.

**Want to see what a scan looks like?** A sample-output transcript is bundled at `EXAMPLE.md` in this package — a walkthrough of a compatibility scan on a synthetic project, including the executive summary and one Critical drill-down with the diff.

## What's covered

- **37 prose patterns** across `CLAUDE.md`, `AGENTS.md`, subagent definitions, and skill bodies (literal-following hazards, contradictions, scaffolding 4.7 no longer needs).
- **31 config patterns** (33 numbered, 2 reserved) across `.claude/settings.json`, hook scripts, package manifests, and Anthropic SDK call sites (deprecated `thinking` shapes, removed sampling parameters, beta header churn, SDK version floors, model ID strings).

Each finding cites a tier-classified source. Critical findings require a Tier 1 (Anthropic-authored) citation.

*Pattern catalog current through 2026-04-26.* Patterns are reviewed against Anthropic's published migration guide, model-deprecations page, and Agent SDK release notes; the date above marks the last review.

## How this differs from `claude-api`

Anthropic ships a bundled `claude-api` skill that automates model-ID swaps and breaking-parameter migrations in your Anthropic SDK call sites — `client.messages.create()` and friends. Run it for code-level changes.

This scanner covers what `claude-api` doesn't: prose patterns in your `CLAUDE.md`, `AGENTS.md`, subagent definitions, and skill bodies — the literal-following hazards that Opus 4.7 reads at face value. The two are complementary. Typical sequence: run `/claude-api migrate` for SDK code, then run this scanner for prose hygiene.

## Compatibility with Opus 4.6 and Opus 4.7

This scanner is built for projects that use both Opus 4.6 and Opus 4.7. The default mode (compatibility) never proposes a change that would degrade either model. A 4.7-only mode is available for users who have fully moved off 4.6.

Opus 4.6 is currently `Active` per Anthropic's [model-deprecations page](https://platform.claude.com/docs/en/about-claude/model-deprecations) (tentative retirement: not sooner than February 5, 2027). The Opus 4.6 / 4.7 differences are documented in Anthropic's [migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide).

**Mode routing.** The skill always asks you to pick a mode before scanning. Your invocation language pre-selects a default — phrases mentioning "compatibility" or the slash command pre-select compat mode; phrases mentioning "migration", "4.7-only", or "fully moved off 4.6" pre-select 4.7-only mode; symptom phrases ("my agent broke after the swap") pre-select Diagnose. You always confirm before any file is read.

**What compat mode does:**
- **Suppresses prose-level 4.7 tradeoffs.** Scaffolding deletions (*"think step by step"*, *"double-check the output"*, *"after every N tool calls, summarize"*), fixed-verbosity caps, and ask-clarifying-questions removals are useful on 4.7 but can degrade 4.6 — the scan holds them back unless you opt in via the end-of-scan offer or a 4.7-only-mode invocation.
- **Downgrades config-level 4.7 tradeoffs to guided apply.** Model-ID swaps, `thinking`-shape rewrites, sampling-parameter strips, and other 🔴-annotated config changes still surface, but as `guided` (you confirm each call site is 4.7-bound before the edit applies) rather than `direct`.

**What 4.7-only mode does:** restores all 🔴 findings to their default rewrites and apply tiers — appropriate when the project has fully moved off 4.6.

## Created by

Built by Mitchel Lairscey at [Ready Solutions](https://readysolutions.ai), a Claude-first AI integration consultancy. For help applying these patterns across a larger codebase, auditing a multi-team setup, or migrating production workloads to Opus 4.7 — reach out at readysolutions.ai.

## Disclaimer

This scanner identifies probable migration issues by pattern-matching against Anthropic's publicly documented Claude Opus 4.6 → 4.7 changes as of its publication date. It is an independent tool. It is not affiliated with, sponsored by, or endorsed by Anthropic. Pattern coverage is best-effort, not exhaustive. Anthropic's API and documentation may change after this scanner ships. Always verify proposed changes against your own codebase and Anthropic's current documentation before applying them. Provided as-is, without warranty of any kind. Use at your own discretion.

## Changelog

### 3.2.2
Pre-ship audit corrections. Foundational source quote in `patterns-prose.md` and the SKILL.md intro swapped to verbatim text from Anthropic's migration guide; the prior wording paraphrased the lead-in. Self-exclusion added so the scanner no longer reads files under any `opus-compatibility-scanner/` path during Phase 1. CB22 now cites the Sub-agents docs URL. MIT license declared in SKILL.md frontmatter and a LICENSE file shipped at the skill root. Token-cost estimate corrected to a per-class breakdown. New README section frames this scanner as complementary to Anthropic's bundled `claude-api` skill.

### 3.2.1
Audit fixes. Install docs lead with manual extraction; the unverified `.skill` drag-and-drop claim removed. End-of-scan offer trigger narrowed to suppressed prose-🔴 patterns only — config-🔴 findings already surface as guided in the drill-down, so the prior trigger could fire when nothing was actually held back. CB12 reserved alongside CB13, consolidating image-coordinate handling into CB27. New Tier 1 source: Anthropic's *Best practices for using Claude Opus 4.7 with Claude Code*. F4 and CB6 source citations upgraded to verbatim. Manifest schema field renamed `tier_counts` → `apply_tier_counts` to match the values it counts.

### 3.2.0
Pre-launch hardening. Internal `apply_tier` enum renamed `auto` → `direct` so the name reflects the safety stance — the scan still asks before every edit. Pattern catalog touch-ups rebalance severity and citation tier across a handful of entries. SKILL.md trigger description tightened and trimmed from 13 phrases to 6. Manifest gains `data_current_through`; the executive summary surfaces the pattern-catalog review date. `EXAMPLE.md` and `INSTALL.txt` added to the package.

### 3.1.0
UX overhaul. Pre-scan intro and three-option mode menu (Compatibility / 4.7-only / Diagnose) replace auto-routing from trigger phrases — the scan does not read any file until the user picks a mode. Conversational outputs render as plain markdown with severity dots instead of fenced code blocks. Manifest and human-readable report are now opt-in at end of scan; no files are written by default.

### 3.0.2
Description rewrite for stronger trigger pull: imperative front-load and expanded symptom-path triggers covering post-4.7 behavior changes. "When to use" section collapsed to defer to the YAML description. No scan-logic changes.

### 3.0.1
Compat-mode intro paragraph corrected to match dual-mode behavior. Categorical-labels table completed for all 🔴 patterns.

### 3.0.0
Default mode shifts from one-way migration to bidirectional compatibility — assumes the user actively uses both Opus 4.6 and 4.7 and never proposes a change that would degrade either model. 4.7-only optimization mode available via opt-in (trigger phrases or end-of-scan offer).

### 2.2.0
Polish pass. Response-shape brevity tightened to fixed line budgets, redundant patterns consolidated, citation fidelity audited end-to-end. Canonical first-response intro paragraph added.

### 2.1.0
Major UX overhaul. Conversational drill-down replaces flag-based batch apply: the scan surfaces every finding, then walks fixes one at a time. Severity-tier and citation-fidelity conventions formalized into the pattern schema.

### 2.0.0
Pattern catalogue expansion (config / API surface added alongside prose). Schema rewrite with per-pattern fidelity classification (`verbatim` / `paraphrased` / `practitioner`). Class-A semantic-judgment step added on top of structural matching to suppress non-directive false positives.

### 1.0.0
Initial pattern catalogue covering Claude Opus 4.6 → 4.7 migration issues in `CLAUDE.md`, `AGENTS.md`, subagents, skill bodies, `.claude/settings.json`, hook scripts, package manifests, and Anthropic SDK call sites.

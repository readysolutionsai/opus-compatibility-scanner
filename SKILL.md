---
name: opus-compatibility-scanner
description: When the user asks for a 4.6 / 4.7 compatibility scan, run this skill rather than improvising the analysis. Covers CLAUDE.md, AGENTS.md, subagents, skill files, settings.json, hooks, package manifests, and Anthropic SDK call sites. Triggers - "compatibility scan", "scan for opus 4.7 migration", "is my project ready for opus 4.7", "my agent is acting weird since I switched to 4.7", "scan for 4.7-only optimizations", slash command /opus-compatibility-scanner.
allowed-tools: Read, Glob, Grep, Bash, Edit
user-invocable: true
argument-hint: '"scan just my CLAUDE.md" / "scan for 4.7-only optimizations"'
license: MIT
---

# Opus 4.6 ⇄ 4.7 Compatibility Scanner

Scans a Claude Code project for two classes of issues that need fixing when migrating from Claude Opus 4.6 to Claude Opus 4.7:

1. **Prose-pattern issues** in CLAUDE.md, AGENTS.md, subagent definitions, and skill bodies — directives 4.6 silently inferred a sensible interpretation for, but 4.7 will read literally. Anthropic's own migration guide flags this as the headline behavioral change: *"Claude Opus 4.7 interprets prompts more literally and explicitly than Claude Opus 4.6 ... It will not silently generalize an instruction from one item to another, and it will not infer requests you didn't make."*
2. **Configuration / API artifact issues** in `.claude/settings.json`, hook scripts, package manifests, and any source code that calls the Anthropic SDK — model IDs, deprecated `thinking` modes, removed parameters, beta headers, SDK version floors, etc.

This skill is **recommend-only by default.** It produces a report; it never modifies files without an explicit per-fix confirmation in conversation.

Announce at start: "I'm using the opus-compatibility-scanner skill to scan your project for Opus 4.6 / 4.7 compatibility issues."

**Core principle:** Never modify a file the user hasn't explicitly approved. The scan is a diagnostic; the user decides what to apply, one fix at a time.

## When to Load References

This skill has substantial domain knowledge split into `references/`. Load each file only when needed:

- **`references/patterns-prose.md`** — Load during Phase 2 when matching patterns against CLAUDE.md, AGENTS.md, subagent definitions, or skill SKILL.md files. The pattern count is enumerated dynamically at scan time.
- **`references/patterns-config.md`** — Load during Phase 2 when matching against `.claude/settings.json`, hook scripts, package manifests, or Anthropic SDK call sites. The pattern count is enumerated dynamically at scan time.
- **`references/sources.md`** — Load when rendering a finding and need to cite a source with its tier (1-4). Every finding must include a citation.
- **`references/report-template.md`** — Load during Phase 4 when rendering the executive-summary report.
- **`references/contradictions.md`** — Load during Phase 3 when running pair-detection patterns (G1, G2, R3, M1). Documents the heuristic + LLM-judgment algorithm.
- **`references/symptoms.md`** — Load when the user describes a symptom ("my agent is acting weird since 4.7") rather than asking for a general scan. Maps symptoms back to pattern IDs.

## When to use

See the trigger list in the YAML description above. Also run proactively whenever the user mentions upgrading from 4.6 to 4.7, when they describe any behavior regression after a model swap, or when you notice them about to flip the model.

## How the scan works

The scan is one pass with five steps: a Phase 0 introduction and mode selection, then three read-only phases (discovery, pattern matching, severity classification), then a fourth phase that renders the executive-summary report. The user picks a mode in Phase 0 before any file is read. After the scan, the response presents findings in priority order, and file modifications happen only when the user accepts a specific fix in conversation, one at a time — see "How to respond to the user" below.

### Phase 0 — Introduce and select mode

Post the intro and option list to the conversation. Wait for the user's reply. Do not run discovery, pattern matching, or any tool until the user has picked a mode.

**Default-mode hint from invocation phrasing.** Use the user's invocation language to pre-select an option, but do not skip the prompt:
- Slash command (`/opus-compatibility-scanner`) or any compatibility-trigger phrase → option 1 is the suggested default.
- Any 4.7-only-trigger phrase ("I'm 4.7-only now", "after upgrading to 4.7", "since switching to 4.7") → option 2 is the suggested default.
- Any symptom phrase ("my agent is acting weird since 4.7", "4.7 broke X", "model swap broke X") → option 3 is the suggested default.

If pre-selection is ambiguous, present the menu without highlighting a default.

**The user sees this — render as plain markdown, do NOT wrap in a fenced code block. Do not re-introduce the skill — the announcement line above has already done that. Lead with the value description, then the menu.**

I check for instructions Opus 4.7 reads more literally than 4.6, plus config and SDK call sites that may now error or behave differently. I'll surface findings, then walk fixes one at a time so you stay in control of every edit.

What would you like to do?

1. **Compatibility scan** — for projects running both Opus 4.6 and 4.7. Holds back 4.7-only optimizations that would degrade 4.6.
2. **4.7-only scan** — for projects fully migrated off 4.6. Surfaces every finding with no holdback.
3. **Diagnose a behavior change** — you switched to 4.7 and something broke. I'll scan in compat mode, then surface symptom-mapped findings first.

Reply `1`, `2`, or `3`.

**Routing rules after the user replies:**
- Option 1 → compat mode for Phase 3 (suppression + apply-tier downgrades active).
- Option 2 → 4.7-only mode for Phase 3 (no suppression, no downgrade).
- Option 3 → compat mode + symptom relevance ranking. Ask one short follow-up first: *"Briefly, what's the behavior change?"* Use the reply to look up symptom-to-pattern mappings in `references/symptoms.md`. After Phase 4, the executive summary prepends a **"Most relevant to your symptom"** subsection listing the matching findings before the standard severity rollup. If no symptom-mapped finding hits, fall through to the standard executive summary plus an offer to walk `references/symptoms.md` interactively.

If the user replies with anything other than `1`, `2`, `3`, or a synonym, ask once more for clarity rather than guessing.

### Phase 1 — Discovery

Enumerate every file in scope. Use Glob and Bash from the project root.

**Prose files (Class A — patterns-prose.md applies):**

```bash
# Project-level CLAUDE.md (any depth)
find . -type f \( -iname "CLAUDE.md" -o -iname ".claude.md" -o -iname ".claude.local.md" -o -iname "AGENTS.md" \) ! -path "*/node_modules/*" ! -path "*/.git/*" ! -path "*opus-compatibility-scanner*" 2>/dev/null
```

```bash
# Subagent + skill definitions
find .claude/agents -type f -name "*.md" ! -path "*opus-compatibility-scanner*" 2>/dev/null
find .claude/skills -type f -name "SKILL.md" ! -path "*opus-compatibility-scanner*" 2>/dev/null
```

Also check for the user-global file: `~/.claude/CLAUDE.md`. If it exists, scan it too — but flag findings there as `[GLOBAL]` so the user knows the change affects all their projects.

```bash
# User-global memory file (flag findings as [GLOBAL] in the report)
ls -la ~/.claude/CLAUDE.md 2>/dev/null
```

When scanning `~/.claude/CLAUDE.md`, every finding from that file is rendered with `[GLOBAL]` prefix in the executive summary and drill-down. Editing the global file requires the user to reply with an explicit phrase that names the file (see "Constraints on what gets edited" below), since the change affects every project on the user's machine.

**Config / API files (Class B — patterns-config.md applies):**

```bash
# Settings + hooks
ls -la .claude/settings.json .claude/settings.local.json 2>/dev/null
find .claude/hooks -type f ! -path "*opus-compatibility-scanner*" 2>/dev/null
find .claude -type f -name "*.json" ! -path "*opus-compatibility-scanner*" 2>/dev/null  # MCP, etc.

# Package manifests
ls -la package.json requirements.txt pyproject.toml Pipfile Cargo.toml go.mod 2>/dev/null

# Source files likely to contain API calls or model IDs
grep -rln "anthropic\|claude-opus\|claude-sonnet\|claude-haiku" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.mjs" --include="*.py" --include="*.go" --include="*.rs" --include="*.java" --include="*.cs" --include="*.toml" --include="*.yaml" --include="*.yml" --include="*.json" --include="*.sh" . 2>/dev/null | head -100
```

Discovery runs silently. The file count surfaces in the executive-summary first line (`Scanned <N> files. Found: …`).

### Phase 2 — Pattern matching

Load the pattern catalogues:

- [references/patterns-prose.md](references/patterns-prose.md) — Class A patterns (count enumerated dynamically at scan time)
- [references/patterns-config.md](references/patterns-config.md) — Class B patterns (count enumerated dynamically at scan time)
- [references/sources.md](references/sources.md) — citation library (Tier 1 = official Anthropic; Tier 3 = practitioner consolidation)
- [lib/regex-engine.md](lib/regex-engine.md) — Grep-tool-only mandate (PCRE2)

**Grep tool mandatory.** Pattern matching against file contents goes through the Claude Code Grep tool (PCRE2). Bash `grep` is NEVER used for pattern matching. Bash is reserved for Phase 1 discovery (`find`, `ls`, file enumeration). See `lib/regex-engine.md` for rationale.

**Per-pattern dispatch on `match.engine`:**
- `pcre2` → Grep tool with the pattern's `match.pattern` regex. Returns structured `{file, line, column, match_text}`.
- `structural` → call the named check function. Currently defined: `messages-array-last-element-is-assistant` (CB6 — parses the source's `messages: [...]` array and verifies the LAST element does NOT have `role: "assistant"`); `cross-file-rule-diff` (M1 — diffs declared rules across parent/child CLAUDE.md files for same-rule contradictions).

**Class B match dispatch additionally consults `applies_to_files` glob** to scope the search before invoking Grep.

**JSONC files** (`.claude/settings.json`, `.claude/settings.local.json`) — when matching Class B patterns inside these files, mentally strip `//` line comments and `/* */` block comments before reasoning about structure. Don't write a parser for it; the patterns themselves are robust to comment whitespace.

Every match produces a finding record per the manifest schema: `{file, line, col, pattern_id, matched_substring, severity, tier, apply_tier}`.

### Phase 3 — Severity classification

**Mode is already set (Phase 0).** Phase 3 branches on the mode the user picked:
- Option 1 → **Compatibility mode**.
- Option 2 → **4.7-only mode**.
- Option 3 → **Compatibility mode** plus symptom relevance ranking applied at Phase 4.

In compatibility mode:
1. Filter out every prose finding whose pattern carries `**Compatibility:** 🔴` in its catalogue record. Suppressed findings are still counted (see "Suppression dedup" below) for the end-of-scan offer.
2. For every config finding whose pattern carries `**Compatibility:** 🔴`, downgrade `apply_tier` from `direct` to `guided`. The drill-down prompt for those findings explicitly asks the user to confirm the call site is 4.7-bound before the edit applies.

In 4.7-only mode: no suppression, no apply-tier downgrade. All 🔴 findings surface with their default rewrites and apply tiers.

**Suppression dedup.** When two or more 🔴 prose patterns match overlapping spans on the same `(file, line)` pair (matched-substring spans overlap by ≥1 character), collapse to a single suppressed-finding count and a single drill-down record. The drill-down entry on opt-in lists all merged pattern IDs in its header for traceability. The dedup applies to the suppressed set only; non-🔴 dedup behavior is unchanged.

**Categorical labels for the end-of-scan offer.** When summarizing suppressed findings:
| Pattern ID | Category label |
|---|---|
| F3, F6, F7 | scaffolding deletion(s) |
| A4, A5 | verbosity-cap removal(s) |
| R6 | ask-clarifying-questions removal(s) |

Group suppressed findings by category, then count and pluralize. Example output: *"2 scaffolding deletions, 1 verbosity-cap removal."*

1. Apply severity per pattern (as-declared in the pattern schema).
2. Enforce Critical-requires-Tier-1 filter (demote any Critical with Tier > 1 to Warning). Also enforce: if `source.fidelity == practitioner`, cap severity at Warning regardless of declared severity.
3. **Class A semantic-judgment step** — for every Class A match where the pattern's `match.high_precision` is `false` (the default):
   - Read ±5 lines around the matched line. Pass to calling Claude with prompt: "Is this match a directive *to Claude* (the model executing this CLAUDE.md / agent / skill), or is it descriptive prose / user-copy / website-behavior description / operational guardrail / example block? Reply DIRECTIVE or NOT_DIRECTIVE."
   - If `NOT_DIRECTIVE`, suppress the finding. Manifest annotation: `suppressed_by: phase3_semantic_check`.
   - If `DIRECTIVE`, retain the finding for severity classification.
   - **Why both `applies_to_files` and this step:** `applies_to_files` is the structural filter (path-based, deterministic, fast). The semantic step catches false positives within in-scope files where the regex matched a non-directive sentence (e.g., "users who **prefer** dark mode" inside a CLAUDE.md describing site behavior; "**NEVER** write an entire long file (100+ lines)" as an operational write-size guardrail rather than a Claude verbosity cap). Two different failure modes, two different mechanisms.
   - **Cost control:** the semantic check is skipped when `match.high_precision: true`. Pattern authors opt in to `high_precision` only when the regex is anchored enough that semantic disambiguation isn't needed (literal API field names, exact directive phrases that only appear in instruction contexts, etc.). Class B patterns ignore the field — the semantic step is Class-A-prose-only.
4. Deduplicate overlapping matches — if one line matches multiple patterns, merge into ONE finding listing all pattern_ids; rewrite is the most conservative applicable.
5. Run pair-detection patterns (G1, G2, R3, M1) via the hybrid heuristic + LLM-judgment algorithm. Full algorithm in `references/contradictions.md`.
6. All findings (Tier 1, 2, 3) are surfaced; the executive summary groups them by severity, not by tier. Tier is shown alongside the citation so the user can judge weight.

### Phase 4 — Report rendering

The first response after the scan is the executive summary, rendered as plain markdown (the intro was already shown in Phase 0; do not repeat it). Concise rewrites are the default per-finding rendering; the expanded rewrite is shown only when the user asks `explain` during drill-down.

**No files are written during the scan.** The drift check at fix time works from the live file (re-reading ±3 lines around the matched line, see "Response shape: applying a fix" below), so the manifest is not required for safe edits. After the scan flow completes, the skill offers — but never silently writes — a manifest (`.claude/opus-4.7-migration-manifest.json`) and/or a human-readable report (`.claude/opus-4.7-migration-report.md`) for users who want an audit trail or a way to resume in a later session. Both offers default to no. The manifest schema in `lib/manifest-schema.json` describes the opt-in shape.

## How to respond to the user

The intro and option list were already shown in Phase 0 — never repeat them. After the scan completes, the first response IS the executive summary, rendered as plain markdown. Wait for the user's reply before drilling down.

**Diagnose mode addendum.** When mode is option 3 (Diagnose), the executive summary prepends a **"Most relevant to your symptom"** subsection listing findings whose pattern IDs map to the symptom the user described. The standard severity rollup follows. If no symptom-mapped finding hits, omit the subsection and offer to walk `references/symptoms.md` interactively at the end.

### Response shape: executive summary (first response)

**The user sees this — render as plain markdown, do NOT wrap in a fenced code block. Use the literal severity emoji 🔴 🟡 🔵 inline so each row is scannable. Replace `<...>` placeholders with actual values; omit any row whose count is zero EXCEPT the Critical row, which always renders.**

**Scanned `<N>` files.**
*Pattern catalog current through 2026-04-26.*

- 🔴 **Critical:** `<C>` — `<one-line gist>`
- 🟡 **Warning:** `<W>` — `<one-line gist>`
- 🔵 **Info:** `<I>` — `<one-line gist>`

Recommend fixing Critical first. Top is `<pattern name>` in `<path>:<line>`.

Walk through the Criticals? (`yes` / `skip` / `stop`)

---

If C is zero, render this shape instead:

**Scanned `<N>` files.** *Pattern catalog current through 2026-04-26.* No critical issues. `<W>` warnings, `<I>` info — want to look?

---

If the mode is Diagnose (option 3) and at least one symptom-mapped finding hit, prepend this subsection above the standard severity rollup:

**Most relevant to "`<short symptom paraphrase>`":**

- `<severity emoji>` `<pattern name>` in `<path>:<line>` — `<one-line gist>`
- `<severity emoji>` `<pattern name>` in `<path>:<line>` — `<one-line gist>`

Then continue with the normal **Scanned `<N>` files.** rollup below.

### Response shape: drill-down (when the user asks)

If the user replies with anything that means "yes, walk me through" or "tell me about the Criticals" or "show me #1", respond with one finding at a time.

**The user sees this — render as plain markdown, do NOT wrap in a fenced code block. The rewrite goes inside a single-line blockquote so it stands apart from the prose without becoming a monospace box:**

🔴 **Critical #1 of `<C>`** — `<pattern name>` · `<path>:<line>`

*What 4.7 will do:* `<one or two sentences from the pattern doc.>`

*Fix:*
> `<concise rewrite — never the expanded form by default>`

*Source:* Tier `<N>` — [`<short citation label>`](`<URL>`)

Apply? (`yes` / `skip` / `explain`)

---

The severity emoji must match the finding's severity: 🔴 Critical, 🟡 Warning, 🔵 Info. The pattern-name line uses `·` (middle dot) as the separator between pattern name and file:line.

The user's reply drives the next turn:
- `yes` → apply the single fix; show diff; move to next finding.
- `skip` (or `next`) → move to next finding without applying.
- `explain` (or `more detail`) → render the expanded rewrite + full rationale for that one finding only; re-prompt for apply.
- `stop` → end the drill-down; summarize what was applied/skipped.
- Any question about the finding → answer, then re-prompt.

**End-of-tier handoff — render as plain markdown, do NOT wrap in a fenced code block:**

Done with Criticals — `<X>` applied, `<Y>` skipped. Move on to Warnings?

Same flow for Warnings, then Info.

### Response shape: end-of-scan offer (compat mode only)

After the final summary in compatibility mode, if any prose-🔴 patterns matched (counted post-dedup), append the offer. Config-🔴 findings are already surfaced as guided in the drill-down — they are not held back, so they do not trigger this offer.

**The user sees this — render as plain markdown, do NOT wrap in a fenced code block:**

There are also `<N>` 4.7-only optimizations I held back: `<categorical summary>`.

Want to see them now? Reply `yes` only if you've fully moved off 4.6.

`<categorical summary>` is generated from the categorical-labels lookup table in Phase 3. If the user replies `yes`, drill-down resumes for the suppressed prose findings using their default rewrites and apply tiers. If the user replies anything else, the scan ends.

If `<N>` would be zero, omit the offer entirely.

### Response shape: end-of-scan manifest offer (both modes)

After the final summary AND after the end-of-scan optimization offer (if shown), append the manifest offer. Default is no file is written. The two output files are independent — the user can ask for one, both, or neither.

**The user sees this — render as plain markdown, do NOT wrap in a fenced code block:**

Want to save these findings to disk? Both default to no.

- **Manifest** (`.claude/opus-4.7-migration-manifest.json`) — structured JSON with finding IDs, file:line anchors, and tier assignments. Useful for resuming this scan in a later session or feeding results into CI.
- **Report** (`.claude/opus-4.7-migration-report.md`) — human-readable Markdown summary of every finding.

Reply `manifest`, `report`, `both`, or `no`.

Write the requested file(s) only after the user replies. The manifest schema is in `lib/manifest-schema.json`. The report follows the layout in `references/report-template.md`. If the user replies `no` or anything other than the four options, write nothing.

### Response shape: applying a fix

When the user accepts a fix:

1. Re-read ±3 lines around the matched line (drift check, inline). If the file has changed since scan, abort that one fix with a one-line note and move on.
2. Confirm the lifted `old_string` is exactly one occurrence in the file. If zero → abort with note. If >1 → re-prompt the user with the surrounding context to disambiguate, or skip.
3. Apply the Edit. Show the diff inline.
4. Move to the next finding.

**Drift-check failure mode.** When the matched substring is no longer present at the recorded line number (file changed since scan), the fix is aborted with a one-line note (e.g., *"Skipped — `path:line` has changed since the scan."*) and the loop moves on to the next finding. The `old_string` uniqueness check in step 2 covers the same failure implicitly; calling it out here so anyone debugging an aborted fix knows the path.

No git snapshot commits. No branch prompts. No batch confirmations. No dry-run mode. The user is in the loop on every single edit, but the loop is cheap because each turn is one short prompt + one short reply.

### Constraints on what gets edited

- **Read-only by default.** The scan reads files; nothing is edited until the user accepts a specific fix in conversation.
- **Source-file edits require explicit confirmation.** For Class B (config/API) findings whose target file is application source code (`*.ts`, `*.py`, etc.), the prompt is explicit: *"This will edit `src/api/client.ts`. Proceed?"* The user always confirms source-file edits.
- **Global-file edits require an explicit phrase.** Editing `~/.claude/CLAUDE.md` requires the user to reply with something like *"yes, edit my global CLAUDE.md"*. A bare "yes" is not enough; the prompt explicitly names the file because the change affects every project on the user's machine.
- **Manual-tier patterns are proposals only.** Patterns flagged `apply_tier: manual` (cross-file diffs, structural changes the scanner can't safely apply) are presented as recommendations: *"Here's what I'd recommend changing manually."* The skill does not Edit them.
- **No retry on Edit failure.** If the Edit tool returns any error, the finding is aborted and the loop moves on. The skill never improvises a different `old_string`.
- **Pattern-overlap dedup.** If one line matches three patterns, the scan emits ONE finding with a combined rewrite. Prevents sequential edits from stomping on each other.
- **Self-exclusion.** Files whose path contains `opus-compatibility-scanner/` are unconditionally skipped during Phase 1 discovery. The scanner does not read its own catalogue. This prevents the scanner's `before` examples (matched substrings inside diff blocks documenting bad patterns) from generating false-positive findings.

### Contradiction management

Pair-detection patterns (G1, G2, R3 in-file; M1 cross-file) run during Phase 3 using a hybrid heuristic + LLM-judgment algorithm. The heuristic catches the obvious cases deterministically; the LLM judgment defaults to **Warning + precedence-block recommendation when uncertain** — Critical is reserved for cases where the directives target the same axis with opposite verbs. The full algorithm specification is in `references/contradictions.md`.

## Usage examples

User says: *"Scan this repo for opus 4.7 migration issues."*

Run all four read-only phases. Show the executive summary. Wait for the user's reply before drilling down or editing anything.

User says: *"Check just my CLAUDE.md."*

Phase 1 narrows to a single file. Phases 2-4 as normal. Same executive-summary response shape.

User says: *"Fix the criticals"* / *"apply all"* / *"fix everything"*

Refuse batch apply: *"I'll walk through them one at a time so you can confirm each — that's the safe way."* Then start the drill-down at Critical #1.

User says: *"I switched to 4.7 and now my agent does X weird thing — what's broken?"*

Run a focused scan first, then if the executive summary doesn't surface the issue, walk the user through the symptom-to-pattern mapping in `references/symptoms.md`.

User asks for batch flags like *"apply all"*, *"--thorough"*, or similar:

Answer naturally without flag syntax: *"All findings are surfaced by default — let's start with the Criticals."* Then start the drill-down.

## Integration

- **`claude-api`** (Anthropic-bundled skill, available in every Claude Code session): If the user wants to migrate application code (Anthropic SDK call sites) in depth — not just surface findings — `claude-api` handles the code-level migration. This skill flags the diffs; `claude-api` applies them.
- **Other CLAUDE.md / debugging skills** (e.g., a `claude-md-improver` for general CLAUDE.md hygiene, or a `systematic-debugging` skill for symptom-driven investigation): if installed in the user's project, complement this scanner. If not installed, the recommendations here stand on their own — every finding includes a citation and a concrete rewrite the user can apply by hand.

# Prose-Pattern Catalogue (Class A)

Patterns that appear in CLAUDE.md, AGENTS.md, subagent definitions, skill SKILL.md bodies, and any other prose system-prompt-style file. Each pattern lists:

- **ID** — short stable identifier (referenced from the report)
- **Match regex** — what to grep for (case-insensitive unless noted)
- **Severity** — Critical / Warning / Info
- **Why 4.6 was fine** — the inferential leap 4.6 made
- **How 4.7 mis-reads** — the literal interpretation problem
- **Rewrite** — a concrete diff template
- **Source** — citation tier and URL

> All patterns share one foundational source:
> **Anthropic — Migration guide:** *"More literal instruction following: Claude Opus 4.7 interprets prompts more literally and explicitly than Claude Opus 4.6, particularly at lower effort levels. It will not silently generalize an instruction from one item to another, and it will not infer requests you didn't make. The upside of this literalism is precision and less thrash."*
> — https://platform.claude.com/docs/en/about-claude/models/migration-guide

---

## Pattern schema

Every Class A pattern (and every Class B pattern) is authored against a fixed YAML schema. The schema is the contract between pattern authors and the scanner runtime — Phase 2 (matching), Phase 3 (severity classification + semantic check), and Phase 4 (report rendering) all read these fields.

```yaml
id: A1
name: Unqualified "never use X" for a language construct
class: A                            # A (prose) | B (config/API)
category: Absolute rules without scope qualifiers
severity: warning                   # critical | warning | info
tier: 3                             # 1 (Anthropic) | 2 (Anthropic staff) | 3 (practitioner) | 4 (forum)
apply_tier: manual                  # direct | guided | manual
template_version: 2.1.0             # bumps when rewrite template changes
match:
  engine: pcre2
  pattern: "(?i)\\b(never|don'?t|do not)\\s+(use|introduce|allow|permit)\\b..."
  flags: [multiline]                # optional
  high_precision: false             # Class A only — when false (default), Phase 3 runs the
                                    # semantic-judgment check on each match. Pattern authors set
                                    # true only when the regex is anchored enough that semantic
                                    # disambiguation isn't needed (literal API field names, exact
                                    # directive phrases that only appear in instruction contexts).
                                    # Class B patterns ignore this field.
rewrite:
  kind: multiline                   # single-token | single-line | multiline
  concise: |                        # ≤3 added lines — what the user sees by default
    Do not introduce `any` in author-written code (src/, packages/*/src/).
    Exceptions: `*.generated.ts`, `types/vendor/*`, `// any-ok:` comments.
  expanded: |                       # full rationale + exception list — surfaced when the user asks "explain"
    Do not introduce `any` in code we author under `src/` or `packages/*/src/`.
    Exceptions: generated files (`*.generated.ts`, `*.pb.ts`), third-party
    type shims in `types/vendor/`, and adapter boundaries documented with
    `// any-ok:` comments explaining why `unknown` won't work.
why_46_was_fine: >
  Inferred the rule meant "in code we author"; exempted generated code, vendor
  shims, adapter boundaries.
why_47_misreads: >
  Applies the rule literally across the entire codebase. May refactor generated
  protobuf, rewrite vendor shims, or strip exemptions where the construct is
  the only working option.
source:
  tier: 3
  url: https://github.com/abhishekray07/claude-md-templates
  foundational_tier_1: https://platform.claude.com/docs/en/about-claude/models/migration-guide
  fidelity: practitioner            # verbatim | paraphrased | practitioner
                                    # `verbatim` requires exact-string Anthropic quote in `quote` field
                                    # `paraphrased` allows author wording; intent must match cited page
                                    # `practitioner` is Tier 3+ — never authorizes Critical severity
  quote: null                       # required when fidelity == verbatim; null otherwise
applies_to_files:                   # glob patterns — deterministic structural filter
  - "CLAUDE.md"
  - ".claude.md"
  - ".claude/agents/*.md"
  - ".claude/skills/*/SKILL.md"
  - "AGENTS.md"
```

## Schema authoring conventions

These conventions govern pattern records.

1. **severity-tier** — `severity: critical` requires `tier: 1`. Any Critical pattern citing Tier 2+ should be demoted to Warning unless a Tier 1 source is found.
2. **fidelity-quote** — `source.fidelity: verbatim` requires `source.quote` to be an exact substring of the cited page body (allowing whitespace/punctuation normalization). If the quote isn't a substring, set `fidelity: paraphrased` and `quote: null`.
3. **fidelity-practitioner-cap** — `source.fidelity: practitioner` caps severity at Warning regardless of declared severity.
4. **kind-multiline-implies-manual** — `rewrite.kind: multiline` forces `apply_tier: manual`. Multi-line rewrites cannot be applied via a single Edit safely.
5. **concise-line-cap** — `rewrite.concise` must add ≤3 lines beyond the original. The concise template is the default user-facing rendering; the cap is the readability protection.

## Field reference

Every field above, enumerated with valid values, defaults, and conditional requirements.

| Field | Valid values | Default | Required when |
|---|---|---|---|
| `id` | unique short identifier (e.g. `A1`, `CB15`, `Global1`) | none | always |
| `name` | human-readable pattern name | none | always |
| `class` | `A` (prose) \| `B` (config/API) | none | always |
| `category` | free-text category label (e.g. "Absolute rules without scope qualifiers") | none | always |
| `severity` | `critical` \| `warning` \| `info` | none | always |
| `tier` | `1` \| `2` \| `3` \| `4` | none | always |
| `apply_tier` | `direct` \| `guided` \| `manual` | none | always |
| `template_version` | semver string (e.g. `2.0.0`) | `2.0.0` | always; bump on rewrite-template change |
| `match.engine` | `pcre2` \| `structural` | `pcre2` | always |
| `match.pattern` | PCRE2 regex string | none | when `engine: pcre2` |
| `match.flags` | list of regex flags (e.g. `[multiline]`) | `[]` | optional |
| `match.high_precision` | `true` \| `false` | `false` | Class A only; Class B ignores. Set `true` only when the regex is anchored enough that semantic disambiguation isn't needed (literal API field names, exact directive phrases that only appear in instruction contexts) |
| `rewrite.kind` | `single-token` \| `single-line` \| `multiline` | none | always |
| `rewrite.concise` | string, ≤3 added lines | none | always; default report output |
| `rewrite.expanded` | string, full rationale + exception list | none | always; surfaced when the user asks "explain" during drill-down |
| `why_46_was_fine` | string explaining 4.6's inferential behavior | none | always |
| `why_47_misreads` | string explaining 4.7's literal interpretation | none | always |
| `source.tier` | `1` \| `2` \| `3` \| `4` | none | always |
| `source.url` | Anthropic or practitioner URL | none | always |
| `source.foundational_tier_1` | Tier 1 anchor URL | none | optional; required when `tier > 1` to allow Warning-cap fallback |
| `source.fidelity` | `verbatim` \| `paraphrased` \| `practitioner` | none | always |
| `source.quote` | exact-string Anthropic quote | `null` | non-null when `fidelity: verbatim`; null otherwise |
| `applies_to_files` | list of glob patterns | none | Class A always; Class B optional |

**Structural-check exception.** A small number of patterns (CB6 prefill detection, M1 cross-file contradictions) cannot be expressed as a single grep. These declare `match.engine: structural` and reference a named check function in lieu of `match.pattern`. Structural patterns are always `apply_tier: manual` — they never call Edit.

---

## Category A — Absolute rules without scope qualifiers

### A1 — Unqualified `never use X` for a language construct

- **Match regex:**
  ```yaml
  - "(?i)\\b(never|don'?t|do not|avoid|ban|prohibit|forbid)\\s+(use|introduce|allow|permit|write|add)\\b.*\\b(any|var|class|export default|console\\.log|comment)\\b"
  ```
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred the rule meant "in code we author" — exempted generated code, vendor type shims, third-party `.d.ts`, and adapter boundaries.
- **How 4.7 mis-reads:** Applies the rule literally across the entire codebase. May refactor generated protobuf, rewrite vendor shims, or strip exemptions where the construct is the only working option.
- **Concise rewrite:**
  ```diff
  - Never use `any` in TypeScript.
  + Do not introduce `any` in author-written code (src/, packages/*/src/).
  + Exceptions: `*.generated.ts`, `types/vendor/*`, `// any-ok:` comments.
  ```
- **Expanded rewrite:**
  ```diff
  - Never use `any` in TypeScript.
  + Do not introduce `any` in code we author under `src/` or `packages/*/src/`.
  + Exceptions: generated files (`*.generated.ts`, `*.pb.ts`), third-party
  +   type shims in `types/vendor/`, and adapter boundaries documented with
  +   `// any-ok:` comments explaining why `unknown` won't work.
  ```
- **Source:** Tier 3 — *MindStudio*, *KeepMyPrompts* practitioner consolidation; foundational Tier 1 quote above. Real example: https://github.com/abhishekray07/claude-md-templates

---

### A2 — Unqualified "always use X" for a tool or framework

- **Match regex:** `(?i)\b(always|every package|in all packages|all of our|everywhere)\b.*\b(use|require|enforce)\b.*\b(typescript|strict|pnpm|npm|yarn|bun|black|prettier|eslint|ruff|mypy|pytest)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred that "all packages" referred to packages of the matching language family.
- **How 4.7 mis-reads:** May try to enforce TS-strict on a Go package, may rewrite a `package-lock.json` to `pnpm-lock.yaml`, or flag non-matching packages as non-compliant.
- **Concise rewrite:**
  ```diff
  - Always use TypeScript strict mode in all packages.
  + In packages with a `tsconfig.json`, require `"strict": true`.
  + Non-TypeScript packages are out of scope.
  ```
- **Expanded rewrite:**
  ```diff
  - Always use TypeScript strict mode in all packages.
  + In TypeScript packages (those with a `tsconfig.json`), require strict mode
  +   (`"strict": true`). Non-TypeScript packages are out of scope for this rule.
  ```
- **Source:** Tier 3 — real example https://github.com/MuhammadUsmanGM/claude-code-best-practices

---

### A3 — "Do not create files" / "Do not create documentation"

- **Match regex:** `(?i)\b(do not|don'?t|never)\s+(create|generate|write|add)\s+(new|any)?\s*(file|files|markdown|docs?|documentation|README|\.md)\b`
- **Severity:** Warning (capped — practitioner-fidelity citation; promote to Critical only with a Tier 1 source)
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred the rule targeted *unsolicited summary/report docs*. Naturally excluded test files, new components, migrations, and config files needed to complete the requested task.
- **How 4.7 mis-reads:** Taken literally, blocks creation of test files, new React components, new migrations — anything the user would reasonably expect as part of the implementation. May ask permission on each file or skip them entirely.
- **Concise rewrite:**
  ```diff
  - Never create documentation files (*.md) or README files unless explicitly requested.
  + Do not create new Markdown summary/report/status files (SUMMARY.md, NOTES.md, CHANGES.md) unless I ask.
  + Source files, tests, migrations, configs needed for the task are allowed.
  ```
- **Expanded rewrite:**
  ```diff
  - Never create documentation files (*.md) or README files unless explicitly requested.
  + Do not create new Markdown summary/report/status files (e.g.,
  +   `SUMMARY.md`, `NOTES.md`, `CHANGES.md`) unless I explicitly ask for one.
  + Creating new source files, test files, migration files, and config files
  +   required to complete the requested implementation is allowed and expected.
  ```
- **Source:** Tier 3 mirror of Anthropic's own default Claude Code system prompt. Real example: https://gist.github.com/markomitranic/26dfcf38c5602410ef4c5c81ba27cce1

---

### A4 — Fixed-line-count or fixed-verbosity caps ("under 4 lines", "MUST be concise")

- **Match regex:** `(?i)\b(MUST|always|never exceed|keep)\b.*\b(under|fewer than|less than|<=?|≤)\s*\d+\s*(lines?|words?|sentences?|tokens?)\b`
  - Also: `(?i)\bbe\s+concise\b|\bminimi[sz]e\s+tokens?\b`
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** **Critical**
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "Positive examples showing how Claude can communicate with the appropriate level of concision tend to be more effective than negative examples or instructions that tell the model what not to do."`
- **Why 4.6 was fine:** 4.6's default verbosity was high; "be concise" trimmed preamble without starving architecture explanations of detail.
- **How 4.7 mis-reads:** Anthropic explicitly: *"Response length calibrates to perceived task complexity rather than defaulting to a fixed verbosity."* 4.7 is already terser. A literal 4-line cap truncates architecture answers, migration explanations, and root-cause analyses.
- **Concise rewrite:**
  ```diff
  - MUST answer concisely with fewer than 4 lines.
  + For lookups and confirmations, keep replies under 4 lines.
  + For architecture, debugging, migrations, design trade-offs: full reasoning.
  + Skip preamble/postamble.
  ```
- **Expanded rewrite:**
  ```diff
  - MUST answer concisely with fewer than 4 lines.
  + For simple lookups and confirmations, keep replies under 4 lines.
  + For architecture questions, migration plans, bug post-mortems, and design
  +   trade-off analyses, give full reasoning. Length should match task complexity.
  + Skip preamble ("Great question!") and postamble ("Hope this helps!").
  ```
- **Source:** Tier 1 — Anthropic prompting best practices: *"Positive examples … tend to be more effective than negative examples or instructions that tell the model what not to do."* https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices

---

### A5 — April 23 verbosity cap regression ("≤25/≤100 words")

- **Match regex:** `(?i)\b(under|fewer than|less than|<=?|≤)\s*(25|50|100)\s*(words?|tokens?)\b`
  - Two-phase: grep, then suppress if the surrounding ±5 lines mention `legacy|deprecated|do not use|removed`.
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** Critical
- **Apply tier:** guided (the rewrite removes a broken constraint; user confirms)
- **Applies to:** standard Class A glob set
- **High-precision regex:** false
- **Fidelity:** `paraphrased` (the postmortem describes the verbosity-cap rollback that motivates this pattern; the rollback shipped on April 20, with usage-limit resets on April 23)
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - Keep responses under 100 words.
  + Match response length to task complexity. Skip preamble/postamble.
  ```
- **Expanded rewrite:** same with full task-complexity-routing rationale.
- **Source:** Tier 1 — https://www.anthropic.com/engineering/april-23-postmortem

---

## Category B — Conditional rules without explicit conditions

### B1 — "Run tests after changes" (no batching guidance)

- **Match regex:** `(?i)\b(run|execute)\s+(tests?|the test suite|npm test|pytest|jest)\b.*\b(after|following|every|each)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **Excludes:** `.claude/skills/test-driven-development/**`, `.claude/skills/*tdd*/**` (TDD methodology depends on the literal reading)
- **High-precision regex:** false
- **Fidelity:** `practitioner` (Tier 3; foundational Tier 1 anchor on literal-following at the migration guide)
- **Why 4.6 was fine:** Batched test runs intelligently — ran once at the end of a multi-file edit, not after every `Edit`.
- **How 4.7 mis-reads:** "After every change" interpreted literally = test run after every single `Edit`. On a 15-file refactor this triples wall-clock time and floods context. Combined with 4.7's *"fewer tool calls by default"* trend, agents may also over-correct and skip tests.
- **Concise rewrite:**
  ```diff
  - Run tests after changes.
  + Run `npm test` once after a coherent unit of work, not after each file edit.
  + Always run before commit. Skip on trivial edits (typos, comments, formatting-only).
  ```
- **Expanded rewrite:**
  ```diff
  - Run tests after changes.
  + Run `npm test` once after you finish a coherent unit of work (all files
  +   for a single task), not after each individual file edit.
  + Always run tests before creating a commit.
  + For trivial changes (typos, comments, formatting-only), skip the test run.
  ```
- **Source:** Tier 3 — real example https://github.com/abhishekray07/claude-md-templates

---

### B2 — "Ask before making changes" (scope unspecified)

- **Match regex:** `(?i)\b(ask|confirm|check with|get permission|prompt me)\b.*\b(before|prior to)\b.*\b(make|making|any|each|every|chang|edit|modif)`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred this meant "before destructive or risky changes" — still made routine edits inside the requested task without pausing.
- **How 4.7 mis-reads:** Literal reading = ask before *every* edit. Destroys flow. Or, scoped only to "commit", agent silently pushes because only "commit" was named.
- **Concise rewrite:**
  ```diff
  - Ask before making changes.
  + Don't `git commit`/`git push` without my go-ahead. Edits, reads, builds, test runs are routine.
  + Pause for destructive operations (rm -rf, git reset --hard, --force, DROP).
  ```
- **Expanded rewrite:**
  ```diff
  - Ask before making changes.
  + Do not `git commit` or `git push` without my explicit go-ahead in the
  +   conversation. Edits, reads, builds, and test runs do not require
  +   confirmation.
  + For destructive operations (`rm -rf`, `git reset --hard`, `git push --force`,
  +   dropping DB tables), pause and ask before running.
  ```
- **Source:** Tier 3 — real example https://github.com/abhishekray07/claude-md-templates

---

### B3 — "Ask when ambiguous" (no examples of what counts)

- **Match regex:** `(?i)\b(when|if)\b.*\b(ambiguous|unclear|uncertain|in doubt)\b.*\b(ask|stop|confirm|clarif)`
- **Severity:** Warning (becomes Critical when paired with C4 — see G1 contradiction below)
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Used "ambiguous" as a soft signal — asked only for high-stakes choices and made reasonable defaults for low-stakes ones.
- **How 4.7 mis-reads:** More literal compliance can turn this into constant interruption on micro-decisions (variable names, file placement, import order).
- **Concise rewrite:**
  ```diff
  - When in doubt, ask first.
  + Ask when (a) architecture changes, (b) hard-to-reverse, (c) two user-visible defaults exist.
  + Otherwise pick the nearest-existing pattern and note it in your reply.
  ```
- **Expanded rewrite:**
  ```diff
  - When in doubt, ask first.
  + Ask before proceeding when (a) the choice changes architecture (new
  +   dependency, new service, new DB table), (b) the choice is hard to reverse,
  +   or (c) two or more user-visible options exist with no clear default.
  + For reversible, low-stakes choices (variable names, internal helper
  +   placement, minor formatting), pick the option that matches nearest
  +   existing code and proceed. Note the choice in your reply.
  ```
- **Source:** Tier 3 — real example https://github.com/iamfakeguru/claude-md

---

## Category C — Rules that rely on 4.6 inferring good defaults

### C1 — "Default to no comments" without exception list

- **Match regex:** `(?i)\b(default to|prefer|write|use)\s*(no|zero|minimal|few)\s+comments?\b|\b(do not|don'?t|never|avoid)\s+(write|add|include)\s+comments?\b`
- **Severity:** Warning (capped — practitioner-fidelity citation; promote to Critical only with a Tier 1 source)
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred exceptions: JSDoc on public APIs, license headers, `// TODO:`, `// eslint-disable`, complex regex explanations.
- **How 4.7 mis-reads:** May strip JSDoc from public APIs, remove license headers, delete `// TODO:` markers — all legitimate "comments" by literal reading.
- **Concise rewrite:**
  ```diff
  - Default to writing no comments.
  + No comments for self-evident code.
  + Keep: JSDoc/TSDoc on exported APIs, license headers, TODO/FIXME, compiler pragmas, non-obvious regex/bitwise.
  ```
- **Expanded rewrite:**
  ```diff
  - Default to writing no comments.
  + Do not add inline explanatory comments for self-evident code.
  + Keep: JSDoc/TSDoc on exported functions and public API, license headers,
  +   `// TODO:` / `// FIXME:` tags, compiler pragmas (`// eslint-disable-line`,
  +   `// @ts-expect-error`), and comments explaining non-obvious regex or
  +   bitwise logic.
  ```
- **Source:** Tier 3 — real example https://github.com/iamfakeguru/claude-md

---

### C2 — "Be thorough" with no task-complexity gating

- **Match regex:** `(?i)\b(be|always be|stay|remain)\s+(thorough|comprehensive|exhaustive|complete)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred "thorough relative to the task." Did not turn a one-line typo fix into a comprehensive code audit.
- **How 4.7 mis-reads:** Per Anthropic, *"4.7 won't infer what 'complete' means without explicit criteria."* A typo-fix task may trigger an exhaustive refactor.
- **Concise rewrite:**
  ```diff
  - Be thorough.
  + Depth matches task complexity. Architecture/debug: explore broadly, evaluate alternatives.
  + Typo/config tweak/copy edit: make the edit, verify it compiles, stop.
  ```
- **Expanded rewrite:**
  ```diff
  - Be thorough.
  + Depth should match task complexity. For architecture work, research, and
  +   debugging: explore broadly, evaluate alternatives, verify fixes thoroughly.
  + For small localized changes (typos, config tweaks, copy edits): make the
  +   edit, verify it compiles, stop.
  ```
- **Source:** Tier 3 — https://www.mindstudio.ai/blog/how-to-prompt-claude-opus-4-7

---

### C3 — "Follow existing patterns" without tie-breaker

- **Match regex:** `(?i)\b(follow|match|mirror|conform to)\b.*\b(existing|established|current|surrounding)\s+(pattern|patterns|style|conventions?|code)\b`
- **Severity:** Info
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "If two files give different guidance for the same behavior, Claude may pick one arbitrarily."`
- **Why 4.6 was fine:** Silently picked the majority pattern when conventions conflicted.
- **How 4.7 mis-reads:** When two patterns coexist (e.g., some files use classes, others use factory functions), 4.7 may halt, ask, or pick inconsistently. Anthropic memory docs: *"If two files give different guidance for the same behavior, Claude may pick one arbitrarily."*
- **Concise rewrite:**
  ```diff
  - Follow existing patterns.
  + Match the nearest sibling file in the same directory.
  + If the directory mixes patterns, prefer the most recent (by git log). Note the call in your reply.
  ```
- **Expanded rewrite:**
  ```diff
  - Follow existing patterns.
  + Follow the pattern of the nearest sibling file in the same directory.
  + If the directory mixes patterns, prefer the most recent pattern (by git
  +   log). Flag the inconsistency in your reply but don't stop to ask.
  ```
- **Source:** Tier 1 — Anthropic memory docs https://code.claude.com/docs/en/memory

---

### C4 — "Don't ask questions" / "Don't interrupt me"

- **Match regex:** `(?i)\b(do not|don'?t|never)\b.*\b(ask|interrupt|pause|stop|prompt|wait)\b.*\b(me|the user|for confirmation|for clarification)\b`
- **Severity:** Warning (capped — practitioner-fidelity citation; promote to Critical only with a Tier 1 source). When paired with a contradicting "always ask" rule, the G1 pair-detection finding may flag Critical via the Phase 3 contradiction algorithm.
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Still surfaced critical ambiguities (irreversible choices, scope surprises) despite such rules.
- **How 4.7 mis-reads:** More literal — may silently pick a bad default on a high-stakes fork rather than interrupt. When combined with "ask when ambiguous", the two rules contradict and 4.7 will resolve unpredictably.
- **Concise rewrite:**
  ```diff
  - Don't ask questions, just do the work.
  + Don't ask on reversible, low-stakes choices — pick the conventional option, note it in your reply.
  + DO ask before: new top-level dependency, public API change, deployment-target change, hard-to-reverse work.
  ```
- **Expanded rewrite:**
  ```diff
  - Don't ask questions, just do the work.
  + Don't ask clarifying questions for reversible, low-stakes choices — pick
  +   the most conventional option and note it in your reply.
  + DO ask before: adding a new top-level dependency, changing a public API,
  +   changing the deployment target, or any change that's hard to reverse.
  ```
- **Source:** Tier 3 — common pattern in global CLAUDE.md files. https://kirill-markin.com/articles/claude-code-rules-for-ai/

---

## Category D — Overly broad prohibitions

### D1 — "Never run destructive commands" with enumerated list

- **Match regex:** `(?i)\b(never|do not|don'?t)\s+(run|execute|use|invoke)\s+(destructive|dangerous|risky)\s+(commands?|git commands?|operations?)\b`
- **Severity:** Warning (capped — practitioner-fidelity citation; promote to Critical only with a Tier 1 source)
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred "destructive" included `rm -rf`, DB drops, force-pushes, etc. — even when the rule only listed git commands.
- **How 4.7 mis-reads:** Reads the list as exhaustive — `rm -rf node_modules` or `DROP TABLE` may not trigger the rule because they aren't enumerated. Or conversely, may flag routine `git restore <unstaged-file>` as destructive.
- **Concise rewrite:**
  ```diff
  - Never run destructive git commands (push --force, reset --hard, ...) unless requested.
  + Confirmation required for: git (push --force*, reset --hard, clean -f*, branch -D), rm -rf, DROP/TRUNCATE, terraform destroy, wrangler delete, aws s3 rm.
  + `git restore <unstaged>` and `git checkout -- <unstaged>` are routine.
  ```
- **Expanded rewrite:**
  ```diff
  - Never run destructive git commands (push --force, reset --hard, ...)
  +   unless the user explicitly requests these actions.
  + Treat the following as "destructive, confirmation required":
  +   - Git: `push --force*`, `reset --hard`, `clean -f*`, `branch -D`,
  +     rewriting published history.
  +   - Filesystem: `rm -rf`, `rm` on anything outside `/tmp` or `node_modules`.
  +   - Database: `DROP`, `TRUNCATE`, destructive migrations.
  +   - Cloud: `wrangler delete`, `aws s3 rm`, terraform `destroy`.
  + Routine `git restore <unstaged-file>` and `git checkout -- <unstaged-file>`
  +   are allowed without confirmation.
  ```
- **Source:** Tier 3 — pattern observed across many CLAUDE.md files including Anthropic's own examples.

---

### D2 — "Don't touch X" (touch is ambiguous)

- **Match regex:** `(?i)\b(do not|don'?t|never)\s+touch\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred "touch" = "modify" — still read files for context.
- **How 4.7 mis-reads:** Literal "touch" can include reads, starving the agent of context it needs to work adjacent to the restricted area.
- **Concise rewrite:**
  ```diff
  - Never touch /app/auth/ without explicit confirmation.
  + Do not modify files under `/app/auth/` without my go-ahead. Reading for context is allowed.
  ```
- **Expanded rewrite:**
  ```diff
  - Never touch /app/auth/ without explicit confirmation.
  + Do not modify files under `/app/auth/` without explicit confirmation.
  +   Reading files for context is allowed.
  ```
- **Source:** Tier 3 — pattern observed in template repositories.

---

## Category E — Vague positive directives

### E1 — "Write clean code" / "Make it production-ready"

- **Match regex:** `(?i)\b(write|produce|deliver|make it|ensure)\s+(clean|elegant|professional|production[- ]ready|enterprise[- ]grade|high[- ]quality)\s+(code|implementation|solution)?\b`
- **Severity:** Info
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Why 4.6 was fine:** Benign — 4.6 ignored as noise.
- **How 4.7 mis-reads:** Mostly still ignored, but wastes bytes of the *"150-200 instruction budget before compliance drops off"*. Worse, 4.7 may over-engineer a simple script into enterprise scaffolding.
- **Concise rewrite:**
  Delete the directive entirely, OR replace with concrete acceptance criteria:
  ```diff
  - Write clean, production-ready code.
  + Every new module: a unit test, a typed public API, an error path for IO failures.
  + No silent try/catch fallback — re-throw or log with context.
  ```
- **Expanded rewrite:** Delete, or replace with specifics:
  ```diff
  - Write clean, production-ready code.
  + Every new module must have: a unit test, a typed public API, and an
  +   error-handling path for network / IO failures. No try/catch as silent
  +   fallback — re-throw or log with context.
  ```
- **Source:** Tier 1 — Anthropic best practices explicitly excludes "self-evident practices like 'write clean code'". https://code.claude.com/docs/en/best-practices
- **Fidelity:** `paraphrased` — Anthropic's Claude Code best-practices guidance flags self-evident "write clean code" / "production-ready" instructions as low-signal CLAUDE.md content; the cited page makes that point in tabular form rather than as an inline quote.
- **Source quote:** `null`

---

## Category F — Temporal / event-based directives

### F1 — "At the end of every task, do X"

- **Match regex:** `(?i)\b(at the end of|after completing|when (you|the task is) (done|finished|complete)|at task completion)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred "end of task" from conversation context.
- **How 4.7 mis-reads:** 4.7 doesn't know when a "task" ends. May run lint after every message, or never (if it reads "task" = entire session).
- **Concise rewrite:**
  ```diff
  - At the end of every task, run lint and typecheck.
  + Before saying "done" or creating a commit, run `npm run lint` + `npm run typecheck`. Fix and re-run on failure.
  ```
- **Expanded rewrite:**
  ```diff
  - At the end of every task, run lint and typecheck.
  + Before saying "done", "ready for review", or creating a commit, run
  +   `npm run lint` and `npm run typecheck`. If either fails, fix and re-run
  +   before reporting completion.
  ```
- **Source:** Tier 3 — real example https://gist.github.com/markomitranic/26dfcf38c5602410ef4c5c81ba27cce1

---

### F2 — Vague hedges ("try to", "if possible", "you might want to")

- **Match regex:**
  ```yaml
  - "(?i)\\b(try to|attempt to|if possible|when possible|where possible|you might want to|preferably)\\s+(\\w+\\s+){0,2}(run|use|write|prefer|enforce|follow|apply|ensure|verify|test|build|deploy|commit|review|implement)\\b"
  ```
- **Severity:** Warning
- **Apply tier:** manual
- **Why 4.6 was fine:** Treated "try to X" as a soft "do X."
- **How 4.7 mis-reads:** Anthropic migration guide explicit warning: hedges weaken instructions. Literal "try to" = "attempt once, then move on if inconvenient."
- **Rewrite template:** Every `try to` → `do`. Every `if possible` → remove or replace with an explicit condition. Every `you might want to` → `do X when Y`.
  ```diff
  - Try to use TypeScript strict mode if possible.
  + Use TypeScript strict mode in all `tsconfig.json` files in this repo.
  ```
- **Source (primary):** Tier 3 — practitioner consolidation. KeepMyPrompts: https://www.keepmyprompts.com/en/blog/claude-opus-4-7-prompting-guide-whats-changed ; MindStudio: https://www.mindstudio.ai/blog/how-to-prompt-claude-opus-4-7 ; RentierDigital: https://medium.com/@rentierdigital/opus-4-7-feels-like-it-broke-your-prompts-it-didnt-you-just-missed-the-2-rewrites-that-matter-826961733888

  **Foundational Tier 1 anchor:** Anthropic migration guide — *"it will not silently generalize an instruction from one item to another, and it will not infer requests you didn't make."* https://platform.claude.com/docs/en/about-claude/models/migration-guide

  **Fidelity:** `practitioner` — practitioner-derived application of the Anthropic literal-following principle. Severity is **Warning** because the Anthropic Tier 1 migration guide explicitly warns that hedges weaken instructions, which moves this above the Info threshold; the practitioner-fidelity cap allows up to Warning.

---

### F3 — 4.6-era scaffolding ("after every N tool calls, summarize"; "double-check before returning"; "think step by step")

- **Match regex:** `(?i)\b(think (step.by.step|carefully|deeply)|reason (carefully|step.by.step)|double[- ]check|after every \d+ (tool calls?|edits?|messages?)|summari[sz]e progress)\b`
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "If you've added scaffolding to force interim status messages, try removing it."`
- **Why 4.6 was fine:** Helped 4.6 stay on track during long agentic loops.
- **How 4.7 mis-reads:** Anthropic explicitly: *"If you've added scaffolding to force interim status messages, try removing it."* and *"If you observe shallow reasoning on complex problems, raise effort to `high` or `xhigh` rather than prompting around it."*
- **Concise rewrite:**
  Delete the scaffolding entirely. For more reasoning on 4.7, raise `effort` in settings.json (do not prompt around it):
  ```diff
  - Think step by step. After every 3 tool calls, summarize progress. Double-check before returning.
  + (deleted — 4.7 self-verifies natively; raise effort to xhigh for complex tasks)
  ```
- **Expanded rewrite:** Delete the scaffolding. If you want more thinking on 4.7, raise `effort` instead.
  ```diff
  - Think step by step. After every 3 tool calls, summarize progress.
  - Double-check the output before returning.
  + (deleted — 4.7 self-verifies and provides interim updates natively;
  +   raise effort to xhigh in settings.json for complex tasks)
  ```
- **Source:** Tier 1 — https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7; reinforced by https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code (scaffolding-removal guidance for Claude Code projects)

---

### F4 — Severity-filter language in code-review prompts

- **Match regex:** `(?i)\b(only (report|surface|raise|flag|mention))\b.*\b(important|significant|major|critical|notable|severe)\b|\b(don'?t|do not|avoid)\s+(nitpick|nitpicking|nitpicks?)\b|\b(be conservative|filter out (minor|low.severity))\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Fidelity:** `verbatim` — `source.quote: "When a review prompt says things like 'only report high-severity issues,' 'be conservative,' or 'don't nitpick,' Claude Opus 4.7 may follow that instruction more faithfully than earlier models did — it may investigate the code just as thoroughly, identify the bugs, and then not report findings it judges to be below your stated bar."`
- **Why 4.6 was fine:** Inferred a sensible importance threshold.
- **How 4.7 mis-reads:** Anthropic best-practices explicitly: 4.7 follows these literally and suppresses findings it judges below the bar — looks like a recall regression.
- **Rewrite template:**
  ```diff
  - Only report important issues. Don't nitpick.
  + Report every issue you find, including ones you are uncertain about or
  +   consider low-severity. Do not filter for importance or confidence at
  +   this stage. I will triage the report after you finish.
  ```
- **Source:** Tier 1 — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices

---

### F5 — Implicit tool-use language ("check", "verify", "look up")

- **Match regex:**
  ```yaml
  - phase 1: "(?i)\\b(check|verify|look up|review|inspect|examine)\\b\\s+(the\\s+)?\\b(project|code|file|structure|repo|codebase|state)\\b"
  - phase 2 (post-match): Read ±5 lines around the matched line. Suppress the finding if any line within ±5 contains an explicit tool name: `Glob`, `Read`, `Grep`, `Bash`, `WebFetch`, `WebSearch`, or a bash command in backticks (e.g., `` `ls` ``, `` `find ...` ``).
  ```
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "Fewer tool calls by default, using reasoning more."`
- **Why 4.6 was fine:** "Check the project structure before suggesting changes" → quietly invoked a tool.
- **How 4.7 mis-reads:** Per Anthropic, 4.7 makes *"fewer tool calls by default, using reasoning more."* "Check" may be interpreted as mental consideration, not a tool call.
- **Concise rewrite:**
  ```diff
  - Check the project structure before suggesting changes.
  + Run the Glob tool to list top-level directories. Read any file referenced in my message.
  ```
- **Expanded rewrite:** Be explicit about which tool:
  ```diff
  - Check the project structure before suggesting changes.
  + Before suggesting changes, run the Glob tool to list top-level
  +   directories and the Read tool on any file referenced in the user's
  +   message.
  ```
- **Source:** Tier 1 — *"Fewer tool calls by default, using reasoning more."* https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7

---

### F6 — "Double-check the output" scaffolding (4.7 self-verifies)

- **Match regex:** `(?i)\b(double[- ]check|verify (twice|again)|review (your|the) (output|response|answer))\b`
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** Warning
- **Apply tier:** guided (delete or rephrase)
- **Applies to:** standard Class A glob set
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "If you've added scaffolding to force interim status messages, try removing it."` (whats-new)
- **Concise rewrite:**
  ```diff
  - Double-check the output before returning.
  + (deleted — 4.7 self-verifies natively; raise effort to xhigh for higher confidence)
  ```
- **Expanded rewrite:** same with explanation.
- **Source:** Tier 1 — whats-new + migration guide; reinforced by https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code (scaffolding-removal guidance for Claude Code projects).

---

### F7 — "After every N tool calls, summarize progress" (redundant on 4.7)

- **Match regex:** `(?i)\bafter every \d+\s+(tool calls?|edits?|messages?)\b`
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** Warning
- **Apply tier:** guided
- **Applies to:** standard Class A glob set
- **High-precision regex:** false
- **Fidelity:** `verbatim` — same Anthropic whats-new "remove scaffolding" quote.
- **Concise rewrite:**
  ```diff
  - After every 3 tool calls, summarize progress.
  + (deleted — 4.7 emits status updates natively; raise effort if mid-task summaries are needed)
  ```
- **Expanded rewrite:** same.
- **Source:** Tier 1 — migration guide; reinforced by https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code (scaffolding-removal guidance for Claude Code projects).

---

## Category G — Contradictions 4.6 resolved via inference

### G1 — "Be thorough" + "Be concise" in the same file

- **Match regex:** Two passes — file must contain both:
  - Pattern A: `(?i)\b(be|always)\s+(thorough|comprehensive|deep|exhaustive)\b` OR `\b(depth over|extended thinking|read entire files|never abbreviate)\b`
  - Pattern B: `(?i)\b(be|keep|stay)\s+(concise|brief|terse|short)\b` OR `\bunder \d+ (lines|words)\b`
- **Severity:** **Critical**
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "If two files give different guidance for the same behavior, Claude may pick one arbitrarily."`
- **Pair-detection:** runs Stage 1 heuristic + Stage 2 LLM judgment per references/contradictions.md. Stage 2 biases toward Warning when uncertain — Critical reserved for confident same-axis-opposite-verbs verdicts.
- **Why 4.6 was fine:** Picked the right mode by task type.
- **How 4.7 mis-reads:** Per Anthropic memory docs: *"If two files give different guidance for the same behavior, Claude may pick one arbitrarily."* Could default to max depth for trivial requests or truncate architecture explanations.
- **Concise rewrite (precedence-block recommendation):**
  ```diff
  + Routing between depth and brevity:
  + - Typo/config/single-file: brief mode (≤10 lines explanation, no extended thinking).
  + - Architecture/debug/migration/multi-file: depth mode (extended thinking, full files, alternatives).
  ```
- **Expanded rewrite (add a routing rule near the top):**
  ```diff
  + Routing between depth and brevity:
  + - Content/copy edits, typo fixes, single-file changes: brief mode
  +   (under 10 lines of explanation, no extended thinking budget).
  + - Architecture decisions, debugging, migrations, anything touching
  +   >1 file or core systems: depth mode (extended thinking, read full
  +   files, evaluate alternatives).
  + When unclear, default to depth mode.
  ```
- **Source:** Tier 1 — https://code.claude.com/docs/en/memory

---

### G2 — "Don't add error handling for impossible cases" + "Be defensive"

- **Match regex:** Two passes. File must contain both:
  - Pattern A: `(?i)\b(don'?t|do not|avoid)\b.*\b(error handling|defensive|validation)\b.*\b(impossible|unreachable|trust internal)\b`
  - Pattern B: `(?i)\b(be defensive|always validate|add error handling|defensive checks?)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Pair-detection:** runs Stage 1 heuristic + Stage 2 LLM judgment per references/contradictions.md. Stage 2 biases toward Warning when uncertain — Critical reserved for confident same-axis-opposite-verbs verdicts.
- **Why 4.6 was fine:** Picked sensible boundaries for validation.
- **How 4.7 mis-reads:** Two contradicting rules; arbitrary resolution.
- **Concise rewrite (precedence-block recommendation):**
  ```diff
  + Validate at trust boundaries (API input, IPC, DB rows, env vars): fail loudly with typed errors.
  + Inside a module's private functions: trust caller-guaranteed invariants. Don't re-validate.
  ```
- **Expanded rewrite:**
  ```diff
  + Validate at trust boundaries (API input, IPC, DB rows, env vars):
  +   fail loudly with typed errors.
  + Inside a single module's private functions, do not re-validate invariants
  +   already guaranteed by the caller.
  ```
- **Source:** Tier 3 — https://github.com/iamfakeguru/claude-md

---

## Category H — Implicit scope rules

### H1 — "Don't commit without permission" — applies to which git ops?

- **Match regex:** `(?i)\b(do not|don'?t|never)\s+(commit|push|merge|rebase)\b.*\b(without|unless)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Inferred which git operations were affected.
- **How 4.7 mis-reads:** May ask permission for `git branch`, `git stash`, `git tag`, or read-only operations.
- **Concise rewrite:**
  ```diff
  - Don't commit without explicit permission.
  + Don't `git commit/push/merge/rebase/reset` without my go-ahead.
  + `git branch -v`, `git status`, `git stash list`, `git log`, `git checkout -b` are routine.
  ```
- **Expanded rewrite:**
  ```diff
  - Don't commit without explicit permission.
  + Do not run `git commit`, `git push`, `git merge`, `git rebase`, or
  +   `git reset` without my explicit go-ahead. Local-only, non-mutating
  +   operations (`git branch -v`, `git status`, `git stash list`,
  +   `git log`, creating a new branch with `git checkout -b`) don't need
  +   confirmation.
  ```
- **Source:** Tier 3 — common pattern.

---

### H2 — "Always lint before committing" — applies when no linter exists?

- **Match regex:** `(?i)\b(always|must|run)\b.*\b(lint|format|prettier|eslint|black|ruff)\b.*\b(before)\b`
- **Severity:** Info
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Skipped silently if no linter was configured.
- **How 4.7 mis-reads:** May add a linter as part of "following the rule." Or fabricate a command and report failure.
- **Concise rewrite:**
  ```diff
  - Always lint before committing.
  + If `npm run lint` exists in package.json, run it before commit. If it doesn't exist, skip silently.
  ```
- **Expanded rewrite:**
  ```diff
  - Always lint before committing.
  + If `npm run lint` exists in package.json, run it before committing.
  +   If it doesn't exist, skip silently.
  ```
- **Source:** Tier 3 — common pattern.

---

## Category I — Memory triggers phrased too strongly

### I1 — "From now on, always X" (no revocation condition)

- **Match regex:**
  ```yaml
  - "(?i)\\b(from now on|going forward|always|forever|in (every|all) future)\\b\\s+(\\w+\\s+){0,3}(use|run|prefer|enforce|require|implement|apply|follow|stick to|default to|ensure|maintain)\\b"
  ```
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `paraphrased` — Anthropic's memory docs describe CLAUDE.md and auto memory as the two mechanisms that "carry knowledge across sessions."
- **Source quote:** `null`
- **Why 4.6 was fine:** Treated as a soft preference.
- **How 4.7 mis-reads:** Compounds across sessions forever. Stale "from now on" rules that contradict the current task cause misbehavior.
- **Concise rewrite:**
  ```diff
  - From now on, always use Pinecone for vector search.
  + In the search service, prefer Pinecone for vector storage. Other services follow project-local choices.
  ```
- **Expanded rewrite:** Add explicit scope or rephrase:
  ```diff
  - From now on, always use Pinecone for vector search.
  + When working on the search service, prefer Pinecone for vector storage.
  +   For other services, follow project-local choices.
  ```
- **Source:** Tier 1 — Anthropic memory docs flag this. https://code.claude.com/docs/en/memory

---

## Category R — Patterns specific to "ambitious" CLAUDE.md files (priority directives, depth directives, parallel dispatch)

These flag patterns common in CLAUDE.md files that try to override default model behavior with strong directives.

### R1 — "Use extended thinking liberally" without task gating

- **Match regex:**
  ```yaml
  - "(?i)\\b(use|enable|engage)\\b.*\\b(extended thinking|deep thinking|max thinking)\\b.*\\b(liberally|always|for any|generously)\\b"
  - "(?i)\\bthink\\s+(deeply|carefully)\\b.*\\b(always|for any|before acting)\\b"
  ```
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "Requests with no thinking field run without thinking."`
- **Why 4.6 was fine:** 4.6's adaptive thinking was on by default; "use it liberally" reinforced existing behavior.
- **How 4.7 mis-reads:** 4.7 thinking is **off by default** (per Anthropic: *"Requests with no `thinking` field run without thinking"*). Response length already calibrates to complexity. An unqualified "think deeply always" fights built-in calibration and runs up thinking tokens on trivial requests.
- **Concise rewrite:**
  ```diff
  - Use extended thinking liberally for any non-trivial decision.
  + Engage extended thinking when: (a) >1 file, (b) unclear acceptance criteria, (c) new architecture, (d) debugging non-obvious bugs.
  + Skip for typo fixes, single-line config tweaks, copy edits — 4.7 auto-calibrates.
  ```
- **Expanded rewrite:**
  ```diff
  - Use extended thinking liberally for any non-trivial decision.
  + Use extended thinking when (a) the task touches >1 file, (b) acceptance
  +   criteria are unclear, (c) introducing new architecture, or (d) debugging.
  + For typo fixes, single-line config tweaks, and copy edits, brief mode is
  +   correct — do not engage extended thinking.
  ```
- **Source:** Tier 1 — https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking

---

### R2 — Parallel dispatch as a blanket imperative

- **Match regex:** `(?i)\b(prefer|always|default to)\b.*\b(parallel|concurrent|spawn(ing)? subagents)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "Fewer subagents spawned by default."`
- **Why 4.6 was fine:** 4.6 spawned subagents readily.
- **How 4.7 mis-reads:** 4.7 *"spawns fewer subagents by default"* per Anthropic. The literal reading of "prefer parallel subagents" may cause spawning even for tasks faster inline.
- **Concise rewrite:**
  ```diff
  - Prefer spawning parallel subagents for independent research tasks.
  + Use subagents when (a) 3+ truly-independent tasks, OR (b) a single task reads >20 files.
  + For 1-2 simple lookups, inline tool use is faster.
  ```
- **Expanded rewrite:**
  ```diff
  - Prefer spawning parallel subagents for independent research tasks.
  + Use subagents when: (a) 3+ independent research tasks exist, OR (b) a
  +   single task will read >20 files and you want to avoid polluting main
  +   context. For 1-2 simple lookups, inline is faster — skip subagents.
  + When you do use N subagents (N >= 2), dispatch in a single assistant
  +   message.
  ```
- **Source:** Tier 1 — https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7

---

### R3 — "Ask First" + "Depth Over Economy" (or similar) — directly contradictory

- **Match regex:** Two passes. File must contain both:
  - Pattern A: `(?i)\b(stop and ask|ask first|always ask|ask before)\b`
  - Pattern B: `(?i)\b(depth over|follow threads|default to depth|default to thoroughness)\b`
- **Severity:** **Critical**
- **Apply tier:** manual
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **High-precision regex:** false
- **Fidelity:** `verbatim` — `source.quote: "If two files give different guidance for the same behavior, Claude may pick one arbitrarily."`
- **Pair-detection:** R3 is LLM-judged with the regex pair as a coarse pre-filter. Pattern A is intentionally broad (heuristic prefilter, expected to over-match); Pattern B narrows the candidate set; Stage 2 LLM judgment per references/contradictions.md decides whether the pair is a real contradiction. Stage 2 biases toward Warning when uncertain — Critical is reserved for confident same-axis-opposite-verbs verdicts.
- **Why 4.6 was fine:** 4.6 picked the right behavior by context.
- **How 4.7 mis-reads:** Two rules pointing opposite directions; arbitrary resolution.
- **Concise rewrite (precedence-block recommendation):**
  ```diff
  + Precedence when "ask-first" and "depth" both apply:
  + - User-facing decisions (intent, scope, design choice): ASK-FIRST wins.
  + - Internal research where direction is approved: DEPTH wins.
  ```
- **Expanded rewrite:** Add explicit precedence:
  ```diff
  + Precedence when "ask-first" and "depth" both apply:
  + - User-facing decisions (intent, scope, design choice): ASK-FIRST wins.
  + - Internal research and analysis where the user has already approved
  +   the direction: DEPTH wins.
  ```
- **Source:** Tier 1 — Anthropic memory docs on contradictions. https://code.claude.com/docs/en/memory

---

### R4 — "Respond warmly" / "use emoji" (4.7 defaults terser)

- **Match regex:** `(?i)\b(respond|reply|answer)\s+(warmly|enthusiastically|with emoji|in a friendly tone)\b|use emoji|with personality`
- **Severity:** Warning
- **Apply tier:** manual (voice direction needs author judgment)
- **Applies to:** standard Class A glob set + role-system prompts (`.claude/agents/*.md`)
- **High-precision regex:** false
- **Fidelity:** `paraphrased` — whats-new describes 4.7's tone as "more direct, opinionated, with less validation-forward phrasing and fewer emoji."
- **Source quote:** `null`
- **Concise rewrite (manual):**
  ```diff
  - Respond warmly with emoji.
  + Use a warm, conversational register. Emoji are appropriate when they aid clarity (status indicators, list bullets); avoid decorative emoji.
  ```
- **Expanded rewrite:** same with examples.
- **Source:** Tier 1 — whats-new.

---

### R5 — Forced subagent spawning ("always spawn for X") without explicit trigger

- **Match regex:** `(?i)\b(always|must|every time)\b\s+(\w+\s+){0,3}(spawn|launch|dispatch|invoke)\b\s+(\w+\s+){0,3}(subagent|agent|sub-agent)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** standard Class A glob set
- **High-precision regex:** false
- **Fidelity:** `verbatim` — same Anthropic whats-new "fewer subagents by default" quote.
- **Concise rewrite (manual):**
  ```diff
  - Always spawn a subagent for research tasks.
  + Spawn subagents when (a) 3+ truly-independent research tasks, OR (b) a single task reads >20 files. Otherwise inline.
  ```
- **Expanded rewrite:** same.
- **Source:** Tier 1 — migration guide; reinforced by https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code (fewer-subagents guidance for Claude Code projects).

---

### R6 — "Ask clarifying questions before starting" (4.7 prefers attempt-first)

- **Match regex:** `(?i)\b(ask|gather)\s+(clarifying\s+)?questions?\s+(before|prior to)\s+(starting|beginning|implementing)\b`
- **Compatibility:** 🔴 4.7-tradeoff (suppressed in compatibility mode; surfaces in 4.7-only mode)
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** standard Class A glob set
- **High-precision regex:** false
- **Fidelity:** `practitioner` — Simon Willison observation in https://simonwillison.net/2026/Apr/18/opus-system-prompt/
- **Concise rewrite (manual):**
  ```diff
  - Ask clarifying questions before starting.
  + Attempt the most reasonable interpretation first; surface ambiguity with your first reply if it materially changes the outcome.
  ```
- **Expanded rewrite:** same.
- **Source:** Tier 3 — Simon Willison; foundational Tier 1 anchor at the migration guide on literal-following.

---

### R7 — Cyber refusal surface keywords without CVP enrollment note

- **Match regex:** `(?i)\b(penetration test(ing)?|red team|exploit (development|writing)|payload (creation|generation)|reverse engineering malware|c2 framework)\b`
  - Two-phase: grep for the keyword; suppress if file also contains `Cyber Verification Program|CVP enrollment|enrolled in CVP|Anthropic CVP`.
- **Severity:** Warning (4.7 refuses these without enrollment context; flagging this proactively saves a refusal cycle)
- **Apply tier:** manual (requires CVP enrollment context decision)
- **Applies to:** standard Class A glob set + `.claude/agents/*.md`
- **High-precision regex:** false
- **Fidelity:** `paraphrased` — migration guide describes 4.7's real-time cybersecurity safeguards and the Cyber Verification Program.
- **Source quote:** `null`
- **Concise rewrite (manual):** Either declare CVP enrollment context inline, OR rephrase the directive to avoid triggering the refusal surface, OR remove the directive if not needed.
- **Expanded rewrite:** same with CVP enrollment instructions.
- **Source:** Tier 1 — migration guide.

---

## Category M — Monorepo-specific patterns

### M1 — Parent CLAUDE.md conflicts with package CLAUDE.md (concatenation, not override)

- **Detection:** When the project has a root `CLAUDE.md` AND nested `packages/*/CLAUDE.md` (or similar), check whether nested files repeat or contradict root rules.
- **Severity:** **Critical** for monorepos
- **Applies to:** `CLAUDE.md`, `.claude.md`, `.claude.local.md`, `AGENTS.md`, `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **Apply tier:** `manual` (cross-file edits are always manual)
- **High-precision regex:** false (structural cross-file check)
- **Fidelity:** `verbatim` — `source.quote: "All discovered files are concatenated into context rather than overriding each other."`
- **Pair-detection (cross-file):** Runs the cross-file algorithm in references/contradictions.md — compares root `CLAUDE.md` rules against each nested `packages/*/CLAUDE.md` for contradiction via Stage 1 heuristic + Stage 2 LLM judgment. Always reported as `manual` apply because cross-file edits cannot be safely auto-applied.
- **Why 4.6 was fine:** 4.6 silently picked the nearest rule.
- **How 4.7 mis-reads:** Files are *"concatenated into context rather than overriding each other"* (Anthropic memory docs). 4.7's literal mode may try to satisfy both contradictory rules.
- **Concise rewrite (manual application — cross-file change):**
  Add `claudeMdExcludes` in `.claude/settings.local.json` to drop irrelevant parent files; OR prefix scope-specific rules with their package path; OR remove duplicated "global" rules from package files (let them inherit from root).
- **Expanded rewrite:**
  1. Add `claudeMdExcludes` to `.claude/settings.local.json` to drop irrelevant parent files.
  2. In any rule that varies by package, prefix with scope: `"In packages/api/, use Zod for validation. In packages/web/, use React Hook Form."`
  3. Remove duplicated "global" rules from package files — let them inherit from root.
- **Source:** Tier 1 — Anthropic memory docs warn about ancestor-CLAUDE.md context. https://code.claude.com/docs/en/memory

---

## Category Global — `~/.claude/CLAUDE.md` patterns

### Global1 — Personal preferences leaking into work repos

- **Match regex (in `~/.claude/CLAUDE.md` only):** `(?i)\b(always|prefer|default to)\b.*\b(pnpm|yarn|bun|black|ruff)\b` *without* a "unless project specifies otherwise" clause
- **Severity:** Warning (capped — practitioner-fidelity citation; promote to Critical only with a Tier 1 source). The `[GLOBAL]` prefix on the rendered finding still makes the blast radius visible to the user.
- **Apply tier:** manual
- **Applies to:** `~/.claude/CLAUDE.md` (user-global memory file ONLY — flag findings as `[GLOBAL]` in the report)
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Silently deferred to the project file.
- **How 4.7 mis-reads:** May insist on the global preference, rewriting lockfiles.
- **Concise rewrite (apply at top of `~/.claude/CLAUDE.md`):**
  ```diff
  + Personal defaults below apply ONLY when the project has no conflicting instruction.
  + If a project CLAUDE.md or lockfile specifies a different tool, follow the project.
  ```
- **Expanded rewrite (apply at the top of `~/.claude/CLAUDE.md`):**
  ```diff
  + Personal defaults below apply ONLY when the project has no conflicting
  +   instruction. If a project CLAUDE.md or lockfile specifies a different
  +   tool, follow the project.
  ```
- **Source:** Tier 3 — common pattern.

---

### Global2 — Unbounded "always explain your reasoning"

- **Match regex (in `~/.claude/CLAUDE.md`):** `(?i)\b(always|every time)\b.*\bexplain\s+(your|the)\s+(reasoning|thinking|choice)\b`
- **Severity:** Warning
- **Apply tier:** manual
- **Applies to:** `~/.claude/CLAUDE.md` (user-global memory file ONLY — flag findings as `[GLOBAL]` in the report)
- **High-precision regex:** false
- **Fidelity:** `practitioner`
- **Why 4.6 was fine:** Skipped for trivial choices.
- **How 4.7 mis-reads:** May produce reasoning blocks on every single edit.
- **Concise rewrite:**
  ```diff
  - Always explain your reasoning.
  + When making a choice with user-visible trade-offs (API shape, tool selection, perf vs clarity), write a 1-2 sentence rationale.
  + Skip rationale for mechanical edits.
  ```
- **Expanded rewrite:**
  ```diff
  - Always explain your reasoning.
  + When you make a choice with user-visible trade-offs (API shape, tool
  +   selection, performance vs. clarity), write a 1-2 sentence rationale.
  + Skip rationale for mechanical edits.
  ```
- **Source:** Tier 3 — common pattern.

---

## Universal rewrite recipes

Apply these across all categories whenever a pattern doesn't have a category-specific rewrite:

1. **Add a scope clause** to every absolute rule. `Never X` → `Never X in <scope>. Exceptions: <list>.`
2. **Replace negative rules with positive examples** where possible. Anthropic best practices: *"Positive examples beat negative rules."*
3. **Delete hedges** (`try to`, `if possible`, `you might want to`).
4. **Add tie-breakers** to any pair of rules that could conflict. *"When rule A and rule B both apply, A wins."*
5. **Name the event** for temporal rules. `"At the end of a task"` → `"Before git commit"` or `"Before saying 'done'"`.
6. **Cap file length at ≤200 lines** (Anthropic best practice) — longer files lose adherence regardless of how well-written the rules are.
7. **Prune rules 4.7 already does correctly** — test by removing a rule and observing behavior over 2-3 sessions.

## Foundational sources for every pattern

- [Anthropic — What's new in Claude Opus 4.7](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7)
- [Anthropic — Migration guide (4.6 → 4.7)](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
- [Anthropic — Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Anthropic — Memory docs (CLAUDE.md, contradictions, hierarchy)](https://code.claude.com/docs/en/memory)
- [Anthropic — Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic — Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Anthropic — Effort](https://platform.claude.com/docs/en/build-with-claude/effort)

See [sources.md](sources.md) for the full citation list with tier classifications.

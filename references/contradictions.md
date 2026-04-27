# Contradiction-detection algorithm

Pair-detection patterns (G1, G2, R3 in-file; M1 cross-file) detect competing directives that 4.6 silently arbitrated and 4.7 may resolve unpredictably. Because false-Critical findings on a well-written CLAUDE.md destroy trust in the rest of the report — the same regression class that motivated the v2 redesign — the algorithm runs in two stages and biases toward Warning when uncertain.

## In-file pair detection (G1, G2, R3)

### Stage 1 — heuristic suppression (cheap, deterministic, runs first)

A pair finding fires only when ALL THREE conditions hold:

1. Both sub-pattern regexes match anywhere in the same file.
2. No scope-clause keyword (`when`, `if`, `for`, `unless`, `except`, `in`, `for the`, `during`) appears within ±2 lines of either match.
3. Both matches are outside fenced code blocks.

If any condition fails, suppress the pair finding with manifest annotation `pair_suppressed_by: heuristic_<condition_id>`.

### Stage 2 — LLM judgment (semantic, runs only when Stage 1 doesn't suppress)

Read both directive blocks in full (including all sub-bullets and surrounding scope clauses). Judge whether the directives describe:

- **The same axis with opposite verbs** (e.g., "be thorough" + "be concise" both target verbosity) → flag Critical with the precedence-block rewrite.
- **Non-overlapping domains** (e.g., "ask before user-facing decisions" + "go deep on internal execution" target different axes) → emit Info finding with note: *"Scope-differentiated; precedence rule recommended for safety but no Critical breakage expected."* Manifest annotation: `pair_downgraded_by: llm_judgment`.

The LLM judgment step runs with the explicit instruction: **default to Warning + precedence-block recommendation when uncertain.** Critical is reserved for cases where the model is confident the directives target the same axis with opposite verbs.

### Why hybrid

The heuristic catches the obvious cases — roughly 90% of properly-scoped rule pairs in the wild — deterministically and cheaply. The LLM judgment catches edge cases the heuristic misses: directives that lack scope-clause keywords near the matched substring but use other linguistic disambiguation (sub-bullet enumeration, "applies to" preambles, contextual framing).

A real CLAUDE.md whose "Ask First" and "Depth Over Economy" directives are scope-differentiated via sub-bullet enumeration (not via in-line `when`/`if` keywords near the trigger phrases) was the motivating case for this hybrid algorithm.

### Failure modes

LLM judgment is non-deterministic; the same file may produce slightly different verdicts across sessions. The Warning-bias rule bounds the consequences: a wrong-Warning still surfaces the same precedence-block rewrite at the same drill-down step, so material harm is near-zero. A wrong-Critical, by contrast, lands as the first thing a stranger sees on their own file — and destroys trust in every subsequent finding.

## Cross-file pair detection (M1)

Monorepo parent ↔ package CLAUDE.md contradictions. Separate algorithm:

1. Enumerate all CLAUDE.md files; build the parent-child tree by filesystem proximity.
2. For each nested file, diff its declared rules against the parent's.
3. Flag any rule present in both files with different verbs (e.g., parent says "never X," package says "always X").

M1 also runs the hybrid Stage 1 + Stage 2 logic — same heuristic-then-LLM-judgment flow, applied across files instead of within one file. M1 findings are always `apply_tier: manual` (cross-file edits require coordination the scanner cannot safely automate).

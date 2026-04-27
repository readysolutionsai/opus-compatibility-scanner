# Symptom → Pattern Map

When a user says "my agent is acting weird since I switched to 4.7" but the scan came up clean, walk this list to see which less-obvious pattern might be the culprit.

| Symptom | Likely pattern | Where it lives |
|---|---|---|
| 4.7 strips JSDoc / comments from public APIs | **C1** "Default to no comments" without exception list | CLAUDE.md, AGENTS.md |
| 4.7 refuses to create test files / new components when implementing a feature | **A3** "Don't create files" / "don't create documentation" | CLAUDE.md |
| 4.7 truncates architecture / migration / debugging answers | **A4** Fixed-line-count caps ("be concise", "under 4 lines") | CLAUDE.md |
| 4.7 refactors generated `.pb.ts` / vendor `.d.ts` files unnecessarily | **A1** Unqualified "never use any" without scope | CLAUDE.md |
| 4.7 asks permission for routine `git restore` or `git branch -v` | **D1** "Never destructive commands" enumeration interpreted as exhaustive, OR **H1** "don't commit" applied to all git ops | CLAUDE.md |
| 4.7 spawns subagents for every tiny lookup | **R2** "Prefer parallel subagents" without trigger conditions | CLAUDE.md |
| 4.7 thinks deeply on every typo fix | **R1** "Use extended thinking liberally" without task gating | CLAUDE.md |
| 4.7 interrupts with clarifying questions on micro-decisions | **B3** "Ask when ambiguous" without examples | CLAUDE.md |
| 4.7 doesn't ask before high-stakes, irreversible actions | **C4** "Don't interrupt me" without exception list | CLAUDE.md |
| 4.7 produces inconsistent code style across edits | **C3** "Follow existing patterns" with no tie-breaker | CLAUDE.md |
| 4.7 returns 400 errors immediately | **CB2** `thinking.type: "enabled"`, **CB3** `temperature` / `top_p` / `top_k`, **CB6** assistant prefills | source code |
| 4.7 returns empty thinking sections in streaming UI | **CB11** `thinking.display` default changed to `"omitted"` | source code |
| 4.7 burns through token budget faster than 4.6 | **CB10** Tokenizer change (1.0-1.35x more tokens for same input) | budget guardrails |
| 4.7 doesn't call tools that 4.6 used to call | **F5** Implicit tool-use language ("check", "verify") read as mental consideration | CLAUDE.md, AGENTS.md |
| Code review agent reports fewer findings than 4.6 did | **F4** Severity-filter language ("only report important issues") read literally | review prompt |
| Multiple competing rules produce different behavior on different runs | **G1, G2, R3, M1** Contradictions that 4.6 silently resolved | CLAUDE.md, monorepo CLAUDE.md |
| Personal preference (e.g. pnpm) overrides project-local choice (npm) | **Global1** Global CLAUDE.md missing "unless project specifies otherwise" clause | `~/.claude/CLAUDE.md` |
| Scaffolding (forced summaries, double-check) feels redundant on 4.7 | **F3** 4.6-era scaffolding | CLAUDE.md, agent prompts |
| Hooks fail with `thinking.type.enabled` errors after upgrading | **CB2** / **CB3** / **CB4** / **CB5** / **CB6** match against `.claude/hooks/**` directly | `.claude/hooks/` |

## When to escalate to a manual review

If none of these match, walk the user through:

1. Compare the failing prompt as 4.6 vs 4.7 saw it (same input, two model IDs).
2. Look for hedges (`try to`, `if possible`) — these silently weaken instructions on 4.7.
3. Look for absolute rules (`always`, `never`) without scope — 4.7 interprets these maximally.
4. Look for any directive that says "Claude" or "the model" — these often need to be re-tuned per model.

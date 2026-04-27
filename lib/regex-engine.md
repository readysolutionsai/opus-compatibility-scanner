# Regex engine mandate

The opus-compatibility-scanner skill uses **only the Claude Code Grep tool** (PCRE2) for pattern matching during Phase 2. Bash `grep` is NEVER used for pattern matching.

## Rationale

- **PCRE2 semantics everywhere.** Lookarounds, non-greedy quantifiers, named groups, `(?i)` inline flags — all work the same regardless of the user's machine. Bash `grep` portability varies by platform (BSD vs GNU vs busybox).
- **No `(?i)` portability trap.** Some POSIX `grep` implementations don't support `(?i)`; others require `-i` as a CLI flag. The Grep tool exposes a `case_insensitive` parameter that's normalized.
- **Output structure.** The Grep tool returns structured matches (file, line, column, match_text). Bash `grep -n` requires post-parsing.

## Allowed Bash usage

Bash is still used for **discovery** (Phase 1) — `find`, `ls`, file enumeration. Pattern matching against file contents goes through the Grep tool.

## Pattern field convention

Every pattern's `match.engine` field is `pcre2` for regex-based patterns or `structural` for patterns that require parsing (e.g., CB6 prefill detection). Structural patterns reference a named check function declared in this directory.

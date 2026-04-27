# Sample Scan: Compatibility Mode on a Synthetic TypeScript Project

This file shows what a compatibility scan looks like end-to-end, on a small synthetic project that has been deliberately seeded with four findings.

## Synthetic project state

The project being scanned contains, among other files:

- `CLAUDE.md` with three seeded issues:
  - An A1 unscoped absolute: `Never use any in TypeScript.` (no scope qualifier — applies even in cases where `any` is the right call, like generics in adapter shims).
  - An R3 contradiction pair: an "ask first" rule and a "default to depth" rule, with no precedence block resolving how they interact.
  - An F7 scaffolding directive: `After every 5 tool calls, summarize progress.` — a 4.6-era pattern 4.7 no longer needs.
- `.claude/settings.json` with two seeded issues:
  - A CB1 model-ID hit: `"model": "claude-opus-4-6"`.
  - A CB2 deprecated `thinking` shape: `"thinking": { "type": "enabled", "budget_tokens": 8000 }`.

The scan runs in Compatibility mode (option 1), which means model-ID and `thinking`-shape findings should surface as `guided` rather than the 4.7-only-mode `direct` apply tier. The F7 scaffolding finding is suppressed in compat mode (it's a prose-🔴 4.7-only optimization) and surfaces only in the end-of-scan offer.

## Transcript

**Claude:** I'm using the opus-compatibility-scanner skill to scan your project for Opus 4.6 / 4.7 compatibility issues.

Before I read anything, pick a mode:

  1. **Compatibility scan** — for projects running both Opus 4.6 and 4.7. Holds back 4.7-only optimizations that would degrade 4.6.
  2. **4.7-only scan** — for projects fully migrated off 4.6. Surfaces every finding with no holdback.
  3. **Diagnose a behavior change** — you switched to 4.7 and something broke. Scans in compat mode, then surfaces symptom-mapped findings first.

Reply `1`, `2`, or `3`.

---

**User:** 1

---

**Claude:** Running compatibility scan.

**Scanned 12 files.**
*Pattern catalog current through 2026-04-26.*

- 🔴 **Critical:** 1 — Deprecated `thinking` shape
- 🟡 **Warning:** 2 — Unscoped `never use`; ask-first / depth contradiction
- 🔵 **Info:** 1 — Old model ID string

Recommend fixing Critical first. Top is `Deprecated thinking shape` in `.claude/settings.json:14`.

Walk through the Criticals? (yes / skip / stop)

---

**User:** yes

---

**Claude:**

🔴 **Critical #1 of 1** — Deprecated `thinking` shape · `.claude/settings.json:14`

*What 4.7 will do:* Reject the request with HTTP 400. Opus 4.7 only accepts `thinking: { type: "adaptive" }` (optionally with `display`). The legacy `{ type: "enabled", budget_tokens: N }` shape was removed.

*Fix:*
> Replace `"thinking": { "type": "enabled", "budget_tokens": 8000 }` with `"thinking": { "type": "adaptive" }`.

*Source:* Tier 1 — [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

Apply? (yes / skip / explain)

---

**User:** yes

---

**Claude:**

```diff
--- a/.claude/settings.json
+++ b/.claude/settings.json
@@ -11,7 +11,7 @@
   "model": "claude-opus-4-6",
   "claudeCode": {
     "effortLevel": "high",
-    "thinking": { "type": "enabled", "budget_tokens": 8000 }
+    "thinking": { "type": "adaptive" }
   },
   "permissions": {
     "allow": ["Bash(npm run build)"]
```

Edit applied to `.claude/settings.json`.

---

**Claude:** Done with Criticals — 1 applied, 0 skipped. Move on to Warnings?

---

**User:** yes

---

**Claude:**

🟡 **Warning #1 of 2** — Unscoped absolute (A1) · `CLAUDE.md:18`

*What 4.7 will do:* Read `Never use any in TypeScript.` literally and refuse `any` even where it is the right call (generic adapter shims, third-party-API boundary types). 4.6 inferred a sensible scope; 4.7 will not.

*Fix:*
> Add a scope qualifier: `Never use any in TypeScript, except in adapter shims at third-party-API boundaries where the upstream type is genuinely unknown.`

*Source:* Tier 1 — [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

Apply? (yes / skip / explain)

---

**User:** skip

---

**Claude:**

🟡 **Warning #2 of 2** — Contradiction pair without precedence (R3) · `CLAUDE.md:5,32`

*What 4.7 will do:* Two competing directives — "ask first when unsure" (line 5) and "default to depth, do not stop to ask during research" (line 32) — target overlapping axes. 4.6 silently arbitrated. 4.7 may resolve unpredictably session-to-session.

*Fix (manual — not auto-applied):*
> Add a `PRECEDENCE` block that names which directive wins in which situation. Example: *"User-facing decisions → Ask First wins. Internal research after the user has approved the direction → Depth wins. Low-stakes reversible choices inside an approved task → pick the nearest existing pattern, do not ask."*

*Source:* Tier 1 — [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

This is `apply_tier: manual` — the scanner does not edit contradiction-pair findings because the right precedence wording is project-specific. Reply `next` to continue.

---

**User:** next

---

**Claude:** Done with Warnings — 0 applied, 1 skipped, 1 manual recommendation. Move on to Info?

🔵 **Info #1 of 1** — Old model ID string (CB1) · `.claude/settings.json:11`

*What 4.7 will do:* `"claude-opus-4-6"` is still a valid Active model per Anthropic's deprecations page; nothing breaks today. Surfaced as Info because compatibility mode assumes some call sites are intentionally pinned to 4.6.

*Fix (guided — config-level 🔴 downgraded by compat mode):*
> Change `"model": "claude-opus-4-6"` to `"model": "claude-opus-4-7"` only if this call site is meant to run on 4.7.

*Source:* Tier 1 — [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

This will edit `.claude/settings.json`. Confirm this call site is 4.7-bound? (yes / skip)

---

**User:** skip

---

**Claude:** Done with Info — 0 applied, 1 skipped.

There is 1 4.7-only optimization I held back: 1 scaffolding deletion.

Want to see it now? Reply `yes` only if you've fully moved off 4.6.

---

**User:** no

---

**Claude:** Want to save these findings to disk? Both default to no.

- **Manifest** (`.claude/opus-4.7-migration-manifest.json`) — structured JSON with finding IDs, file:line anchors, and tier assignments. Useful for resuming this scan in a later session or feeding results into CI.
- **Report** (`.claude/opus-4.7-migration-report.md`) — human-readable Markdown summary of every finding.

Reply `manifest`, `report`, `both`, or `no`.

---

**User:** no

---

**Claude:** Scan complete. 1 applied, 3 skipped. Nothing else written.

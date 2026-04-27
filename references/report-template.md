# Report template

The skill responds in conversation as plain markdown — never wrapped in a fenced code block. The shapes below are the canonical layouts for each conversational turn. Severity emoji are inline (🔴 🟡 🔵) so each row is scannable at a glance. Keep responses tight; the user drives depth via reply.

The intro paragraph is shown only once, in **Phase 0** (before the scan starts), as part of the mode-selection prompt. Every shape below is post-scan and assumes the intro has already been delivered.

---

## Executive summary (first response after scan)

Render as plain markdown. Replace `<...>` placeholders with actual values; omit any row whose count is zero EXCEPT the Critical row, which always renders even at zero.

**Scanned `<N>` files.**
*Pattern catalog current through 2026-04-26.*

- 🔴 **Critical:** `<C>` — `<one-line gist>`
- 🟡 **Warning:** `<W>` — `<one-line gist>`
- 🔵 **Info:** `<I>` — `<one-line gist>`

Recommend fixing Critical first. Top is `<pattern name>` in `<path>:<line>`.

Walk through the Criticals? (`yes` / `skip` / `stop`)

---

## Executive summary — zero criticals variant

**Scanned `<N>` files.** *Pattern catalog current through 2026-04-26.* No critical issues. `<W>` warnings, `<I>` info — want to look?

---

## Diagnose-mode prepend (option 3 only, when symptom-mapped findings exist)

When the user picked option 3 in Phase 0 and at least one symptom-mapped finding hit, prepend this subsection above the standard severity rollup:

**Most relevant to "`<short symptom paraphrase>`":**

- `<severity emoji>` `<pattern name>` in `<path>:<line>` — `<one-line gist>`
- `<severity emoji>` `<pattern name>` in `<path>:<line>` — `<one-line gist>`

Then continue with the normal **Scanned `<N>` files.** rollup.

If no symptom-mapped finding hits, omit the subsection. Add an offer at the end of the scan to walk `references/symptoms.md` interactively.

---

## Drill-down (one finding per turn, after the user asks)

`<severity emoji>` **`<Severity>` #N of `<total>`** — `<pattern name>` · `<path>:<line>`

*What 4.7 will do:* `<one or two sentences from the pattern doc.>`

*Fix:*
> `<concise rewrite — never the expanded form by default>`

*Source:* Tier `<N>` — [`<short citation label>`](`<URL>`)

Apply? (`yes` / `skip` / `explain`)

The severity emoji must match the finding's severity: 🔴 Critical, 🟡 Warning, 🔵 Info. Use `·` (middle dot) to separate pattern name from `path:line`.

---

## After the user accepts a fix

Show a one-line confirmation followed by the diff. The diff itself stays in a fenced `diff` code block — that is the only place a fence is appropriate, because `+` and `-` prefixes need monospace alignment to read correctly.

Applied `<pattern_id>` in `<path>:<line>`:

```diff
- <old line>
+ <new line>
```

Then move straight into the next finding.

---

## After the user types "explain"

Render the expanded rewrite plus the full "why 4.6 was fine / how 4.7 mis-reads" rationale for that one finding only. Use plain markdown subheadings (`*Why 4.6 was fine:*`, `*How 4.7 mis-reads:*`, `*Expanded rewrite:*`) — no fenced code block. Re-prompt for apply at the end.

---

## End-of-tier handoff

Done with Criticals — `<X>` applied, `<Y>` skipped. Move on to Warnings?

Same one-line shape at the end of Warnings → Info, and at the end of Info → final summary.

---

## Final summary

Done. Applied `<total>`, skipped `<skipped>`. Run your build to verify, then re-scan to confirm zero findings remain.

---

## Variation: zero findings

**Scanned `<N>` files.** No issues flagged for Opus 4.7. If your agent is still behaving oddly, ask me to walk through `references/symptoms.md` — it maps behaviors to less-obvious patterns the scan can miss.

---

## Variation: single-file scan

When the user asks to scan one file (e.g., "check just my CLAUDE.md"), the executive summary names the file in line one:

**Scanned `<path>`.**

- 🔴 **Critical:** `<C>`
- 🟡 **Warning:** `<W>`
- 🔵 **Info:** `<I>`

(...same recommendation + walk-through prompt as above)

---

## End-of-scan offer (compat mode, when 🔴 findings exist)

There are also `<N>` 4.7-only optimizations I held back: `<categorical summary>`.

Want to see them now? Reply `yes` only if you've fully moved off 4.6.

`<categorical summary>` is a comma-separated list of `<count> <label>` entries, e.g., `2 scaffolding deletions, 1 verbosity-cap removal`. Labels come from the SKILL.md categorical-labels lookup table.

---

## End-of-scan manifest offer (both modes)

After the final summary AND after the optimization offer (if shown), append:

Want to save these findings to disk? Both default to no.

- **Manifest** (`.claude/opus-4.7-migration-manifest.json`) — structured JSON with finding IDs, file:line anchors, and tier assignments. Useful for resuming this scan in a later session or feeding results into CI.
- **Report** (`.claude/opus-4.7-migration-report.md`) — human-readable Markdown summary of every finding.

Reply `manifest`, `report`, `both`, or `no`.

Write the requested file(s) only after the user replies. The manifest schema is in `lib/manifest-schema.json`. If the user replies `no` or anything other than the four options, write nothing.

---

## Dedup display (drill-down on opt-in)

When multiple 🔴 patterns dedup to one finding (overlapping match spans on the same line), the drill-down header lists all merged IDs:

🟡 **4.7-only optimization #N of `<total>`** — `F3 + F6 + F7 (merged)` · `<path>:<line>`

*Category:* `<pattern category name>`

(...rest of drill-down shape unchanged)

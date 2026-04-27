# Config / API Pattern Catalogue (Class B)

Patterns that appear in `.claude/settings.json`, hook scripts, package manifests, and source code that calls the Anthropic SDK. Unlike Class A (prose), most Class B findings are **structural and verifiable** — model IDs are wrong or right, API parameters are accepted or 400-error, SDK versions are above or below a floor.

> All findings here are Tier 1 — verified against Anthropic's official migration guide and the relevant SDK release notes.

> **Compatibility-mode behavior:** any pattern annotated `🔴 4.7-tradeoff` has its `apply_tier` downgraded from `direct` to `guided` in compatibility mode. The drill-down prompt for those findings asks the user to confirm the call site is 4.7-bound before applying the edit. In 4.7-only mode, the default `apply_tier` is restored.

## Schema reference

Class B patterns follow the same record schema as Class A (see `patterns-prose.md` schema docblock). Class-B-specific notes:

1. `match.high_precision` is ignored — the Phase 3 semantic-judgment step is a Class-A-prose-only mechanism.
2. `applies_to_files` typically includes source globs (`*.ts`, `*.js`, `*.mjs`, `*.py`, `*.go`, `*.rs`, etc.) and config files (`.claude/settings.json`, `package.json`, `wrangler.toml`), not Markdown files.
3. `apply_tier` defaults to `direct` for single-line API parameter strips and `guided` for multi-line refactors.

---

## CB1 — Old model ID strings (`claude-opus-4-6`, etc.)

- **Match regex:** `claude-(opus|sonnet)-4-6(-\d{8})?`
  - Also: `anthropic\.claude-(opus|sonnet)-4-6(-v\d+)?` (Bedrock)
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** Info (4.6 is **not** deprecated; this is opt-in)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `**/*.toml`, `**/*.yaml`, `**/*.yml`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** New ID is `claude-opus-4-7`. Bedrock: `anthropic.claude-opus-4-7`. Vertex: `claude-opus-4-7`. No dated `-YYYYMMDD` snapshot exists for 4.7.
- **Why it's only Info:** 4.6 is still Active (not deprecated as of 2026-04-24); Anthropic guarantees 60 days' notice before retirement and the earliest 4.6 retirement is 2027-02-05. Migrating to 4.7 is recommended for capability, not urgency.
- **Where it usually lives:** `.claude/settings.json` (`model` field), `worker/wrangler.toml`, `.env`, source files calling `client.messages.create({ model: "..." })`, package versions of the SDK.
- **Concise rewrite:** Update model ID: `claude-opus-4-6` → `claude-opus-4-7`. Bedrock: `anthropic.claude-opus-4-6` → `anthropic.claude-opus-4-7`.
- **Expanded rewrite:**
  ```diff
  - model: "claude-opus-4-6"
  + model: "claude-opus-4-7"
  ```
- **Source:** [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview); [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

---

## CB2 — Deprecated `thinking` shape (`{type: "enabled", budget_tokens: N}`)

- **Match regex:**
  ```yaml
  - "thinking[^}]*type[^}]*['\"]enabled['\"]"
  - "\\bbudget_tokens\\s*:"
    proximity_constraint: within 200 chars of `messages.create|anthropic\\.|claude-opus|thinking\\s*:`
  ```
  The `budget_tokens` alt is anchored to Anthropic context — avoids matches on `max_budget_tokens`, `user_budget_tokens`, comments, and unrelated config files where the identifier coincidentally appears.
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** **Critical** (returns HTTP 400 on 4.7)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `.claude/hooks/**`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** `thinking.type: "enabled"` is removed. `adaptive` is the only thinking-ON mode on 4.7. `thinking: {type: "disabled"}` and omitting the `thinking` field entirely both remain valid (no thinking; no error). `budget_tokens` is gone entirely.
- **Concise rewrite:** Replace `thinking: { type: "enabled", budget_tokens: N }` with `thinking: { type: "adaptive", display: "summarized" }`. Drop `budget_tokens`. To deepen reasoning, raise `output_config.effort` to `"xhigh"`.
- **Expanded rewrite:**
  ```diff
  - thinking: { type: "enabled", budget_tokens: 16000 }
  + thinking: { type: "adaptive", display: "summarized" }
  ```
  - Drop `budget_tokens`. To get more reasoning on 4.7, raise `output_config.effort` to `"xhigh"` instead.
  - Note: adaptive thinking is **off by default** on 4.7 — requests with no `thinking` field run without thinking. This is a behavior change from 4.6.
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide), [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)

---

## CB3 — Non-default `temperature` / `top_p` / `top_k`

- **Match regex:**
  ```yaml
  - "\\b(temperature|top_p|top_k)\\s*[:=]\\s*[0-9.]+"
  - proximity_constraint: within 500 chars of `anthropic|Anthropic|messages.create|claude-opus|claude-sonnet|claude-haiku`
  ```
  The proximity constraint avoids false matches in IoT/ML code that uses these parameter names for unrelated meanings (e.g., a DHT22 temperature sensor reading).
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** **Critical** (any non-default value returns 400 on 4.7)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `.claude/hooks/**`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** These sampling parameters are removed. Any non-default value sent in the request body returns HTTP 400.
- **Concise rewrite:** Strip `temperature`, `top_p`, `top_k` entries from request bodies on 4.7 calls.
- **Expanded rewrite:** Strip them entirely.
  ```diff
  - temperature: 0.7,
  - top_p: 0.95,
  ```
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB4 — Removed beta headers

- **Match regex:** `effort-2025-11-24|fine-grained-tool-streaming-2025-05-14|interleaved-thinking-2025-05-14`
- **Severity:** Warning (no error, but unnecessary and may shadow GA behavior)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `.claude/hooks/**`
- **Apply tier:** `guided`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** All three are now GA. Sending the beta headers is harmless but should be removed for cleanliness.
- **Concise rewrite:** Remove `effort-2025-11-24`, `fine-grained-tool-streaming-2025-05-14`, `interleaved-thinking-2025-05-14` from `betas` arrays. Move from `client.beta.messages.create` to `client.messages.create`.
- **Expanded rewrite:** Strip from `headers` / `betas` array, and move from `client.beta.messages.create` back to `client.messages.create`.
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB5 — Deprecated `output_format` key

- **Match regex:** `\boutput_format\s*[:=]`
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** Warning (deprecated; still works but should migrate)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `.claude/hooks/**`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** Replaced by `output_config.format`.
- **Concise rewrite:** Migrate `output_format: {...}` → `output_config: { format: {...} }`.
- **Expanded rewrite:**
  ```diff
  - output_format: { type: "json_schema", schema: ... }
  + output_config: { format: { type: "json_schema", schema: ... } }
  ```
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB6 — Assistant-message prefills (last-turn only)

- **Match:**
  ```yaml
  match:
    engine: structural
    check_function: messages-array-last-element-is-assistant
    description: |
      Parse the source file's `messages: [...]` array (when present in a `client.messages.create()` call).
      If the LAST element has `role: 'assistant'`, this is a trailing-prefill that returns 400 on 4.7.
      Mid-conversation assistant messages are valid and not flagged.
  ```
  Last-turn detection requires AST-level parsing that Edit-mode cannot safely apply, so `apply_tier: manual`.
- **Severity:** **Critical** (last-turn assistant-message prefills return HTTP 400 on 4.7)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`, `.claude/hooks/**`
- **Apply tier:** `manual` (structural check — requires AST analysis of messages array)
- **Fidelity:** `verbatim`
- **Source quote:** `"Claude Mythos Preview, Claude Opus 4.7, Claude Opus 4.6, and Claude Sonnet 4.6 do not support prefilling assistant messages. Sending a request with a prefilled last assistant message to any of these models returns a 400 invalid_request_error"`
- **Source URL:** `https://platform.claude.com/docs/en/api/errors`
- **What 4.7 changes:** Last-turn assistant-message prefills return HTTP 400 on 4.7. Prefill removal carries over from Opus 4.6 — it is not new in 4.7. The migration guide describes this as "Prefill removal (carried over from Opus 4.6)." The errors page documents the verbatim 400 response for the prefilled-last-message case.
- **Concise rewrite:** Drop trailing assistant-message prefills from `messages` arrays. Use `output_config.format` for structured output, or system prompts for tone steering.
- **Expanded rewrite:** Use `output_config.format` for structured outputs, or system prompts for tone/format steering. Mid-conversation assistant turns remain valid — only drop the LAST element if it has `role: "assistant"`.
- **Source:** [API errors page](https://platform.claude.com/docs/en/api/errors); reinforced by the [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB7 — SDK version below 4.7 floor

- **What to check:** parse manifests for SDK versions:
  - `package.json` `dependencies` / `devDependencies` — `@anthropic-ai/sdk` must be **≥ 0.90.0**
  - `package.json` — `@anthropic-ai/claude-agent-sdk` must be **≥ 0.2.111** (TypeScript) or **≥ 0.2.112** confirmed working
  - `requirements.txt` / `pyproject.toml` — `anthropic` must be **≥ 0.96.0**, `claude-agent-sdk` must be **≥ 0.2.111**
- **Severity:** **Critical** if a 4.7 model ID is used with a too-old SDK (you'll see `thinking.type.enabled` API errors)
- **Applies to:** `package.json`, `requirements.txt`, `pyproject.toml`, `Pipfile`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **Concise rewrite:** Bump SDK floors: `@anthropic-ai/sdk` ≥0.90.0 (TS); `anthropic` ≥0.96.0 (Python); `claude-agent-sdk` ≥0.2.111. Run `npm update` or `pip install -U anthropic`.
- **Expanded rewrite:** bump the version in the manifest. Run `npm update @anthropic-ai/sdk` or `pip install -U anthropic`.
- **Source:** [anthropic-sdk-python releases](https://github.com/anthropics/anthropic-sdk-python/releases) v0.96.0 (2026-04-16); [anthropic-sdk-typescript releases](https://github.com/anthropics/anthropic-sdk-typescript/releases) v0.90.0 (2026-04-16)

---

## CB8 — Renamed SDK package (Code → Agent) + breaking behavior change

**You'll likely hit this during a 4.7 migration even though it predates 4.7** — the `@anthropic-ai/claude-code` → `claude-agent-sdk` rename happened at v0.1.0 in 2025. Listed here because the silent settings-loading change is precisely the kind of breakage that surfaces when teams audit their stack during a model swap.

- **Match regex:** `@anthropic-ai/claude-code-sdk|^claude-code-sdk\b|claude_code_sdk|ClaudeCodeOptions`
- **Severity:** **Critical** (breaking change at v0.1.0 — old imports may keep resolving, but the new package no longer reads CLAUDE.md / settings.json by default)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`
- **Apply tier:** `manual` (multi-file refactor — package rename + opt-in flag requires coordination)
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:**
  - **Package rename:** `@anthropic-ai/claude-code` → `@anthropic-ai/claude-agent-sdk` (TypeScript). Python: `claude_code_sdk` → `claude_agent_sdk`.
  - **Python type rename:** `ClaudeCodeOptions` → `ClaudeAgentOptions`.
  - **Behavior change at v0.1.0:** the Agent SDK stopped reading `CLAUDE.md` and `.claude/settings.json` by default. Code that relies on those files being auto-loaded silently breaks until the caller opts back in.
  - **Opt back in** via the `settingSources` constructor option: `settingSources: ['user', 'project', 'local']` restores the previous auto-load behavior.
- **Concise rewrite:** Rename imports: `@anthropic-ai/claude-code` → `@anthropic-ai/claude-agent-sdk`, `claude_code_sdk` → `claude_agent_sdk`. Rename Python type `ClaudeCodeOptions` → `ClaudeAgentOptions`. Add `settingSources: ['user', 'project', 'local']` to opt back into CLAUDE.md/settings.json reads on Agent SDK ≥v0.1.0.
- **Expanded rewrite:**
  ```diff
  - import { query } from "@anthropic-ai/claude-code-sdk";
  + import { query } from "@anthropic-ai/claude-agent-sdk";
  ```
  ```diff
  - from claude_code_sdk import ClaudeCodeOptions
  + from claude_agent_sdk import ClaudeAgentOptions
  ```
  ```diff
  - new ClaudeAgent({ apiKey })
  + new ClaudeAgent({ apiKey, settingSources: ['user', 'project', 'local'] })
  ```
- **Source:** [Agent SDK migration guide](https://code.claude.com/docs/en/agent-sdk/migration-guide)

---

## CB9 — `.claude/settings.json` `effortLevel` worth raising on 4.7

- **What to check:** parse `.claude/settings.json`. If `effortLevel` is `"low"` or `"medium"` AND `model` is `"opus"`, `"opus[1m]"`, `claude-opus-4-7*`, or unset (default), recommend raising to `"xhigh"`.
- **Severity:** Info (advisory)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** New `xhigh` effort level is available only on Opus 4.7. Anthropic guidance: *"If you observe shallow reasoning on complex problems, raise effort to high or xhigh rather than prompting around it."* Claude Code v2.1.117+ defaults to `xhigh` on Opus 4.7.
- **Concise rewrite:** In `.claude/settings.json`, raise `effortLevel` from `low`/`medium` to `xhigh` for Opus 4.7 pinning. Claude Code v2.1.117+ defaults to xhigh on 4.7.
- **Expanded rewrite:**
  ```diff
  - "effortLevel": "medium"
  + "effortLevel": "xhigh"
  ```
- **Source:** [Effort docs](https://platform.claude.com/docs/en/build-with-claude/effort), [Claude Code changelog v2.1.117](https://code.claude.com/docs/en/changelog); reinforced by [Best practices for using Claude Opus 4.7 with Claude Code](https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code) (xhigh-default guidance)

---

## CB10 — Token-estimation code (4.7 tokenizer changed)

- **Match regex:** `(chars\s*\/\s*4|length\s*\/\s*4|tiktoken|llama\.tokenize)` near anywhere model usage estimates exist
- **Severity:** Warning
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`
- **Apply tier:** `guided`
- **Fidelity:** `paraphrased` — paraphrased from migration guide on new tokenizer behavior
- **What 4.7 changes:** New tokenizer; same input text uses **1.0x to 1.35x** more tokens than 4.6. Client-side estimates are now wrong.
- **Concise rewrite:** Replace local token-estimation math with `count_tokens` endpoint, OR widen budget guardrails by 35%.
- **Expanded rewrite:** Replace local estimation with the Anthropic `count_tokens` endpoint, OR widen budget guardrails by 35%.
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB11 — `thinking.display` default change (silent)

- **Match regex:** Streaming UI code that reads `thinking` content blocks WITHOUT explicitly setting `display: "summarized"`.
- **Severity:** Warning (silent change — no error, but UI shows blank thinking section)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`, `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mjs`, `**/*.py`, `package.json`, `requirements.txt`, `pyproject.toml`, `wrangler.toml`
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **What 4.7 changes:** Default `display` is now `"omitted"` (was `"summarized"` on 4.6). UI that expected to render summarized thinking will see empty fields unless you opt back in.
- **Concise rewrite:** Set `thinking.display: "summarized"` explicitly. Default changed to `"omitted"` on 4.7.
- **Expanded rewrite:**
  ```diff
  - thinking: { type: "adaptive" }
  + thinking: { type: "adaptive", display: "summarized" }
  ```
- **Source:** [Migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

---

## CB12 — [reserved]

An earlier draft pattern at this slot duplicated CB27's image-coordinate scale-factor handling. The slot is intentionally left vacant rather than renumbering CB14–CB33; pattern IDs are stable references in findings and shouldn't shift.

---

## CB13 — [reserved]

An earlier draft pattern at this slot was withdrawn during pre-launch review. The slot is intentionally left vacant rather than renumbering CB14–CB33; pattern IDs are stable references in findings and shouldn't shift.

---

## CB14 — Claude Code version below 4.7 floor

- **What to check:** the Claude Code CLI version (`claude --version`). Must be **v2.1.111+** for Opus 4.7 support.
- **Severity:** **Critical** if the project pins to 4.7 but the local CLI is older
- **Applies to:** *(structural check — no file glob; runs `claude --version` to verify)*
- **Apply tier:** `direct`
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **Concise rewrite:** Run `claude update` to ensure CLI ≥v2.1.111 (4.7 floor).
- **Expanded rewrite:** `claude update`
- **Source:** [Claude Code model-config docs](https://code.claude.com/docs/en/model-config)

---

## CB15 — `task_budget.total < 20000` returns 400

- **Match regex:** `task_budget\s*[:=]\s*\{[^}]*total\s*[:=]\s*([0-9]+)`
  - Match captures the integer; flag when capture < 20000.
- **Severity:** Critical
- **Apply tier:** direct (config tweak — change the integer literal)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the Anthropic task-budgets docs.
- **Source quote:** `null`
- **What 4.7 changes:** Per task-budgets docs, minimum `task_budget.total` is 20000 tokens. Lower values return HTTP 400.
- **Concise rewrite:**
  ```diff
  - task_budget: { total: 10000 }
  + task_budget: { total: 20000 }
  ```
- **Expanded rewrite:** same as concise; raise the integer to ≥20000.
- **Source:** Tier 1 — https://platform.claude.com/docs/en/build-with-claude/task-budgets

---

## CB16 — `task_budget` without `task-budgets-2026-03-13` beta header

- **Match regex:** files containing `task_budget\s*[:=]` AND NOT containing `task-budgets-2026-03-13`
  - Two-phase: grep for `task_budget`, then verify the file (or imports) include the beta header string.
- **Severity:** Critical
- **Apply tier:** guided (header insertion in betas array; user confirms which file)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the task-budgets docs.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - betas: ["interleaved-thinking-2025-05-14"]
  + betas: ["interleaved-thinking-2025-05-14", "task-budgets-2026-03-13"]
  ```
- **Expanded rewrite:** same as concise.
- **Source:** Tier 1 — task-budgets docs.

---

## CB17 — `task_budget` on non-4.7 model (returns 400)

- **Match regex:** files containing `task_budget` AND a model field set to a non-4.7 ID (`claude-sonnet-4-6`, `claude-haiku-4-5*`, `claude-opus-4-6`)
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** Critical
- **Apply tier:** direct
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the task-budgets docs.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - model: "claude-sonnet-4-6", task_budget: {...}
  + model: "claude-opus-4-7", task_budget: {...}
  ```
- **Expanded rewrite:** same as concise; OR drop the `task_budget` field if the call must stay on a non-4.7 model.
- **Source:** Tier 1 — task-budgets docs.

---

## CB18 — Deprecated tool name strings

- **Match regex:** `text_editor_20250124|str_replace_editor|code_execution_20250124`
- **Severity:** Critical
- **Apply tier:** direct (string substitution)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the migration guide.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - { type: "text_editor_20250124", name: "str_replace_editor" }
  + { type: "text_editor_20250728", name: "text_editor" }
  ```
  For code_execution: `code_execution_20250124` → `code_execution_20250825`.
- **Expanded rewrite:** same as concise with full tool reference.
- **Source:** Tier 1 — migration guide.

---

## CB19 — `undo_edit` command references (no longer exists)

- **Match regex:** `\bundo_edit\b`
  - proximity constraint: within 200 chars of `text_editor` or tool-use context
- **Severity:** Critical
- **Apply tier:** direct (delete the command branch / case)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the migration guide.
- **Source quote:** `null`
- **Concise rewrite:** Delete `undo_edit` command branches. The new text_editor has no undo primitive — handle reverts via re-edit with the previous content.
- **Expanded rewrite:** same; consider tracking edit history client-side if undo UX is required.
- **Source:** Tier 1 — migration guide.

---

## CB20 — `display` combined with `thinking.type: "disabled"`

- **Match regex:** files containing both `thinking[^}]*type[^}]*['"]disabled['"]` AND `display\s*:`
- **Severity:** Critical
- **Apply tier:** guided (drop the `display` field; user confirms)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the adaptive-thinking docs.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - thinking: { type: "disabled", display: "summarized" }
  + thinking: { type: "disabled" }
  ```
- **Expanded rewrite:** same; if you need summarized display, switch to `thinking: { type: "adaptive", display: "summarized" }`.
- **Source:** Tier 1 — adaptive-thinking docs.

---

## CB21 — `xhigh_effort` missing from `_SUPPORTED_CAPABILITIES` for pinned 4.7

- **Match regex:** `.claude/settings.json` contains `"model"\s*:\s*"claude-opus-4-7` AND `_SUPPORTED_CAPABILITIES` does NOT contain `xhigh_effort`
- **Severity:** Critical (pinned model + missing capability advertisement)
- **Apply tier:** guided
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`
- **Fidelity:** `paraphrased` — claim derives from the model-config docs.
- **Source quote:** `null`
- **Concise rewrite:** Add `"xhigh_effort"` to the `_SUPPORTED_CAPABILITIES` array in settings.json.
- **Expanded rewrite:** same; verify the rest of the array against current model-config docs.
- **Source:** Tier 1 — model-config docs.

---

## CB22 — `permission_mode` snake_case (should be `permissionMode`)

- **Match regex:** agent frontmatter (`.claude/agents/*.md` head) contains `^permission_mode\s*:`
- **Severity:** Critical (silently ignored)
- **Apply tier:** direct (rename key)
- **Applies to:** `.claude/agents/*.md`
- **Fidelity:** `paraphrased` — claim derives from the sub-agents docs.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - permission_mode: edit
  + permissionMode: edit
  ```
- **Expanded rewrite:** same.
- **Source:** Tier 1 — [Sub-agents docs](https://code.claude.com/docs/en/sub-agents)

---

## CB23 — Tool-allowlist key swapped between agents and skills

- **Match regex:** agent frontmatter (`.claude/agents/*.md` head) contains `^allowed-tools\s*:` — should be `tools:`. Skill frontmatter (`.claude/skills/*/SKILL.md` head) contains `^tools\s*:` — should be `allowed-tools:`. The pattern fires on either match; the two checks are independent.
- **Severity:** Critical (key silently ignored — listed tools are not actually allow-listed)
- **Apply tier:** direct
- **Applies to:** `.claude/agents/*.md`, `.claude/skills/*/SKILL.md`
- **Fidelity:** `paraphrased` — sub-agents docs document `tools:` as the canonical agent field; skills docs document `allowed-tools:` as the canonical skill field.
- **Source quote:** `null`
- **Concise rewrite:** Agent files: rename `allowed-tools:` → `tools:`. Skill files: rename `tools:` → `allowed-tools:`.
- **Expanded rewrite:** same — the keys are not interchangeable. The agent-side field is `tools` (subagents docs); the skill-side field is `allowed-tools` (skills docs). Either misnaming silently disables the allowlist for that file.
- **Source:** Tier 1 — [sub-agents docs](https://code.claude.com/docs/en/sub-agents) (`tools` for agents) + [skills docs](https://code.claude.com/docs/en/skills) (`allowed-tools` for skills).

---

## CB24 — Plugin subagents with `hooks`/`mcpServers`/`permissionMode` silently dropped

- **Match regex:** plugin agent files (`.claude/plugins/*/agents/*.md`) frontmatter contains any of `hooks:`, `mcpServers:`, `permissionMode:`
- **Severity:** Critical (silently dropped at runtime)
- **Apply tier:** manual (refactor to plugin-config-level scope)
- **Applies to:** `.claude/plugins/*/agents/*.md`
- **Fidelity:** `verbatim` — `source.quote: "plugin subagents do not support the \`hooks\`, \`mcpServers\`, or \`permissionMode\` frontmatter fields"`
- **Concise rewrite:** Remove the dropped fields from plugin agent frontmatter; declare them at plugin-config level instead.
- **Expanded rewrite:** same with plugin.json restructuring details.
- **Source:** Tier 1 — sub-agents docs.

---

## CB25 — `clear_thinking_20251015` header without per-turn verification

- **Match regex:** files containing `clear_thinking_20251015` (in betas or headers)
- **Severity:** Warning — the underlying bug was patched in Claude Code v2.1.101 per the April 23 postmortem. **Conditional rendering:** if CB14's `claude --version` probe reports v2.1.101+, surface this finding as **Info**. If older or version unknown, render as Warning.
- **Apply tier:** manual (requires careful per-turn audit code on older CLIs; no-op on patched CLIs)
- **Version probe:** reuse the result of CB14's `claude --version` invocation. If unavailable (CB14 didn't run, e.g., scan scoped to a non-Claude-Code project), default to Warning.
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the April 23 postmortem.
- **Source quote:** `null`
- **Concise rewrite (manual):** On Claude Code v2.1.101+, no action required — the bug is patched. On older CLI versions, either remove the header entirely (preferred for new code) OR add the per-turn verification logic per the postmortem's recommended pattern.
- **Expanded rewrite:** same with the verification snippet.
- **Source:** Tier 1 — https://www.anthropic.com/engineering/april-23-postmortem

---

## CB26 — `MAX_THINKING_TOKENS` or `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` set for 4.7

- **Match regex:** `.claude/settings.json` env block contains `MAX_THINKING_TOKENS` OR `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** Critical (silently breaks adaptive thinking on 4.7)
- **Apply tier:** direct (delete the env entries)
- **Applies to:** `.claude/settings.json`, `.claude/settings.local.json`
- **Fidelity:** `paraphrased` — claim derives from the model-config docs.
- **Source quote:** `null`
- **Concise rewrite:**
  ```diff
  - "env": { "MAX_THINKING_TOKENS": "16000", "CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING": "1" }
  + "env": {}
  ```
  Use `effortLevel` instead to control reasoning depth.
- **Expanded rewrite:** same.
- **Source:** Tier 1 — model-config docs.

---

## CB27 — Image coordinate scale-factor math (1568, 1.15MP, etc.)

- **Match regex:** `\b1568\b|1\.15\s*MP|max_image_dimension|scaleImageCoords|resizeForVision`
- **Severity:** Critical (math actively breaks for users uploading >1568px images on 4.7)
- **Apply tier:** manual (vision code refactor)
- **Applies to:** Class B standard glob set + `**/vision*.ts`, `**/image*.ts`
- **Fidelity:** `paraphrased` — claim derives from the migration guide.
- **Source quote:** `null`
- **Concise rewrite (manual):** Remove the scale-factor math; pass image coordinates directly. 4.7 supports 2576px / 3.75MP at 1:1 pixel coordinates.
- **Expanded rewrite:** same with refactor examples.
- **Source:** Tier 1 — migration guide.

---

## CB28 — Stop-reason handlers missing `refusal` or `model_context_window_exceeded`

- **Match regex:** files containing `stop_reason` switch/match/if blocks that handle `end_turn|max_tokens|tool_use` but NOT `refusal` or `model_context_window_exceeded`
- **Severity:** Critical (unhandled stop reasons throw or silently misroute on 4.7)
- **Apply tier:** manual (case-by-case handler insertion)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the migration guide.
- **Source quote:** `null`
- **Concise rewrite (manual):** Add `case "refusal":` and `case "model_context_window_exceeded":` branches. Refusal: surface the model's explanation; do not retry. Context exceeded: trim conversation or split into multiple calls.
- **Expanded rewrite:** same with handler skeletons for each language.
- **Source:** Tier 1 — migration guide.

---

## CB29 — Multi-turn thinking-block filtering strips `signature`

- **Match regex:** files containing thinking-block manipulation (`block.type === "thinking"`) AND a filter/map operation that excludes the `signature` field
- **Severity:** Critical (silently breaks adaptive-thinking continuity across turns; 4.7 uses signature for verification)
- **Apply tier:** manual (requires understanding the filter intent)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the adaptive-thinking docs.
- **Source quote:** `null`
- **Concise rewrite (manual):** When passing thinking blocks back to the API in subsequent turns, preserve the `signature` field on each thinking block exactly as received. Do not redact, regenerate, or strip it.
- **Expanded rewrite:** same with code examples for typical streaming pipelines.
- **Source:** Tier 1 — adaptive-thinking docs.

---

## CB30 — `output_format` deprecated in favor of `output_config.format` (deprecation warning)

- **Match regex:** `\boutput_format\s*[:=]` (same as CB5)
- **Severity:** Warning (deprecation; CB5 covers the migration mechanic; CB30 surfaces the deprecation explicitly so users see it as a soft block)
- **Apply tier:** direct (key rename per CB5)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased`
- **Source quote:** `null`
- **Concise rewrite:** see CB5.
- **Expanded rewrite:** see CB5.
- **Source:** Tier 1 — migration guide.

---

## CB31 — `effort: xhigh` on non-4.7 model (silently falls back to high)

- **Match regex:** files containing `effort\s*[:=]\s*['"]xhigh['"]` AND a model field set to a non-4.7 ID
- **Compatibility:** 🔴 4.7-tradeoff (apply_tier downgraded to guided in compatibility mode; restored to default in 4.7-only mode)
- **Severity:** Warning (silent quality degradation, not an error)
- **Apply tier:** direct
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — claim derives from the effort docs.
- **Source quote:** `null`
- **Concise rewrite:** Either pin `model: "claude-opus-4-7"` for that call, OR drop `xhigh` to `high` to make the fallback explicit.
- **Expanded rewrite:** same.
- **Source:** Tier 1 — effort docs.

---

## CB32 — Low `max_tokens` paired with `effort: xhigh` or `effort: max` (premature truncation)

- **Match regex:** files containing both `effort\s*[:=]\s*['"](xhigh|max)['"]` AND `max_tokens\s*[:=]\s*([0-9]+)` where the captured integer < 65536
- **Severity:** Warning (output gets cut mid-reasoning)
- **Apply tier:** guided (raise max_tokens; user confirms)
- **Applies to:** Class B standard glob set
- **Fidelity:** `paraphrased` — Anthropic's migration guide explicitly recommends *"start at 64k tokens and tune from there"* when running 4.7 at `max` or `xhigh` effort.
- **Source quote:** `null`
- **Concise rewrite:** Raise `max_tokens` to ≥ 65536 when running at `xhigh` or `max` effort. Anthropic recommends starting at 64k and tuning from there.
- **Expanded rewrite:** same with output_config.effort hints.
- **Source:** Tier 1 — migration guide.

---

## CB33 — Image `max_tokens` budget too low for 4.7's ~3x image token cost

- **Match regex:** files containing image input (`type: "image"` in messages) AND `max_tokens` < 4096
- **Severity:** Warning
- **Apply tier:** guided
- **Applies to:** Class B standard glob set + `**/vision*.ts`
- **Fidelity:** `paraphrased` — claim derived from migration-guide tokenizer note (1.0–1.35x text) + image-coverage notes; intent preserved.
- **Concise rewrite:** Raise `max_tokens` to ≥ 4096 for image-input calls on 4.7. Verify against the actual image's token count using `count_tokens`.
- **Expanded rewrite:** same.
- **Source:** Tier 1 — migration guide (tokenizer notes).



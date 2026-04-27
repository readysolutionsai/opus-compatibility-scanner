# Sources & citation tiers

Every finding in the scanner cites a source from this list. Tier 1 sources carry the most authority (Anthropic-authored, current). Lower tiers are practitioner observations and should be presented as such.

## Tier 1 — Official Anthropic documentation

These are the load-bearing sources for the entire pattern catalogue.

- [What's new in Claude Opus 4.7](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7)
- [Migration guide (4.6 → 4.7)](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)
- [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Effort](https://platform.claude.com/docs/en/build-with-claude/effort)
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Best practices for using Claude Opus 4.7 with Claude Code](https://claude.com/blog/best-practices-for-using-claude-opus-4-7-with-claude-code) — published April 16, 2026; covers the xhigh default, adaptive thinking, scaffolding removal, fewer subagents.
- [Memory / CLAUDE.md docs](https://code.claude.com/docs/en/memory)
- [Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code model configuration](https://code.claude.com/docs/en/model-config)
- [Claude Code subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code settings](https://code.claude.com/docs/en/settings)
- [Claude Code changelog](https://code.claude.com/docs/en/changelog)
- [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Agent SDK migration guide](https://code.claude.com/docs/en/agent-sdk/migration-guide)
- [April 23 Claude Code postmortem](https://www.anthropic.com/engineering/april-23-postmortem)
- [Introducing Claude Opus 4.7 (news)](https://www.anthropic.com/news/claude-opus-4-7)
- [anthropic-sdk-python releases](https://github.com/anthropics/anthropic-sdk-python/releases) (v0.96.0 = 4.7 floor)
- [anthropic-sdk-typescript releases](https://github.com/anthropics/anthropic-sdk-typescript/releases) (v0.90.0 = 4.7 floor)
- [AWS Bedrock Claude Opus 4.7 announcement](https://aws.amazon.com/about-aws/whats-new/2026/04/claude-opus-4.7-amazon-bedrock/)
- [AWS Bedrock model card — Claude Opus 4.7](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-opus-4-7.html)
- [Google Vertex AI partner-models — Claude](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/partner-models/claude)

## Tier 2 — Anthropic staff public posts

- Boris Cherny (Anthropic, Claude Code creator), quoted: *"It took a few days for me to learn how to work with [4.7] effectively."* — primary citation: [X post](https://x.com/bcherny/status/2044822408826380440), April 16, 2026. Secondary reference: productcompass.pm.

## Tier 3 — Reputable practitioner consolidations

- Simon Willison — [Changes in the system prompt between Opus 4.6 and 4.7](https://simonwillison.net/2026/Apr/18/opus-system-prompt/)
- MindStudio — [How to Prompt Claude Opus 4.7 Differently Than 4.6](https://www.mindstudio.ai/blog/how-to-prompt-claude-opus-4-7)
  - *Note:* MindStudio's stated Opus 4.6 retirement date conflicts with Anthropic's Tier 1 model-deprecations page. Defer to the deprecations page on date claims.
- KeepMyPrompts — [Claude Opus 4.7 prompting guide: breaking changes and prompt edits](https://www.keepmyprompts.com/en/blog/claude-opus-4-7-prompting-guide-whats-changed)
- ClaudeFast — [Opus 4.7 best practices](https://claudefa.st/blog/guide/development/opus-4-7-best-practices)
- iBuildWith.ai — [Effort, Thinking, and How Claude Opus 4.7 Changed the Rules](https://www.ibuildwith.ai/blog/effort-thinking-opus-4-7-changed-the-rules/)
- HumanLayer — [Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) (instruction budget guidance)

## Tier 4 — Forum aggregators (use with caution)

- botmonster — [Claude Opus 4.7: What X and Reddit Users Are Saying](https://botmonster.com/posts/claude-opus-4-7-x-reddit-reception/)
- findskill.ai, merchmindai — Reddit/X chatter consolidations
- HN thread `news.ycombinator.com/item?id=47823270` — referenced indirectly

## Quote attribution rules

> Patterns marked `paraphrased` are intent-preserved relative to the cited Anthropic page. The cited URL and underlying claim anchor the pattern; the field signals that the wording in the pattern body is the author's, not a literal Anthropic quote.

Every pattern's `source` block carries a `fidelity` field with one of three values, in addition to the `tier` field. The two fields are orthogonal: `tier` describes source authority (1=Anthropic, 4=forum); `fidelity` describes how faithfully the pattern's cited text matches the source page.

- **`verbatim`** — the pattern's `source.quote` field is an exact-string substring of the cited Anthropic page body (allowing whitespace/punctuation normalization). Required when the pattern's "Why this matters" copy presents a quoted phrase as Anthropic's own words.
- **`paraphrased`** — the pattern's "Why this matters" copy expresses the intent of a Tier 1 source in the author's own words. The `quote` field is null (or omitted). Permitted at any severity level.
- **`practitioner`** — the pattern's primary source is a Tier 3 practitioner consolidation; the underlying principle may anchor to a Tier 1 source via `foundational_tier_1` (separate URL field). Severity is capped at Warning regardless of declared severity (cannot be Critical).

**Why three values:** `tier` alone cannot distinguish "Anthropic said this exactly" (`verbatim` Tier 1) from "Anthropic said something whose intent matches this" (`paraphrased` Tier 1) from "Practitioner extension of an Anthropic principle" (`practitioner` Tier 3 + `foundational_tier_1`).

**Pattern authoring rule:** when adding or editing a pattern, choose `fidelity` based on what the "Why this matters" copy presents:
- Quoted Anthropic phrase in copy → `verbatim` + populate `quote`.
- Author paraphrase of an Anthropic page → `paraphrased`.
- Author claim derived from practitioner blogs (mindstudio, keepmyprompts, simonwillison.net, etc.) → `practitioner`.

## How the scanner uses tiers

- **Critical findings** must cite Tier 1 (or Tier 1 + Tier 2 reinforcement). If only Tier 3 evidence exists, downgrade to Warning.
- **Warning findings** can cite Tier 1, 2, or 3.
- **Info findings** can cite any tier.
- When showing a finding to the user, include the citation URL and the tier. Example: *"Source: Tier 1 — Anthropic migration guide (link)."*

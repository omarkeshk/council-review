# council-review — multi-model code review in Claude Code

A distributable Claude Code skill that has your diff reviewed by frontier models from multiple AI labs at once — Anthropic, OpenAI, Google, xAI, DeepSeek, and more — through the Council AI MCP server. Every model reviews independently; a moderator cross-checks and returns a **verdict-first** synthesis:

1. **SHIP / NO-SHIP** with a consensus score
2. **Confirmed findings** — issues 2+ models independently agree on
3. **Dissents** — issues only one model flagged, with that model's reasoning

Claude Code acts on confirmed findings and leaves dissents as judgment calls for you.

## Requirements

- A [Council AI](https://council-ai.app) account on the **Ultra tier** (the MCP server is Ultra-only).
- A Council Personal Access Token (`csa_…`), minted at <https://council-ai.app/settings?tab=mcp>.
- Claude Code (any recent version with MCP support).

Every review bills against your existing Council monthly budget — same meter as the web app, no separate charges.

## Install

**1. Connect the Council AI MCP server:**

```bash
claude mcp add --transport http council-ai https://mcp.council-ai.app/mcp \
  --header "Authorization: Bearer csa_YOUR_TOKEN"
```

**2. Install the plugin** — inside Claude Code:

```
/plugin marketplace add omarkeshk/council-review
/plugin install council-review@council-ai
```

Or install the skill manually by copying it to your Claude Code skills directory:

```bash
# personal (all projects)
mkdir -p ~/.claude/skills/council-review
curl -fsSL https://raw.githubusercontent.com/omarkeshk/council-review/main/skills/council-review/SKILL.md \
  -o ~/.claude/skills/council-review/SKILL.md

# or per-project (shared with your team via the repo)
mkdir -p .claude/skills/council-review
curl -fsSL https://raw.githubusercontent.com/omarkeshk/council-review/main/skills/council-review/SKILL.md \
  -o .claude/skills/council-review/SKILL.md
```

**3. Verify:** in Claude Code, run `/mcp` — `council-ai` should show as connected with a `council_review` tool.

## Usage

Just ask, in any repo:

> "Get a council review of my staged changes"
> "Multi-model review of this diff, focus on security"
> "I want a second opinion on this diff before I merge"

Or invoke the skill directly: `/council-review`.

## Example session

```
> council review my staged changes, I'm worried about security

⏺ Bash(git diff --staged)
⏺ council-ai — council_review (diff: 4,812 chars, focus: "security",
    context: "Adds a password-reset endpoint with token emailing")

⏺ ## Council Verdict
  **NO-SHIP** — a confirmed HIGH severity token-handling flaw.
  Consensus score: 78%

  Confirmed findings (2+ models):
  - HIGH  src/routes/reset.ts:41 — reset token compared with `==` against the
    stored value; timing-unsafe and the token never expires. Fix: hash the
    token, compare with timingSafeEqual, add a 15-minute TTL. (flagged by
    Claude Opus 4.8, GPT-5.6, Gemini 3.5)

  Dissents:
  - Only Grok 4.5 flagged MEDIUM src/routes/reset.ts:18 — the email lookup
    reveals account existence via response timing. Its reasoning: the 404
    returns ~40ms faster than the send path. Judgment call — worth it if
    account enumeration matters to you.

  Want me to apply the confirmed fix?
```

## What gets sent where

The diff, your optional context sentence, and the focus are sent to the Council AI backend, which fans them to the selected models via their APIs. Reviews are logged for admin observability on Council's side (`mcpQueries`), not stored as chat history. Don't send diffs containing live credentials — the skill instructs Claude Code to redact obvious secrets first.

## Limits

- Diff cap: **14,000 characters** per call. The skill automatically retries with `--unified=1` and then splits by file.
- Latency: expect roughly a minute per call — several frontier models run in parallel plus a synthesis pass.
- Focus values: `bugs`, `security`, `performance`, `architecture`, `all` (default).

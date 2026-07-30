---
name: council-review
description: Multi-model code review via the Council AI MCP server. Use when the user asks for a "council review", "multi-model review", "second opinion on this diff", "have the council look at this", or wants a code review cross-checked by models from multiple AI labs before shipping. Gathers the current git diff, calls the council-ai MCP council_review tool, and reports a verdict-first synthesis with confirmed findings and dissents.
---

# Council Review

Run a multi-model code review: models from multiple labs (Anthropic, OpenAI, Google, xAI, and more) each review the same diff independently, and a moderator returns a verdict-first synthesis — SHIP/NO-SHIP, then findings confirmed by 2+ models, then single-model dissents.

## Prerequisites

The `council-ai` MCP server must be connected (tool `council_review` available). If it is not, tell the user to install it — see the README next to this skill — and stop. Requires a Council AI Ultra-tier Personal Access Token.

## Workflow

### 1. Gather the diff

Pick the diff that matches what the user asked about:

- Default (uncommitted work): `git diff` — if empty, fall back to `git diff --staged`.
- "staged changes": `git diff --staged`.
- "this branch" / "my PR": `git diff <default-branch>...HEAD` (resolve the default branch; usually `main`).
- A specific commit: `git show <sha> --format=""`.
- A pasted snippet or specific file: use it directly.

If there is no diff at all, tell the user and stop — do not invent code to review.

### 2. Respect the size cap

`council_review` caps diffs at **14,000 characters**. If the diff is larger:

1. First retry with reduced context: `git diff --unified=1`.
2. Still too large: split by file (`git diff -- <path>` per file or logical group) and call `council_review` once per chunk, then merge the results in your report (a NO-SHIP on any chunk is a NO-SHIP overall).

Skip lockfiles, generated files, and vendored code when splitting — reviewers add no value there.

### 3. Derive the focus from the user's ask

- Mentions of security, auth, injection, secrets → `focus: "security"`
- Mentions of speed, latency, memory, scale → `focus: "performance"`
- Mentions of design, structure, coupling, patterns → `focus: "architecture"`
- Mentions of bugs, correctness, "will this break" → `focus: "bugs"`
- Otherwise → `focus: "all"` (default)

### 4. Call the tool

Call the council-ai MCP tool `council_review` with:

- `diff`: the diff from step 1/2
- `focus`: from step 3
- `context`: one or two sentences on what the change is supposed to do — pull this from the user's words, the branch name, or recent commit messages. Always provide it when you know the intent; it materially reduces false positives. (Max 3,000 chars.)
- `models`: omit unless the user named specific models (then use `get_models` to resolve IDs).

The call takes on the order of a minute (several frontier models run in parallel) and bills against the user's Council AI monthly budget.

### 5. Present the result

Report in this order, keeping the verdict on the first line:

1. **Verdict** — SHIP or NO-SHIP, with the consensus score.
2. **Confirmed findings** — issues 2+ models agreed on, with severity, location, and the concrete fix.
3. **Dissents** — issues only one model flagged, attributed by model name with that model's reasoning, so the user can judge them.

Then: **act only on confirmed findings** (offer to apply the fixes, or apply them if the user already asked you to). Present dissents as judgment calls for the user — do not act on them unless the user asks. If the verdict is SHIP with no confirmed findings, say so plainly and stop.

## Notes

- Never send secrets: if the diff contains obvious credentials (.env contents, private keys), redact those lines before calling and note the redaction.
- If the tool returns a budget error, report the message as-is — the budget resets monthly on the user's Council plan.

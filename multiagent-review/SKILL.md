---
name: multiagent-review
description: Orchestrates a parallel code review by spawning Claude, Codex, and Cursor Solo agents, then synthesizes their findings. Use when the user asks for /multiagent-review, multi-agent review, Solo reviewers, or parallel Claude/Codex/Cursor review of local changes, commits, branches, or PRs.
---

# Multiagent Review

You are only the orchestrator; Solo agents are the reviewers. This is review-only: do not edit files, format, patch, commit, change branches, or otherwise mutate the codebase. Keep reviewer Solo processes open unless the user explicitly asks to close them.

## Workflow

1. Resolve scope.
   - Accepted targets: commit hash, branch name, PR link, or local changes.
   - For explicit targets, prefer `gh`; fall back to `git` only if `gh` is unavailable. Resolve latest `main` hash, target hash, and diff `main..target` without checking out branches or touching the working tree. For PRs, diff latest `main` against the PR head ref/commit. Ignore unrelated local changes.
   - With no target, review local changes only: include `git diff --cached` and `git diff`. If both are empty, ask for a commit, branch, or PR link.
   - Gather bounded context: `AGENTS.md`, `CONTRIBUTING.md`, `README.md`, `docs/`, formatter/linter config, test conventions, and nearby test patterns.

2. Publish one Solo scratchpad tagged `multiagent-review` when supported. Include:
   - Review scope and assumptions.
   - For explicit targets: latest `main` hash, target hash, and target source.
   - The exact orchestrator-collected diff.
   - Relevant standards excerpts or paths.
   - Required reviewer output schema.

3. Spawn reviewers with Solo, not standard subagents.
   - Use `list_agent_tools` to identify Claude, Codex, and Cursor runtimes.
   - Use `spawn_agent` once per runtime, named clearly such as `review-claude`, `review-codex`, and `review-cursor`.
   - Send the same prompt to all reviewers, prepending any Solo `agent_instructions`.

4. Instruct reviewers:
   - Review only the scratchpad diff; do not run their own diff.
   - They may inspect repository files and standards for context.
   - They must not check out branches, edit files, format, patch, commit, or mutate the codebase.

```markdown
You are one of three independent Solo code reviewers. Review only the changes in Solo scratchpad <scratchpad id/name>; do not run git diff yourself.
Review-only. Do not check out branches, edit files, run formatters, apply patches, create commits, or mutate the codebase. Inspect files only when needed for context.

Review on two axes:
1. Standards: conformance to documented standards, conventions, guidelines, and local patterns.
2. Functionality: correctness, bugs, safety, performance, and best-practice risks.

Return exactly:

## Summary of findings

For each finding:
- File and line(s): <path:line or path:line-line>
- Severity: critical | high | medium | low
- Axis: Standards | Functionality | Both
- Issue: <one sentence>
- Reasoning: <detailed description and reasoning>

If there are no findings, write "No findings."

## Overall verdict

Approve | Require changes | Disapprove

Reason: <brief explanation>
```

5. Synthesize outputs.
   - Normalize equivalent findings even when wording or line numbers differ.
   - Sort by reporting-agent count descending, then maximum severity: `critical`, `high`, `medium`, `low`.
   - Use `Approve` when no material issues remain.
   - Use `Require changes` for any critical, high, or credible medium issue that should block merge, or standards violations that create maintainability risk.
   - Use `Disapprove` when the change is fundamentally unsafe, unworkable, or directionally wrong.

## Final Response

Include one compact GitHub-flavored Markdown findings table outside code fences:

| # | Agents | Location | Severity | Axis | Issue |
|---|---|---|---|---|---|
| F1 | 🔴 🔵 ⚪ | path:line | high | Functionality | short issue summary |

Put three-agent findings first and keep table cells short enough for terminal width.

Then include:

`Legend: 🔴 Claude · 🔵 Codex · ⚪ Cursor`

`Details`
- `F1 : Synthesized reasoning from all reporting agents.`

Finish with a compact verdicts table:

| Agent | Verdict | Reason |
|-------|---------|--------|
| 🔴 Claude | Approve | Implementation is sound. |
| 🔵 Codex | Require changes | Selector can match unintended elements. |
| ⚪ Cursor | Approve | Low residual risk. |
| Orchestrator | Require changes | Unanimous functionality concern should be fixed. |

---
name: multiagent-review
description: Orchestrate a parallel code review by spawning Claude, Codex, and Cursor Solo agents and synthesizing their findings. Use when the user asks for /multiagent-review, multi-agent review, Solo reviewers, or parallel Claude/Codex/Cursor review of local changes, commits, branches, or PRs.
---

# Multiagent Review

Run as the orchestrator only. The Solo agents are the reviewers. This is review-only: do not edit files, run formatters, apply patches, create commits, or change branches.

## Workflow

### 1. Resolve review scope

The review target may be a commit hash, branch name, PR link, or local changes. For commit, branch, and PR targets, use the GitHub CLI (`gh`) to resolve refs and collect diffs without checking out branches; fall back to `git` only when `gh` is not installed.
If the user provides a commit hash, branch name, or PR link, use `gh` to resolve the latest `main` hash, resolve the target hash, then collect the diff from latest `main` to the target. Record both hashes in the scratchpad. With the `git` fallback, perform the same resolution and diff locally without checking out branches or editing the working tree.
If the user provides no target, review local changes only: include staged changes with `git diff --cached` and unstaged changes with `git diff`. If no local changes exist, stop and ask the user for a commit hash, branch name, or PR link.
For PR links, use `gh` to identify the PR head ref/commit and diff the PR head against the latest `main` hash. Do not review unrelated local working-tree changes when an explicit target is provided.
Find governing standards, such as `AGENTS.md`, `CONTRIBUTING.md`, `README.md`, `docs/`, formatter/linter config, test conventions, and nearby test patterns. Keep context bounded.

### 2. Publish one Solo scratchpad

Create a Solo scratchpad containing:
- Review scope and base assumptions.
- For explicit targets: latest `main` hash, target hash, and target source.
- The exact diff, gathered only by the orchestrator.
- Relevant standards excerpts or paths to the governing docs.
- Required reviewer output schema.
Tag it with `multiagent-review` when tags are supported.
Do not ask reviewer agents to run their own diff command. They may inspect repository files and documented standards for context, but the diff they review must come from the scratchpad.
Reviewer agents must not check out branches, edit files, run formatters, apply patches, create commits, or otherwise mutate the codebase.

### 3. Spawn reviewers in parallel

Use Solo, not standard subagents:
1. `list_agent_tools` to identify Claude, Codex, and Cursor runtimes.
2. `spawn_agent` once for each runtime, naming them clearly, for example `review-claude`, `review-codex`, and `review-cursor`.
3. Send each reviewer the same prompt, prepending any `agent_instructions` returned by Solo and invoking `/caveman` in the prompt.
Keep all three agents open after the review. Do not close their Solo processes unless the user explicitly asks.

### 4. Reviewer prompt

Use this prompt shape for every reviewer:

```markdown
/caveman

You are one of three independent Solo code reviewers. Review only the changes in the Solo scratchpad <scratchpad id/name>; do not run git diff yourself.
Review-only. Do not check out branches, edit files, run formatters, apply patches, create commits, or mutate the codebase. Inspect files only when needed for context.
Use caveman mode for token savings, but preserve the exact headings, fields, verdict words, file paths, line numbers, severities, and technical details required below.

Review on two axes:
1. Standards: do the changes conform to documented standards, conventions, guidelines, and local patterns?
2. Functionality: do the changes work, introduce bugs, affect safety/performance, or violate best practice?

Return exactly this structure:

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

### 5. Synthesize

Read all reviewer outputs. Normalize equivalent findings that refer to the same defect even if wording or line numbers differ.
Sort findings by:
1. Number of agents that reported the issue, highest first.
2. Maximum severity, ordered `critical`, `high`, `medium`, `low`.
3. File path and line number.

Use a single Excel-like GitHub-flavored Markdown pipe table every time. Do not output findings as key/value blocks or per-finding cards.
```markdown
Legend: 🔴 Claude, 🔵 Codex, ⚪ Cursor
| Finding | Agents | File/Line(s) | Severity | Axis | Issue | Reasoning Summary |
|---|---|---|---|---|---|---|
| F1 | 🔴 🔵 ⚪ | path:line | high | Functionality | ... | ... |
```

`Agents` shows which reviewers reported the normalized finding. Place all three-agent findings at the top. Keep cells concise so the table remains readable; put additional detail in follow-up sections only when needed.
After the table, include:
- **Reviewer verdicts**: one line each for Claude, Codex, and Cursor.
- **Orchestrator verdict**: `Approve`, `Require changes`, or `Disapprove`.
- **Reasoning**: concise explanation for the orchestrator verdict, explicitly weighing standards and functionality.
- **Follow-up availability**: state that the Solo reviewer agents remain open for follow-up questions.

## Verdict guidance

Use `Approve` when no material issues remain. Use `Require changes` when any critical, high, or credible medium finding should be fixed before merge, or when standards violations create maintainability risk. Use `Disapprove` when the change is fundamentally unsafe, unworkable, or directionally wrong enough that patching individual issues is not the right next step.

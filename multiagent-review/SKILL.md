---
name: multiagent-review
description: Orchestrate a parallel code review by spawning Claude, Codex, and Cursor Solo agents and synthesizing their findings. Use when the user asks for /multiagent-review, multi-agent review, Solo reviewers, or parallel Claude/Codex/Cursor review of local changes, commits, branches, or PRs.
---

# Multiagent Review

Run as an orchestrator only. The Solo agents are the reviewers. This is review-only: do not edit files, run formatters, apply patches, create commits, or change branches.

## Workflow

### 1. Resolve review scope

Review target may be a commit hash, branch name, PR link, or local changes. For commit, branch, and PR targets, use GitHub CLI (`gh`) to resolve refs and collect diffs without checking out branches; fall back to `git` only when `gh` is not installed.
If the user provides a commit hash, branch name, or PR link, use `gh` to resolve latest `main` to a hash, resolve the target to a hash, then collect the target-vs-latest-`main` diff. Record both hashes in the scratchpad. With the `git` fallback, perform the same resolution and diff locally without checkout or working-tree edits.
If the user provides no target, review local changes only: include staged changes with `git diff --cached` and unstaged changes with `git diff`. If no local changes exist, stop and ask the user for a commit hash, branch name, or PR link.
For PR links, use `gh` to identify the PR head ref/commit and diff that head against latest `main`. Do not review unrelated local working-tree changes when an explicit target is provided.
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
Reviewer agents must not checkout branches, edit files, run formatters, apply patches, create commits, or otherwise mutate the codebase.

### 3. Spawn reviewers in parallel

Use Solo, not normal subagents:
1. `list_agent_tools` to identify Claude, Codex, and Cursor runtimes.
2. `spawn_agent` once for each runtime, naming them clearly, for example `review-claude`, `review-codex`, and `review-cursor`.
3. Send each reviewer the same prompt, prepending any `agent_instructions` returned by Solo and invoking `/caveman` in the prompt.
Keep all three agents open after the review. Do not close their Solo processes unless the user explicitly asks.

### 4. Reviewer prompt

Use this prompt shape for every reviewer:

```markdown
/caveman

You are one of three independent Solo code reviewers. Review only the changes in Solo scratchpad <scratchpad id/name>; do not run git diff yourself.
Review-only. Do not checkout branches, edit files, run formatters, apply patches, create commits, or mutate the codebase. Inspect files only when needed for context.
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

approved | requires changes | disapproved

Reason: <brief explanation>
```

### 5. Synthesize

Read all reviewer outputs. Normalize equivalent findings that refer to the same defect even if wording or line numbers differ.
Sort findings by:
1. Number of agents that reported the issue, highest first.
2. Maximum severity, ordered `critical`, `high`, `medium`, `low`.
3. File path and line number.

Use this comparison table every time:
```markdown
Legend: 🔴 Claude, 🔵 Codex, ⚪ Cursor
| Finding | Agents | File/Line(s) | Severity | Axis | Issue | Reasoning Summary |
|---|---|---|---|---|---|---|
| F1 | 🔴 🔵 ⚪ | path:line | high | Functionality | ... | ... |
```

`Agents` shows which reviewers reported the normalized finding. Place all three-agent findings at the top.
After the table, include:
- **Reviewer verdicts**: one line each for Claude, Codex, and Cursor.
- **Orchestrator verdict**: `approved`, `requires changes`, or `disapproved`.
- **Reasoning**: concise explanation for the orchestrator verdict, explicitly weighing standards and functionality.
- **Follow-up availability**: state that the Solo reviewer agents remain open for follow-up questions.

## Verdict guidance

Use `approved` when no material issues remain. Use `requires changes` when any critical, high, or credible medium finding should be fixed before merge, or when standards violations create maintainability risk. Use `disapproved` when the change is fundamentally unsafe, unworkable, or directionally wrong enough that patching individual issues is not the right next step.

---
name: code-review
description: >
  Review workspace changes, branches, or commits by dispatching the entire review
  to an isolated subagent (`code-reviewer`) so the main conversation context stays
  clean and unbiased.
---

# Code Review

Use this skill whenever the user asks to review code changes (workspace, branch comparison, commit, or "review and fix").

## Core Principle: Context Isolation

Do **not** inspect diffs, read changed files, or perform the review directly in the main conversation.
Hand off the entire review workflow to an isolated subagent so that:
1. Large diffs, rule checklists, and tool outputs do not bloat the main conversation.
2. The reviewer evaluates the actual changes objectively without confirmation bias from earlier edits in the main conversation.

## Workflow

1. **Extract intent from the user request**
   - **Target scope**:
     - Workspace changes (default when no range/commit is given)
     - Branch/range comparison (e.g., `from <ref>` / `to <ref>`)
     - Single commit (e.g., `HEAD` or a commit hash)
     - Any user-specified exclusions or custom rule paths
   - **Background context**:
     - If the user provided background context or a background file, pass it through.
     - Otherwise, if the main conversation already discussed the purpose of these changes, write a concise 1–3 sentence summary to pass to the reviewer.
   - **Mode**: report-only by default, or `"review and fix"` if the user explicitly asked to fix issues.

2. **Invoke the isolated reviewer subagent**
   - Call `invoke_subagent` with `TypeName: "code-reviewer"` (if `code-reviewer` is not available in the current runtime, fall back to `TypeName: "self"` and instruct it to follow `agents/code-reviewer.md`).
   - Provide `Role: "Isolated Code Reviewer"` and a clear `Prompt` containing:
     - The repository path
     - The review target scope (and any exclusions, custom rules, or background context)
     - Instruction to return a structured review report (coverage summary + severity-grouped findings with file paths and line numbers) without editing repository files.
   - End your turn and wait for the subagent to finish.

3. **Present the report & close the loop**
   - Present the subagent's coverage summary and findings with clickable `file:///...#L...` links.
   - If the user asked to `"review and fix"` (or once the user confirms), apply the safe Critical/High/Medium fixes in the main workspace and verify with tests.

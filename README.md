# Antigravity Code Review

[English](README.md) | [简体中文](README.zh-CN.md)

An automated code review plugin for **Google Antigravity** that reviews workspace changes, branch comparisons, and individual commits inside an isolated subagent—keeping the main conversation context clean and unbiased.

Under the hood, this plugin integrates Alibaba's open-source **[Open Code Review](https://github.com/alibaba/open-code-review)** (`ocr` CLI) via its `delegate` workflow (`ocr delegate preview` and `ocr delegate rule`) for file scope filtering and rule distribution, while letting the Antigravity subagent handle diff inspection, fact-checking, and report generation without requiring an external `ocr` LLM or API key.

## Key Features

- **Context Isolation**: The main conversation never reads raw diffs or rule checklists directly. Instead, the entire review is dispatched to an isolated `code-reviewer` subagent to prevent context bloat and confirmation bias.
- **Open Code Review Integration**: Uses `ocr delegate preview` and `ocr delegate rule` to determine reviewable files, apply exclusions, and deduplicate applicable review rules.
- **Diff Fact-Checking (Anti-Hallucination)**: The subagent fetches actual diffs and surrounding context via `git --no-pager`, focuses only on changed (`+`) lines or behavior directly introduced by the diff, and verifies every finding against the diff before reporting.
- **Flexible Review Scopes**: Supports workspace changes (staged, unstaged, and untracked files), branch/range comparisons, and single commits.
- **Report-Only by Default**: The `code-reviewer` subagent never modifies repository files. Fixes are only applied in the main workspace when you explicitly ask to `"review and fix"` or confirm after reviewing the report.

## Installation

### 1. Prerequisite: Install Open Code Review CLI

This plugin requires the `ocr` CLI from `@alibaba-group/open-code-review`. Install it globally:

```bash
npm install -g @alibaba-group/open-code-review
```

Verify that `ocr` is available on your `PATH`:

```bash
command -v ocr
```

> **Note**: This plugin only runs `ocr delegate preview` and `ocr delegate rule`—it never runs `ocr review`, so you do **not** need to configure an LLM or API key for `ocr`.

### 2. Install the Antigravity Plugin

Clone this repository and install the `antigravity-code-review` directory either globally or per workspace:

#### Option A: Global Installation (Recommended)

Symlink (recommended if you want updates via `git pull`) or copy `antigravity-code-review` into `~/.gemini/config/plugins/`:

```bash
# Symlink
ln -s "$(pwd)/antigravity-code-review" ~/.gemini/config/plugins/antigravity-code-review

# Or copy
cp -r ./antigravity-code-review ~/.gemini/config/plugins/antigravity-code-review
```

#### Option B: Workspace Installation

Copy `antigravity-code-review` into `.agents/plugins/` inside your target repository:

```bash
mkdir -p /path/to/your-project/.agents/plugins
cp -r ./antigravity-code-review /path/to/your-project/.agents/plugins/antigravity-code-review
```

Once installed, verify in Antigravity's **Customizations / Plugins** panel that **Antigravity Code Review** (`antigravity-code-review`) is enabled.

## Usage

Trigger the `code-review` skill in Antigravity with natural language prompts, such as:

- **Workspace changes (default)**
  - `Review my current workspace changes`
- **Branch or range comparison**
  - `Review my current branch against main`
- **Single commit**
  - `Review commit HEAD`
- **Review and fix**
  - `Review the latest commit and fix high-confidence issues`
- **Optional parameters**: You can also mention file exclusion patterns, custom rule paths, or background context for the changes, which will be forwarded to the `code-reviewer` subagent.

## How It Works

1. **Intent Extraction**: The main conversation parses your request for target scope, background context, and mode (report-only by default, or `"review and fix"`).
2. **Isolated Review**: The `code-reviewer` subagent runs:
   - `command -v ocr` to verify the CLI is installed;
   - `ocr delegate preview` to resolve the review scope and `reviewable_files`;
   - `ocr delegate rule` to load deduplicated rules for those files;
   - `git --no-pager` (and direct file reads for untracked files) to inspect diffs, check context, and verify findings.
3. **Structured Report**: Results are returned with clickable file/line links, covering:
   - **Scope & Coverage**: mode, refs / `merge_base`, `total_files`, `reviewed_files`, `skipped_files` (with reasons), and `coverage_rate`.
   - **Findings by Severity**: `Critical/High`, `Medium`, and high-value `Low` issues, each with file path, line range, category (`bug`, `security`, `performance`, `maintainability`, `test`, etc.), explanation, and concrete fix suggestion.

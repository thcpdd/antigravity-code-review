# Antigravity Code Review

[English](README.md) | [简体中文](README.zh-CN.md)

这是一个为 **Google Antigravity** 打造的自动代码审查插件（Plugin），通过隔离的子代理（Isolated Subagent）对工作区改动、分支对比或指定 Commit 进行客观、结构化的代码审查，避免主对话上下文膨胀与确认偏误。

本插件底层集成了阿里巴巴开源的 **[Open Code Review](https://github.com/alibaba/open-code-review)**（`ocr` CLI）的 `delegate` 能力，借助其文件范围过滤与规则分发机制，由 Antigravity 子代理直接完成 Diff 获取、事实核验与审查报告生成，无需额外配置 `ocr` 的 LLM / API Key。

## 核心特性

- **上下文隔离（Context Isolation）**：主对话不直接读取大量 Diff 或规则清单，而是将审查任务完整委派给独立的 `code-reviewer` 子代理执行，保持主对话上下文干净且审查视角客观。
- **基于 Open Code Review 的规则与范围控制**：通过 `ocr delegate preview` 与 `ocr delegate rule` 自动识别可审查文件、过滤无关文件，并按内容去重分发审查规则。
- **防幻觉事实核验**：子代理会基于 `git --no-pager` 获取真实 Diff 与上下文，只针对变更行（`+`）或 Diff 直接引入的行为提出问题，并在输出前逐条核验代码是否存在、是否与 Diff 矛盾。
- **多种审查范围**：支持当前工作区改动（staged + unstaged + untracked）、分支/范围对比以及单个 Commit 审查。
- **默认只读报告，按需修复**：`code-reviewer` 子代理不会修改仓库文件；仅在用户明确要求（如 `"review and fix"`）或确认后，才由主对话应用修复并验证测试。

## 安装方式

### 1. 前置依赖：安装 Open Code Review CLI

本插件依赖 `@alibaba-group/open-code-review` 提供的 `ocr` 命令，请先全局安装：

```bash
npm install -g @alibaba-group/open-code-review
```

安装完成后，确认 `ocr` 可用：

```bash
command -v ocr
```

> **说明**：本插件仅使用 `ocr delegate preview` 和 `ocr delegate rule`，不会运行 `ocr review`，因此**无需**为 `ocr` 配置额外的模型或 API Key。

### 2. 安装 Antigravity 插件

克隆本仓库后，可选择全局安装或工作区级安装：

#### 方式一：全局安装（推荐，所有项目可用）

将 `antigravity-code-review` 目录软链接（便于随仓库更新）或复制到 Antigravity 全局插件目录 `~/.gemini/config/plugins/`：

```bash
# 软链接方式
ln -s "$(pwd)/antigravity-code-review" ~/.gemini/config/plugins/antigravity-code-review

# 或直接复制
cp -r ./antigravity-code-review ~/.gemini/config/plugins/antigravity-code-review
```

#### 方式二：工作区级安装（仅当前项目可用）

将 `antigravity-code-review` 目录复制到目标项目的 `.agents/plugins/` 目录下：

```bash
mkdir -p /path/to/your-project/.agents/plugins
cp -r ./antigravity-code-review /path/to/your-project/.agents/plugins/antigravity-code-review
```

安装完成后，可在 Antigravity 的 **Customizations / Plugins** 页面确认 **Antigravity Code Review**（`antigravity-code-review`）已启用。

## 使用方法

在 Antigravity 对话中直接用自然语言提出代码审查需求即可触发 `code-review` Skill，例如：

- **审查当前工作区改动（默认）**
  - `Review my current workspace changes`
  - `帮我审查一下当前工作区的代码改动`
- **对比分支或范围**
  - `Review my current branch against main`
  - `对比 origin/main 到 HEAD 审查当前分支的改动`
- **审查指定 Commit**
  - `Review commit HEAD`
  - `审查最近一次提交的代码`
- **审查并修复高置信度问题**
  - `Review the latest commit and fix high-confidence issues`
  - `审查当前改动并修复发现的问题（review and fix）`
- **附加选项**：可在请求中指定要排除的文件模式、自定义规则路径或本次改动的背景说明，主对话会将其一并传递给 `code-reviewer` 子代理。

## 工作流程与输出格式

1. **意图解析**：主对话从你的请求中提取审查范围、背景上下文和模式（默认仅输出报告，或 `"review and fix"`）。
2. **隔离审查**：调度 `code-reviewer` 子代理依次执行：
   - 检查 `ocr` CLI 是否可用；
   - 运行 `ocr delegate preview` 确定可审查文件与基准范围；
   - 运行 `ocr delegate rule` 获取对应文件的审查规则；
   - 使用 `git --no-pager`（及直接读取未跟踪文件）获取 Diff 与上下文并完成事实核验。
3. **结构化报告**：返回给主对话并呈现给你，包含：
   - **Scope & Coverage**：审查模式、基准引用 / `merge_base`、文件总数、已审查数、跳过文件及原因、覆盖率；
   - **Findings by Severity**：按 `Critical/High`、`Medium`、`Low`（仅高价值项）分组的问题列表，每条均包含文件路径与行号范围、问题类别（`bug`、`security`、`performance`、`maintainability`、`test` 等）、原因说明与具体修复建议。

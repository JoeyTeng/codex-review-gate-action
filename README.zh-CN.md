# Codex Review Gate

语言：[British English (en-GB)](README.md) | [简体中文 (zh-CN)](README.zh-CN.md)

`codex-review-gate` 是一个可复用 GitHub Action，用 Codex review signal 为 PR 提供 deterministic `codex/review-gate` status check。

## 生成式 AI 提示

这个 Action 会请求并评估 Codex 生成式 AI review 输出。当它发布受控 `@codex review` marker comment 后，Codex 可能在 PR 中回复 AI 生成的 comments 或 reviews。请先人工核验这些 AI 生成内容，再把它用于安全性、正确性或 merge 决策。

这个 Action 自身不会执行 PR 代码。它只协调 GitHub comments、reviews、reactions 和 commit statuses，让维护者可以把 Codex review 作为 required branch-protection signal。

## Workflow 用法

```yaml
name: Codex Review Gate

on:
  pull_request_target:
    types: [opened, reopened, synchronize, ready_for_review]
  issue_comment:
    types: [created]
  pull_request_review:
    types: [submitted]
  pull_request_review_comment:
    types: [created]
  schedule:
    - cron: "0 */2 * * *"
  workflow_dispatch:
    inputs:
      pull_request:
        description: Optional pull request number to gate
        required: false
        type: string

permissions:
  contents: read
  issues: write
  pull-requests: write
  statuses: write

concurrency:
  group: codex-review-gate-${{ github.repository }}
  cancel-in-progress: false

jobs:
  codex-review-gate:
    name: codex/review-gate runner
    runs-on: ${{ fromJSON(vars.CODEX_REVIEW_GATE_RUNNER_LABELS || '["ubuntu-slim"]') }}
    timeout-minutes: 15
    steps:
      - uses: JoeyTeng/codex-review-gate-action@v1
        with:
          github-token: ${{ github.token }}
          pull-request: ${{ github.event.pull_request.number || github.event.issue.number || github.event.inputs.pull_request }}
          head-sha: ${{ github.event.pull_request.head.sha || '' }}
          event-mode: ${{ vars.CODEX_REVIEW_GATE_EVENT_MODE }}
          codex-bot-logins: ${{ vars.CODEX_REVIEW_GATE_BOT_LOGINS }}
          completion-signal-buffer-seconds: ${{ vars.CODEX_REVIEW_GATE_COMPLETION_SIGNAL_BUFFER_SECONDS }}
```

英文 README 包含完整 job-level event filter 示例。建议生产仓库使用完整示例，避免无关 bot、Dependabot 或 fork review events 分配 runner。

## 运行模型

- 通过 default branch 上的 `pull_request_target` 运行。
- 默认把 `codex/review-gate` status 写到 PR head SHA。
- 串行创建受控 `@codex review` marker comment。
- 每个 marker comment 都包含可见的生成式 AI 提示。
- 只有 active marker 之后出现 Codex clean completion comment 或 `APPROVED` review，且当前 head 没有 Codex findings，才会通过。
- Codex review output 可能不完整或不准确，不能替代 maintainer judgment。

## 高级运行模型

Event-driven review gate 的状态机、自动重试开关、**GHA 成本模型 (cost model)** 和手动恢复行为见 [DESIGN.zh-CN.md](DESIGN.zh-CN.md)。

## 反馈

Action bug、gate 行为问题、文档问题或 Marketplace listing 问题，请在 [GitHub issues](https://github.com/JoeyTeng/codex-review-gate-action/issues) 反馈。如果 PR 收到有问题的 AI 生成 review 内容，请优先使用 GitHub 对具体 comment 或 review 的正常反馈/举报入口。

## Source and Development

这个仓库是 Marketplace release package。开发、CI 和 self-gating workflows 维护在 [JoeyTeng/codex-review-gate](https://github.com/JoeyTeng/codex-review-gate)。

# Codex + Auxiliary Agents + GitHub Actions 自动化 Pipeline 配置指南

## 概述

本 Pipeline 的目标是以 Codex 为主，辅以 Minimax、Kimi、Claude Code 或本机 Hermes：

1. **Codex 主实现** - 写代码、修 bug、补测试。
2. **本地快速验证** - 运行相关测试、lint、typecheck。
3. **Codex / Hermes 二次审查** - 重点找 correctness、安全、测试缺口、回归风险和设计问题。
4. **GitHub Actions 完整验证** - PR 上运行 review、cleanup、CI、可选自动修复。
5. **人工最终确认** - 生产仓库不建议让 AI 自动 merge。

核心原则：

- Codex 是主 agent，负责实现和 review。
- Minimax、Kimi、Claude Code、本机 Hermes 是辅助 agent，可用于对照实现、失败修复或深度 review。
- 不要让同一个 AI 自己写、自己审、自己合并。
- PR 目标分支是 `dev`，`main`/`dev` 不直接推送。

## 文件结构

```
.codex/
└── config.toml               # Codex 本地审查默认配置
CLAUDE.md                     # Claude Code 可选辅助实现规则
AGENTS.md                     # Codex/agent 项目说明与审查规则
docs/development/code_review.md # 统一审查标准
.github/workflows/
├── codex-task.yml            # 新增：任务入口，创建分支/PR 并通知外部实现 runner
├── ci.yml                    # 已有：测试 + mypy + coverage
├── code-cleanup.yml          # 已有：pre-commit 自动修复
├── codex-review.yml          # 新增：Codex 审查 Pipeline
└── codex-auto-fix.yml        # 新增：自动修复失败
```

## 配置步骤

### 1. 启用 Workflow（已完成）

工作流文件已创建，推送到 GitHub 后自动生效。

### 2. 配置 Secrets（必须）

在 GitHub 仓库设置中添加以下 Secrets：

| Secret | 用途 | 必需 |
|--------|------|------|
| `HERMES_WEBHOOK_URL` | Codex/Hermes 深度 review 触发地址 | 可选 |
| `CODEX_TASK_WEBHOOK_URL` | 外部任务 runner 触发地址，可指向 Codex/Hermes/自建调度器 | 可选 |
| `ANTHROPIC_API_KEY` | Claude Code 辅助 runner | 可选 |
| `MINIMAX_API_KEY` | Minimax 辅助 runner | 可选 |
| `KIMI_API_KEY` | Kimi 辅助 runner | 可选 |
| `GITHUB_TOKEN` | 自动提交修复 | 自动提供 |
| `OPENAI_API_KEY` | Codex CLI / Codex review 使用 | 已有 |

#### 设置 HERMES_WEBHOOK_URL

如果你希望 Hermes 自动审查 PR，需要：

1. 在 Hermes 配置中启用 webhook 接收：
   ```bash
   hermes config set webhooks.codex_review.path /codex-review
   ```

2. 获取 webhook URL（如果使用 ngrok 等工具）：
   ```bash
   ngrok http 8080
   # 复制 https URL 到 GitHub Secrets
   ```

3. 或者使用 Hermes Cloud 提供的 webhook URL

### 3. 分支保护规则（推荐）

在 GitHub 仓库设置中配置：

```
Settings -> Branches -> Add rule
- Branch name pattern: main
- Require a pull request before merging: ✅
- Require status checks to pass: ✅
  - Search for checks: "security-scan", "quality-check"
- Require conversation resolution before merging: ✅
```

### 4. 推荐日常流程

#### 方式 A：GitHub Actions 发起任务 → Codex runner 写代码 → Codex/Hermes Review/CI

1. 打开 GitHub Actions → `codex-task` → `Run workflow`
2. 输入任务描述、base branch（默认 `dev`）
3. workflow 会自动创建 `codex/task-*` 分支和 PR
4. 如果配置了 `CODEX_TASK_WEBHOOK_URL`，workflow 会把任务、PR 号、分支名发送给外部 runner
5. 外部 runner 默认用 Codex 实现，也可调用 Minimax、Kimi 或 Hermes 辅助
6. `codex-review`、`code-cleanup`、`ci` 自动运行

Webhook payload 示例：

```json
{
  "event": "codex_task_request",
  "repository": "dimensionalOS/dimos",
  "base_branch": "dev",
  "branch": "codex/task-123456789-add-feature",
  "pr_number": 123,
  "pr_url": "https://github.com/dimensionalOS/dimos/pull/123",
  "task": "Implement obstacle avoidance in navigation module",
  "actor": "username"
}
```

#### 方式 B：本地 Codex 写代码 → 本地 Codex review → GitHub PR

```bash
# 1. 在 feature 分支上使用 Codex
git checkout -b feat/new-feature
codex

# 2. 本地快速验证
./bin/pytest-fast
uv run ruff check dimos tests scripts
uv run mypy dimos/

# 3. 用 Codex review 模式做二次审查
codex
# 在 Codex 中运行 /review，选择 against base branch dev

# 4. 推送并创建 PR
git push -u origin feat/new-feature
gh pr create --title "feat: obstacle avoidance" --body "..."

# 5. Pipeline 自动触发
# - 安全扫描
# - 代码质量检查
# - Codex/Hermes Review（如果配置了 webhook）
```

#### 方式 C：手动触发 Review

```bash
# 在已有 PR 上手动触发
cd /home/lenovo/working/dimos-auto-test

# 获取当前 PR 号
PR_NUMBER=$(gh pr view --json number -q .number)

# 手动触发 workflow
gh workflow run codex-review -f pr_number=$PR_NUMBER
```

### 5. PR Pipeline 执行流程

```
PR 创建/更新
    │
    ▼
┌─────────────────┐
│ 1. Security Scan │  ← 硬编码密钥、危险函数
│    (必过)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Quality Check │  ← ruff lint/format, mypy
│    (必过)        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. Trigger      │  ← 发送 webhook 到 Hermes
│    Hermes       │
└─────────────────┘
```

### 6. 结果查看

Pipeline 会在 PR 中自动添加评论：

- **Security Scan Results** - 安全扫描结果
- **Code Quality Check** - 代码质量报告
- **Hermes Review Status** - Hermes 审查状态

### 7. 自动修复

如果配置了 `codex-auto-fix.yml`，当 lint/format 失败时：

1. Pipeline 自动运行 `ruff check --fix`
2. 自动提交修复到同一分支
3. 在 PR 中评论告知已修复

## 本地测试

在推送前本地运行检查：

```bash
cd /home/lenovo/working/dimos-auto-test

# 安装依赖
uv sync --extra all --frozen

# 运行安全扫描（模拟）
git diff main...HEAD | grep "^+" | grep -iE "(api_key|secret|password|token)\s*[=:]\s*['\"][^'\"]{8,}['\"]"

# 运行 lint
source .venv/bin/activate
ruff check dimos/ tests/ scripts/

# 运行 format 检查
ruff format --check dimos/ tests/ scripts/

# 运行快速测试
./bin/pytest-fast
```

## 故障排除

### Pipeline 未触发

检查：
1. 文件是否推送到 GitHub：`git push`
2. PR 是否针对正确分支（main/dev）
3. 文件变更是否在监控路径内（dimos/, tests/, scripts/）

### Security Scan 误报

如果安全扫描误报了合法代码，可以在 PR 评论中回复：
```
@github-actions bot ignore-secret-scan
```

### Hermes Webhook 未收到

检查：
1. Secret 是否正确设置
2. Webhook URL 是否可访问
3. Hermes 是否正在运行并监听 webhook

## 进阶配置

### 添加自定义检查

编辑 `.github/workflows/codex-review.yml`，在 `quality-check` job 中添加步骤：

```yaml
- name: Custom Check
  run: |
    # 你的自定义检查命令
    python scripts/custom_check.py
```

### 修改触发条件

编辑 workflow 文件的 `on:` 部分：

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'dimos/**'      # 监控这些路径
      - 'your-path/**'  # 添加新路径
```

## 参考

- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Codex CLI 文档](https://github.com/openai/codex)
- [ruff 文档](https://docs.astral.sh/ruff/)

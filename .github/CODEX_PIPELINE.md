# Codex + GitHub Actions 自动化 Pipeline 配置指南

## 概述

本 Pipeline 实现三层自动化审查：
1. **安全扫描** - 检测硬编码密钥和危险函数
2. **代码质量** - Lint、Format、Type Check
3. **Hermes Review** - AI 深度审查逻辑和架构

## 文件结构

```
.github/workflows/
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
| `HERMES_WEBHOOK_URL` | Hermes Review 触发地址 | 可选 |
| `GITHUB_TOKEN` | 自动提交修复 | 自动提供 |
| `OPENAI_API_KEY` | Codex CLI 使用 | 已有 |

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

### 4. Codex CLI 使用方式

#### 方式 A：Codex 写代码 → 自动触发 Review

```bash
# 1. 在 feature 分支上使用 Codex
git checkout -b feat/new-feature
codex exec --full-auto "Implement obstacle avoidance in navigation module"

# 2. 推送并创建 PR
git push -u origin feat/new-feature
gh pr create --title "feat: obstacle avoidance" --body "..."

# 3. Pipeline 自动触发
# - 安全扫描
# - 代码质量检查
# - Hermes Review（如果配置了 webhook）
```

#### 方式 B：手动触发 Review

```bash
# 在已有 PR 上手动触发
cd /home/lenovo/working/dimos-auto-test

# 获取当前 PR 号
PR_NUMBER=$(gh pr view --json number -q .number)

# 手动触发 workflow
gh workflow run codex-review -f pr_number=$PR_NUMBER
```

### 5. Pipeline 执行流程

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

# 运行测试
pytest --tb=no -q
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

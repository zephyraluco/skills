---
name: git-commit
description: >
  执行 git commit，带 Conventional Commit 消息分析、智能暂存与消息生成。当用户要求提交变更、创建 git commit，或提及 "/commit" 时使用。支持：
  (1) 从变更中自动检测 type 和 scope
  (2) 从 diff 生成 conventional commit 消息
  (3) 可交互式提交，并可选覆盖 type/scope/description
  (4) 智能文件暂存以形成逻辑分组
license: MIT
metadata:
  author: zeal
  version: "0.0.1"
allowed-tools: Bash Zsh Git
---

# 使用 Conventional Commits 进行 Git 提交

## 概述

使用 Conventional Commits 规范创建标准化、语义化的 git 提交。分析实际 diff 以确定合适的 type、scope 和消息。

> ❗ **Commit message 统一使用英文**。type、scope、description、body、footer 均用英文撰写，即使代码注释、文档或对话使用中文。这保证提交历史在各工具链、CI 与国际协作环境中保持一致。

## Conventional Commit 格式

```
<type>[optional scope]: <description>

- [optional body 1]
- [optional body 2]

[optional footer(s)]
```

## Commit 类型

| 类型       | 用途                        |
| ---------- | ------------------------------ |
| `feat`     | 新功能                    |
| `fix`      | Bug 修复                        |
| `docs`     | 仅文档             |
| `style`    | 格式/风格（无逻辑变更）    |
| `refactor` | 代码重构（无新功能/修复） |
| `perf`     | 性能改进 |
| `test`     | 添加/更新测试               |
| `build`    | 构建系统/依赖      |
| `ci`       | CI/配置变更              |
| `chore`    | 维护/杂项               |
| `revert`   | 回滚提交                  |

## 破坏性变更

```
# Exclamation mark after type/scope
feat!: remove deprecated endpoint

# BREAKING CHANGE footer
feat: allow config to extend other configs

BREAKING CHANGE: `extends` key behavior changed
```

## 工作流程

### 1. 分析 Diff

```bash
# If files are staged, use staged diff
git diff --staged

# If nothing staged, use working tree diff
git diff

# Also check status
git status --porcelain
```

### 2. 暂存文件（如需要）

如果没有任何内容被暂存，或者你想以不同方式对变更分组：

```bash
# Stage specific files
git add path/to/file1 path/to/file2

# Stage by pattern
git add *.test.*
git add src/components/*

# Interactive staging
git add -p
```

**绝不提交密钥**（.env、credentials.json、私钥）。

### 3. 生成 Commit 消息

分析 diff 以确定：

- **类型（Type）**：这是什么类型的变更？
- **范围（Scope）**：影响哪个区域/模块？
- **描述（Description）**：一行摘要说明改了什么（现在时、祈使语气、<72 个字符）

> 以上内容均以**英文**书写，不要因为项目文档是中文就改成中文提交消息。

### 4. 执行提交

```bash
# Single line
git commit -m "<type>[scope]: <description>"

# Multi-line with body/footer
git commit -m "$(cat <<'EOF'
<type>[scope]: <description>

- <optional body 1>
- <optional body 2>

<optional footer>
EOF
)"
```

## 最佳实践

- **Commit message 使用英文**：type/scope/description/body/footer 均为英文
- 每次提交只包含一个逻辑变更
- 现在时：用 "add" 而不是 "added"
- 祈使语气：用 "fix bug" 而不是 "fixes bug"
- 引用 issue：`Closes #123`、`Refs #456`
- 描述保持在 72 个字符以内

## Git 安全守则

- 绝不要修改 git config
- 未经明确要求，绝不运行破坏性命令（--force、hard reset）
- 除非用户要求，绝不跳过钩子（--no-verify）
- 绝不强制推送到 main/master
- 如果提交因钩子而失败，请修复并创建一个**新的**提交（不要 amend）

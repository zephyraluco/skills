---
name: github-pr
description: >
  当用户要求创建分支、提交当前变更、打开 PR 或草稿 PR、发布 Pull Request，或需要从 gh pr create / PR 模板问题中恢复时用；涵盖.github/pull_request_template.md、--body 格式、<agent-name>/ 分支命名以及 Codex 应用的 git 指令
metadata:
  internal: true
---

# 创建 PR

当需要把本地工作转化为 GitHub Pull Request 时使用本技能

## Fork PR 与分支 PR 的区别

在撰写 PR 描述之前，先确认这是分支 PR（来自同一仓库的分支）
还是 fork PR（来自 fork 的外部贡献）。你可以为分支 PR 撰写完整描述，
但**不可以**为 fork PR 撰写 —— 此时应告知用户，主动提出帮助评审他们的描述
或提供技术细节，并把 GitHub 链接交给他们自行创建 PR

## 工作流

1. 修改 Git 状态之前，先检查当前状态：

   ```bash
   git status --short
   git branch --show-current
   git diff -- <paths>
   ```

   只暂存属于本次请求变更的文件，保留用户无关的改动

2. 创建或确认分支：

   ```bash
   git switch -c <commit-type>/<short-topic>
   ```

   分支信息按 Conventional Commit 约定取 `<commit-type>/<short-topic>`：

   - `<commit-type>` 取自 `git-commit` 技能的类型表（见下表）
   - `<short-topic>` 用 kebab-case 简短描述本次变更主题，例如 `feat/durable-agent-runtime`、`fix/reflow-selection`

   若当前分支已存在且主题一致，直接复用，不要为了改名而新建分支

3. 校验并提交：

   ```bash
   git add <paths>
   git diff --cached --check
   git commit -m "<concise message>"
   ```

   提交信息按 `git-commit` 技能的规范生成：`<type>[optional scope]: <description>`，type 与分支前缀保持一致，scope 取变更所在的模块
   保持简洁，不要添加生成工具或共同作者的尾注（footer）

4. 推送分支：

   ```bash
   git push -u origin <branch>
   ```

5. 除非用户明确要求，否则以草稿形式创建 PR：

   ```bash
   gh pr create --draft --base canary --head <branch> --title "<title>" --body '<body>'
   ```

   本仓库请优先使用 `canary` 作为基础分支。若沙箱内访问 GitHub 失败，
   请提权重试

## PR 描述

使用以下 PR 描述格式：

````markdown
## Summary

<One paragraph: what changed and why, described in terms of the effect visible to end users or developers.>

## Changes

### <Area, e.g. Sessions and automation>

- <A specific change>
- <A specific change>

### <Area, e.g. Tool execution and file changes>

- <A specific change>

## Verification

- `<Full command that actually ran>` — <result, e.g. 952 passed>
- Not run: `<command>` (`<reason>`)

<One paragraph: scope and boundaries — what is included, what is not, known limitations, and who owns the merge decision.>

<!-- NEXT_JS_LLM -->
````

填写好内容后通过 `--body` 传入

## 异常恢复

- 若 PR 可能已存在，先检查再避免重复创建：

  ```bash
  gh pr view --head <branch> --json url,isDraft,title
  ```

- 若 `gh pr create` 未获批准，应说明分支已推送但 PR 未创建，并给出准确的修正命令
- Codex 应用的 Git 操作成功后，请附上对应的最终回复指令，用于分支创建、暂存、
  提交、推送和 PR 创建

## 审查流程

PR 创建后进入「审查 → 修复 → 回复」循环，直到审查方不再提出问题

### 1. 拉取审查意见

先定位本轮审查针对的**提交号**，再逐条读取意见：

```bash
gh pr view <number> --json reviews,comments \
  --jq '.reviews[] | {author: .author.login, state: .state, body: .body}'

gh api repos/{owner}/{repo}/pulls/<number>/comments \
  --jq '.[] | {path, line, commit: .original_commit_id, author: .user.login, body}'
```

要点：

- 只处理**本轮新增**的意见，历史已解决（`Resolved`）的条目跳过
- 记录每条意见的 `path` + `line` + `commit`，后续回帖时用作定位依据
- `gh pr view` 的评论正文可能截断，按需用 `gh api` 取全文

### 2. 判断意见是否成立

对每条意见给出明确结论，不要笼统"已修复"：

| 结论 | 处理方式 |
|------|----------|
| 确实有问题 | 改代码 |
| 是误报 | 写清为什么代码是对的，可附复现或测试证明 |
| 是设计取舍 | 说明取舍理由，必要时同步更新 PR 描述的范围段落 |
| 超出本 PR 范围 | 指出独立跟进项，不强行塞进本 PR |

不成立的意见也要回复，避免审查方重复提出

### 3. 修改并本地验证

```bash
git status --short
git diff -- <paths>
cargo test --workspace --all-features --locked
cargo +nightly fmt --all -- --check
cargo clippy --all-targets --all-features --locked -- -D warnings
git diff --cached --check
```

- 一条意见一个提交，提交信息写清修的是哪个问题，按 `git-commit` 技能生成（如 `fix(policy): honor explicit ask rules for read-only calls`）
- 修复引入的新行为必须有对应测试；修 bug 要先有能失败的测试
- 跨平台相关改动（路径、换行、socket、信号、文件锁）在本地至少验证一个平台，其余交给 CI

### 4. 推送并回复

```bash
git push
gh pr comment <number> --body '<回复正文>'
```

回复按意见逐条对应，每条包含：**问题** → **结论** → **对应提交**（或"误报"理由）
结尾附上本轮验证结果：

```markdown
### Review fix at `<short-sha>`

- `cargo test --workspace --all-features --locked` — <n> passed
- `cargo clippy ... -D warnings`、`cargo fmt --check`、`git diff --check` 均通过
- 跨平台 CI 状态：<运行中 / 全部通过>
```

### 5. 请求下一轮审查

```bash
gh pr comment <number> --body '@codex review'
```

等 CI 与审查都返回后回到第 1 步。**同时更新 PR 描述**中的 `## Verification` 与结尾范围段落，保持与最新提交一致

### 6. 收敛判定

以下条件同时满足才停止循环：

- [ ] 本轮所有意见均已回复且有结论
- [ ] 无新增未处理意见，最新审查显示 `Resolved` 或审查方明确认可
- [ ] 全部 CI 检查通过（含跨平台矩阵与最低支持版本编译检查）
- [ ] PR 描述与实际变更一致，验证命令可复现
- [ ] 无遗留的临时调试代码、被跳过的测试、被静默的 lint 豁免

若审查额度耗尽或审查方长时间无响应，在 PR 中记录当前状态与待确认事项，交由仓库所有者决定是否合并

## 相关技能

- `git-commit` — 本技能的分支前缀与提交信息均沿用其 Conventional Commit 类型表与消息格式；需要交互式暂存、拆分逻辑分组或从 diff 生成消息时，转用该技能
- `github-cli` — PR 创建后的 CI 排查、issue 分流、发布与安全告警监控


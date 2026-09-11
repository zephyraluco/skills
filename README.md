# 技能仓库（Skills Repository）

## 安装

从本仓库安装某个技能：

**Bun**

```bash
bunx skills add zephyraluco/skills --skill skill-name
```

**npm**

```bash
npx skills add zephyraluco/skills --skill skill-name
```

**pnpm**

```bash
pnpm dlx skills add zephyraluco/skills --skill skill-name
```

例如安装 C++ 编码规范：

```bash
bunx skills add zephyraluco/skills --skill cpp-coding-standards
```

也可以先克隆仓库后本地安装：

```bash
git clone https://github.com/zephyraluco/skills.git
```

## 卸载

使用 `skills remove`（别名 `skills rm`）卸载已安装的技能。不传技能名时会进入交互式选择：

```bash
bunx skills remove                    # 交互式选择要卸载的技能
bunx skills remove cpp-coding-standards   # 按名称卸载
```

指定作用域或 agent：

```bash
bunx skills rm --global cpp-coding-standards   # 从全局（用户级）卸载
bunx skills rm --agent claude-code             # 仅清理指定 agent 的链接
bunx skills rm --all                           # 卸载全部（隐含 -y）
```

也可先查看已安装的技能，再决定卸载哪些：

```bash
bunx skills list          # 列出项目级已安装技能
bunx skills ls -g         # 列出全局技能
bunx skills ls -a claude-code   # 按 agent 过滤
```

**npm / pnpm 等价写法**：将上面的 `bunx` 换成 `npx`（npm）或 `pnpm dlx`（pnpm）即可

常用选项：

| 选项 | 说明 |
| ---- | ---- |
| `-g, --global` | 从全局（用户级）作用域卸载 |
| `-a, --agent <agents>` | 仅从指定 agent 卸载（省略则清理所有 agent 链接） |
| `-s, --skill <skills>` | 指定要卸载的技能名（`*` 表示全部） |
| `-y, --yes` | 跳过确认提示 |
| `--all` | 卸载所有已安装技能（隐含 `-y`，不可与指定技能名组合） |

## 可用技能

| 技能 | 说明 | 语言/生态 |
| ---- | ---- | --------- |
| `cpp-coding-standards` | 基于 [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) 的 C++ 编码规范。在编写、审查或重构 C++ 代码时使用，推行类型安全、资源安全、不可变性与清晰性。 | C++17/20/23 |
| `git-commit` | 执行 git commit，带 Conventional Commit 消息分析、智能暂存与消息生成。当用户要求提交变更、创建 git commit，或提及 `/commit` 时使用。支持自动检测 type/scope、从 diff 生成消息、交互式提交、智能文件暂存分组。 | Git |
| `python-patterns` | Pythonic 惯用法、PEP 8 标准、类型提示，以及构建健壮、高效、可维护 Python 应用的最佳实践。 | Python 3.9+ |
| `ros2-development` | ROS2 开发的全面最佳实践、设计模式与常见陷阱。覆盖节点与组件设计、launch 文件、QoS/DDS 配置、生命周期节点、action、colcon/ament 构建系统、调试工具与生产部署。 | ROS2（Humble / Iron / Jazzy / Rolling） |
| `rust-coding-standards` | Rust 最佳实践指南。含 9 章参考文档：编码风格与惯用法、Clippy 与 Lint、性能思维、错误处理、自动化测试、泛型与分发、类型状态模式、注释与文档、理解指针。 | Rust 1.88+ |
| `web-coding-standards` | 面向 TypeScript、JavaScript、React 与 Node.js 开发的通用编码规范、最佳实践与模式。 | TS/JS/React/Node.js |

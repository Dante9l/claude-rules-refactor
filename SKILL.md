---
name: claude-rules-refactor
description: "Use when the user wants to restructure, audit, expand, or initialize their CLAUDE.md and rules files. Triggers on: 优化claude.md、重构claude.md、更新全局配置、整理规则文件、从会话数据提炼规则、把历史对话写进配置、规则健康检查、规则冲突、跨项目同步规则、给项目建claude.md、初始化项目配置、这个项目没有claude.md. Modes: (A) restructure CLAUDE.md into modular rule files; (B) mine session history for new rules; (C) audit existing rules for quality issues and conflicts; (D) bootstrap a new project-level CLAUDE.md from scratch. Use whenever the user mentions improving, auditing, learning from past sessions, or setting up Claude config for a project."
---

# Claude Rules Refactor Skill

## 概述

三种模式，按用户意图自动选择，也可组合使用：

- **模式 A — 重构**：把现有 CLAUDE.md 按规范拆成模块化规则文件
- **模式 B — 从会话挖掘**：扫描历史 session，提炼值得写进配置的规则
- **模式 C — 健康检查**：审计现有规则文件，找出质量问题、冲突和跨项目重复

推荐组合：先 C 了解现状 → B 挖掘新规则 → A 统一重构。

---

## 规范原则（两种模式都遵守）

详见 [references/config-spec.md](references/config-spec.md)，核心要点：

- 全局偏好放 `~/.claude/CLAUDE.md`，项目规则放项目根目录 `.claude/rules/`
- 每个规则文件一个主题，不超过 50 行
- 写具体可验证的指令，不写模糊描述
- `CLAUDE.md` 只做引用入口，用 `@path/to/rule.md` 引用规则文件

---

## 模式 A：重构现有 CLAUDE.md

### 触发信号
用户说"重构"、"整理"、"优化 claude.md"，或提供了一个臃肿的配置文件。

### 步骤

**1. 读取目标文件**

读取用户指定的 CLAUDE.md（默认 `~/.claude/CLAUDE.md`）和已有规则文件。

**2. 分析现有内容**

识别以下问题：
- 模糊指令（"代码要简洁" → 需改为"函数不超过 30 行"）
- 混在一起的主题（git + 测试 + 工程规范 → 应拆开）
- 重复或矛盾的规则
- 应该在项目级而非全局级的规则

**3. 规划拆分方案**

输出拆分计划，格式：

```
拟创建/修改的文件：
- ~/.claude/CLAUDE.md          → 保留全局偏好 + @引用列表
- ~/.claude/rules/xxx.md       → [主题]（约 N 行）
- ~/.claude/rules/yyy.md       → [主题]（约 N 行）

待删除/合并：
- [文件名]：原因

模糊指令需具体化：
- "代码要简洁" → "函数不超过 30 行；不添加注释到未修改的代码行"
```

等用户确认后再执行。

**4. 执行重构**

- 逐文件写入，每次写完说明改了什么
- 保留用户原有的有效规则，不删除有意义的内容
- CLAUDE.md 末尾用 `@` 引用所有规则文件

**5. 验证**

读取所有新文件，确认：无重复规则、每文件 ≤ 50 行、CLAUDE.md 引用完整。

---

## 模式 B：从会话数据挖掘规则

### 触发信号
用户说"从以前的对话"、"根据历史 session"、"把出错记录写进配置"。

### 步骤

**1. 扫描 session 文件**

用 Python 脚本扫描 `~/.claude/projects/**/*.jsonl`。

**Windows 编码注意事项**（必须遵守，否则脚本会崩溃）：
- 用 `python3 -c "..."` 不用 heredoc
- 脚本开头加：`import sys; sys.stdout.reconfigure(encoding='utf-8', errors='replace')`
- 读文件用：`open(f, encoding='utf-8', errors='ignore')`
- 不在脚本中使用 emoji 字符

**2. 提取信号**

从 session 中提取：

| 信号类型 | 提取方式 |
|---------|---------|
| 用户纠正 | 用户消息含"不要"、"别"、"你又"、"停止"、"为什么" |
| 工具报错 | assistant 消息含 "Error"、"Traceback"、"exit code" |
| 反复提醒 | 同类用户消息在多个 session 中出现 |
| 正面确认 | 用户消息含"对"、"好"、"就这样"、"完美" |

**3. 归纳候选规则**

将信号归纳为候选规则，格式：

```
候选规则（来自 N 个 session）：
1. [规则描述]
   证据："[用户原话片段]"（出现 N 次）
   建议写入：~/.claude/rules/[文件名].md

2. ...
```

**4. 用户确认**

展示候选规则，询问：
- 哪些要加入配置
- 归属哪个规则文件（或新建文件）
- 是否需要调整措辞

**5. 写入配置**

按用户确认的内容写入对应规则文件，遵守规范（具体可验证、≤ 50 行/文件）。

---

## 通用规则

- 执行前先读取目标文件，不凭记忆操作
- 修改 `~/.claude/CLAUDE.md` 属于 Level 2 操作，需说明影响范围后执行
- 每次写入前输出将要写入的内容，等用户确认（除非用户说"直接执行"）
- 不创建空文件，不写无实质内容的占位规则

---

## 模式 C：规则健康检查

### 触发信号
用户说"检查规则"、"规则有没有问题"、"健康检查"，或在重构前想先了解现状。

### 步骤

**1. 收集所有规则文件**

扫描以下位置：
- `~/.claude/CLAUDE.md` 和 `~/.claude/rules/`
- 当前项目 `.claude/rules/`（如果在项目目录中）
- 用户指定的其他路径

**2. 逐项检查，输出报告**

| 检查项 | 判断标准 |
|--------|---------|
| 模糊指令 | 没有可量化/可验证标准的规则（如"代码要好"） |
| 文件超长 | 单文件超过 50 行 |
| 主题混杂 | 一个文件包含 2 个以上不相关主题 |
| 重复规则 | 同一规则在多个文件中出现 |
| 全局/项目错位 | 项目特定规则放在全局，或全局规则重复写在项目里 |
| 孤立引用 | CLAUDE.md 引用了不存在的规则文件 |
| 跨项目重复 | 多个项目的规则文件有相同内容，应提升为全局规则 |

**3. 输出报告格式**

```
规则健康检查报告
================
文件数：N，总行数：N

[问题] 模糊指令（M 处）
  - ~/.claude/rules/engineering.md:5  "代码要简洁" → 建议改为"函数不超过 30 行"

[问题] 文件超长（M 处）
  - ~/.claude/rules/xxx.md  68 行 → 建议拆分为 xxx-a.md / xxx-b.md

[问题] 规则冲突（M 处）
  - 全局 rules/engineering.md: "用中文回复"
  - 项目 .claude/rules/api.md: "reply in English"
  → 建议：项目规则优先级更高，确认是否符合预期

[问题] 跨项目重复（M 处）
  - project-a/.claude/rules/git.md:3 与 project-b/.claude/rules/git.md:3 内容相同
  → 建议提升到 ~/.claude/rules/git.md

健康评分：X/10
```

**4. 询问处理方式**

报告完成后询问：是否要直接修复这些问题，还是先看完再决定？

---

## 模式 D：建立项目级 CLAUDE.md

### 触发信号
用户说"给这个项目建 claude.md"、"初始化项目配置"、"这个项目没有 claude.md"，或当前项目目录下不存在 `.claude/` 目录。

### 步骤

**1. 探索项目**

并行读取以下信息（只读，不修改）：
- 项目根目录文件列表（识别技术栈：package.json / pom.xml / go.mod / requirements.txt 等）
- 现有 README（如果有）
- git log --oneline -20（了解提交风格和主要贡献者）
- 现有 `.claude/` 目录（如果已存在，避免覆盖）

**2. 推断项目特征**

根据探索结果识别：
- 语言和框架（Go / Python / Java / React 等）
- 测试框架（jest / pytest / testify 等）
- 构建工具（make / gradle / npm 等）
- 代码风格线索（从现有代码推断，不猜测）

**3. 输出建议配置**

展示拟写入的内容，格式：

```
拟创建：
- {project}/.claude/CLAUDE.md       → 项目入口
- {project}/.claude/rules/stack.md  → 技术栈规范（语言版本、框架约定）
- {project}/.claude/rules/workflow.md → 工作流规范（构建、测试、提交）

内容预览：
[CLAUDE.md]
...
[rules/stack.md]
...
```

等用户确认后再写入。

**4. 写入文件**

- 只写项目特定的规则，不重复全局 `~/.claude/rules/` 里已有的内容
- 规则必须具体可验证（参考 references/config-spec.md）
- 每个规则文件 ≤ 50 行
- CLAUDE.md 只做引用入口，不堆砌内容

**5. 提示后续**

写入完成后提示：
- 可以用模式 B 扫描该项目的历史 session，补充项目特定的 feedback
- 可以用模式 C 对新建的配置做健康检查

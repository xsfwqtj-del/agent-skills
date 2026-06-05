---
name: mio-sbagent-self-contained
description: Mio-SBAgent-Self-Contained — AI agent 可读的代码模块规范。写新代码、新建模块、组织文件结构时自动触发。每一个 .py/.ts 文件 Ctrl+A 全选丢进 AI 窗口，读完头部就能干活——不翻别的文件，不需要人解释项目背景。
trigger: auto
---

# Mio-SBAgent-Self-Contained

> 版本: v2.0.0 | 日期: 2026-06-05 | 分支 A：术语体系 + 核心原则重写
> 替换: ai-native-dev v1.1.0

---

## 北极星

**为谁？** 非程序员用户 + AI agent 助手。

**解决什么？** 人和 AI 都无法单独处理大型代码库——人的工作记忆上限 ~4 chunks，AI 的上下文窗口有限，几十个文件来回跳就退化。

**Mio-SBAgent-Self-Contained 是什么？** 一个代码质量属性。跟"可测试""可维护""低耦合"并列——区别是那些为人类程序员定，这条为 AI agent 上下文窗口定。

**判断标准：** 任何一个 `.py` / `.ts` 文件，Ctrl+A 全选 → 丢进 AI 窗口 → AI 读完头部就能干活。不翻别的文件，不需要人解释项目背景。

---

## 术语表

以下五个术语是核心概念。每个有英文定义和中文解释。

### Mio-SBAgent-Self-Contained Module（自包含模块）

> A module whose file header contains everything an AI agent needs to understand its role, contract, and navigation — without reading any other file or knowing project context.

一个模块 = 一个文件。打开文件、读头部（~20 行）、知道合约（消费/产出/依赖）、按标签跳到目标段——全程不翻别的文件。**自包含不等于孤立**：模块可以依赖外部接口，但接口签名写在头部里，不需要翻实现文件。

### Cognitive Load Budget（认知容量预算）

> The hard upper bound on how much context an AI agent can hold at once. Every file structure decision must account for this limit.

认知容量是硬上限。人 ~4 chunks，AI 上下文窗口同理。每多跳一个文件、每多一个未声明的依赖，都在消耗预算。**预算不是行数上限——是有内部导航的文件多大都行，没有导航的文件再小也浪费。**

### Module Contract（模块合约）

> A structured declaration at the top of every file specifying: what this module is, what it consumes, what it produces, what it depends on but doesn't control.

每个文件头部的结构化声明：我是谁 → 我消费什么 → 我产出什么 → 我依赖但不控制什么。合约是 AI agent 的"握手协议"——读完合约就知道能不能改、改哪里、改完会影响谁。**合约先于代码。**

### In-File Navigation Map（文件内导航地图）

> A tag-indexed table of contents within the file header, allowing AI agents to jump directly to a labeled section instead of reading the entire file.

文件头部的小地图，用 `[A]` `[B]` `[C]` 标签 + 行号索引。AI agent 读小地图 → 定位目标标签 → 跳到对应段落 → 只读该段。**有内部导航 = 文件多大都行。没有导航 = AI 被迫全文读完。**

### Frozen Cognitive Unit（冻结认知单元）

> A module whose contract and interface have stabilized — no functional changes for 30+ days, depended on by ≥2 other modules. Once frozen, AI agents only need to remember the contract, not the internals.

稳定后的模块。三个验收条件：① 30 天内无功能修改 ② ≥2 个模块依赖它 ③ 合约字段无变化。冻结后 AI agent 只需记住合约，释放认知容量给还在演化的模块。

---

## v1.1.0 的根因修正

v1.1.0 九项缺陷，按三分类根因修正：

### 结构缺失 → 补字段

| v1.1 | v2.0 |
|------|------|
| 头部缺"我依赖什么" | 新增 **依赖声明**——"我依赖但不控制" |
| "我不需要知道"方向反了 | 改为 **接口合约**——声明依赖的接口签名，不泄漏实现路径 |
| 缺 token 预算 | 每个段标注 **token 估算**，AI 判断是否值得读 |
| 缺内部导航 | 新增 **文件内导航地图**——`[A]` `[B]` `[C]` 标签 + 行号 |

### 约束模糊 → 给标准

| v1.1 | v2.0 |
|------|------|
| ≤300 行一刀切 | **有内部导航 = 文件多大都行** |
| 冻结构件无验收标准 | 三个可度量条件：30 天 / ≥2 依赖方 / 合约不变 |
| "按数据流排列"无方法 | 具体操作：沿着合约的消费/产出字段追踪上下游 |

### 可执行性差 → 可判定

| v1.1 | v2.0 |
|------|------|
| 检查清单面向人类（"陌生人能理解吗？"） | 每项 YES/NO，AI 可逐项判定 |
| 缺非程序员指南 + 语言锁定 | 语言无关，示范覆盖 Python + TypeScript |

---

## 三层导航

AI agent 进项目后不需要记住全貌，走三层递进：

```
MAP.md（找模块在哪）
  → 文件头部合约（读 20 行，知道我是谁/消费/产出/依赖）
    → 内部标签（跳到目标段，只读需要的部分）
```

全程不超两次上下文窗口加载。

---

## 检查清单（AI 可执行）

每项判定 YES/NO。不需要人类判断。

| # | 检查项 | 判定 |
|---|--------|------|
| 1 | 文件头部含「我是谁」——一句话说清角色，不引用项目名 | YES / NO |
| 2 | 文件头部含「我消费什么」——列出输入类型 + 示例值 | YES / NO |
| 3 | 文件头部含「我产出什么」——列出输出类型 + 示例值 | YES / NO |
| 4 | 文件头部含「我依赖但不控制」——列出外部接口签名（不写实现路径） | YES / NO |
| 5 | 文件头部有导航地图——`[A]` `[B]` `[C]` 标签 + 行号索引 | YES / NO |
| 6 | 每个段标注 token 估算 | YES / NO |
| 7 | 公共接口 ≤ 5 个函数 | YES / NO |
| 8 | 函数参数 ≤ 3 | YES / NO |
| 9 | 嵌套条件 ≤ 2 层 | YES / NO |
| 10 | 合约字段无悬空引用（所有引用的规则/流程均有定义） | YES / NO |

---

## 非程序员验收（人类三条）

不需要懂代码。打开任意一个模块文件：

1. **打开文件头 20 行能懂吗？** → 知道这文件干什么的
2. **能找到标签吗？** → 看到 `[A]` `[B]` `[C]` 标记
3. **要翻别的文件才能懂吗？** → 如果必须翻 → 自包含不合格

---

## 后续分支

本文件仅完成分支 A（术语体系 + 核心原则重写）。后续：

- **分支 B**：模块头部模板重设计（完整字段 + 示例）
- **分支 C1**：内部导航规范（标签语法 + 跳段规则）
- **分支 C2**：系统地图 MAP.md 规范
- **分支 D**：冻结构件验收标准
- **分支 E**：横切关注点处理
- **分支 F**：非程序员使用指南 + AI 检查清单
- **分支 G**：Python + TypeScript 双示范模块

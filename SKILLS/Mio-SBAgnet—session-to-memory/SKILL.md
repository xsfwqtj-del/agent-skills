---
name: Mio-SBAgnet—session-to-memory
description: Use when the user wants to archive sessions, extract knowledge from conversation history, convert chat logs to structured notes, or says "存档" "归档" "转化" "提取记忆" "整理会话". Also use when the user mentions wanting to preserve what was discussed or recover insights from past conversations. Processes local CC/Codex/Hermes/WorkBuddy session JSONL files.
---

# Mio-SBAgent — Session → Memory 转化引擎

> 版本: v1.0.1 | 更新: CSO 修复——description 移除 pipeline 总结，仅保留触发条件

将 CC 会话 JSONL 转化为结构化记忆文件。外部 agent 打开本文件即开始执行。

## 前置条件

确认以下路径存在，不存在则询问用户：

- **JSONL 目录**: `~/.claude/projects/` 或用户指定的会话文件目录（CC 默认存放位置。若用户未配置 Junction，询问路径）
- **记忆目录**: 当前项目根目录下的 `Mio-SBAgnet—memory/`（首次运行自动创建）
- **manifest**: `{记忆目录}/.manifest.json`（首次运行自动创建）

## 分层分工

管线分两层——脚本做机械操作，LLM 做需要判断的步骤：

| 层 | 步骤 | 角色 |
|----|------|------|
| 脚本 | ❶ 预检、❷ 源适配+去噪、❻ 写入+manifest+aggregate | 不做判断，规则明确 |
| LLM | ❸ 识主题、❹ 聚拢+提取+分类+打标、❺ 去重融合 | 需理解对话内容 |

## 执行管线

严格按 6 步顺序执行。

### ❶ 预检（脚本）

对每个待处理会话：

1. **文件检**: 大小、行数、时间范围。是否 JSONL 格式（逐行可解析）、是否被截断/中断
2. **源识别**: 识别 agent 类型（CC/Codex/Hermes/WorkBuddy...），确定字段映射表
3. **会话概要**: 预读前 5 条 exchange → 判断第一条用户消息；从 `cwd` 判断项目
4. **处理决策**: 纯闲聊（<10 exchange，无工具调用）→ 跳过；大会话（>5MB）→ 标记需分片

输出: `[{sessionId, agentType, project, intent, size, exchanges, skip, needsSharding}]`

### ❷ 源适配 + 去噪（脚本）

1. 按 agent 类型映射字段 → 统一中间格式（见 `references/06-extraction-format.md`）
2. 去噪：砍 hook 注入文本、skill 列表、tool-use metadata、系统消息、中断信号
3. 统计工具调用：记录所有 tool call（类型、次数、目标文件/命令）

**输出**: 清洁对话流 + `operationsSummary`（工具/命令/文件统计）

**参考**: 读取 `references/06-extraction-format.md`

### ❸ 识别主题（LLM）

遍历清洁对话流，识别对话中讨论的独立主题。每个主题标注：

- `topicId`: `topic-{sessionId前8位}-NNN`
- `summary`: 一句话主题摘要
- `uniqueKeywords`: 3-5 个在该主题下独有的关键词（用于后续聚拢）
- `category`: 预判分类方向（一个主题可产出多个分类的记忆）

**识别原则**: 不是按时间切——同一主题即使分散在多处也会被关键词聚拢回来。寻找对话中的"话题切换点"作为边界提示，但不强制切分。

**参考**: 读取 `references/04-topic-identification.md`

### ❹ 聚拢 + 提取 + 分类 + 打标 + 生成 .md（LLM）

**对每个主题，一步完成**：

1. **关键词回搜聚拢**: 用 `uniqueKeywords` 回搜全对话流，聚拢所有相关 exchange（包括对话中间隔断的同一主题片段）
2. **提取**: 起因、过程、结果、因果、产出物。**质量门禁**——不得出现"用户发起对话""待补充""N/A"。
3. **分类**: 按 `references/01-classification.md` 的提取信号 + 决策树判定（一个主题可产出多条不同分类的记忆）
4. **打标**: 项目(cwd)、时间(timestamp)、主题ID(topicId)、关键词(3-8个)、前置主题ID、关联
5. **生成 .md**: 按 `references/02-file-format.md` 完整模板生成，**末尾追加"操作环境"段**

**操作环境段**（每条 .md 末尾必带）:
```markdown
## 操作环境
- 会话: {sessionId}
- 时间: {HH:MM - HH:MM}
- 技能: [列出本主题涉及的所有 skill 调用]
- 工具: Read(N), Bash(N), Write(N), WebSearch(N)
- 外部命令: [npx/powershell 等]
- 新发现: [本主题中发现的 CC 特性/外部知识/bug]
- 前置主题: {topicId of preceding topic，如无则写 null}
```

**参考**: 读取 `references/01-classification.md` `references/02-file-format.md` `references/03-keywords.md`

### ❺ 去重融合（LLM）

全部主题的 .md 生成完毕后统一执行：

1. **同会话内**: 检查生成的 .md 列表有无主题重叠 → 重叠的合并为一条，补充细节
2. **跨会话**: 与已有记忆文件比对关键词重叠 ≥ 3 → 同一事件则更新时间线（追加"## {日期} 更新"段），不同事件则写关联
3. **阈值检查**: 同类错误 ≥ 3 触发立法提醒，问题 ≥ 5 触发审查提醒

**参考**: 读取 `references/03-keywords.md`

### ❻ 写入 + 更新 manifest + 汇聚 aggregate（脚本）

1. 按 `references/02-file-format.md` 的模板逐个写入 .md
2. 路径: `{记忆目录}/{分类}/{序号}_{标题}.md`，序号按已有文件自动递增
3. 从每条 .md 的"操作环境"段汇聚 → 更新 `{记忆目录}/_aggregate/stats.json`
4. 更新 `.manifest.json`，当前会话标记为 `processed`
5. 按输出摘要格式报告结果

**参考**: 读取 `references/02-file-format.md` `references/05-pipeline.md`

## 输出摘要格式

每次运行完毕输出：

```
📊 会话归档完成 — YYYY-MM-DD HH:MM

处理会话: N 个 (id1, id2, ...)
提取记忆: M 条
  错误: a  决策: b  偏好: c  模式: d  排除: e
  不变量: f  讨论: g  问题: h  知识: i  配置: j  使用: k

去重合并: n 条

阈值提醒:
  [如有] 🔴 XX类累计 ≥ 3 → 触发立法提醒
  [如有] 🟡 问题累计 ≥ 5 → 建议规划审查

写入文件:
  Mio-SBAgnet—memory/问题/01_Harness层hook全部缺失.md
  Mio-SBAgnet—memory/决策/01_四波修复优先级.md
  ...
```

## 外部 agent 兼容指南

本 skill 设计为任何支持 Markdown 指令的 LLM agent 均可执行。如果外部 agent 没有文件写入工具：

1. 步骤 ❶-❺ 照常执行（读文件和分析）
2. 步骤 ❻ 输出完整 Markdown 文本而非写入
3. 在回复末尾说："以上是记忆文件内容。请手动创建对应文件，或授权写入权限后重新运行。"

如果外部 agent 的上下文窗口较小：
- 一次只处理一个会话
- 处理完一个后，暂停，让用户确认后继续下一个

## 并发安全

多 agent 同时运行时，写入步骤 ❽ 存在竞态。按以下策略执行：

1. **按 session 分片**：每个 agent 处理不重叠的会话组，避免同 session 冲突。同 sessionId 的所有主题必须由同一 agent 处理
2. **序号预分配**：步骤 ❻ 写入前先读取目标分类目录，计算序号范围并声明占用
3. **manifest 锁**：更新 `.manifest.json` 前，先读当前内容，合并新条目后一次性写入。若写入时发现内容已被其他 agent 修改（JSON 内容与读取时不一致），重试最多 3 次
4. **会话级单写**：同 sessionId 的所有事件必须由同一 agent 处理并写入——不跨 agent 拆分

## Gotchas

- 同一会话可产生多个主题，由 LLM 识别而非按时间切分——同一主题即使分散多处也会被关键词聚拢
- 会话间可能有关联（同一话题跨会话讨论），由步骤 ❺ 跨会话去重 + `_aggregate` 自动捕获
- manifest 是会话级追踪——处理过的会话第二次跑会跳过；如需重新处理，从 manifest 删除对应条目
- 去重是增量安全的——重复跑不会产生重复文件
- 步骤 ❹ 提取时宁可多写不少写——步骤 ❺ 去重融合会处理冗余
- "操作环境"段从 JSONL 的工具调用中提取，不需要 LLM 判断——脚本在步骤 ❷ 已整理好

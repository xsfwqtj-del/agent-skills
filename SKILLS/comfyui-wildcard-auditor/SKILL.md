---
name: comfyui-wildcard-auditor
description: 审查 ComfyUI ImpactWildcardProcessor 的 wildcard txt 库质量。当用户说：审查wildcard、检查提示词库、审计wildcard、wildcard质量检查、wildcard有没有问题、纠错wildcard、检查XX.txt时触发。检测重复行、格式污染、雷区违规、无效标签。
---

# ComfyUI Wildcard 库审查器

读取全部 wildcard txt 文件，逐行扫描检测质量问题，列出问题清单并修复。

## 前置信息

### Wildcard 目录

```
F:\ComfyUI\ComfyUI-aki-v3\ComfyUI\custom_nodes\ComfyUI-Impact-Pack\wildcards\一键随机\
```

### 9 个受检文件

体位.txt、发型.txt、场景.txt、服装.txt、氛围.txt、画质.txt、表情.txt、视角.txt、身材.txt

以及任何带 `_` 前缀的辅助文件（如 `_通用正向提示词_色气.txt`）

### 文件格式

每行是一个完整 wildcard 选项，逗号分隔。ImpactWildcardProcessor 随机抽取。

## 检测项目

### 🔴 致命（必须修复）

**格式污染**
- 文件中的 Markdown 风分隔符（`== ... ==`、`---`、`# 标题`）
- 空行（空白行/只有空白字符的行）
- 纯注释行（不会被承认为标签的说明文字）
- **为什么致命**：ImpactWildcardProcessor 会随机抽到这些行，把它们当标签注入 prompt，生成破烂图

**雷区违规**
按用户明确雷区扫描每行。发现任何违规则标记：

全局雷区：
- 西漫/美式风：`comic`、`toon`、`cartoon`、`western_comics`、`kubrick_stare`、`troll_face`、emoji 表情符
- 写实/照片：`photorealistic`、`realistic`、`3D`、`photo_(medium)`、`photo_background`
- Pony 标签：`score_9`、`score_8_up`、`score_7_up`、`source_pony`、`source_furry`
- R18G：`gore`、`blood`、`violence`、`scar`、`wound`、`death`、`torture`、`guro`、`vomit`、`scat`、`feces`、`urine`

类别专属雷区：
- 发型.txt：`short_hair`、`bald`、`buzz_cut`、`crew_cut`、`mohawk`、`dreadlocks`、`cornrows`
- 身材.txt：`fat`、`obese`、`chubby`、`plump`、`emaciated`、`anorexic`
- 表情.txt：`cold_stare`（用户明确删除过）、`crying`
- 画质.txt：`score_9`、`score_8_up`（Pony 标签系统，Illustrious 不适用）

### 🟡 警告（建议修复）

**完全重复行**
两行内容完全相同（逐字符比对）。

**高度雷同行**（相似度 > 85%）
- 仅个别词汇排列顺序不同
- 例如 `slender, model figure, narrow waist, long legs, elegant` 和 `slender, model figure, narrow waist, long legs, small breasts, elegant`

**近重复场景/体位**
- 场景和体位文件中，同一场景只改变了一个次要细节的多行
- 例：同一个位置只换了时间（day → night），保留最多 2 个变体

### 🟢 信息（不修）

**偏 SFW 标签**
- 不违规但不够「色気」的标签
- 如 `peaceful, gentle smile, blushing` — 保留但标记，让用户决定

**文件统计**
- 每个文件的行数
- 总标签数
- 与上次审计的差异（如有记录）

## 工作流

### 第一步：读取全部

用 Read 工具逐个读取 9 个文件（优先并行读取）。

### 第二步：逐行扫描

对每行执行检查：
1. 第一个字符 → 如果是 `=` `#` `-` 且行首没有标签特征 → 🔴 格式污染
2. 空行（trim 后为空）→ 🔴
3. 遍历雷区关键词列表 → 🔴 雷区违规
4. 与所有已扫描行做相似度比对 → 🟡 重复/雷同

### 第三步：生成报告

按以下格式输出：

```
## 审计报告

### 概要
| 指标 | 数值 |
|------|------|
| 检查文件数 | 9 |
| 总行数 | XXX |
| 🔴 致命问题 | N 个 |
| 🟡 警告 | N 个 |
| 🟢 信息 | N 条 |

### 🔴 致命问题
#### 格式污染
| 文件 | 行号 | 内容 | 问题 |
|------|------|------|------|

#### 雷区违规
| 文件 | 行号 | 违规标签 | 违规类型 |
|------|------|---------|---------|

### 🟡 警告
| 文件 | 行号 | 内容 | 问题描述 |
|------|------|------|---------|

### 📊 各文件统计
| 文件 | 行数 |
|------|------|
```

### 第四步：修复

对每个 🔴 致命问题和 🟡 警告，直接修复。顺序：
1. 🔴 先行：删除分隔符行、空行、违规行
2. 🟡 后行：删除重复行、合并雷同行

使用 Edit 工具逐个修复，修复后说明改了什么。

### 第五步：验证

修复后重新读取所有文件，确认问题行数 = 0。

---

## 雷区参考（完整清单）

以下全部来自用户的明确指示（ComfyUI 环境与出图偏好记忆文件 + 会话记录）：

- **身材禁**：胖(plump/fat/chubby)、极瘦(emaciated/anorexic)
- **发型禁**：短发及以下、光头系。最短到 medium hair/bob cut
- **表情禁**：cold stare、crying、夸张颜艺（ahegao 除外）
- **R18G 禁**：血腥/暴力/致残/死亡/粪尿
- **标签体系**：Illustrious 用 `masterpiece, best quality`，不用 Pony 的 `score_9`
- **风格禁**：西漫、美式卡通、写实、3D、照片风
- **目标**：日系动漫/Illustrious 风格，色气方向

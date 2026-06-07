---
name: mio-sbagent-self-contained
description: Use when writing new code, creating new modules, or organizing file structure and need AI agents to understand modules without reading other files or needing project background. Applies to .py and .ts files following the self-contained module pattern.
trigger: auto
---

# Mio-SBAgent-Self-Contained

> 版本: v3.0.1 | 日期: 2026-06-05 | 全部 8 分支完成 | 更新: CSO修复+常见错误+Red Flags+不适用场景
> 替换: ai-native-dev v1.1.0

## 北极星

**为谁？** 非程序员用户 + AI agent 助手。

**解决什么？** 人和 AI 都无法单独处理大型代码库——人的工作记忆上限 ~4 chunks，AI 的上下文窗口有限，几十个文件来回跳就退化。

**Mio-SBAgent-Self-Contained 是什么？** 一个代码质量属性。跟"可测试""可维护""低耦合"并列——区别是那些为人类程序员定，这条为 AI agent 上下文窗口定。

**判断标准：** 任何一个 `.py` / `.ts` 文件，Ctrl+A 全选 → 丢进 AI 窗口 → AI 读完头部就能干活。不翻别的文件，不需要人解释项目背景。

---

## 导航

[A] 术语表 | [B] 模块头部模板 | [C] 内部导航规范 | [D] 系统地图 MAP.md | [E] 冻结构件 | [F] 横切关注点 | [G] AI 检查清单 | [H] 非程序员验收 | [I] Python 示范 | [J] TypeScript 示范

---

## [A] 术语表 ~800 tokens

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

## [B] 模块头部模板 ~500 tokens

每个 `.py` / `.ts` 文件必须以下列模板开头。AI agent 读完头部就能决定要不要继续读、读哪段。

```
# ───────────────────────────────────────────
# [模块名]
#   我是谁: [一句话角色说明]
#   我消费: [输入类型 + 示例值]
#   我产出: [输出类型 + 示例值]
#   我依赖: [外部接口签名，不写实现路径]
#   冻结: YES/NO | 冻结日期: YYYY-MM-DD | 依赖方数量: N
#   横切: [我处理的关注点] | 豁免: [我不管的关注点]
# ───────────────────────────────────────────
# 导航: [A] 核心逻辑 | [B] 数据结构 | [C] 公共接口 | [D] 依赖适配 | [E] 错误处理
# token: 头部~150 | A~300 | B~200 | C~250 | D~150 | E~100 | 合计~1150
# ───────────────────────────────────────────
```

### 字段说明

| 字段 | 必填 | 说明 | 反例 |
|------|------|------|------|
| 我是谁 | ✅ | 不说项目名，只说自己干什么 | ❌ "Mio 项目的 agent 模块" → ✅ "Agent 执行器——接收 task → 执行 → 返回结果" |
| 我消费 | ✅ | 类型 + 示例值 | ❌ "消费数据" → ✅ "`TaskRequest {id: str, type: str, payload: dict}`" |
| 我产出 | ✅ | 类型 + 示例值 | ❌ "产出结果" → ✅ "`TaskResult {id: str, status: 'ok'|'fail', output: str}`" |
| 我依赖 | 按需 | 接口签名，不写路径 | ✅ "需要 `ILogger.log(level, msg)`" ❌ "需要 `utils/logger.py:log_to_file()`" |
| 冻结 | 按需 | 满足三条件才标 YES | — |
| 横切 | 按需 | 处理什么 + 豁免什么 | "处理: 日志 | 豁免: 认证、限流" |
| 导航 | ✅ | 标签 + 段落名 | — |
| token | ✅ | 按段估算，AI 据此判断是否值得读 | — |

### 合约规则

1. **公共接口 ≤ 5 个函数** —— 深模块：简单接口 + 复杂实现（Ousterhout）
2. **函数参数 ≤ 3** —— 超了用配置对象/数据类
3. **嵌套条件 ≤ 2 层** —— 超了提取函数
4. **合约字段无悬空引用** —— 标签必须在文中有对应段

---

## [C] 内部导航规范 ~450 tokens

### 标签语法

```
通用标签: [A] [B] [C] [D] [E] [F] [G] [H] [I] [J]
语义标签: [合约] [接口] [测试] [配置] [错误] [日志]
```

标签放在行首（或注释内），全局唯一。推荐通用标签 `[A]`-`[J]` 避免语义歧义。

### 导航地图格式

文件头部必须有一行"导航:"列出所有标签：

```
# 导航: [A] 核心逻辑(L12) | [B] 数据结构(L45) | [C] 公共接口(L78) | [D] 依赖适配(L102) | [E] 错误处理(L130)
```

### 段标记格式

每个段开头用标签标记：

```python
# [A] 核心逻辑 ──────────────────────────────────────
def process(task: TaskRequest) -> TaskResult:
    ...
```

### 跳段规则

| AI 要做什么 | 读哪个标签 | 理由 |
|------------|-----------|------|
| 理解模块做什么 | [A] 核心逻辑 | 先看全景 |
| 调用模块 | [C] 公共接口 | 只读签名和合约 |
| 改内部逻辑 | [A] + [B] | 逻辑 + 数据结构 |
| 排查错误 | [E] 错误处理 + [C] | 错误类型 + 接口行为 |
| 替换依赖 | [D] 依赖适配 | 适配器位置 |
| 做依赖方改动 | 头部（20行就够了） | 只需知道合约 |

### 标签递进

```
读导航行（1 行） → 定位标签 → 跳到目标段 → 只读该段 → 完成
全程：头部 ~150 tokens + 目标段 ~250 tokens = ~400 tokens
对比无导航全文读完：~1200+ tokens
节省：~66%
```

---

## [D] 系统地图 MAP.md ~500 tokens

项目根目录 `MAP.md`，AI agent 进项目读的第一个文件。**本身必须是 Mio-SBAgent-Self-Contained——一个窗口装下。**

### MAP.md 模板

```markdown
# [项目名] 系统地图

> token: ~400 | 模块数: N | 冻结: M | 最后更新: YYYY-MM-DD

## 目录树
src/
├── core/           ← [A] 核心域
├── services/       ← [B] 服务层
├── adapters/       ← [C] 适配层
└── shared/         ← [D] 共享工具

## 模块合约速查

| 模块 | 合约 | 冻结 | 文件 |
|------|------|------|------|
| AgentRunner | 消费 TaskRequest → 产出 TaskResult | ✅ | `src/core/runner.py` |
| ToolRegistry | 消费 tool_name → 产出 ToolHandler | ❌ | `src/core/registry.py` |
| LLMAdapter | 依赖 ILLMClient.generate(prompt)→str | ❌ | `src/adapters/llm.py` |

## 数据流

TaskRequest → AgentRunner → ToolRegistry → LLMAdapter → TaskResult
```

### MAP.md 规则

1. **一个窗口装下** — 全部内容 ≤ 800 tokens（头 200 行）
2. **只写合约，不写实现** — 每个模块一行：消费什么 → 产出什么
3. **冻结标注** — ✅/❌ 一眼判断哪些可以少关注
4. **数据流** — 沿着合约消费/产出字段追踪，不写具体调用链
5. **目录树** — 按功能域分，不是按文件类型分

### MAP.md 与模块头部的关系

```
MAP.md 说"AgentRunner: 消费 TaskRequest → 产出 TaskResult，在 src/core/runner.py"
  → AI 打开 runner.py → 读头部 20 行 → 知道完整合约
    → 按标签跳到 [C] 公共接口 → 只读需要的函数
```

**互补不重复**：MAP.md 说"哪个模块有什么合约"，头部说"这个合约怎么实现"。

---

## [E] 冻结构件 ~400 tokens

### 三验收条件（必须全部满足）

| # | 条件 | 判定方法 |
|---|------|---------|
| 1 | 30 天内无功能修改 | `git log --since="30 days ago" -- <file>` 只有 refactor/docs/test 提交 |
| 2 | ≥2 个模块依赖它 | grep 其他模块头部的"我依赖"字段，数引用次数 |
| 3 | 合约字段无变化 | 头部"我消费/我产出/我依赖"字段 = 30 天前的版本 |

### 冻结流程

```
模块满足三条件 → 改头部 "冻结: NO" → "冻结: YES | 冻结日期: YYYY-MM-DD | 依赖方数量: N"
              → 在 MAP.md 对应行改 ❌ → ✅
              → 提交 commit: "freeze: [模块名] 冻结认知单元"
```

### 解冻流程（合约变更时）

```
需要改合约 → 解冻标记 "冻结: YES" → "冻结: NO (解冻原因: [合约变更说明])"
          → MAP.md 改 ✅ → ❌
          → 通知所有依赖方
          → 改完后重新走冻结流程
```

### 为什么冻结

冻结不是"这模块完美了"——是"它的合约稳定了，AI agent 不需要每次读它的内部实现"。冻结后释放的认知容量可以分配给还在演化的模块。**冻结的是认知，不是代码。** 内部重构（不改合约）不需要解冻。

---

## [F] 横切关注点 ~350 tokens

横切关注点 = 多个模块都要碰但谁都不想"拥有"的公共行为：日志、认证、限流、指标、缓存、错误上报。

### 两种处理模式

**模式 1：服务注入（推荐）**

模块声明依赖接口，运行时注入实现：

```
# 我依赖: ILogger.log(level, msg) → void
# 横切: 处理: 日志（通过 ILogger） | 豁免: 认证、限流
```

```python
# [D] 依赖适配
class AgentRunner:
    def __init__(self, logger: ILogger):  # 注入，非导入
        self._log = logger
```

**模式 2：契约豁免**

明确说不处理什么，让上层/中间件负责：

```
# 横切: 处理: 无 | 豁免: 认证（由 auth_middleware 处理）、限流、日志
```

### 横切字段格式

```
# 横切: 处理: [关注点列表] | 豁免: [关注点列表]
```

例子：
```
# 横切: 处理: 日志(ILogger)、重试(内置) | 豁免: 认证、限流、指标
# 横切: 处理: 无 | 豁免: 日志、认证、限流、缓存
```

### 规则

- 每个模块必须声明横切字段——处理了什么 + 豁免了什么
- 豁免 != 忽略——必须说明谁负责（如"由 auth_middleware 处理"）
- 依赖注入优于全局导入——服务注入维护模块的自包含性

---

## [G] AI 检查清单 ~300 tokens

AI agent 打开模块后逐项判定，每项 YES/NO。不需要人类判断。

| # | 检查项 | 判定 |
|---|--------|------|
| 1 | 头部含「我是谁」——一句话说清角色 | YES / NO |
| 2 | 头部含「我消费」——输入类型 + 示例值 | YES / NO |
| 3 | 头部含「我产出」——输出类型 + 示例值 | YES / NO |
| 4 | 头部含「我依赖」——外部接口签名（不写实现路径） | YES / NO |
| 5 | 头部有导航标签——`[A]`...`[E]` 且有对应段 | YES / NO |
| 6 | 每个段有 token 估算 | YES / NO |
| 7 | 公共接口 ≤ 5 个函数 | YES / NO |
| 8 | 单函数参数 ≤ 3 | YES / NO |
| 9 | 嵌套条件 ≤ 2 层 | YES / NO |
| 10 | 合约字段无悬空引用（所有标签均有对应段） | YES / NO |
| 11 | 横切字段已填（处理了什么 + 豁免了什么） | YES / NO |
| 12 | 冻结字段准确（三条件全部满足才标 YES） | YES / NO |
| 13 | 头部 token ≤ 200（~25 行） | YES / NO |
| 14 | MAP.md 中能找到此模块的合约速查行 | YES / NO |

**通过标准**：14 项全部 YES → 模块合格。有 NO → 修复后重跑。

---

## [H] 非程序员验收 ~250 tokens

不需要懂代码。打开任意一个模块文件，只靠文件头部：

### 验收三步

| 步骤 | 问题 | 通过标准 |
|------|------|---------|
| 1 | 打开文件头 20 行能懂吗？ | 说出"这个文件干什么的" |
| 2 | 能看到 `[A]` `[B]` `[C]` 标记吗？ | 能指出来——"这里有标签" |
| 3 | 必须翻别的文件才能懂吗？ | 不需要——"头 20 行就够了" |

### 红牌

- "这行是什么意思？" → 合约用了代码术语 → 不合格
- "我得先懂 XX 模块才能看这个" → 依赖泄漏了实现 → 不合格
- "头 20 行看不懂，但我后面看懂了" → 头部没写好 → 不合格

### 对比验收

找两个模块：一个按此规范写的，一个没按。打开头 20 行，差别肉眼可见。

---

## [I] Python 示范 ~500 tokens

```python
# ═══════════════════════════════════════════════
# AgentRunner — Agent 执行器
#   我是谁: 接收 TaskRequest → 编排 Tool + LLM → 返回 TaskResult
#   我消费: TaskRequest {id: str, type: str, payload: dict}
#   我产出: TaskResult {id: str, status: 'ok'|'fail', output: str}
#   我依赖: IToolRegistry.lookup(name: str) -> ToolHandler
#           ILLMClient.generate(prompt: str) -> str
#   冻结: NO | 依赖方数量: 1 (AgentPipeline)
#   横切: 处理: 重试(内置) | 豁免: 日志(ILogger注入)、认证、限流
# ═══════════════════════════════════════════════
# 导航: [A] 核心逻辑(L35) | [B] 内部状态(L72) | [C] 公共接口(L95) | [D] 依赖适配(L130) | [E] 错误处理(L155)
# token: 头部~150 | A~300 | B~150 | C~200 | D~150 | E~150 | 合计~1100
# ═══════════════════════════════════════════════

from dataclasses import dataclass
from typing import Protocol


# ── 类型定义 ─────────────────────────────────

@dataclass
class TaskRequest:
    """我消费的主类型"""
    id: str
    type: str          # 例: "chat", "tool_call", "system"
    payload: dict      # 例: {"message": "hello", "tool_name": "search"}


@dataclass
class TaskResult:
    """我产出的主类型"""
    id: str
    status: str        # "ok" 或 "fail"
    output: str


class IToolRegistry(Protocol):
    """我依赖的接口签名（不写实现路径）"""
    def lookup(self, name: str) -> object: ...


class ILLMClient(Protocol):
    """我依赖的接口签名"""
    def generate(self, prompt: str) -> str: ...


# [A] 核心逻辑 ──────────────────────────────────

class AgentRunner:
    """接收 task → 按 type 分发 → 收集结果 → 返回."""

    def __init__(self, tools: IToolRegistry, llm: ILLMClient):
        self._tools = tools
        self._llm = llm
        self._retries = 0  # [B]

    def run(self, task: TaskRequest) -> TaskResult:
        """一次完整的 task 执行。不超过 2 层嵌套。"""
        handler = self._dispatch(task.type)
        try:
            output = handler(task)
            return TaskResult(id=task.id, status="ok", output=str(output))
        except Exception as exc:
            return self._handle_error(task, exc)

    def _dispatch(self, task_type: str):
        """按任务类型返回处理函数。"""
        handlers = {
            "chat": self._run_chat,
            "tool_call": self._run_tool,
            "system": self._run_system,
        }
        if task_type not in handlers:
            raise ValueError(f"未知任务类型: {task_type}")
        return handlers[task_type]

    def _run_chat(self, task: TaskRequest) -> str:
        return self._llm.generate(task.payload["message"])

    def _run_tool(self, task: TaskRequest) -> str:
        tool = self._tools.lookup(task.payload["tool_name"])
        return tool(task.payload)

    def _run_system(self, task: TaskRequest) -> str:
        return "system task done"


# [B] 内部状态 ─────────────────────────────────
    @property
    def retry_count(self) -> int:
        return self._retries


# [C] 公共接口 ─────────────────────────────────
    # 3 个公共方法: run, retry_count, 构造函数   [≤5 ✅]


# [D] 依赖适配 ─────────────────────────────────
    # IToolRegistry 和 ILLMClient 由调用方注入
    # 切换实现（mock / real / cloud）不需要改本模块


# [E] 错误处理 ─────────────────────────────────
    def _handle_error(self, task: TaskRequest, exc: Exception) -> TaskResult:
        self._retries += 1
        return TaskResult(
            id=task.id,
            status="fail",
            output=f"[{type(exc).__name__}] {exc}"
        )
```

---

## [J] TypeScript 示范 ~500 tokens

```typescript
// ═══════════════════════════════════════════════
// AgentRunner — Agent 执行器
//   我是谁: 接收 TaskRequest → 编排 Tool + LLM → 返回 TaskResult
//   我消费: TaskRequest {id: string, type: TaskType, payload: Record<string, unknown>}
//   我产出: TaskResult {id: string, status: 'ok'|'fail', output: string}
//   我依赖: IToolRegistry.lookup(name: string) => ToolHandler
//           ILLMClient.generate(prompt: string) => Promise<string>
//   冻结: NO | 依赖方数量: 1 (AgentPipeline)
//   横切: 处理: 重试(内置) | 豁免: 日志(ILogger注入)、认证、限流
// ═══════════════════════════════════════════════
// 导航: [A] 核心逻辑(L35) | [B] 内部状态(L72) | [C] 公共接口(L95) | [D] 依赖适配(L130) | [E] 错误处理(L155)
// token: 头部~150 | A~300 | B~150 | C~200 | D~150 | E~150 | 合计~1100
// ═══════════════════════════════════════════════

// ── 类型定义 ─────────────────────────────────

type TaskType = "chat" | "tool_call" | "system";

interface TaskRequest {
  /** 我消费的主类型 */
  id: string;
  type: TaskType;            // 例: "chat", "tool_call", "system"
  payload: Record<string, unknown>;  // 例: {"message": "hello"}
}

interface TaskResult {
  /** 我产出的主类型 */
  id: string;
  status: "ok" | "fail";
  output: string;
}

interface IToolRegistry {
  /** 我依赖的接口签名（不写实现路径） */
  lookup(name: string): ToolHandler;
}

type ToolHandler = (task: TaskRequest) => string;

interface ILLMClient {
  /** 我依赖的接口签名 */
  generate(prompt: string): Promise<string>;
}


// [A] 核心逻辑 ──────────────────────────────────

class AgentRunner {
  // [B] 内部状态
  private retries = 0;

  constructor(
    private tools: IToolRegistry,
    private llm: ILLMClient,
  ) {}

  /** 一次完整的 task 执行。不超过 2 层嵌套。 */
  async run(task: TaskRequest): Promise<TaskResult> {
    const handler = this.dispatch(task.type);
    try {
      const output = await handler(task);
      return { id: task.id, status: "ok", output: String(output) };
    } catch (exc) {
      return this.handleError(task, exc as Error);
    }
  }

  private dispatch(taskType: TaskType): (t: TaskRequest) => Promise<string> {
    const handlers: Record<TaskType, (t: TaskRequest) => Promise<string>> = {
      chat: (t) => this.runChat(t),
      tool_call: (t) => this.runTool(t),
      system: (t) => this.runSystem(t),
    };
    if (!(taskType in handlers)) {
      throw new Error(`未知任务类型: ${taskType}`);
    }
    return handlers[taskType];
  }

  private async runChat(task: TaskRequest): Promise<string> {
    return this.llm.generate(task.payload["message"] as string);
  }

  private async runTool(task: TaskRequest): Promise<string> {
    const tool = this.tools.lookup(task.payload["tool_name"] as string);
    return tool(task);
  }

  private async runSystem(_task: TaskRequest): Promise<string> {
    return "system task done";
  }


  // [C] 公共接口 ─────────────────────────────────
  /** 重试次数（只读） */
  get retryCount(): number {
    return this.retries;
  }
  // 3 个公共成员: run, retryCount, constructor   [≤5 ✅]


  // [D] 依赖适配 ─────────────────────────────────
  // IToolRegistry 和 ILLMClient 由调用方注入
  // 切换实现（mock / real / cloud）不需要改本模块


  // [E] 错误处理 ─────────────────────────────────
  private handleError(task: TaskRequest, exc: Error): TaskResult {
    this.retries++;
    return {
      id: task.id,
      status: "fail",
      output: `[${exc.constructor.name}] ${exc.message}`,
    };
  }
}
```

---

## 常见错误

| 错误 | 预防 |
|------|------|
| 写得太多——把自包含误解为"完整文档" | 头部是合约不是文档。AI 读完合约就该能决定要不要继续读 |
| 只写不导航——没加标签分段 | `[A]` `[B]` 标签是硬要求。没有导航 AI 必须全文读完 |
| 头部信息过时——改代码没同步改合约 | 合约先于代码。改了函数签名必须同步更新头部 |
| 把自包含误解为隔离 | 模块可以依赖外部接口，但接口签名写在头部里 |

---

## Red Flags

- "这个模块很简单不需要头部"
- "头部以后再补"
- "把整个 README 当头部"

**这些全部意味着你正在违反自包含原则。**

---

## 不适用场景

- 配置文件（JSON/YAML/TOML）→ 不是 .py/.ts 模块
- 一次性脚本 → 不会反复被 AI agent 读取
- 第三方库代码 → 不归本模块规范管辖

---

## 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.1.0 | 2026-05 | 原始 ai-native-dev 版本 |
| v2.0.0 | 2026-06-05 | 分支 A — 术语体系 + 核心原则重写 |
| v3.0.0 | 2026-06-05 | 分支 B~G — 头部模板 + 导航规范 + MAP.md + 冻结 + 横切 + 检查清单 + 双语言示范 |

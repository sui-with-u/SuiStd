# SuiBot 正式框架文档 v2.0

> **SuiBot：基于 LLM 的拟人化智能体——正式框架概述**
> 文档版本：v2.0 · 2026.05

---

## 目录

- [引言：为什么要重构](#引言为什么要重构)
- [第一章：整体架构](#第一章整体架构)
  - [1.1 设计哲学](#11-设计哲学)
  - [1.2 技术栈](#12-技术栈)
  - [1.3 五层架构概览](#13-五层架构概览)
  - [1.4 新版运行流程图](#14-新版运行流程图)
- [第二章：运行流程详解](#第二章运行流程详解)
  - [2.1 五步核心流程](#21-五步核心流程)
  - [2.2 半阻塞总线流说明](#22-半阻塞总线流说明)
  - [2.3 Skill 执行分支](#23-skill-执行分支)
- [第三章：模块规范](#第三章模块规范)
  - [3.1 Core 主控模块](#31-core-主控模块)
  - [3.2 Hand Manager](#32-hand-manager)
  - [3.3 Hand 接入规范](#33-hand-接入规范)
  - [3.4 Skill Manager](#34-skill-manager)
  - [3.5 Skill 编写规范](#35-skill-编写规范)
- [第四章：引擎层详解](#第四章引擎层详解)
  - [4.1 Emotion Engine · PAD 情感引擎](#41-emotion-engine--pad-情感引擎)
  - [4.2 Memory Engine · 记忆引擎](#42-memory-engine--记忆引擎)
  - [4.3 Self-Reflection Engine · 自我审视引擎](#43-self-reflection-engine--自我审视引擎)
  - [4.4 角色档案（基础设定）](#44-角色档案基础设定)
- [第五章：WebSocket 通信协议规范 v1.0](#第五章websocket-通信协议规范-v10)
  - [5.1 连接基础](#51-连接基础)
  - [5.2 消息格式](#52-消息格式)
  - [5.3 消息类型与 Payload](#53-消息类型与-payload)
  - [5.4 通信流示例](#54-通信流示例)
  - [5.5 SDK 快速参考](#55-sdk-快速参考)
  - [5.6 错误码表](#56-错误码表)
- [附录 A：变更历史](#附录-a变更历史)
- [附录 B：快捷指令表](#附录-b快捷指令表)

---

## 引言：为什么要重构

MMVP 阶段验证了核心方向的可行性——PAD 情感模型可以工作，记忆分层机制可以工作，穗穗「活了」。但在架构讨论中，原始设计的问题被清晰指出：

**原版的问题：**

- Core 太大，既计算情感值、又写数据库、又调用 LLM、又管理 Skill——职责混乱，极难扩展
- 模块串行耦合过深，任何一处出错都会卡死整条链路，排查困难
- 没有统一的中心调度，Hands、Skill、引擎各自为政，缺乏优雅的插拔机制
- 所谓「猫爪（Hands）」没有统一管理器，无法横向扩展到 QQ 以外的平台

**重构目标：**

- Core 只做五件事：接收 → 查引擎 → 组 Prompt → 调 LLM → 决策分发
- 情感、记忆、性格演化全部剥离为独立引擎，被 Core 调用而非内嵌其中
- Skill 与 Hand 均由专属 Manager 统一管理，支持热插拔
- 整个系统像「人」一样一次只做一件事（半阻塞总线流）

---

## 第一章：整体架构

### 1.1 设计哲学

SuiBot v2 采用**类冯·诺依曼半阻塞总线流**架构。

冯·诺依曼架构的「诟病」之一是指令总线与数据总线的互斥——一次只能做一件事，要么取指令，要么取数据。我们把这个「缺陷」变成设计目标：让 SuiSui 的行为模式更贴近真实人类——**她一次只处理一件事，处理完了才接下一件**。

这带来两个好处：

1. 行为一致性更强，不会出现「同时在两个对话里情绪不一样」的尴尬
2. 系统状态清晰，调试容易，不需要追踪并发竞态

**核心原则（一句话版本）：**

> Core 是调度者，不是大包大揽者。所有可独立封装的能力，均下沉到对应模块。

### 1.2 技术栈

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| 后端语言 | TypeScript 7 Beta | 强类型，与前端统一语言 |
| 运行时 | Bun | 比 Node.js 更快，原生支持 TS |
| 前端框架 | React + Vite | 管理界面（WebUI） |
| UI 组件库 | ShadCN UI | 轻量、可定制 |
| 图表库 | Recharts | 情感曲线可视化 |
| 向量数据库 | ChromaDB | 长期记忆存储与检索 |
| 短期记忆 | 内存 Buffer（Map） | 最近 5~10 轮对话 |
| 通信协议 | WebSocket + JSON | 全双工，所有模块统一接入 |
| LLM 接入 | DeepSeek API（可换） | 便宜好用，支持国产 |

> **为什么从 Python 迁移到 TypeScript？**
> MMVP 用 Python 快速验证了方向。正式版考虑到生产环境长期运营的性能需求，以及前后端语言统一的工程便利，切换到 TypeScript + Bun。

### 1.3 五层架构概览

```
┌─────────────────────────────────────────────────────┐
│                    外部层 HANDS                      │
│         QQ      Telegram   Minecraft   WebUI   ...  │
└──────────────────────┬──────────────────────────────┘
                       │ WebSocket
┌──────────────────────▼──────────────────────────────┐
│                 接入层 HAND MANAGER                   │
│      WS 连接管理 · 优先级路由 · 消息规范化            │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  核心层 SUICORE                       │
│  接收 → 查引擎 → 组 Prompt → 调 LLM → 决策分发       │
└──────┬───────────────┬────────────────┬─────────────┘
       │               │                │
┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────────────┐
│   Emotion   │ │   Memory    │ │  Self-Reflection     │
│   Engine    │ │   Engine    │ │  Engine              │
│  PAD 三维   │ │ Buffer+VDB  │ │  定时审视→更新Prompt │
└─────────────┘ └─────────────┘ └─────────────────────┘
       ↑ 引擎层 CORE ENGINES（被 Core 调用，同步回调）
┌─────────────────────────────────────────────────────┐
│                 扩展层 SKILL MANAGER                  │
│         插拔式 · 统一调度 · 结果回调 Core             │
│    搜索Skill   日历Skill   自定义Skill   ...          │
└─────────────────────────────────────────────────────┘
```

**各层职责一览：**

| 层级 | 职责 | 可插拔 |
|------|------|--------|
| 外部层 HANDS | 各平台的消息收发适配器 | ✅ 任意增删 |
| 接入层 HAND MANAGER | WS 连接管理、优先级队列、消息规范化 | ❌ 核心基础设施 |
| 核心层 SUICORE | 调度、Prompt 组装、LLM 调用、决策 | ❌ 系统大脑 |
| 引擎层 CORE ENGINES | 情感计算、记忆管理、性格演化 | ✅ 可替换实现 |
| 扩展层 SKILLS | 可选功能扩展（搜索、日历等） | ✅ 热插拔 |

### 1.4 新版运行流程图

```
                    ┌─────────────────────────────────┐
                    │  外部消息（QQ/TG/MC/WebUI等）    │
                    └──────────────┬──────────────────┘
                                   │ WebSocket
                                   ▼
                    ┌─────────────────────────────────┐
                    │         HAND MANAGER             │
                    │  验证格式 → 补充元数据 → 优先级入队 │
                    └──────────────┬──────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────┐
│                      SUICORE 主控                         │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ ① 接收   │→ │ ② 查引擎 │→ │ ③ 组Prompt│→ │ ④ LLM  │ │
│  │ 解析消息 │  │memory+   │  │ sys+ctx  │  │ 推理   │ │
│  └──────────┘  │emotion   │  └──────────┘  └────┬────┘ │
│                └──────────┘                      │      │
│                     ↑ 同步回调                    ▼      │
│                                         ┌────────────┐  │
│                                         │ ⑤ 决策分发 │  │
│                                         └─────┬──────┘  │
└───────────────────────────────────────────────┼─────────┘
                              ┌─────────────────┴──────────────────┐
                              │                                     │
                    需要 Skill？NO                              YES  │
                              │                                     ▼
                              │                       ┌────────────────────┐
                              │                       │   SKILL MANAGER    │
                              │                       │ 调度对应 Skill 执行 │
                              │                       └────────┬───────────┘
                              │                                │ 结果回调 Core
                              │                                ▼
                              │                       ┌────────────────────┐
                              │                       │  Core 二次调用 LLM │
                              │                       │  整合 Skill 结果   │
                              │                       └────────┬───────────┘
                              │                                │
                              ▼                                ▼
                    ┌──────────────────────────────────────────────┐
                    │  更新引擎状态（Emotion + Memory 写入）        │
                    └──────────────────┬───────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────┐
                    │     HAND MANAGER → Hand          │
                    │  output(chat_reply) 下发回复     │
                    └─────────────────────────────────┘
```

> **引擎层（Emotion + Memory + Self-Reflection）** 以虚线/同步回调方式被 Core 调用，不主动发起通信，不属于主流程的串行节点，但其响应会阻塞 Core 继续执行（保证数据一致性）。

---

## 第二章：运行流程详解

### 2.1 五步核心流程

#### 步骤一：接收 & 规范化

1. Hand 通过 WebSocket 发送 `input(chat)` 消息
2. Hand Manager 验证 JSON 格式，缺少字段返回 `ERR_VALIDATION`
3. Hand Manager 补充 `msg_id`（UUID v4）、`timestamp`（Unix 秒）
4. 按优先级（PPP/P0/P1/P2/P3）加入消息队列
5. Core 取出消息，解析 `user_id`、`content`、`group_id` 等字段

#### 步骤二：查询引擎

Core 同步请求两个引擎，等待两者均返回后继续：

- **Emotion Engine**：返回当前 PAD 三维坐标和情绪标签（如「轻松愉快」）
- **Memory Engine**：以当前消息为查询向量，检索相关的短期 Buffer 和长期 VectorDB 记忆片段，返回格式化的上下文文本

#### 步骤三：组装 System Prompt

Core 将以下内容按顺序拼接为完整 System Prompt：

```
[角色档案 bio]           ← 固定，来自 character.json
[动态性格描述]           ← 由 Self-Reflection Engine 生成并定期更新
[当前情绪状态]           ← Emotion Engine 返回的 PAD 值与标签
[相关记忆片段]           ← Memory Engine 检索结果
[对话上下文]             ← 最近 N 轮历史（短期 Buffer）
[当前用户消息]           ← 本次输入
```

#### 步骤四：LLM 推理

调用 DeepSeek API，提交完整 System Prompt + 用户消息。

LLM 返回结构化响应，包含：

- `reply`：自然语言回复文本
- `tool_call`（可选）：需要调用的 Skill 名称与参数
- `emotion_delta`（可选）：对情绪的影响向量（LLM 判断本次对话的情感倾向）

#### 步骤五：决策 & 输出

**分支 A——无需 Skill：**

1. 触发引擎写入：
   - Emotion Engine 根据 `emotion_delta` 更新 PAD 值
   - Memory Engine 将本次对话存入 Buffer，必要时向 VectorDB 写入长期记忆
2. 通过 Hand Manager 将 `output(chat_reply)` 下发给对应 Hand

**分支 B——需要调用 Skill：**

1. Core 将 `tool_call` 参数转发给 Skill Manager
2. Skill Manager 调度对应 Skill 执行，等待结果
3. Skill 结果回传 Core
4. Core 携带 Skill 结果再次调用 LLM，生成最终自然语言回复
5. 执行与分支 A 相同的引擎写入 + 下发流程

### 2.2 半阻塞总线流说明

Core 在以下阶段会阻塞等待：

| 阻塞点 | 等待对象 | 超时处理 |
|--------|----------|----------|
| 步骤二 | Emotion Engine + Memory Engine 均返回 | 超时用默认值继续（不中断对话） |
| 步骤四 | LLM API 响应 | 超时返回固定错误提示，不重试 |
| 步骤五（分支B） | Skill 执行完成 | 超时返回「技能执行失败」提示 |

阻塞期间，新进入队列的消息会等待当前轮次处理完毕。PPP 级消息可打断（强制插队）。

### 2.3 Skill 执行分支

LLM 通过 Tool Calling 模式决定是否调用 Skill。Core 解析 LLM 返回的 `tool_call` 字段：

```typescript
// LLM 返回示例
{
  reply: "好的，我来帮你查一下天气！",
  tool_call: {
    skill_name: "weather_search",
    params: { city: "岳阳", date: "today" }
  }
}
```

Skill Manager 收到调用请求后：

1. 查找已注册的 `weather_search` Skill
2. 执行 `skill.execute({ city: "岳阳", date: "today" })`
3. 等待结果，回调 Core
4. Core 将结果注入第二次 LLM 调用的上下文

---

## 第三章：模块规范

### 3.1 Core 主控模块

**职责（只有这些，不多）：**

- 从 Hand Manager 接收规范化消息
- 调用 Emotion Engine 和 Memory Engine 获取上下文
- 组装 System Prompt
- 调用 LLM API
- 解析 LLM 返回，决定是否触发 Skill
- 触发引擎状态更新
- 将回复通过 Hand Manager 下发

**明确不做的事：**

- ❌ 不直接操作数据库（交给 Memory Engine）
- ❌ 不计算情感数值（交给 Emotion Engine）
- ❌ 不执行 Skill 业务逻辑（交给 Skill Manager）
- ❌ 不管理 WebSocket 连接（交给 Hand Manager）

**对外接口：**

Core 不直接对外暴露接口，所有通信通过 Hand Manager 中转。内部提供以下方法供引擎层回调：

```typescript
core.onEngineReady(emotionState, memoryContext)   // 引擎数据就绪
core.onSkillResult(skillName, result)              // Skill 执行完毕
```

**状态管理：**

Core 维护一个简单的状态对象，广播给所有 WebUI 类连接：

```typescript
interface CoreState {
  status: "idle" | "processing"    // 当前是否在处理消息
  currentEmotion: PADVector        // 当前情绪坐标
  connectedHands: string[]         // 已连接的 Hand 列表
  handCount: number
}
```

### 3.2 Hand Manager

**职责：**

- 管理所有 Hand 的 WebSocket 连接生命周期
- 实现握手协议（HELLO / WELCOME）
- 维护心跳检测（每 30s 发送 ping，10s 内未收到 pong 则计一次失败，连续 3 次强制断开）
- 按优先级维护消息队列，依次交给 Core 处理
- 将 Core 的 `output` 路由回正确的 Hand

**优先级队列规则：**

| 优先级 | 含义 | 处理规则 |
|--------|------|----------|
| PPP | Admin 控制指令 | 强制插队，立即处理 |
| P0 | 紧急交互（电话/直播） | 尽量立即处理 |
| P1 | 普通私聊 / @ 消息 | 正常处理 |
| P2 | Core 自行调整的优先级 | 空闲时处理 |
| P3 | 群聊非 @ 等低优先消息 | 空闲时处理 |

**连接标识规则：**

- 同一 `hand_name` 重复连接时，旧连接自动断开，新连接接管
- `hand_name` 使用小写字母 + 下划线：`napcat_qq`、`sui_web`、`mc_bot`

### 3.3 Hand 接入规范

任何新的 Hand 接入只需三步：

**Step 1：复制 SDK**

```bash
cp sdk/sui_sdk.ts my_hand/
# 或 Python 版
cp sdk/sdk.py my_hand/
```

**Step 2：实现三个回调**

```typescript
import { SuiClient } from "./sui_sdk"

const client = new SuiClient("napcat_qq", { handType: "hand" })

// 收到 Core 的回复，发给用户
client.onReply(async (payload) => {
  await sendQQMessage(payload.reply)
})

// 收到 Core 的主动通知（如提醒吃药）
client.onNotify(async (payload) => {
  await sendQQMessage(`[通知] ${payload.content}`)
})

// 出错了
client.onError((payload) => {
  console.error("SuiBot error:", payload)
})
```

**Step 3：连接并发送消息**

```typescript
await client.connect("ws://localhost:8000")

// 用户发消息了
await client.sendChat("今天天气真不错", { userId: "qq_123456" })
```

**hand_type 取值：**

| 值 | 含义 |
|----|------|
| `hand` | 外部通道（QQ、TG、Minecraft 等） |
| `webui` | 管理界面（SuiWeb、DevWeb） |
| `test` | 测试客户端 |

### 3.4 Skill Manager

**职责：**

- 维护已注册 Skill 的列表与元数据
- 接收 Core 的调用请求，找到对应 Skill 并执行
- 执行超时处理，结果回调 Core
- 支持热插拔（运行时注册/注销 Skill，无需重启 Core）

**Skill 注册方式：**

```typescript
skillManager.register(new WeatherSkill())
skillManager.register(new CalendarSkill())
skillManager.unregister("weather_search")   // 热卸载
```

**LLM 如何知道有哪些 Skill？**

Core 在组装 System Prompt 时，会将所有已注册 Skill 的 `name` 和 `description` 注入 Prompt，LLM 根据 description 自主决定是否调用。这是标准的 Tool Calling 模式。

### 3.5 Skill 编写规范

每个 Skill 是一个独立的 TypeScript 类，实现以下接口：

```typescript
interface SuiSkill {
  name: string               // 技能唯一标识，如 "weather_search"
  description: string        // 告诉 LLM 什么时候调用这个技能
  paramsSchema: object       // JSON Schema，描述参数格式
  execute(params: object): Promise<string>   // 执行并返回结果字符串
}
```

**示例：天气查询 Skill**

```typescript
// skills/weatherSearch.ts —— 这个文件只负责查天气

export class WeatherSkill implements SuiSkill {

  name = "weather_search"

  description = "当用户询问天气情况时调用，可以查询指定城市的当日或未来天气"

  paramsSchema = {
    city: { type: "string", description: "城市名称" },
    date: { type: "string", description: "日期，today 或 YYYY-MM-DD" }
  }

  async execute(params: { city: string; date: string }): Promise<string> {

    // 检查参数，缺少城市名就直接返回错误
    if (!params.city) {
      return "查询失败：没有指定城市名称"
    }

    // 调用天气 API
    const result = await fetchWeatherAPI(params.city, params.date)

    // 返回格式化的天气信息，LLM 会把它整合进回复里
    return `${params.city}${params.date === "today" ? "今天" : params.date}：${result.weather}，${result.temp}°C`
  }
}
```

**Skill 编写规则：**

- `execute` 只返回字符串，由 Core 交给 LLM 整合为自然语言，Skill 不负责措辞
- `execute` 内部必须有 Early Return 处理参数缺失情况
- 一个 Skill 文件只做一件事，文件名 = Skill 功能

---

## 第四章：引擎层详解

### 4.1 Emotion Engine · PAD 情感引擎

#### PAD 三维情感模型

PAD 模型由 Mehrabian 和 Russell 于 1974 年提出，将情感抽象为三个可量化的维度：

| 维度 | 全称 | 范围 | 正值 | 负值 |
|------|------|------|------|------|
| P | Pleasure（愉悦度） | [-1, 1] | 开心、愉悦、满足 | 难过、痛苦、失落 |
| A | Arousal（激活度） | [-1, 1] | 激动、兴奋、愤怒 | 疲惫、沉静、冷漠 |
| D | Dominance（支配度） | [-1, 1] | 自信、主动、掌控 | 被动、恐惧、顺从 |

#### 情绪衰减

人类的情绪不会永远维持高强度，会随时间趋于平静。SuiSui 的情绪以指数方式向稳态（`baseline_pad`）衰减：

```
E_{t+1} = (E_t - baseline) × e^{-λt} + baseline
```

- `λ`：衰减系数，默认 0.1，可通过 `config_set` 调整
- `t`：距上次更新的时间间隔（分钟）
- `baseline`：来自角色档案的 `homeostasis.baseline_pad`

#### 情绪波动

每次对话结束后，LLM 返回 `emotion_delta` 向量（三维），叠加到当前 PAD 坐标：

```
E_new = clamp(E_current + emotion_delta, -1, 1)
```

`clamp` 确保三个维度始终在 [-1, 1] 范围内。

#### 对外接口

```typescript
emotionEngine.getCurrentState(): { pad: PADVector; label: string }
emotionEngine.applyDelta(delta: PADVector): void        // Core 调用，更新情绪
emotionEngine.setPAD(pad: PADVector): void              // WebUI command 调用，手动设置
emotionEngine.decay(minutesPassed: number): void        // 定时任务调用，情绪衰减
```

#### 情绪标签映射（示例）

| P | A | D | 标签 |
|---|---|---|------|
| 高 | 中 | 中 | 轻松愉快 |
| 低 | 高 | 低 | 焦虑不安 |
| 低 | 低 | 低 | 低落沉闷 |
| 高 | 高 | 高 | 兴奋激动 |
| 中 | 低 | 中 | 平静中性 |

### 4.2 Memory Engine · 记忆引擎

#### 分层记忆机制

```
┌─────────────────────────────────────────────────────┐
│  短期记忆 Buffer（内存 Map）                         │
│  · 最近 5~10 轮对话                                  │
│  · 默认保留 2 小时                                   │
│  · 保障实时对话连贯性                                │
└──────────────────────┬──────────────────────────────┘
                       │ 超出时限或容量 → 写入
┌──────────────────────▼──────────────────────────────┐
│  长期记忆 VectorDB（ChromaDB）                       │
│  · 用户核心信息、重要偏好、关键经历                   │
│  · 以向量形式存储，支持语义检索                       │
│  · 通过遗忘机制动态调整，不无限增长                   │
└─────────────────────────────────────────────────────┘
```

#### 数学化遗忘（Forgetting Curve）

基于艾宾浩斯遗忘曲线，每个长期记忆片段有一个「强度」评分 `S ∈ [0, 1]`：

```
P(保持) = e^{-t/S}
```

- `t`：记忆存储时间（小时）
- `S`：强度评分，由写入时的重要性判断赋值（0.1~1.0）
- 当 `P < 0.2` 时，记忆片段进入「模糊状态」

**模糊状态的处理：**
记忆不会直接删除，而是标记为 `status: "fuzzy"`。Core 检索到模糊记忆时，LLM 会在回复中表现出「有点记不清了」「好像印象里有这么回事」等拟人化表达。这比直接丢失记忆更贴近人类行为。

#### 记忆混淆（Interference）

模拟人类「相似信息容易混淆」的特点。当检索结果中存在余弦相似度 > 0.85 的记忆片段时，向其向量加入高斯噪声：

```
vector_retrieved = vector_original + N(0, σ²)
σ = 0.05（当相似度 > 0.85 时启用）
```

结果：LLM 在回复中可能混淆相似事件的细节（如两次「去公园」的时间、地点），更贴近真实记忆状态。

#### 对外接口

```typescript
memoryEngine.getContext(query: string): Promise<MemoryContext>     // Core 查询
memoryEngine.writeShortTerm(turn: DialogTurn): void                // Core 写入短期
memoryEngine.writeLongTerm(fragment: MemoryFragment): Promise<void> // Core 写入长期
memoryEngine.search(query: string): Promise<MemoryFragment[]>      // WebUI 命令调用
memoryEngine.decay(): Promise<void>                                // 定时任务，遗忘处理
```

#### 解决 MMVP 的「黑盒问题」

MMVP 阶段 ChromaDB 的黑盒导致无法验证记忆是否正常工作。v2 解决方案：

- WebUI 提供 `memory_search` 命令，可实时查看 VectorDB 内容
- 所有记忆写入/读取操作写详细日志，可通过 `/get_memory` 指令导出
- 记忆强度评分和模糊状态可在 WebUI 中可视化展示

### 4.3 Self-Reflection Engine · 自我审视引擎

这是 SuiBot 拟人化设计的灵魂——SuiSui 的性格不是写死的固定模板，而是从她的经历和情感变化中逐渐「涌现」出来的。

#### 触发条件

两种触发方式，满足任一即执行：

1. 每收到 **10 条**用户消息后（可配置）
2. 每天 **23:00** 定时触发（可配置）
3. 用户执行 `/self-re` 快捷指令（手动触发，调试用）

#### 执行流程

```
触发自我审视
    ↓
读取近期 PAD 情绪变化曲线（过去 N 条消息的情绪轨迹）
    ↓
读取长期记忆中的核心记忆碎片（重要性评分最高的 Top-K）
    ↓
调用 LLM，生成一段「自我描述」
    ↓
将自我描述存储，在下次对话时注入 System Prompt
```

#### 自我描述示例

```
最近我总是很容易感到不安（情绪曲线：P 值持续偏低，A 值中等），
也许是因为想起了以前在镇子里那段漫长的雨季（核心记忆）。
我发现自己对别人的话变得更敏感了，也更需要被认可。
```

这段描述会影响下次对话的 System Prompt：

```
[性格当前状态]
最近变得更加敏感，容易对他人的态度产生情绪波动，
回复时语气应偏向温和，避免过于直接或强硬的表达，多给予共情。
```

#### 性格演化的边界

- 物理设定（名字、外貌、身世背景）是**固定锚点**，不受演化影响
- 说话风格、情感表达方式、主动性强弱会随经历**动态变化**
- 演化是**渐进的**，不会单次剧烈跳变

### 4.4 角色档案（基础设定）

存储在 `character.json`，可通过 WebUI 的 `character_get / character_update` 命令查看和修改：

```json
{
  "bio": {
    "name": "穗穗",
    "gender": "Female",
    "birthday": "万历四十六年 五月初六（1618年6月11日）",
    "age": "14，有时会谎称只有 9 岁",
    "height": "138cm",
    "physical_traits": ["黑发盘发", "齐刘海", "深蓝瞳", "有泪痣"],
    "background": "身处明末饥荒乱世，家人全部在饥馑中离世，成为孤女。在贫瘠荒凉的乡间辗转跋涉，一路孤身行走，饱经世间疾苦。"
  },
  "homeostasis": {
    "baseline_pad": [0.1, -0.2, 0.0],
    "sensitivity": 1.2
  },
  "current_state": "（由 Self-Reflection Engine 动态更新，初始为空）"
}
```

**参数说明：**

| 字段 | 说明 |
|------|------|
| `bio` | 固定身份锚点，不随性格演化改变 |
| `homeostasis.baseline_pad` | 情绪稳态坐标：[0.1, -0.2, 0.0] = 轻微愉悦、偏低激活、中性支配 |
| `homeostasis.sensitivity` | 情感敏感度，1.2 表示比默认值稍敏感，更易产生情绪波动 |
| `current_state` | 由 Self-Reflection Engine 写入，每次审视后更新 |

---

## 第五章：WebSocket 通信协议规范 v1.0

Hand、WebUI 与 SuiCore 三者之间所有实时通信走 **WebSocket + JSON**，无其他协议。

### 5.1 连接基础

#### 端点

```
ws://<host>:<port>/ws/<hand_name>
```

- `hand_name`：连接方唯一标识，如 `napcat_qq`、`sui_web`、`minecraft_bot`
- 同一 `hand_name` 重复连接时旧连接自动断开

#### 连接生命周期

```
Client                          Core
  │                               │
  │──── WS connect ──────────────>│  ① TCP 连接建立
  │<─── WS accept ───────────────│
  │                               │
  │──── HELLO ──────────────────>│  ② 握手：声明身份和能力
  │<─── WELCOME ─────────────────│  ③ 确认：返回 session_id
  │                               │
  │<─── ping ────────────────────│  ④ 心跳：Core 每 30s 发 ping
  │──── pong ───────────────────>│     Client 须在 10s 内回复
  │                               │
  │──── input/status/command ───>│  ⑤ 正常通信
  │<─── output/status/command ───│
  │                               │
  │──── status(offline) ────────>│  ⑥ 优雅断线
  │<─── WS close ────────────────│
```

Core 检测到连续 3 次 ping 未收到 pong → 强制断开并清理资源。

#### 握手消息

Hand 连接后**第一条消息必须是 HELLO**：

```json
{
  "type": "status",
  "from": "napcat_qq",
  "payload": {
    "action": "hello",
    "hand_type": "hand",
    "version": "1.0"
  }
}
```

Core 回复 WELCOME：

```json
{
  "type": "status",
  "from": "core",
  "to": "napcat_qq",
  "payload": {
    "action": "welcome",
    "session_id": "sess_abc123",
    "core_version": "0.2.0",
    "server_time": 1712345678
  }
}
```

### 5.2 消息格式

#### 通用字段

```json
{
  "type": "input | output | status | command | error",
  "from": "发送者名称",
  "to": "接收者名称 | null",
  "payload": { "action": "...", "...": "..." },
  "msg_id": "可选，消息唯一标识（UUID v4）",
  "reply_to": "可选，回复的目标 msg_id",
  "priority": "P3 | P2 | P1 | P0 | PPP",
  "timestamp": 1712345678,
  "version": "1.0"
}
```

**规则：**

- **所有消息 payload 必须有 `action` 字段**，缺少则返回 `ERR_VALIDATION`
- `from` 和 `to` 统一使用小写字母 + 下划线命名
- `msg_id` 推荐 UUID v4
- `timestamp` 为 Unix 秒级时间戳

#### 优先级详细说明

| 级别 | 含义 | 处理规则 | 典型场景 |
|------|------|----------|----------|
| PPP | 最高优先 | 强制插队，立即处理 | Admin 控制指令 |
| P0 | 紧急交互 | 尽量立即处理 | 电话、游戏、vtuber 直播 |
| P1 | 一般交互 | 正常处理 | 私聊 Chat、@ 消息 |
| P2 | Core 自调 | 空闲时处理 | Core 对消息主动降级 |
| P3 | 低优先 | 空闲时处理 | 群聊非 @ 消息 |

### 5.3 消息类型与 Payload

#### INPUT（Hand → Core）

| action | 方向 | 说明 |
|--------|------|------|
| `chat` | Hand→Core | 用户对话消息 |

**chat 示例：**

```json
{
  "type": "input",
  "from": "napcat_qq",
  "payload": {
    "action": "chat",
    "content": "今天天气真不错",
    "user_id": "qq_123456",
    "group_id": "group_789"
  },
  "msg_id": "msg_abc123",
  "priority": "P1",
  "timestamp": 1712345678,
  "version": "1.0"
}
```

#### OUTPUT（Core → Hand / WebUI）

| action | 方向 | 说明 |
|--------|------|------|
| `chat_reply` | Core→Hand | 对话回复 |
| `notify` | Core→Hand | 主动通知推送 |

**chat_reply 示例：**

```json
{
  "type": "output",
  "from": "core",
  "to": "napcat_qq",
  "payload": {
    "action": "chat_reply",
    "reply": "嗯嗯今天阳光特别好~",
    "emotion": {
      "P": 0.3,
      "A": 0.1,
      "D": -0.1,
      "label": "轻松愉快"
    },
    "topic_ended": false
  },
  "reply_to": "msg_abc123",
  "timestamp": 1712345680,
  "version": "1.0"
}
```

**notify 示例：**

```json
{
  "type": "output",
  "from": "core",
  "to": "napcat_qq",
  "payload": {
    "action": "notify",
    "content": "该吃药了哦",
    "level": "info"
  }
}
```

`level` 取值：`info` | `warning` | `urgent`

#### STATUS（双向）

| action | 方向 | 说明 |
|--------|------|------|
| `hello` | Hand/WebUI→Core | 连接握手 |
| `welcome` | Core→Hand/WebUI | 握手确认 |
| `hand_state` | Hand/WebUI→Core | 汇报自身状态 |
| `core_state` | Core→WebUI | 广播 Core 状态 |

**hand_state 示例：**

```json
{
  "type": "status",
  "from": "napcat_qq",
  "payload": {
    "action": "hand_state",
    "state": "online",
    "details": { "qq_bot_online": true, "groups": 5 }
  }
}
```

`state` 取值：`online` | `offline` | `busy` | `idle`

**core_state 示例（广播给所有 WebUI）：**

```json
{
  "type": "status",
  "from": "core",
  "payload": {
    "action": "core_state",
    "status": "idle",
    "emotion": {
      "P": 0.0,
      "A": -0.1,
      "D": -0.2,
      "label": "平静中性"
    },
    "connected_hands": ["napcat_qq", "sui_web"],
    "hand_count": 2
  }
}
```

#### COMMAND（WebUI ↔ Core）

WebUI 通过 command 管理 Core 运行状态和配置。

| action | 方向 | 说明 |
|--------|------|------|
| `config_get` | WebUI→Core | 读取配置项 |
| `config_set` | WebUI→Core | 修改配置项 |
| `character_get` | WebUI→Core | 读取角色档案 |
| `character_update` | WebUI→Core | 修改角色档案 |
| `memory_search` | WebUI→Core | 搜索记忆库 |
| `emotion_get` | WebUI→Core | 读取当前情绪 |
| `emotion_set` | WebUI→Core | 手动设置情绪值 |
| `hand_list` | WebUI→Core | 列出已连接 Hand |

**config_get / config_set：**

```json
// WebUI → Core
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "config_get", "keys": ["MODEL", "DECAY_LAMBDA"] },
  "msg_id": "cmd_001"
}

// Core → WebUI
{
  "type": "command",
  "to": "sui_web",
  "payload": {
    "action": "config_get",
    "result": { "MODEL": "deepseek-v4-flash", "DECAY_LAMBDA": 0.1 }
  },
  "reply_to": "cmd_001"
}
```

```json
// WebUI → Core
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "config_set", "config": { "DECAY_LAMBDA": 0.2 } },
  "msg_id": "cmd_002"
}

// Core → WebUI
{
  "type": "command",
  "to": "sui_web",
  "payload": { "action": "config_set", "success": true },
  "reply_to": "cmd_002"
}
```

**emotion_get / emotion_set：**

```json
// 读取当前情绪
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "emotion_get" },
  "msg_id": "cmd_005"
}

// Core 回复
{
  "type": "command",
  "to": "sui_web",
  "payload": {
    "action": "emotion_get",
    "result": { "P": 0.3, "A": 0.1, "D": -0.1, "label": "轻松愉快" }
  },
  "reply_to": "cmd_005"
}

// 手动设置情绪（调试/剧本模式）
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "emotion_set", "pad": [0.5, 0.3, 0.0] },
  "msg_id": "cmd_006"
}
```

**memory_search：**

```json
// WebUI → Core
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "memory_search", "query": "生日", "limit": 5 },
  "msg_id": "cmd_007"
}

// Core → WebUI
{
  "type": "command",
  "to": "sui_web",
  "payload": {
    "action": "memory_search",
    "results": [
      { "content": "用户说今年生日想去吃火锅", "strength": 0.8, "status": "active", "stored_at": 1712000000 },
      { "content": "去年生日用户独自过的", "strength": 0.3, "status": "fuzzy", "stored_at": 1680000000 }
    ]
  },
  "reply_to": "cmd_007"
}
```

**character_get / character_update：**

```json
// 读取角色档案
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "character_get" },
  "msg_id": "cmd_008"
}

// 修改角色档案（只传需要修改的字段）
{
  "type": "command",
  "from": "sui_web",
  "payload": {
    "action": "character_update",
    "changes": { "current_state": "最近变得更加敏感，需要更多认可..." }
  },
  "msg_id": "cmd_009"
}
```

**hand_list：**

```json
// WebUI → Core
{
  "type": "command",
  "from": "sui_web",
  "payload": { "action": "hand_list" },
  "msg_id": "cmd_010"
}

// Core → WebUI
{
  "type": "command",
  "to": "sui_web",
  "payload": {
    "action": "hand_list",
    "hands": [
      { "name": "napcat_qq", "type": "hand", "state": "online", "session_id": "sess_001" },
      { "name": "sui_web", "type": "webui", "state": "online", "session_id": "sess_002" }
    ]
  },
  "reply_to": "cmd_010"
}
```

#### ERROR（双向）

```json
{
  "type": "error",
  "from": "core",
  "to": "napcat_qq",
  "payload": {
    "code": "ERR_VALIDATION",
    "message": "payload 缺少 action 字段",
    "hint": "所有消息 payload 需要包含 action 字符串"
  },
  "reply_to": "msg_abc123"
}
```

### 5.4 通信流示例

#### 用户对话完整流

```
Hand (QQ)              Hand Manager           Core                  Emotion/Memory
  │                        │                    │                        │
  │── input(chat) ────────>│                    │                        │
  │                        │── 验证+路由 ───────>│                        │
  │                        │                    │── 查询情绪+记忆 ───────>│
  │                        │                    │<── 情绪状态+记忆上下文 ─│
  │                        │                    │── 组 Prompt + LLM ──   │
  │                        │                    │── 更新引擎状态 ────────>│
  │<── output(chat_reply) ─┤<── 下发回复 ────── │                        │
```

#### Skill 调用流

```
Core               Skill Manager          Skill             Core（二次）
  │                     │                   │                   │
  │── 调用请求 ─────────>│                   │                   │
  │                     │── execute() ──────>│                   │
  │                     │<── 结果字符串 ─────│                   │
  │<── 结果回调 ─────── │                   │                   │
  │── 携带结果再次 LLM ──────────────────────────────────────────>│
  │<── 最终回复 ─────────────────────────────────────────────── │
```

#### WebUI 管理操作

```
WebUI                  Core
  │                      │
  │── command(config_get) │
  │                      ├── 读取配置
  │<── command(result) ──│
  │                      │
  │── command(emotion_set)│
  │                      ├── 写入引擎
  │<── command(success) ─│
```

#### Core 状态广播

```
WebUI_1            Core               WebUI_2
  │                  │                    │
  │<── core_state ───┤                    │
  │                  │── core_state ─────>│
```

### 5.5 SDK 快速参考

#### TypeScript SDK

```typescript
import { SuiClient } from "./sui_sdk"

// Hand 接入
const client = new SuiClient("napcat_qq", { handType: "hand" })

client.onReply(async (payload) => {
  // payload.reply 是回复文本
  // payload.emotion 是当前情绪状态
  await sendQQMessage(payload.reply)
})

client.onNotify(async (payload) => {
  await sendQQMessage(`[通知] ${payload.content}`)
})

client.onError((payload) => {
  console.error("错误：", payload.code, payload.message)
})

await client.connect("ws://localhost:8000")

// 发送用户消息
const reply = await client.sendChat("你好呀", { userId: "qq_123456" })
console.log(reply.reply)
```

```typescript
// WebUI 接入
const admin = new SuiClient("sui_web", { handType: "webui" })
await admin.connect("ws://localhost:8000")

// 读取配置
const config = await admin.sendCommand("config_get", { keys: ["MODEL"] })
console.log(config.result)

// 修改情绪
await admin.sendCommand("emotion_set", { pad: [0.8, 0.5, 0.2] })

// 搜索记忆
const memories = await admin.sendCommand("memory_search", { query: "生日", limit: 5 })
console.log(memories.results)
```

#### Python SDK（兼容旧版 Hand）

```python
from sdk import SuiClient

client = SuiClient("napcat_qq", hand_type="hand")

@client.on_reply
async def handle_reply(reply_payload):
    await send_qq_message(reply_payload["reply"])

@client.on_notify
async def handle_notify(notify_payload):
    await send_qq_message(f"[通知] {notify_payload['content']}")

@client.on_error
async def handle_error(error_payload):
    log_error(error_payload)

await client.connect("ws://localhost:8000")
await client.wait_forever()
```

### 5.6 错误码表

| 错误码 | 含义 | 常见原因 |
|--------|------|----------|
| `ERR_PARSE` | JSON 解析失败 | 消息不是合法 JSON |
| `ERR_VALIDATION` | 格式校验失败 | 缺少 `action` 字段或必填字段 |
| `ERR_UNKNOWN_TYPE` | 未知的 `type` 值 | type 不在五种合法值内 |
| `ERR_UNKNOWN_ACTION` | 未知的 `action` 值 | payload.action 不在已定义列表内 |
| `ERR_NOT_FOUND` | 目标 Hand 不在线 | to 指向的 hand_name 未连接 |
| `ERR_INTERNAL` | Core 内部错误 | LLM 调用失败、引擎异常等 |

---

## 附录 A：变更历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0 | 2026-05-01 | 初始协议版本（MMVP 阶段，Python + WebSocket） |
| v2.0 | 2026-05-09 | 正式框架重构，迁移至 TypeScript + Bun；Core 职责剥离；引入 Manager 模式；完整协议规范；新增 UOP 编码规范 |

---

## 附录 B：快捷指令表

以下指令用于调试和开发阶段，通过命令行或 WebUI 发送：

| 指令 | 功能 |
|------|------|
| `/exit` | 退出并保存完整日志 |
| `/self-re` | 手动触发一次自我审视 |
| `/get_memory` | 将全部记忆导出到日志文件 |
| `/p_log` | 保存当前会话日志 |
| `/init` | 初始化：删除所有记忆和数据，恢复到初始状态（危险操作） |
| `/emotion` | 打印当前 PAD 值和情绪标签 |
| `/status` | 打印 Core 当前状态和已连接 Hand 列表 |

---

*SuiBot 正式框架文档 v2.0 · FLINTCORE 版权所有 · CC BY-NC-ND 4.0*
*夜来南风起，小麦覆陇黄。*

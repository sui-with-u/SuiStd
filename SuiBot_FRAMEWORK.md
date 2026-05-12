# SuiBot 正式框架文档 v3.4

> **SuiBot：基于 LLM 的拟人化智能体——正式框架概述**
> sui-with-u 穗穗碎碎念
> 文档版本：v3.4 · 2026.05

---

## 目录

- [引言：为什么要重构](#引言为什么要重构)
- [第一章：整体架构](#第一章整体架构)
  - [1.1 设计哲学](#11-设计哲学)
  - [1.2 技术栈](#12-技术栈)
  - [1.3 仓库结构](#13-仓库结构)
  - [1.4 五层架构概览](#14-五层架构概览节选完整流程见第二章)
  - [1.5 新版运行流程图](#15-新版运行流程图)
- [第二章：运行流程详解](#第二章运行流程详解)
  - [2.1 五步核心流程](#21-五步核心流程)
  - [2.2 半阻塞总线流说明](#22-半阻塞总线流说明)
  - [2.3 Tool 执行分支](#23-tool-执行分支)
- [第三章：模块规范](#第三章模块规范)
  - [3.1 Core 主控模块](#31-core-主控模块)
  - [3.2 Hand Manager](#32-hand-manager)
  - [3.3 Hand 接入规范](#33-hand-接入规范)
  - [3.4 Tool Manager](#34-tool-manager)
  - [3.5 Tool 编写规范](#35-tool-编写规范)
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
- [第六章：Vibe Coding 规范——AI 编码规范（修订版）](#第六章vibe-coding-规范ai-编码规范修订版)
  - [6.1 什么是 UOP](#61-什么是-uop)
  - [6.2 命名规范](#62-命名规范)
  - [6.3 注释规范](#63-注释规范)
  - [6.4 函数规范](#64-函数规范)
  - [6.5 文件与目录规范](#65-文件与目录规范)
  - [6.6 性能与视觉规范](#66-性能与视觉规范)
  - [6.7 复用规范](#67-复用规范)
  - [6.8 SuiBot 项目落地示例](#68-suibot-项目落地示例)
  - [6.9 给 AI 的提示词（直接复制使用）](#69-给-ai-的提示词直接复制使用)
- [附录 A：组织仓库总览](#附录-a组织仓库总览)
- [附录 B：变更历史](#附录-b变更历史)
- [附录 C：快捷指令表](#附录-c快捷指令表)

---

## 引言：为什么要重构

**原版的问题：**

- Core 太大，既计算情感值、又写数据库、又调用 LLM、又管理 Tool——职责混乱，极难扩展
- 模块串行耦合过深，任何一处出错会卡死整条链路，排查困难
- 没有统一的中心调度，Hands、Tool、引擎各自为政，缺乏插拔机制
- Hand 没有统一管理器，无法横向扩展到 QQ 以外的平台

**重构目标：**

- Core 只做五件事：接收 → 查引擎 → 组 Prompt → 调 LLM → 决策分发
- 情感、记忆、性格、优先级全部剥离为独立引擎，被 Core 调用而非内嵌
- Tool 与 Hand 均由专属 Manager 统一管理，支持按需拉取与热插拔
- 整个系统像「人」一样一次只做一件事（半阻塞总线流）

---

## 第一章：整体架构

### 1.1 设计哲学

SuiBot v3 采用**类冯·诺依曼半阻塞总线流**架构。

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
| 前端框架 | React + Vite | 管理界面（WebUI / DevWeb） |
| UI 组件库 | ShadCN UI | 轻量、可定制 |
| 图表库 | Recharts | 情感曲线可视化 |
| 向量数据库 | ChromaDB | 长期记忆存储与检索 |
| 短期记忆 | 内存 Buffer（Map） | 最近 5~10 轮对话 |
| 通信协议 | WebSocket + JSON | 全双工，所有模块统一接入 |
| LLM 接入 | DeepSeek API（可换） | 便宜好用，支持国产 |
| 仓库方案 | 多仓库（主仓库 + 独立 Hand/Tool 仓库） | 主仓库自带 Manager，按需拉取外部仓库 |

> **为什么从 Python 迁移到 TypeScript？**
> MMVP 用 Python 快速验证了方向。正式版考虑到生产环境长期运营的性能需求，以及前后端语言统一的工程便利，切换到 TypeScript + Bun。Python 原型代码作为算法参考保留在 `/legacy` 目录。

### 1.3 仓库结构

v3 采用**多仓库**方案。主仓库包含 Core、Engines、Manager 以及内置的基础组件，独立 Hand / Tool 仓库按需拉取部署。

#### 主仓库 `suibot`

一般部署只需要克隆主仓库，它包含完整的运行时核心：

```
suibot/                             ← 主仓库，标准部署的全部内容
├── package.json
├── bunfig.toml
├── tsconfig.json
├── claude.md                       # AI 编码规范
├── README.md
│
├── core/                           # Core 主控
│   ├── index.ts
│   ├── message-handler.ts          # 五步主流程
│   └── prompt-builder.ts           # System Prompt 组装
│
├── engines/                        # 引擎层，Core 直接调用，无 Manager
│   ├── emotion-engine.ts           # PAD 三维情感：衰减 + 波动
│   ├── memory-engine.ts            # 短期 Buffer + 长期 ChromaDB
│   ├── reflect-engine.ts           # 定时自我审视 → 更新 System Prompt
│   └── priority-engine.ts          # 优先级队列 + 消息路由规则
│
├── tools/
│   ├── tool-manager/               # Tool Manager（管理工具的注册、调度、拉取）
│   │   ├── index.ts
│   │   ├── registry.ts             # 已注册 Tool 列表与元数据
│   │   ├── deployer.ts             # 拉取/安装/启停外部 Tool 仓库
│   │   └── ...
│   └── tools/                      # 拉取到的 Tool 落地目录
│       ├── T-TTS/                  # 示例：TTS Tool（从 T-TTS 仓库拉取）
│       ├── T-Weather/
│       └── ...                     # 其他外部 Tool（.gitignore 排除）
│
├── hands/
│   ├── hand-manager/               # Hand Manager（管理连接、调度、拉取）
│   │   ├── index.ts
│   │   ├── registry.ts             # 已连接 Hand 列表与元数据
│   │   ├── deployer.ts             # 拉取/安装/启停外部 Hand 仓库
│   │   └── ...
│   └── hands/                      # 拉取到的 Hand 落地目录
│       ├── H-SuiWeb/               # 示例：管理面板（从 H-SuiWeb 仓库拉取）
│       ├── H-SuiDevWeb/
│       └── ...                     # 其他外部 Hand（.gitignore 排除）
│
├── types/                          # 全局类型定义
│   └── index.ts
│
└── utils/                          # 通用工具
    ├── index.ts
    ├── math.ts
    ├── async.ts
    └── id.ts
```

> `tools/tools/` 和 `hands/hands/` 是**运行时落地目录**，由各自的 Manager 在部署时自动填充，内容被 `.gitignore` 排除，不进入主仓库版本控制。

#### 独立仓库命名规范

| 前缀 | 类型 | 示例 |
|------|------|------|
| `H-` | Hand | `H-SuiWeb`、`H-SuiDevWeb`、`H-OneBot`、`H-Telegram`、`H-Minecraft` |
| `T-` | Tool | `T-TTS`、`T-Weather`、`T-Calendar` |

每个独立仓库包含自己的 `package.json`、`index.ts` 入口，以及一个 `sui.config.json` 描述文件，供 Manager 识别和注册：

```json
// sui.config.json（每个独立 Hand / Tool 仓库必须有）
{
  "name": "H-SuiWeb",
  "type": "hand",
  "version": "1.0.0",
  "entry": "index.ts",
  "description": "SuiBot 管理面板，情感曲线可视化与参数配置"
}
```

#### 部署流程

**标准部署（只需主仓库，无 WebUI）：**

```bash
git clone https://github.com/sui-with-u/suibot
cd suibot && bun install && bun run start
```

**按需拉取 / 启停外部 Hand 或 Tool：**

所有操作均通过 Core 中转，再由对应 Manager 执行，WebUI 本身也是一个需要安装的 Hand：

```
CLI 或已安装的 WebUI
        │
        ▼  command(hand_install / tool_install / start / stop)
      Core
        │
        ├──► Hand Manager   管理 hands/hands/ 下的外部 Hand 仓库
        │
        └──► Tool Manager   管理 tools/tools/ 下的外部 Tool 仓库
```

**拉取流程（Manager 内部执行）：**

```
① Core 收到 command(hand_install, { name: "H-SuiWeb" })
② Core 转发给 Hand Manager
③ Hand Manager 在 github.com/sui-with-u 下搜索同名仓库
④ git clone 到 hands/hands/H-SuiWeb/
⑤ bun install（子目录依赖安装）
⑥ 读取 sui.config.json，注册到 Manager 注册表
⑦ 启动进程，Hand 建立 WebSocket 连接
```

**启停流程：**

```
① CLI / WebUI 发送 command(hand_start / hand_stop, { name: "H-SuiWeb" })
② Core 转发给 Hand Manager
③ Hand Manager 启动或终止对应进程
```

**命令行参考：**

```bash
bun run sui add H-SuiWeb       # 拉取并安装
bun run sui start H-SuiWeb     # 启动
bun run sui stop H-SuiWeb      # 停止
bun run sui add T-TTS          # 拉取 Tool 同理
```

### 1.4 五层架构概览（节选，完整流程见第二章）

```
┌──────────────────────────────────────────────────────────────┐
│                       外部层 HANDS                            │
│   独立仓库（H-*），部署后落地在 hands/hands/，各自独立运行    │
│                                                              │
│  H-SuiWeb    H-SuiDevWeb    H-OneBot    H-Telegram    ...   │
└───────────────────────────┬──────────────────────────────────┘
                            │ WebSocket
┌───────────────────────────▼──────────────────────────────────┐
│              接入层 HAND MANAGER（hands/hand-manager/）        │
│   WS 连接生命周期 · 调度委托 Priority Engine                  │
│   经 Core 指令触发外部 Hand 仓库的拉取 / 注册 / 启停          │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                    核心层 SUICORE（core/）                     │
│      接收 → 查引擎 → 组 Prompt → 调 LLM → 决策分发           │
└──────┬──────────────┬─────────────┬────────────┬─────────────┘
       │              │             │            │ Core 直接调用，无 Manager
       ▼              ▼             ▼            ▼
  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Emotion │  │  Memory  │  │ Reflect  │  │ Priority │
  │ Engine  │  │  Engine  │  │ Engine   │  │ Engine   │
  │PAD情感  │  │Buffer+VDB│  │审视→Prompt│  │优先级路由│
  └─────────┘  └──────────┘  └──────────┘  └──────────┘
       ↑ 引擎层 ENGINES（engines/，Core 直接调用）
┌──────────────────────────────────────────────────────────────┐
│                       扩展层 TOOLS                            │
│   tools/tool-manager/  经 Core 指令触发外部 Tool 仓库的拉取/启停│
│   tools/tools/         外部 Tool 落地目录（T-TTS、T-Weather…）│
└──────────────────────────────────────────────────────────────┘
```

**各层职责一览：**

| 层级 | 位置 | 职责 | 可扩展 |
|------|------|------|--------|
| 外部层 HANDS | `hands/hands/H-*/`（独立仓库） | 各平台消息收发适配器，独立进程运行 | ✅ 按需拉取 |
| 接入层 HAND MANAGER | `hands/hand-manager/` | WS 连接管理；经 Core 指令触发外部 Hand 仓库的拉取/注册/启停 | ❌ 主仓库固定 |
| 核心层 SUICORE | `core/` | 调度、Prompt 组装、LLM 调用、决策 | ❌ 系统大脑 |
| 引擎层 ENGINES | `engines/` | 情感、记忆、审视、优先级路由，Core 直接调用 | ❌ 核心固定组件 |
| 扩展层 TOOLS | `tools/tool-manager/` + `tools/tools/T-*/` | 经 Core 指令触发外部 Tool 仓库的拉取/注册/启停 | ✅ 按需拉取 |

### 1.5 新版运行流程图

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
                    需要 Tool？NO                              YES  │
                              │                                     ▼
                              │                       ┌────────────────────┐
                              │                       │   TOOL MANAGER    │
                              │                       │ 调度对应 Tool 执行 │
                              │                       └────────┬───────────┘
                              │                                │ 结果回调 Core
                              │                                ▼
                              │                       ┌────────────────────┐
                              │                       │  Core 二次调用 LLM │
                              │                       │  整合 Tool 结果   │
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
- `tool_call`（可选）：需要调用的 Tool 名称与参数
- `emotion_delta`（可选）：对情绪的影响向量（LLM 判断本次对话的情感倾向）

#### 步骤五：决策 & 输出

**分支 A——无需 Tool：**

1. 触发引擎写入：
   - Emotion Engine 根据 `emotion_delta` 更新 PAD 值
   - Memory Engine 将本次对话存入 Buffer，必要时向 VectorDB 写入长期记忆
2. 通过 Hand Manager 将 `output(chat_reply)` 下发给对应 Hand

**分支 B——需要调用 Tool：**

1. Core 将 `tool_call` 参数转发给 Tool Manager
2. Tool Manager 调度对应 Tool 执行，等待结果
3. Tool 结果回传 Core
4. Core 携带 Tool 结果再次调用 LLM，生成最终自然语言回复
5. 执行与分支 A 相同的引擎写入 + 下发流程

### 2.2 半阻塞总线流说明

Core 在以下阶段会阻塞等待：

| 阻塞点 | 等待对象 | 超时处理 |
|--------|----------|----------|
| 步骤二 | Emotion Engine + Memory Engine 均返回 | 超时用默认值继续（不中断对话） |
| 步骤四 | LLM API 响应 | 超时返回固定错误提示，不重试 |
| 步骤五（分支B） | Tool 执行完成 | 超时返回「技能执行失败」提示 |

阻塞期间，新进入队列的消息会等待当前轮次处理完毕。PPP 级消息可打断（强制插队）。

### 2.3 Tool 执行分支

LLM 通过 Tool Calling 模式决定是否调用 Tool。Core 解析 LLM 返回的 `tool_call` 字段：

```typescript
// LLM 返回示例
{
  reply: "好的，我来帮你查一下天气！",
  tool_call: {
    tool_name: "weather_search",
    params: { city: "岳阳", date: "today" }
  }
}
```

Tool Manager 收到调用请求后：

1. 查找已注册的 `weather_search` Tool
2. 执行 `tool.execute({ city: "岳阳", date: "today" })`
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
- 解析 LLM 返回，决定是否触发 Tool
- 触发引擎状态更新
- 将回复通过 Hand Manager 下发

**明确不做的事：**

- ❌ 不直接操作数据库（交给 Memory Engine）
- ❌ 不计算情感数值（交给 Emotion Engine）
- ❌ 不执行 Tool 业务逻辑（交给 Tool Manager）
- ❌ 不管理 WebSocket 连接（交给 Hand Manager）

**对外接口：**

Core 不直接对外暴露接口，所有通信通过 Hand Manager 中转。内部提供以下方法供引擎层回调：

```typescript
core.onEngineReady(emotionState, memoryContext)   // 引擎数据就绪
core.onToolResult(toolName, result)              // Tool 执行完毕
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

**优先级队列规则（由 Priority Engine 提供，Hand Manager 委托调用）：**

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

### 3.4 Tool Manager

**职责：**

- 维护已注册 Tool 的列表与元数据
- 接收 Core 的调用请求，找到对应 Tool 并执行
- 执行超时处理，结果回调 Core
- 支持热插拔（运行时注册/注销 Tool，无需重启 Core）
- 经 Core 指令触发后管理外部 Tool 仓库：拉取（git clone）、依赖安装、读取 `sui.config.json` 注册、启停进程
- 允许自行处理简单的调度层事务（超时重试、结果缓存），无需经过 Core

**Tool 注册方式：**

```typescript
toolManager.register(new WeatherTool())
toolManager.register(new CalendarTool())
toolManager.unregister("weather_search")   // 热卸载
```

**LLM 如何知道有哪些 Tool？**

Core 在组装 System Prompt 时，会将所有已注册 Tool 的 `name` 和 `description` 注入 Prompt，LLM 根据 description 自主决定是否调用。这是标准的 Tool Calling 模式。

### 3.5 Tool 编写规范

每个 Tool 是一个独立的 TypeScript 类，实现以下接口：

```typescript
interface SuiTool {
  name: string               // 技能唯一标识，如 "weather_search"
  description: string        // 告诉 LLM 什么时候调用这个技能
  paramsSchema: object       // JSON Schema，描述参数格式
  execute(params: object): Promise<string>   // 执行并返回结果字符串
}
```

**示例：天气查询 Tool**

```typescript
// tools/weatherSearch.ts —— 这个文件只负责查天气

export class WeatherTool implements SuiTool {

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

**Tool 编写规则：**

- `execute` 只返回字符串，由 Core 交给 LLM 整合为自然语言，Tool 不负责措辞
- `execute` 内部必须有 Early Return 处理参数缺失情况
- 一个 Tool 文件只做一件事，文件名 = Tool 功能

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

#### Tool 调用流

```
Core               Tool Manager          Tool             Core（二次）
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

## 第六章：Vibe Coding 规范——AI 编码规范（修订版）

> 本章规范适用于所有参与 SuiBot 开发的成员，无论是自己写代码还是用 AI 生成代码（Vibe Coding）。
> 基于 UOP（面向理解编程）修订，结合 SuiBot 项目实际情况调整。
> UOP 原始规范：[github.com/kernel4632/UOP](https://github.com/kernel4632/UOP)
> 独立规范文件：见项目根目录 `claude.md`

### 6.1 什么是 UOP

**UOP（Understanding-Oriented Programming，面向理解编程）** 把「任意背景的人能快速读懂」作为代码的第一优先级。

这不是要求代码写得烂。恰恰相反——真正成熟的 UOP 追求更高的长期可维护性、更强的复用性、更低的整体心智负担。它把「理解成本」放在首位，但其余工程指标在这个前提下优化，而不是被牺牲。

**为什么 SuiBot 要用 UOP？**

SuiBot 是高中生研究性学习项目，团队成员编程基础参差不齐。项目大量使用 AI（Cursor、Trae 等）辅助开发，AI 生成的代码专业但对新手难以 review。UOP 确保：

- 组员都能看懂核心代码在做什么
- AI 生成代码符合统一风格
- 半年后回来改代码，不需要从头理解

**SuiBot 对原版 UOP 的修订：**

| 维度 | 原版 UOP | SuiBot 修订版 |
|------|----------|--------------|
| 命名 | 理解优先，可放宽规范 | 理解优先，**规范强制**，不可因「好理解」破坏大小写规则 |
| 注释 | 所有紧凑写法都加注释 | 必要位置必须写，普通清晰代码不强制 |
| 视觉 | 不提及 | 少量性能可换视觉，**性能 > 视觉** |
| 文件组织 | 平铺为主，最多两层 | 遵循 TypeScript 社区标准，不强制平铺 |
| 复用 | 出现两次再提取 | 遵循标准工程规范，可预见复用提前提取 |

---

### 6.2 命名规范

#### 大小写规则（强制，不可违反）

遵循 TypeScript / JavaScript 社区标准：

| 场景 | 规则 | 示例 |
|------|------|------|
| 变量、函数 | camelCase（小驼峰） | `currentPAD`、`applyDelta`、`getUserInfo` |
| 类、接口、类型 | PascalCase（大驼峰） | `EmotionEngine`、`HandManager`、`PADVector` |
| 模块级常量 | SCREAMING_SNAKE_CASE | `DECAY_LAMBDA`、`MAX_RETRY` |
| 文件名 | kebab-case | `emotion-engine.ts`、`hand-manager.ts` |
| 文件夹名 | kebab-case | `core-engines/`、`tool-manager/` |

> **命名要符合直觉，但规范优先。** 不能因为「更好理解」就用 `emotionengine` 代替 `EmotionEngine`，规范本身就是直觉的一部分。

#### 函数命名

用「动词 + 名词」，一眼知道在做什么：

```typescript
// ✅
sendReply()
buildSystemPrompt()
applyEmotionDelta()
getMemoryContext()
checkConnectionAlive()

// ❌ 动词不明确
emotion()
prompt()
memory()
```

首选动词：`get / set / add / remove / update / build / send / check / apply / load / save / init`

#### 变量命名

布尔变量写成疑问句形式：

```typescript
isConnected    isProcessing    hasPermission    canRetry
```

用完整常见英语单词，`id`、`api`、`url`、`PAD` 等约定俗成的缩写保留：

```typescript
// ✅
const manager = new HandManager()
const result = await callLLM(prompt)

// ❌
const mgr = new HandManager()
const res = await callLLM(pmt)
```

---

### 6.3 注释规范

注释不用每一行都写，但以下情况**必须写**：

**1. 非显而易见的业务逻辑**

```typescript
// PAD 三个维度都要 clamp，情绪值不能超出 [-1, 1]，否则后续衰减计算会溢出
currentPAD.P = clamp(currentPAD.P + delta.P * sensitivity, -1, 1)
```

**2. 魔法数字和魔法字符串**

```typescript
const DECAY_LAMBDA = 0.1          // 情绪衰减系数，0.1 约等于 1 小时后情绪减半
const FUZZY_THRESHOLD = 0.2       // 记忆保持概率低于此值时进入「模糊状态」
const SIMILARITY_THRESHOLD = 0.85 // 余弦相似度超过此值认为是相似记忆，触发混淆
```

**3. 坑点注释——看起来奇怪但有原因的代码**

```typescript
await delay(200)    // LLM API 有限流，连续请求必须间隔，去掉会概率性 429
connection.close()  // 同一 hand_name 重复连接必须关掉旧的，否则两边都会收到重复消息
```

**4. 复杂公式或算法**

```typescript
// 艾宾浩斯遗忘曲线：P = e^(-t/S)，t 为存储时长（小时），S 为记忆强度 [0,1]
const retentionProb = Math.exp(-hoursPassed / memory.strength)

// PAD 情绪指数衰减：向 baseline 靠拢，factor = e^(-λt)
const factor = Math.exp(-DECAY_LAMBDA * minutesPassed)
```

**5. 文件头注释（每个文件必须有）**

```typescript
// emotion-engine.ts
// 负责穗穗的 PAD 三维情感状态管理：读取、更新（对话触发）、衰减（定时任务）
// 对外导出：getCurrentState() / applyDelta() / setPAD() / decay()
```

**较长函数用段落注释分区：**

```typescript
async function handleMessage(message: IncomingMessage) {

  // —— 参数校验 ——
  if (!message.payload?.action) return sendError("ERR_VALIDATION", message)
  const { content, userId } = message.payload

  // —— 查询引擎状态 ——
  const emotion = await emotionEngine.getCurrentState()
  const memory = await memoryEngine.getContext(content)

  // —— 调用 LLM ——
  const prompt = buildSystemPrompt(emotion, memory)
  const llmResult = await callLLM(prompt, content)

  // —— 更新状态并下发回复 ——
  emotionEngine.applyDelta(llmResult.emotionDelta)
  memoryEngine.writeShortTerm({ content, reply: llmResult.reply, userId })
  sendReply(userId, llmResult.reply, emotion)
}
```

---

### 6.4 函数规范

每个函数聚焦单一职责，长度以 **4–15 行**为甜区。

用**卫语句（Early Return）**把异常情况先处理掉，主逻辑写在最后，缩进保持在两层以内：

```typescript
// ✅ 卫语句版本
function sendChat(content: string, userId: string) {
  if (!content) return
  if (!userId) return
  if (!isConnected) {
    console.warn("未连接 Core，消息丢弃")
    return
  }

  ws.send(buildChatMessage(content, userId))
}

// ❌ 嵌套版本
function sendChat(content: string, userId: string) {
  if (content) {
    if (userId) {
      if (isConnected) {
        ws.send(buildChatMessage(content, userId))
      }
    }
  }
}
```

---

### 6.5 文件与目录规范

文件组织**遵循 TypeScript 社区标准规范**，按功能模块分层：

```
suicore/
├── core/
│   ├── index.ts              // Core 主控入口
│   ├── message-handler.ts    // 消息处理逻辑
│   └── prompt-builder.ts     // System Prompt 组装
├── engines/
│   ├── emotion-engine.ts     // PAD 情感引擎
│   ├── memory-engine.ts      // 记忆管理
│   └── reflect-engine.ts     // 自我审视引擎
├── managers/
│   ├── hand-manager.ts       // Hand 连接管理
│   └── tool-manager.ts      // Tool 调度管理
├── tools/
│   ├── weather-search.ts
│   └── calendar.ts
├── types/
│   └── index.ts              // 全局类型定义
└── utils/
    └── index.ts              // 通用工具函数
```

文件名 kebab-case，名称直接说明内容，不用缩写。

---

### 6.6 性能与视觉规范

**性能优先级高于视觉，这是不可妥协的原则。**

#### 允许的「少量性能换视觉」

```typescript
// ✅ 日志可读性提升明显，且只在非热路径执行
console.log(`情绪：P=${pad.P.toFixed(2)}, A=${pad.A.toFixed(2)}, D=${pad.D.toFixed(2)}`)

// ✅ 调试日志格式化输出
console.log(JSON.stringify(state, null, 2))
```

#### 禁止的「为视觉牺牲性能」

```typescript
// ❌ 热路径里深拷贝只为日志好看
const snapshot = JSON.parse(JSON.stringify(currentPAD))

// ❌ 每次消息触发全量重渲染
messages.forEach(m => reRenderAll(m))

// ❌ 在 LLM 路径上做无人查看的格式化
const prettyPrompt = formatPromptForDisplay(prompt)
await callLLM(prettyPrompt)
```

#### SuiBot 热路径（严禁非必要操作）

- `handleMessage()` 主流程
- `buildSystemPrompt()` 组装过程
- `callLLM()` 调用链路
- `emotionEngine.applyDelta()` 情绪更新

热路径以外（日志、WebUI 渲染、调试命令）可适当用性能换可读性。

---

### 6.7 复用规范

**遵循标准工程规范：**

- 能预见到会被多处调用的工具函数，提前提取到 `utils/index.ts`
- 同一模块内部复用的逻辑，提取为私有函数（不导出）
- 跨模块复用的类型定义，统一放在 `types/index.ts`
- 业务逻辑不做「万能函数」，一个函数解决一类问题

```typescript
// utils/index.ts —— 项目通用工具

// 把数值限制在 [min, max] 范围内，PAD 计算、UI 进度条等多处会用到
export function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max)
}

// 等待指定毫秒，用于限流控制
export function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

// 生成 UUID v4，用于 msg_id 生成
export function generateId(): string {
  return crypto.randomUUID()
}
```

---

### 6.8 SuiBot 项目落地示例

以 `emotion-engine.ts` 为例，展示规范在项目中的实际应用：

```typescript
// emotion-engine.ts
// 负责穗穗的 PAD 三维情感状态管理：读取、更新（对话触发）、衰减（定时任务）
// 对外导出：getCurrentState() / applyDelta() / setPAD() / decay()

import { PADVector, EmotionState } from "../types"
import { clamp } from "../utils"

// 情绪衰减系数，0.1 约等于 1 小时后情绪减半，可通过 config_set 动态调整
const DECAY_LAMBDA = 0.1
// 情感敏感度：1.2 表示穗穗比默认值稍敏感，更易产生情绪波动
const SENSITIVITY = 1.2
// 情绪稳态，来自 character.json，情绪衰减后趋向此值
const BASELINE_PAD: PADVector = { P: 0.1, A: -0.2, D: 0.0 }

let currentPAD: PADVector = { ...BASELINE_PAD }


export function getCurrentState(): EmotionState {
  return {
    pad: { ...currentPAD },
    label: getEmotionLabel(currentPAD),
  }
}


export function applyDelta(delta: PADVector): void {
  // 乘以敏感度再叠加，clamp 确保不超出 [-1, 1]，否则衰减计算会溢出
  currentPAD.P = clamp(currentPAD.P + delta.P * SENSITIVITY, -1, 1)
  currentPAD.A = clamp(currentPAD.A + delta.A * SENSITIVITY, -1, 1)
  currentPAD.D = clamp(currentPAD.D + delta.D * SENSITIVITY, -1, 1)
}


export function setPAD(pad: PADVector): void {
  currentPAD = {
    P: clamp(pad.P, -1, 1),
    A: clamp(pad.A, -1, 1),
    D: clamp(pad.D, -1, 1),
  }
}


export function decay(minutesPassed: number): void {
  // 指数衰减公式：E_new = baseline + (E_current - baseline) * e^(-λt)
  const factor = Math.exp(-DECAY_LAMBDA * minutesPassed)
  currentPAD.P = BASELINE_PAD.P + (currentPAD.P - BASELINE_PAD.P) * factor
  currentPAD.A = BASELINE_PAD.A + (currentPAD.A - BASELINE_PAD.A) * factor
  currentPAD.D = BASELINE_PAD.D + (currentPAD.D - BASELINE_PAD.D) * factor
}


// 把 PAD 数值映射为可读的中文标签，用于日志和 WebUI 展示
function getEmotionLabel(pad: PADVector): string {
  if (pad.P > 0.5 && pad.A > 0.3)  return "兴奋愉快"
  if (pad.P > 0.2 && pad.A <= 0.3) return "轻松愉快"
  if (pad.P < -0.3 && pad.A > 0.3) return "焦虑不安"
  if (pad.P < -0.3 && pad.A <= 0)  return "低落沉闷"
  return "平静中性"
}
```

---

### 6.9 给 AI 的提示词（直接复制使用）

将以下内容复制到 Cursor / Trae / Roo Code 的「规则」或「自定义指令」中，与项目根目录的 `claude.md` 配合使用：

```
# SuiBot 项目编码规范

- 变量和函数：camelCase（小驼峰），如 currentPAD、applyDelta
- 类、接口、类型：PascalCase（大驼峰），如 EmotionEngine、PADVector
- 模块级常量：SCREAMING_SNAKE_CASE，如 DECAY_LAMBDA、MAX_RETRY
- 文件名：kebab-case，如 emotion-engine.ts、hand-manager.ts
- 命名要直觉，但规范优先，不能为了「好理解」破坏大小写规则

- 不用每一行都写注释，以下情况必须写：
  1. 非显而易见的业务逻辑
  2. 魔法数字和魔法字符串，必须注释说明含义和取值理由
  3. 看起来奇怪但有原因的代码（限流延迟、特殊顺序等），注释说明删掉会发生什么
  4. 复杂公式和算法，注释说明公式来源和参数含义
  5. 每个文件头部必须有注释说明这个文件负责什么、对外暴露哪些接口
- 较长函数用「// —— 段落名 ——」做区块分隔，方便扫读
- 禁止写重复代码字面意思的废话注释

- 单一职责，4–15 行为甜区，超出考虑拆分
- 用卫语句（Early Return）把所有异常先处理，主逻辑写在最后
- 缩进控制在两层以内

- 按功能模块分层，遵循 TypeScript 社区标准目录结构
- 文件名 kebab-case，名称直接说明内容
- 类型定义集中在 types/index.ts，通用工具集中在 utils/index.ts

- 性能优先级高于视觉，不可妥协
- 可以在非热路径（日志、调试、WebUI 渲染）用少量性能换可读性
- 禁止在热路径（handleMessage、buildSystemPrompt、callLLM、applyDelta）做任何非必要计算或 I/O
- 禁止为视觉效果在热路径做深拷贝、全量重渲染、非必要字符串格式化

- 遵循标准工程规范：能预见多处使用的工具函数，提前提取到 utils/
- 跨模块复用的类型统一放 types/
- 业务逻辑不做万能函数，一个函数解决一类问题
```

## 附录 A：组织仓库总览

| 仓库 | 类型 | 说明 |
|------|------|------|
| [suibot](https://github.com/sui-with-u/suibot) | 主仓库 | Core + Engines + Manager，标准部署的全部内容 |
| [SuiStd](https://github.com/sui-with-u/SuiStd) | 规范仓库 | 项目编码规范、最终愿景与开发路径 |
| [SuiFw](https://github.com/sui-with-u/SuiFw) | 脚手架 | Bun + TypeScript 7 后端空项目模板，供团队学习与快速开始 |
| [.github](https://github.com/sui-with-u/.github) | 组织主页 | 组织 README、Issue 模板、许可证等 |
| [H-SuiWeb](https://github.com/sui-with-u/H-SuiWeb) | Hand | 管理面板，情感曲线可视化、记忆查询、参数配置 |
| [H-SuiDevWeb](https://github.com/sui-with-u/H-SuiDevWeb) | Hand | 开发调试面板 |
| [H-OneBot](https://github.com/sui-with-u/H-OneBot) | Hand | QQ 双向消息（NapCat） |
| [H-Telegram](https://github.com/sui-with-u/H-Telegram) | Hand | Telegram 机器人 |
| [H-Minecraft](https://github.com/sui-with-u/H-Minecraft) | Hand | Minecraft 聊天 / 指令 / 自动化 |
| [T-TTS](https://github.com/sui-with-u/T-TTS) | Tool | 语音合成（Text-to-Speech） |
| [T-Weather](https://github.com/sui-with-u/T-Weather) | Tool | 天气查询 |
| [T-Calendar](https://github.com/sui-with-u/T-Calendar) | Tool | 日历与提醒 |

> `H-*` 和 `T-*` 仓库通过 `bun run sui add <name>` 或 WebUI 扩展中心安装，所有安装/启停指令均须经 Core 中转后由对应 Manager 执行。

## 附录 B：变更历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0 | 2026-05-05 | 初始协议版本（MMVP 阶段，Python + WebSocket） |
| v2.0 | 2026-05-09 | 正式框架重构，迁移至 TypeScript + Bun；Core 职责剥离；引入 Manager 模式；完整协议规范；新增 UOP 编码规范 |
| v3.0 | 2026-05-11 | 迁移至 Bun Workspaces Monorepo；目录结构全面更新；tools/ = Tool 扩展，engines/ = 情感/记忆/审视引擎；所有 Hand 纳入单仓管理；types/ 与 utils/ 跨包共享 |
| v3.1 | 2026-05-11 | Hand Manager 和 Tool Manager 改为 workspace 子包，整体可热插拔；Priority Engine 从 Hand Manager 下沉至 engines/；Engine Manager 不引入，Core 直接调用引擎 |
| v3.2 | 2026-05-12 | 回归多仓库方案；Hand/Tool 独立仓库按需拉取，落地在 hands/hands/ 和 tools/tools/；Manager 各自独立管理外部仓库的拉取/注册/启停；sui.config.json 规范引入；命令行与 WebUI 双入口部署 |
| v3.3 | 2026-05-12 | 修正部署流程为 CLI/WebUI→Core→Manager→操作；WebUI 改为外部 Hand，不内置；引言去除故事背景，直接列问题与目标；版权署名统一改为 sui-with-u 组织 |
| v3.4 | 2026-05-12 | 明确所有 Manager 操作须经 Core 指令触发，Manager 只允许自行处理简单的本层事务；新增 SuiStd、SuiFw、.github 仓库说明 |

---

## 附录 C：快捷指令表

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

*SuiBot 正式框架文档 v3.4 · Sui-with-u 版权所有 · CC BY-NC-ND 4.0*
*夜来南风起，小麦覆陇黄。*

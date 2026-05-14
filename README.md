# SuiBot 项目编码规范

> 适用于所有 AI 辅助开发（Cursor、Trae、Roo Code 等 Vibe Coding 工具）。
> 基于 UOP（面向理解编程）与 HOP（面向人类编程）整合修订，结合 SuiBot 项目实际。
> 与项目框架文档 `SuiBot_FRAMEWORK.md` 配合使用。

---

## 核心原则

**让刚接触项目的人只看局部代码就能理解、上手、修改。**

代码即文档，文件头就是使用手册，注释就是教学材料，命名就是说明书。

优先级排序：**可读性 > 性能 > 视觉美观**（热路径除外，热路径性能优先）。

---

## 架构思想：触发 → 指令 → 数据 → 反馈

所有交互和业务流程按这条链条组织，让数据流动清晰可追踪：

```
外部触发（CLI / WebUI / Hand 消息）
    ↓
Core 接收并路由（触发层，只判断谁触发了什么）
    ↓
引擎 / Manager 执行具体业务（指令层，完成一次清楚的业务动作）
    ↓
状态数据被修改（数据层，显式读写，可追踪）
    ↓
回复下发 / 状态广播（反馈层，把结果交回调用者）
```

**SuiBot 各层对应：**

| 链条环节 | SuiBot 对应位置 |
|----------|----------------|
| 触发 | Hand 发送消息 / CLI 命令 / WebUI 指令 |
| 指令执行 | `core/message-handler.ts` 五步主流程 |
| 数据修改 | `engines/` 各引擎的状态写入 |
| 效果反馈 | Hand Manager 将回复下发给对应 Hand |

任何人看到一个事件触发，都能顺着链条找到：调用了哪个方法 → 读写了哪些数据 → 最终影响了什么。

---

## 命名规范（强制）

### 大小写规则

遵循 TypeScript / JavaScript 社区标准：

| 场景 | 规则 | 示例 |
|------|------|------|
| 变量、函数 | camelCase（小驼峰） | `currentPAD`、`applyDelta` |
| 类、接口、类型 | PascalCase（大驼峰） | `EmotionEngine`、`PADVector` |
| 模块级常量 | SCREAMING_SNAKE_CASE | `DECAY_LAMBDA`、`MAX_RETRY` |
| 文件名 | kebab-case | `emotion-engine.ts`、`hand-manager.ts` |
| 文件夹名 | kebab-case | `core-engines/`、`hand-manager/` |

### 缩写规则（HOP 规范，强制）

约定俗成的缩写必须作为整体保留，**禁止写成首字母大写其余小写的假单词**：

| 场景 | 正确 | 错误 |
|------|------|------|
| 缩写单独出现或在开头 | `id`、`url`、`api`、`json`、`pad` | — |
| 缩写在普通单词后面 | `userID`、`fileURL`、`parseJSON`、`currentPAD` | `userId`、`fileUrl`、`parseJson`、`currentPad` |
| 类名或专有名称 | `HandManager`、`PADVector`、`JSONData` | `HandManger`、`PadVector`、`JsonData` |

允许的缩写：`id`、`url`、`api`、`http`、`json`、`sse`、`sql`、`pad`、`llm`、`tts`、`ws`。
禁止不常见缩写：`mgr`、`cfg`、`svc`、`repo`、`proc`。

> **注意**：SuiBot 架构中 `HandManager`、`ToolManager` 是既定名称，保留不改。
> HOP 规范建议避免 `manager` 这类技术词，但在 SuiBot 中这是有明确含义的架构术语。

### 函数与变量命名

- 函数名用「动词 + 名词」：`applyDelta`、`buildSystemPrompt`、`getMemoryContext`
- 首选动词：`get / set / add / remove / update / build / send / check / apply / load / save / init`
- 布尔变量用 `is / has / can / should` 开头：`isConnected`、`hasPermission`、`canRetry`
- 变量名直接说是什么：`userName`、`currentPAD`、`retentionProb`

---

## 注释规范

### 文件头注释（每个文件必须有）

用流畅自然的大白话说清楚：这个文件做什么、设计思想、有哪些重要方法、怎么调用、注意事项。
**不用标注格式**（不写「负责什么」「主要方法」），用自然语言直接描述。调用示例必须完整真实：

```typescript
// emotion-engine.ts
//
// 管穗穗的情绪状态。情绪用 PAD 三维坐标表示（愉悦度、激活度、支配度，各在 [-1,1] 范围内）。
// 每次对话结束后叠加情感波动，同时随时间指数衰减回基准值，模拟人类情绪的自然消退。
//
// 调用示例：
//   const state = getCurrentState()          // 获取当前情绪，返回 { pad, label }
//   applyDelta({ P: 0.3, A: 0.1, D: 0.0 })  // 叠加一次对话带来的情绪变化
//   decay(30)                                // 经过 30 分钟后的情绪衰减
//   setPAD({ P: 0.5, A: 0.3, D: 0.0 })      // 手动设置（调试/剧本模式）
```

### 行注释 vs 尾随注释

两种注释各有用途，不能混用：

```typescript
// ✅ 尾随注释：一行注释对应一行代码，紧凑排列，形成流程整体，不留空行
const factor = Math.exp(-DECAY_LAMBDA * minutesPassed)  // 衰减因子 e^(-λt)，t 越大越接近 0
currentPAD.P = baselinePAD.P + (currentPAD.P - baselinePAD.P) * factor  // P 向基准值靠拢
currentPAD.A = baselinePAD.A + (currentPAD.A - baselinePAD.A) * factor  // A 同理
currentPAD.D = baselinePAD.D + (currentPAD.D - baselinePAD.D) * factor  // D 同理

// ✅ 行注释：一行注释对应下面多行代码，写在代码块上方
// 把新的情感波动叠加到当前状态，乘以敏感度系数让穗穗比普通人更敏感
// clamp 确保结果始终在 [-1, 1]，超出范围会导致后续衰减计算溢出
currentPAD.P = clamp(currentPAD.P + delta.P * SENSITIVITY, -1, 1)
currentPAD.A = clamp(currentPAD.A + delta.A * SENSITIVITY, -1, 1)
currentPAD.D = clamp(currentPAD.D + delta.D * SENSITIVITY, -1, 1)
```

### 必须写注释的情况

1. **魔法数字和魔法字符串**——注释说明含义和取值理由：

```typescript
const DECAY_LAMBDA = 0.1          // 情绪衰减系数，0.1 约等于 1 小时后情绪减半
const FUZZY_THRESHOLD = 0.2       // 记忆保持概率低于此值时进入「模糊状态」
const SIMILARITY_THRESHOLD = 0.85 // 余弦相似度超过此值认为是相似记忆，触发混淆机制
```

2. **坑点注释**——看起来奇怪但有原因，注释说明删掉会发生什么：

```typescript
await delay(200)    // LLM API 有限流，连续请求必须间隔，去掉会概率性 429
connection.close()  // 同一 hand_name 重复连接必须关掉旧的，否则两边都会收到重复消息
```

3. **复杂公式和算法**——注释说明公式来源和参数含义：

```typescript
// 艾宾浩斯遗忘曲线：P = e^(-t/S)
// t = 存储时长（小时），S = 记忆强度 [0,1]，S 越大遗忘越慢
const retentionProb = Math.exp(-hoursPassed / memory.strength)
```

4. **非显而易见的业务逻辑**——注释说清楚为什么这么做：

```typescript
// 穗穗的记忆不会直接消失，而是进入「模糊状态」，让她表现出「有点记不清了」
// 这比直接删除记忆更贴近人类的遗忘行为
if (retentionProb < FUZZY_THRESHOLD) memory.status = "fuzzy"
```

5. **复杂逻辑的段落分区**——较长函数用段落注释分隔：

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

### 不需要注释的情况

```typescript
// ❌ 废话，代码本身就说明了
i = i + 1    // i 加 1
return user  // 返回 user
```

---

## 函数规范

### 固定结构顺序

每个函数按这个顺序组织，不能打乱：

```
① 函数头注释（自然语言，说清楚做什么、怎么用、注意事项）
② 卫语句（Early Return，每条加注释说明原因）
③ 空一行（视觉上明确区分卫语句和主逻辑）
④ 读取数据 / 处理逻辑 / 修改数据
⑤ 反馈 / 返回结果
```

```typescript
// ✅ 完整示例
/** 把一次对话的情绪影响叠加进来。delta 由 LLM 返回，sensitivity 放大穗穗的情绪反应。 */
export function applyDelta(delta: PADVector): void {
  if (!delta) return   // 没有情绪变化就不处理

  // 乘以敏感度再叠加，clamp 确保不超出 [-1, 1]，否则衰减计算会溢出
  currentPAD.P = clamp(currentPAD.P + delta.P * SENSITIVITY, -1, 1)
  currentPAD.A = clamp(currentPAD.A + delta.A * SENSITIVITY, -1, 1)
  currentPAD.D = clamp(currentPAD.D + delta.D * SENSITIVITY, -1, 1)
}
```

### 长度与拆分

- **4–16 行**为甜区，超出考虑拆分
- 提前返回的简单判断写成紧凑一行：`if (!nodes.length) return []`
- 只有当一整块逻辑已经形成独立语义、拆出来后更容易理解时才拆
- **禁止**拆成少于 3 行或只用一次的碎片函数

### 卫语句规则

每条卫语句加注释说明原因，卫语句之间不留空行，卫语句结束后空一行再写主逻辑：

```typescript
function sendChat(content: string, userId: string) {
  if (!content) return                        // 空消息不发
  if (!userId) return                         // 没有目标不发
  if (!isConnected) {
    console.warn("未连接 Core，消息丢弃")
    return                                    // 连接断开时丢弃，不报错
  }

  ws.send(buildChatMessage(content, userId))
}
```

---

## 文件与目录规范

遵循 TypeScript 社区标准分层，结合 SuiBot 的实际目录结构：

```
suibot/
├── core/                   # 触发→指令链条的核心调度
│   ├── index.ts
│   ├── message-handler.ts  # 五步主流程
│   └── prompt-builder.ts
├── engines/                # 数据层：状态读写都在这里
│   ├── emotion-engine.ts
│   ├── memory-engine.ts
│   ├── reflect-engine.ts
│   └── priority-engine.ts
├── tools/                  # 扩展指令层
│   ├── tool-manager/
│   └── tools/
├── hands/                  # 触发层适配器
│   ├── hand-manager/
│   └── hands/
├── types/                  # 全局类型，所有文件共享
│   └── index.ts
└── utils/                  # 无业务身份的纯工具，拿到别的项目也能用
    ├── math.ts             # clamp、normalize
    ├── async.ts            # delay、retry
    └── id.ts               # generateID（注意：ID 大写）
```

**文件命名**：kebab-case，名称直接说明内容。

**工具函数放置规则**：
- 业务逻辑模块里**禁止出现工具函数**，发现了立即提取到 `utils/`
- 只有多个业务模块都会用、且拿到别的项目也能用的函数才进 `utils/`
- 纯业务辅助函数留在对应业务模块内作为私有函数

---

## 性能规范

**热路径性能优先**，其他地方可读性优先。

SuiBot 的热路径（严禁非必要操作）：
- `handleMessage()` 主流程
- `buildSystemPrompt()` 组装
- `callLLM()` 调用链路
- `applyDelta()` 情绪更新

```typescript
// ✅ 允许：非热路径的日志格式化，可读性收益大
console.log(`情绪：P=${pad.P.toFixed(2)}, A=${pad.A.toFixed(2)}, D=${pad.D.toFixed(2)}`)

// ❌ 禁止：热路径里深拷贝只为日志好看
const snapshot = JSON.parse(JSON.stringify(currentPAD))

// ❌ 禁止：LLM 路径上做无人查看的格式化
const prettyPrompt = formatPromptForDisplay(prompt)
await callLLM(prettyPrompt)
```

---

## 复用规范

**第一次出现的逻辑优先就地写**，保持上下文完整可读。
同样逻辑稳定出现**两次及以上**再提取。
提取的前提：提取后更容易理解、更容易搜索、更容易修改。
**禁止**提前为「可能复用」而抽象的函数，等需求真实出现再处理。

```typescript
// utils/math.ts
// 把数值限制在 [min, max] 范围内。PAD 情绪计算、UI 进度条等多处稳定用到。
// clamp(1.5, -1, 1) → 1   clamp(-2, -1, 1) → -1
export function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max)
}

// utils/async.ts
// 等待指定毫秒。用于 LLM API 限流控制，去掉会概率性 429。
// await delay(200)
export function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

// utils/id.ts
// 生成 UUID v4，用于消息的 msg_id 字段。
// generateID() → "550e8400-e29b-41d4-a716-446655440000"
export function generateID(): string {
  return crypto.randomUUID()
}
```

---

## 架构规范：纯函数优先，组合优于继承

### 纯函数优先

纯函数：相同输入永远返回相同输出，不读写任何外部状态。易于测试、易于复用、行为完全可预测：

```typescript
// ✅ 纯函数：只依赖入参，可独立测试
export function applyEmotionDelta(
  current: PADVector,
  delta: PADVector,
  sensitivity: number
): PADVector {
  return {
    P: clamp(current.P + delta.P * sensitivity, -1, 1),
    A: clamp(current.A + delta.A * sensitivity, -1, 1),
    D: clamp(current.D + delta.D * sensitivity, -1, 1),
  }
}

// ❌ 副作用函数：直接修改外部状态，难以测试，行为难以追踪
function applyEmotionDelta(delta: PADVector) {
  currentPAD.P = clamp(currentPAD.P + delta.P * sensitivity, -1, 1)
}
```

### 副作用隔离到边界

副作用（写数据库、调 API、改全局状态）隔离到模块**最外层**，内部逻辑全用纯函数：

```typescript
// 内部：纯函数计算
export function calcDecay(current: PADVector, baseline: PADVector, minutes: number): PADVector {
  const factor = Math.exp(-DECAY_LAMBDA * minutes)  // 指数衰减因子
  return {
    P: baseline.P + (current.P - baseline.P) * factor,
    A: baseline.A + (current.A - baseline.A) * factor,
    D: baseline.D + (current.D - baseline.D) * factor,
  }
}

// 边界：只有这一行有副作用，其余都是纯计算
let _state: PADVector = { ...BASELINE_PAD }
export function decay(minutes: number): void {
  _state = calcDecay(_state, BASELINE_PAD, minutes)
}
```

### 组合优于继承

依赖通过参数显式传入，而不是通过 `this` 或继承链隐式传递：

```typescript
// ✅ 组合：依赖显式传入，一眼看清楚
export function storeWithDecay(
  store: (key: string, value: string) => void,
  key: string,
  value: string,
  strength: number
): MemoryFragment {
  const fragment = buildFragment(value, strength)  // 纯函数构建片段
  store(key, JSON.stringify(fragment))
  return fragment
}
```

### 什么时候可以用类

- 需要维护内部状态，且状态与行为强绑定（如 `WebSocketConnection`）
- 实现第三方库要求的接口（如具体的 Tool 实现 `SuiTool` 接口）
- 管理需要生命周期的资源（如数据库连接池）

即使用类，方法内部也要尽量调用纯函数，把副作用控制在最小范围。

---

## 视觉规范

- 同一语义段的代码紧凑排列，使用尾随注释形成流程整体，**不留空行**
- 不同语义段之间**空 1 行**
- 每个函数之间**空 2 行**
- 卫语句结束后空 1 行再写主逻辑

---

## 给 AI 的提示词（直接复制粘贴）

将以下内容复制到 Cursor / Trae / Roo Code 的「规则」或「自定义指令」中：

```
# SuiBot 项目编码规范

## 核心原则
让刚接触项目的人只看局部代码就能理解、上手、修改。
优先级：可读性 > 性能 > 视觉（热路径除外，热路径性能优先）。

## 架构思想
触发 → 指令 → 数据 → 反馈。
外部触发（CLI/WebUI/Hand）→ Core 路由 → 引擎/Manager 执行业务 → 状态数据修改 → 回复下发。
任何人看到一个事件触发，都能顺着链条找到调用了什么、读写了哪些数据、最终影响了什么。

## 命名规则（强制）
- 变量和函数：camelCase（小驼峰）
- 类、接口、类型：PascalCase（大驼峰）
- 模块级常量：SCREAMING_SNAKE_CASE
- 文件名：kebab-case
- 缩写规则（HOP 规范，强制）：
  - 缩写单独出现或在开头：小写，如 id、url、api、json、pad
  - 缩写在普通单词后面：整体大写，如 userID、fileURL、parseJSON、currentPAD
  - 类名中：整体大写，如 PADVector、JSONData
  - 禁止写成假单词：userId ❌ → userID ✅，parseJson ❌ → parseJSON ✅
- 函数名用「动词+名词」：applyDelta、buildSystemPrompt、getMemoryContext
- 布尔变量用 is/has/can/should：isConnected、hasPermission
- 禁止不常见缩写：mgr、cfg、svc、repo

## 注释规则
- 文件头必须有：自然语言描述文件做什么、怎么调用，给出完整真实的调用示例
- 尾随注释：一行注释对应一行代码，紧凑排列，形成流程整体，不留空行
- 行注释：一行注释对应多行代码，写在代码块上方
- 必须写注释：魔法数字（说明含义和取值理由）、坑点（说明删掉会发生什么）、复杂公式（说明来源和参数）、非显而易见的业务逻辑
- 复杂函数用「// —— 段落名 ——」做区块分隔
- 禁止写重复代码字面意思的废话注释

## 函数规则
- 固定结构顺序：函数头注释 → 卫语句（每条加注释） → 空一行 → 主逻辑 → 返回
- 4–16 行为甜区，超出考虑拆分
- 提前返回的简单判断写成紧凑一行：if (!nodes.length) return []
- 禁止拆成少于 3 行或只用一次的碎片函数

## 文件规则
- 按功能分层：core/（调度）、engines/（数据状态）、tools/（扩展指令）、hands/（触发适配）
- 文件名 kebab-case，名称直接说明内容
- 类型定义在 types/index.ts，通用工具在 utils/ 下对应文件
- 业务模块里禁止出现工具函数，发现了立即提取到 utils/
- utils/ 只放真正通用、拿到别的项目也能用的函数

## 性能规则
- 热路径（handleMessage、buildSystemPrompt、callLLM、applyDelta）严禁非必要计算或 I/O
- 非热路径可用少量性能换可读性（日志格式化、调试输出）
- 禁止热路径里深拷贝、全量重渲染、非必要字符串格式化

## 复用规则
- 第一次出现优先就地写，稳定出现两次及以上再提取
- 禁止提前为「可能复用」而抽象，等需求真实出现再处理
- 提取前提：提取后更容易理解、搜索、修改

## 视觉规则
- 同一语义段紧凑排列，不留空行，用尾随注释形成流程整体
- 不同语义段之间空 1 行
- 每个函数之间空 2 行
- 卫语句结束后空 1 行再写主逻辑

## 架构规则
- 优先写纯函数：相同输入永远返回相同输出，不读写外部状态
- 副作用（数据库、API、全局状态）隔离到模块最外层边界，内部全用纯函数
- 组合优于继承：依赖通过参数显式传入，不通过 this 或继承隐式传递
- 用类的场景：维护内部状态且强绑定、实现第三方接口、管理生命周期资源
```
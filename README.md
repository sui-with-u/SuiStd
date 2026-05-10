# SuiBot 项目编码规范

> 适用于所有 AI 辅助开发（Cursor、Trae、Roo Code 等 Vibe Coding 工具）。
> 基于 UOP（面向理解编程）修订版，结合 SuiBot 项目实际情况调整。
> 与项目框架文档 `SuiBot_FRAMEWORK.md` 配合使用。

---

## 核心原则

**理解优先，规范托底，性能压倒视觉，组合优于继承，纯函数优先。**

- 代码首先要让任意背景的人能快速读懂
- 命名和结构必须符合语言规范，不能以「直觉」为由破坏规范
- 优先使用纯函数和函数组合，避免不必要的类和副作用
- 视觉可以用少量性能换取，但绝不允许为了视觉大量牺牲性能
- 性能 > 可读性 > 视觉美观

---

## 命名规则（强制）

大小写遵循 TypeScript / JavaScript 社区标准，不可因「更好理解」而违反：

| 场景 | 规则 | 示例 |
|------|------|------|
| 变量、函数 | camelCase（小驼峰） | `currentPAD`、`applyDelta`、`getUserInfo` |
| 类、接口、类型 | PascalCase（大驼峰） | `EmotionEngine`、`HandManager`、`PADVector` |
| 模块级常量 | SCREAMING_SNAKE_CASE | `DECAY_LAMBDA`、`MAX_RETRY` |
| 文件名 | kebab-case | `emotion-engine.ts`、`hand-manager.ts` |
| 文件夹名 | kebab-case | `core-engines/`、`skill-manager/` |

- 函数名使用「动词 + 名词」结构，首选动词：`add / make / get / find / show / list / update / remove / check / send / build / apply / load / save / init`
- 布尔变量写成疑问句形式：`isReady`、`hasPermission`、`canEdit`、`isConnected`
- 变量名使用完整常见英语单词：`result`、`user`、`manager`、`config`、`index`、`items`
- `id`、`api`、`url`、`PAD` 等约定俗成的缩写保留，其余一律写完整单词
- 类名用名词，直接代表它管理的对象：`User`、`HandManager`、`EmotionState`

---

## 注释规则

不用每一行都写注释，但以下情况**必须写**：

1. **非显而易见的业务逻辑**

```typescript
// PAD 三个维度都要 clamp，情绪值不能超出 [-1, 1]，否则后续衰减计算会溢出
currentPAD.P = clamp(currentPAD.P + delta.P * SENSITIVITY, -1, 1)
```

2. **魔法数字和魔法字符串**，必须注释说明含义和取值理由

```typescript
const DECAY_LAMBDA = 0.1          // 情绪衰减系数，0.1 约等于 1 小时后情绪减半
const FUZZY_THRESHOLD = 0.2       // 记忆保持概率低于此值时进入「模糊状态」
const SIMILARITY_THRESHOLD = 0.85 // 余弦相似度超过此值认为是相似记忆，触发混淆机制
```

3. **坑点注释**：看起来奇怪但有原因的代码，注释说明删掉会发生什么

```typescript
await delay(200)    // LLM API 有限流，连续请求必须间隔，去掉会概率性 429
connection.close()  // 同一 hand_name 重复连接必须关掉旧的，否则两边都会收到重复消息
```

4. **复杂公式或算法**，注释说明公式来源和参数含义

```typescript
// 艾宾浩斯遗忘曲线：P = e^(-t/S)，t 为存储时长（小时），S 为记忆强度 [0,1]
const retentionProb = Math.exp(-hoursPassed / memory.strength)
```

5. **文件头部**（每个文件必须有），说明这个文件负责什么、对外暴露哪些接口

```typescript
// emotion-engine.ts
// 负责穗穗的 PAD 三维情感状态管理：读取、更新（对话触发）、衰减（定时任务）
// 对外导出：getCurrentState() / applyDelta() / setPAD() / decay()
```

较长函数用 `// —— 段落名 ——` 做区块分隔，方便扫读。

禁止写重复代码字面意思的废话注释：

```typescript
// ❌
i = i + 1    // i 加 1
return user  // 返回 user
```

---

## 函数规则

- 单一职责，**4–15 行**为甜区，超出时考虑拆分
- 用**卫语句（Early Return）**把所有异常先处理，主逻辑写在最后
- 缩进控制在两层以内

```typescript
// ✅
function sendChat(content: string, userId: string) {
  if (!content) return
  if (!userId) return
  if (!isConnected) {
    console.warn("未连接 Core，消息丢弃")
    return
  }

  ws.send(buildChatMessage(content, userId))
}
```

---

## 文件规则

- 按功能模块分层，遵循 TypeScript 社区标准目录结构
- 文件名 kebab-case，名称直接说明内容，不用缩写
- 类型定义集中在 `types/index.ts`
- **业务逻辑模块里禁止出现工具函数**，发现了立即提取到 `utils/` 下的独立模块
- `utils/` 内部按职责再拆文件，不要全堆在 `utils/index.ts` 里

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
│   └── skill-manager.ts      // Skill 调度管理
├── skills/
│   ├── weather-search.ts
│   └── calendar.ts
├── types/
│   └── index.ts              // 全局类型定义
└── utils/
    ├── index.ts              // 统一重导出入口
    ├── math.ts               // clamp、normalize 等数学工具
    ├── async.ts              // delay、retry 等异步工具
    └── id.ts                 // generateId 等 ID 工具
```

---

## 性能规则（重要）

**性能优先级高于视觉，不可妥协。**

允许的「少量性能换视觉」——仅限非热路径：

```typescript
// ✅ 日志可读性提升明显，且只在调试路径执行
console.log(`情绪：P=${pad.P.toFixed(2)}, A=${pad.A.toFixed(2)}, D=${pad.D.toFixed(2)}`)
```

禁止的「为视觉牺牲性能」：

```typescript
// ❌ 热路径里深拷贝只为日志好看
const snapshot = JSON.parse(JSON.stringify(currentPAD))

// ❌ 每次消息触发全量重渲染
messages.forEach(m => reRenderAll(m))

// ❌ LLM 路径上做无人查看的格式化
const prettyPrompt = formatPromptForDisplay(prompt)
await callLLM(prettyPrompt)
```

**SuiBot 热路径**（严禁非必要操作）：

- `handleMessage()` 主流程
- `buildSystemPrompt()` 组装过程
- `callLLM()` 调用链路
- `emotionEngine.applyDelta()` 情绪更新

---

## 复用规则

- 遵循标准工程规范：能预见多处使用的工具函数，提前提取到 `utils/` 下对应的独立模块
- 跨模块复用的类型统一放 `types/`
- 业务逻辑不做「万能函数」，一个函数解决一类问题

---

## 视觉规则

- 功能段落之间空 1–2 行，相关代码紧靠在一起，不相关的代码之间留空行隔开
- 每个函数之间空两行，给阅读者视觉喘息空间
- 卫语句每条之间空一行，主逻辑前再空一行，视觉上明确区分卫语句和主逻辑

---

## 架构规则：组合优于继承，纯函数优先

这是本规范最重要的架构原则。

### 优先使用纯函数

**纯函数**：相同输入永远返回相同输出，不读写任何外部状态，没有副作用。纯函数易于测试、易于复用、行为完全可预测。

```typescript
// ✅ 纯函数：只依赖入参，不触碰外部状态
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

// ✅ 纯函数：给定同样的 PAD，永远返回同样的标签
export function getEmotionLabel(pad: PADVector): string {
  if (pad.P > 0.5 && pad.A > 0.3)  return "兴奋愉快"
  if (pad.P > 0.2 && pad.A <= 0.3) return "轻松愉快"
  if (pad.P < -0.3 && pad.A > 0.3) return "焦虑不安"
  if (pad.P < -0.3 && pad.A <= 0)  return "低落沉闷"
  return "平静中性"
}

// ❌ 副作用函数：直接修改外部状态，难以测试，行为难以追踪
function applyEmotionDelta(delta: PADVector) {
  currentPAD.P = clamp(currentPAD.P + delta.P * sensitivity, -1, 1)
  currentPAD.A = clamp(currentPAD.A + delta.A * sensitivity, -1, 1)
  currentPAD.D = clamp(currentPAD.D + delta.D * sensitivity, -1, 1)
}
```

### 副作用隔离到边界

副作用（写数据库、调用 API、修改全局状态）不可避免，但要隔离到模块的**最外层边界**，内部逻辑全部用纯函数处理：

```typescript
// emotion-engine.ts 内部逻辑全部是纯函数，只有最外层有副作用

// ✅ 纯函数：计算新状态，不改任何东西
export function calcDecay(
  current: PADVector,
  baseline: PADVector,
  minutesPassed: number
): PADVector {
  // 指数衰减：向 baseline 靠拢，factor = e^(-λt)
  const factor = Math.exp(-DECAY_LAMBDA * minutesPassed)
  return {
    P: baseline.P + (current.P - baseline.P) * factor,
    A: baseline.A + (current.A - baseline.A) * factor,
    D: baseline.D + (current.D - baseline.D) * factor,
  }
}

// 副作用边界：只有这里读写模块状态，其余全是纯计算
let _state: PADVector = { ...BASELINE_PAD }

export function decay(minutesPassed: number): void {
  _state = calcDecay(_state, BASELINE_PAD, minutesPassed)   // 纯函数算，副作用只在这一行
}
```

### 组合优于继承

用函数组合代替类继承链，依赖关系通过参数显式传入：

```typescript
// ❌ 继承：子类隐式依赖父类内部实现，改父类可能悄悄破坏子类
class BaseMemory {
  protected store(key: string, value: string) { ... }
}
class AdvancedMemory extends BaseMemory {
  storeWithDecay(key: string, value: string, strength: number) {
    this.store(key, value)
  }
}

// ✅ 组合：依赖通过参数显式传入，一眼看清楚
export function storeWithDecay(
  store: (key: string, value: string) => void,   // 依赖当参数传进来
  key: string,
  value: string,
  strength: number
): MemoryFragment {
  const fragment = buildFragment(value, strength)   // 纯函数构建片段
  store(key, JSON.stringify(fragment))
  return fragment
}
```

### 什么时候可以用类

以下情况可以用类，其余优先用模块级函数：

- 需要维护内部状态，且状态与行为强绑定（如 `WebSocketConnection`）
- 实现第三方库要求的接口（如 `SuiSkill` 接口的具体 Skill 实现）
- 需要通过 `new` 管理生命周期的资源（如数据库连接池）

即使用类，方法内部也要尽量调用纯函数，把副作用控制在最小范围。

---

## 给 AI 的提示词（直接复制粘贴）

将以下内容复制到 Cursor / Trae / Roo Code 的「规则」或「自定义指令」中：

```
# SuiBot 项目编码规范

## 核心原则
理解优先，规范托底，性能压倒视觉，组合优于继承，纯函数优先。

## 命名规则（强制）
- 变量和函数：camelCase（小驼峰），如 currentPAD、applyDelta
- 类、接口、类型：PascalCase（大驼峰），如 EmotionEngine、PADVector
- 模块级常量：SCREAMING_SNAKE_CASE，如 DECAY_LAMBDA、MAX_RETRY
- 文件名：kebab-case，如 emotion-engine.ts、hand-manager.ts
- 函数名用「动词+名词」：get / add / remove / update / build / send / check / apply 是首选动词
- 布尔变量写成疑问句：isReady、hasPermission、canEdit、isConnected
- 变量名用完整常见英语单词，id / api / url 等约定缩写保留，其余一律写完整

## 注释规则
- 不用每一行写注释，以下情况必须写：
  1. 非显而易见的业务逻辑
  2. 魔法数字和魔法字符串，注释说明含义和取值理由
  3. 看起来奇怪但有原因的代码，注释说明删掉会发生什么
  4. 复杂公式和算法，注释说明公式来源和参数含义
  5. 每个文件头部必须有注释说明文件职责和对外接口
- 较长函数用「// —— 段落名 ——」做区块分隔
- 禁止写重复代码字面意思的废话注释

## 函数规则
- 单一职责，4–15 行为甜区，超出考虑拆分
- 用卫语句（Early Return）把所有异常先处理，主逻辑写在最后
- 缩进控制在两层以内

## 文件规则
- 按功能模块分层，遵循 TypeScript 社区标准目录结构
- 文件名 kebab-case，名称直接说明内容
- 类型定义集中在 types/index.ts
- 禁止在业务逻辑模块里写工具函数，发现了立即提取到 utils/ 下的独立文件
- utils/ 内部按职责拆文件：math.ts / async.ts / id.ts 等，不要全堆在 index.ts

## 性能规则（重要）
- 性能优先级高于视觉，不可妥协
- 可以在非热路径（日志、调试、WebUI 渲染）用少量性能换可读性
- 禁止在热路径（handleMessage、buildSystemPrompt、callLLM、applyDelta）做任何非必要计算或 I/O
- 禁止为视觉效果在热路径做深拷贝、全量重渲染、非必要字符串格式化

## 复用规则
- 能预见多处使用的工具函数，提前提取到 utils/ 下对应的独立模块
- 跨模块复用的类型统一放 types/
- 业务逻辑不做万能函数，一个函数解决一类问题

## 视觉规则
- 功能段落之间空 1–2 行，相关代码紧靠，不相关的留空行隔开
- 每个函数之间空两行
- 卫语句之间空一行，主逻辑前再空一行

## 架构规则：组合优于继承，纯函数优先
- 默认写纯函数：相同输入永远返回相同输出，不读写任何外部状态
- 副作用（写数据库、调 API、改全局状态）隔离到模块最外层边界，内部逻辑全用纯函数
- 用函数组合代替类继承链，依赖关系通过参数显式传入，不通过 this 或继承隐式传递
- 只在以下情况使用类：需维护内部状态且状态与行为强绑定、实现第三方库接口、管理需要生命周期的资源
- 即使用类，方法内部也要尽量调用纯函数，把副作用控制在最小范围
```

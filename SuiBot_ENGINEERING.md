# SuiBot 工程优化方案（采纳并执行）

> 文档版本：v1.0 · 2026.09
> 定位：`SuiBot_FRAMEWORK.md` 定义「要做什么」，本文档定义「怎么把它做扎实」
> 状态：**建议已全部采纳，第一批已落地并验证**（见第五章执行记录）

---

## 目录

- [第一章：核心结论](#第一章核心结论)
- [第二章：插件契约](#第二章插件契约)
- [第三章：扩展点与钩子](#第三章扩展点与钩子)
- [第四章：仓库分布与工程化](#第四章仓库分布与工程化)
- [第五章：执行记录](#第五章执行记录)
- [第六章：后续路线](#第六章后续路线)
- [附录：参考实现的取舍](#附录参考实现的取舍)

---

## 第一章：核心结论

### 1.1 定位不变

SuiBot 走**自研大脑 + 自研框架**的路线。理由不是「别人的框架不好」，而是**目标不同**：

| | 编码 Agent 框架 | SuiBot |
|---|---|---|
| 思考单位 | 任务（task） | 陪伴（companionship） |
| 世界模型 | workspace + 文件 + shell | 对话 + 情绪 + 记忆 |
| 一次交互 | 用户提问 → 一个 turn → 交答案 | 持续消息流 → 状态连续演化 |
| 终态 | 任务完成 | 永不「完成」 |

这个差异会导致一个具体冲突：编码框架的上下文压缩是**无差别裁剪省 token**，而 SuiBot 的艾宾浩斯遗忘曲线是**选择性拟人化遗忘**。强行对齐会做出「每半小时失忆一次的穗穗」。

**结论**：骨架自己拿。将来若要把编码 Agent 框架作为外部能力接入，正确姿势是**当作一个 Hand**（见 `SuiBot_FRAMEWORK.md` 第三章 3.3），而不是替换掉 Core。

### 1.2 但插件契约要对齐成熟实践

自研框架最常见的失败方式不是设计错，而是**插件接口没定好**——插件作者写的东西和宿主期望的对不上，最后所有扩展都得回来改 Core。

因此本文档第二章起的规范，是把已经被大规模验证过的插件契约形态**剥离掉业务语义后**，落到 SuiBot 上。

### 1.3 一句话原则

> Core 提供**扩展点**，插件提供**实现**。加一个新能力不应该需要修改 Core 的任何一行代码。

---

## 第二章：插件契约

### 2.1 统一形态

每个 Hand / Tool 插件都通过 `sui.config.json` 向 Manager 声明自己：

```jsonc
{
  "name": "H-SuiWeb",
  "type": "webui",              // hand | webui | test | tool
  "apiVersion": 1,              // 【必须】契约版本，见 2.3
  "version": "0.1.0",
  "entry": "vite",              // 入口提示
  "permissions": ["config:read", "config:write"],   // 声明要碰什么
  "description": "SuiBot 管理面板，情感曲线可视化与参数配置"
}
```

### 2.2 注册必须返回精确卸载函数

**规范（强制）**：任何注册动作都返回 disposer。

```typescript
const unregister = toolManager.register(new WeatherTool())
unregister()   // 只卸载这一次注册
```

**为什么**：靠 `unregister(name)` 要求调用方记住名字，且无法表达「同一个 Tool 装了两份」。注册方持有 disposer，所有权才清晰，整体卸载（插件被禁用、HMR 重载）时才能干净回滚。

`ToolManager.register()` 已按此实现，并且 disposer 会比对引用，避免「后注册的同名 Tool 被先注册的 disposer 误删」。

### 2.3 `apiVersion`：契约版本

**规范（强制）**：`apiVersion` 声明插件依赖的协议版本；Core 在安装和加载时校验，不匹配**直接拒绝**。

```typescript
// tools/plugin-contract.ts
export const SUPPORTED_API_VERSION = 1
checkContract(name, manifest)   // 不兼容时抛错
```

**为什么这条最紧急**：插件是独立仓库、独立演进。主仓库改了 WS 协议或 Tool 接口后，旧插件的表现是**静默错乱**——连接能建立，但字段对不上、消息被丢弃，排查成本极高。有了 `apiVersion`，不兼容在安装那一刻就暴露。

未声明 `apiVersion` 的旧插件按 v1 处理并**打印警告**（不让老插件直接装不上，但提示作者补齐）。

### 2.4 权限声明

`permissions` 目前**只做审计记录**（安装时打印），不做拦截。

**为什么先声明后实现**：Hand 是由 `Bun.spawn` 跑起来的**任意代码**，而 `config.json`（含 API Key）就在进程工作目录旁。声明字段先建立起来，让「这个插件要求了什么」可见；真正的沙箱/审批是后续工作。这一点上必须诚实：**当前没有任何强制力**。

### 2.5 Tool 的返回边界（待重构）

现状 `SuiTool.execute()` 返回 `string`，而规范又要求「Tool 不负责措辞」——这两条互相打架：返回字符串恰恰要求 Tool 负责措辞。

**目标形态**（未实施，见第六章）：

```typescript
interface ToolResult {
  ok: boolean
  summary: string                      // 给 LLM 看的一句话
  data?: Record<string, unknown>       // 结构化数据，供 WebUI 可视化/缓存
}
```

同时 `paramsSchema` 从手写对象换成 schema 库，让**参数校验与 TypeScript 类型同源**，避免二者漂移。

---

## 第三章：扩展点与钩子

### 3.1 为什么需要钩子

如果插件只能注册 Tool，那「过滤敏感词」「调整语气」「记忆写入前过滤」这类需求就必须回来改 `message-handler.ts`。Core 会重新变成大包大揽者，违背 `SuiBot_FRAMEWORK.md` 1.1 节的核心原则。

### 3.2 钩子点定义

`core/hooks.ts` 定义了四个扩展点：

| 钩子 | 时机 | 参数 → 返回 |
|---|---|---|
| `message/received` | 消息进入五步流程前 | `Message` → `Message` |
| `prompt/assembled` | System Prompt 组装完成后 | prompt 字符串 → 追加/改写后的字符串 |
| `emotion/before-delta` | LLM 情绪增量写入引擎前 | PAD 增量 → 修正后的增量 |
| `reply/ready` | 回复下发前 | 回复文本 → 改写后的文本 |

### 3.3 waterfall 语义

处理者**按注册顺序链式执行**，每环拿到上一环的输出：

```typescript
registerHook(HOOKS.REPLY_READY, (r) => `${r}[A]`)
registerHook(HOOKS.REPLY_READY, (r) => `${r}[B]`)
await applyHooks(HOOKS.REPLY_READY, "base")   // => "base[A][B]"
```

**为什么是 waterfall 而不是多播**：多播会让多个插件互相覆盖；waterfall 让它们**叠加生效**（先过滤敏感词，再调整语气）。

### 3.4 隔离保障

热路径上的扩展点必须防「一个插件坏掉拖垮整场对话」：

- 单个处理者抛错 → **捕获并记日志**，其余处理者继续，原值保持不变
- 返回 `undefined` → 保持原值
- 没有处理者 → 立即原值返回（不在热路径上做无谓开销）

### 3.5 插件装配入口

插件在 `register.ts` 里声明自己要挂什么，宿主只负责统一装配：

```typescript
// tools/tools/builtin/register.ts
export function registerBuiltinPlugins(manager: ToolManager): () => void {
  const disposers = [
    ...setupTools(manager),           // 注册 Tool
    ...setupConversationHooks(),      // 挂钩子
  ]
  return () => { for (const dispose of disposers) dispose() }
}
```

新增一个插件只需在 `setup*` 里加一行，**不需要动 Core，也不需要动入口逻辑**。

---

## 第四章：仓库分布与工程化

### 4.1 仓库布局

采用**主仓库 + 独立扩展仓库**方案（`SuiBot_FRAMEWORK.md` 1.3 节的延续）：

```
sui-with-u/
├── SuiBot          主仓库    Core + 四引擎 + 两个 Manager + 协议 + SDK + CLI
├── SuiStd          规范仓库   框架文档 + 编码规范（唯一真源）
├── H-SuiWeb        Hand      管理面板（webui 类型）
├── H-SuiArena      Hand      赛博图灵博弈平台（未完工）
├── H-SuiDevWeb     Hand      调试面板
├── H-OneBot        Hand      QQ
├── H-Telegram      Hand      Telegram
├── T-Weather       Tool      天气
└── ...
```

### 4.1.1 Hand 登记表

| 仓库 | 类型 | 状态 | 说明 |
|---|---|---|---|
| `H-SuiWeb` | `webui` | 可用 | 纯 WS 客户端，浏览器直连 Core；管理 + 聊天 + 情感可视化 |
| `H-SuiArena` | `hand` | **未完工** | 赛博图灵博弈平台。自带游戏服务与前端，AI 侧待接入 Core |

### 4.1.2 Hand 接入规范

任何 Hand 接入 SuiBot 需要满足以下四条：

**① 提供 `sui.config.json` 清单**

```jsonc
{
  "name": "H-SuiArena",
  "type": "hand",                 // hand | webui | test
  "apiVersion": 1,                // 必须与 Core 的 SUPPORTED_API_VERSION 一致
  "version": "0.1.0",
  "entry": "bun run --filter h-suiarena-backend dev",
  "permissions": ["core:chat"],   // 声明需要的能力
  "description": "一句话说明这个 Hand 做什么"
}
```

**② 明确分工：Hand 只做触发层**

Hand 负责「外部触发 → 规范化 → 发给 Core」。**不该在 Hand 里重复实现**情绪、记忆、
LLM 对话——那是 Core 的职责。判断标准：如果两个 Hand 各自实现了一套情绪模型，
这个 Hand 的角色就跑偏了。

**③ 通过 WS 协议与 Core 通信**

端点 `ws://<host>:<port>/ws/<hand_name>`，HELLO/WELCOME 握手，
消息类型 `input | output | status | command | error`。
**必须应答 Core 的 ping**（否则约 120 秒内会被判超时断开）。
可直接复制主仓库的 `sdk/sui-sdk.ts`。

**④ 用 `_` 命名连接标识**

`hand_name` 用小写字母 + 下划线：`sui_web`、`sui_arena`、`napcat_qq`。

### 4.1.3 设计约定：`Core = 一个角色`

**一个 Core 进程 = 一个角色**：一组全局 PAD、一份 `character.json`、一条 WS 端口。

> **这是设计，不是限制。** 它不是待偿还的技术债，也不需要「多角色支持」来修复。
>
> 由此推出的正确架构是：**要接入 N 个不同人设的智能体，就起 N 个 Core 实例**
> （各配一份 `character.json` 与端口），由上层平台按**端点清单**分别连接。
>
> 这样智能体与平台完全解耦：平台不需要知道智能体内部怎么实现情绪与记忆，
> 任何实现了 SuiBot WS 协议的智能体都能接入，不限于 SuiBot 自家的 Core。

一个典型的多智能体场景（`H-SuiArena` 赛博图灵博弈平台）：

```
┌──────────────────────────────────────────────────────────┐
│                  H-SuiArena（平台）                       │
│  入场 / 房间 / 回合 / 投票 / 界面                          │
└───────┬──────────────────┬──────────────────┬────────────┘
        │ WS               │ WS               │ WS
   ┌────▼─────┐      ┌─────▼────┐      ┌──────▼─────┐
   │ SuiBot   │      │ Astro Bot│      │ 其他智能体  │
   │ Core     │      │ Core     │      │ ...        │
   │ :8001    │      │ :8002    │      │ :800N      │
   │ 角色=穗穗 │      │ 角色=Astro│      │            │
   └──────────┘      └──────────┘      └────────────┘
```

**平台需要提供的东西**：一份端点清单（每个 AI 叫什么、连哪个 Core URL、
是否由平台代管进程与角色档案）+ 按清单建立 WS 连接。**不需要**在 Core 里引入角色维度。

> **更正记录（2026.10）**：本节的标题原为「**已知限制**：Core 是单角色的」，
> 并把「多角色支持」列入第六章的待办。那个定性是错的——单角色是设计约定，
> 不是缺陷；把它当缺陷会误导人去重构 Core 的引擎单例，而那是没必要的。


### 4.2 三条落地规矩

**① 主仓库只留内置件**

`core/`、`engines/`、`hands/hand-manager/`、`tools/tool-manager/`、`tools/tools/builtin/`、`sdk/`、`types/`、`utils/`、`cli.ts`。

`hands/hands/` 和 `tools/tools/` 是**运行时落地目录**，由各自 Deployer 拉取填充，内容不进版本控制。

**② `.gitignore` 的坑（已修正）**

规则写 `tools/tools/`（带尾斜杠）会**排除整个目录**，而 git 规定「父目录被排除后无法再重新包含其中文件」——`!tools/tools/builtin/` 这个例外会**静默失效**。正确写法是只排除内容：

```gitignore
tools/tools/*          # 注意：不是 tools/tools/
!tools/tools/builtin/  # 随主仓库发布的内置 Tool 必须进版本控制
hands/hands/*
data/                  # 情绪/记忆快照，用户私有数据
```

**③ 规范文档单一真源**

`SuiStd` 是编码规范的唯一来源。其他仓库不要再复制一份 `CLAUDE.md` 全文——规范改一次要改 N 处，必然漂移。已发现 `SuiBot/AGENTS.md`、`SuiBot/CLAUDE.md` 内容逐字重复，应改为指向 `SuiStd`。

### 4.3 版本锁定

Deployer 原本是「clone 默认分支」，**没锁版本**：插件作者一 push，所有人下次安装拿到的都是新代码，出问题无法定位、无法回滚。

引入 `sui.lock.json`（随主仓库提交）：

```jsonc
{
  "H-SuiWeb": {
    "repo": "https://github.com/sui-with-u/H-SuiWeb.git",
    "commit": "a1b2c3d",              // 安装时的精确 commit
    "installedAt": 1790348661
  }
}
```

安装时记录（`tools/plugin-lock.ts` 的 `recordInstall`），卸载时移除。**收益**：可复现、可回滚。**成本**：一个 JSON 文件 + 一次 `git rev-parse`。

### 4.4 来源组织可配置

Deployer 原本把 `sui-with-u` 写死在源码里，换 fork 或私有组织必须改代码。现改为：

```typescript
const ORG = process.env.SUI_PLUGIN_ORG || "sui-with-u"
```

### 4.5 跨平台

Deployer 原先用 `Bun.spawnSync(["rm", "-rf", dir])` 卸载——**Windows 上没有 `rm` 命令**，直接失败。已统一改为 `fs/promises` 的 `rm(dir, { recursive: true, force: true })`。

---

## 第五章：执行记录

> 本章记录**实际做了什么、验证结果如何**。已完成项与未完成项严格区分，不含「应该可以」。

### 5.1 仓库整理

| 项目 | 处理 | 结果 |
|---|---|---|
| `H-SuiWeb` | 从 `SuiBot/hands/hands/` 移出为顶层独立仓库 | ✅ 保持 `.git` 与全部提交历史，remote 不变，工作区干净；226.8MB `node_modules` 未搬（新位置按 `bun.lock` 重建） |
| `SuiBot/hands/hands/` | 恢复为空运行时目录 | ✅ 符合设计 |
| `who-is-the-ai` | 待删除 | ⏸ **未执行**——见下方说明 |

**关于 `who-is-the-ai`**：该仓库 `docs/plan.md` 存在**未提交的完整重写**（93 增 / 120 删，新增「自我审视」章节并引用 SuiStd）。删除会永久丢失这份内容（未经 commit，git 救不回）。已向负责人说明，**等确认后再删**。

### 5.2 已落地的代码改动

| # | 改动 | 文件 |
|---|---|---|
| 1 | 保留客户端 `msg_id`，回复用 `reply_to` 回传 | `hands/hand-manager/index.ts` |
| 2 | ping/pong 兼容字符串与 JSON 两种形式 | `src/index.ts` |
| 3 | WebUI 应答 Core 心跳 | `H-SuiWeb/src/hooks/useWebSocket.ts`、`src/lib/protocol.ts` |
| 4 | 连接关闭时立即清理注册表 | `src/index.ts` |
| 5 | `ToolManager.register()` 返回精确 disposer | `tools/tool-manager/index.ts` |
| 6 | `onChange` 变更通知，注册后即时同步技能表 | `tools/tool-manager/index.ts`、`src/index.ts` |
| 7 | 契约版本 `apiVersion` 校验 | `tools/plugin-contract.ts`（新增） |
| 8 | `sui.lock.json` 版本锁定 | `tools/plugin-lock.ts`（新增） |
| 9 | 来源组织可配置 | 两个 `deployer.ts` |
| 10 | 跨平台卸载（`rm -rf` → `fs.rm`） | 两个 `deployer.ts` |
| 11 | 钩子系统 + 4 个扩展点 | `core/hooks.ts`（新增） |
| 12 | 钩子接入五步主流程 | `core/message-handler.ts` |
| 13 | 内置插件装配入口 | `tools/tools/builtin/register.ts`（新增） |
| 14 | 首个内置 Tool（`get_current_time`） | `tools/tools/builtin/time-tool.ts`（新增） |
| 15 | 引擎状态落盘与恢复 | `engines/state-store.ts`（新增） |
| 16 | 引擎恢复接口 | `memory-engine.ts`、`chromadb-store.ts`、`reflect-engine.ts` |
| 17 | `.gitignore` 修正（builtin 可入库） | `.gitignore` |
| 18 | Priority Engine 接入主流程（入队 + 半阻塞串行 + 优先级预取） | `hands/hand-manager/index.ts`、`engines/priority-engine.ts`、`src/index.ts` |
| 19 | Core 状态真实上报（`processing` / `idle`） | `core/index.ts` |
| 20 | **Reflect Engine 数据供给打通**（不再传空数组，情绪曲线与核心记忆真正进入审视） | `core/index.ts` |
| 21 | **长期记忆强度按情绪强度计算**（不再写死 0.3，核心记忆筛选才有意义） | `engines/memory-engine.ts` |
| 22 | **记忆混淆机制修复**（去掉只写不读的死缓存，改为同批次内比较） | `engines/memory-engine.ts` |
| 23 | **每日 23:00 定时审视**（文档承诺但从未实现） | `engines/reflect-engine.ts`、`src/index.ts` |
| 24 | 审视可观测性（`previewReflectPrompt()` 用于断言 prompt 内容） | `engines/reflect-engine.ts` |
| 25 | 快照启动即写一次 + 启动日志 | `engines/state-store.ts` |
| 26 | H-SuiArena 包名规范（根/frontend 的旧包名遗漏，导致 `bun install` 失败） | `H-SuiArena/package.json` × 3 |

### 5.3 验证结果

**类型检查**：`bunx tsc --noEmit` → 通过（exit 0）

**单元级验证：9/9 通过**

```
✅ 内置 Tool 注册            ✅ onChange 触发并带描述
✅ disposer 精确卸载          ✅ TimeTool 执行
✅ 非法时区回退不抛错          ✅ 同版本契约通过
✅ 不兼容契约被拒绝            ✅ chat_reply 回传客户端 msg_id
✅ 心跳 45s 后仍未被断开
```

**钩子与持久化验证：18/18 通过**

```
✅ waterfall 按注册顺序叠加   ✅ 单个钩子抛错被隔离
✅ 返回 undefined 保持原值     ✅ 钩子精确卸载
✅ 快照含 PAD/短期记忆/自我描述/版本号
✅ 恢复 PAD / 短期记忆 / 自我描述
✅ 存档损坏时不抛错
```

**端到端重启验证（最关键的一条）**

| 阶段 | 情绪 P 值 |
|---|---|
| 启动初始 | `0.1` |
| 一轮对话后 | `0.16` |
| 定时快照落盘 | `0.15429024508215758` |
| **杀进程重启后** | **`0.15429024508215758`** ✅ |

重启后 API Key 无需重新走一遍对话，情绪与短期记忆均精确还原——**「关机再打开就不记得你」的问题已解决**。

**SDK/CLI 复活验证**：`history_get` 与 `hand-list` 均正常返回（此前必然 10s 超时）。

**Priority Engine 接入验证：11/11 通过**

```
✅ 按优先级出队 + 同优先级 FIFO      ✅ 未指定优先级时自动定级为 P1
✅ 半阻塞：前一条完成前不开始          ✅ 半阻塞：完成后才跑下一条
✅ 单条抛错后队列继续处理             ✅ status 直接分发不进队列
✅ isBusy 反映正在处理                ✅ getQueueSize 反映积压
✅ 排空后回到空闲
```

**端到端优先级预取验证**：同一批内 P3 先发、P1 后发，处理顺序为
`ID_P1 → ID_P3 → ID_P3B`（P1 成功插队），三条消息均未丢失。

**Core 状态广播验证**：`webui` 类型连接收到 `core_state` 序列
`["processing","idle"]`——WebUI 的忙碌指示反映真实状态。

**Reflect Engine 数据供给验证：18/18 通过**

```
✅ 未记录情绪时 prompt 显示「暂无数据」   ✅ 记录情绪后 prompt 含情绪曲线
✅ getEmotionHistory 能取回历史          ✅ 核心记忆被选中（strength > 0.5）
✅ 弱记忆未被选中                        ✅ 核心记忆内容进了 prompt
✅ 平静时写入的强度接近下限（0.300）      ✅ 情绪剧烈时强度明显更高（0.930）
✅ 强度不超过上限 1.0                    ✅ 相似记忆触发混淆标记
✅ 空记忆库检索不崩且无混淆               ✅ 累计 10 条消息触发自我审视
✅ 触发后计数归零（不重复审视）           ✅ startDailyReflect 可启停
```

**端到端验证**：

- 连发 10 条真实对话 → 第 10 条触发 `[ReflectEngine] Self-reflection complete`
- 真实链路下长期记忆强度实测 **0.96 ~ 0.98**（此前恒为 0.3），
  `memory_search` 返回 10 条全部超过 0.5 的核心记忆门槛
- 定时快照实测被 Core 覆盖写入（验证 `setInterval` 确实在跑）

### 5.4 过程中发现的额外问题（已一并修复）

- **连接关闭不清理**：`close` 处理器没调 `removeConnection`，连接要等心跳连续失败 3 次（约 120 秒）才被移除，期间 `hand_list` 与 WebUI 的「已连接 Hand」计数虚高。
- **`readFileSync` 死代码**：`hands/hand-manager/deployer.ts` 底部有一个用 `require("fs")` 的 helper，在 ESM 下不可用且无人调用，已删。
- **`H-SuiWeb` 启动脚本指向不存在的路径**：`dev:skydog` 引用 `../SuiCore`（Python 时代的遗留），已改为 `../SuiBot && bun run .`。
- **`sui.config.json` 的 `entry` 与实现不符**：原写 `index.ts`，但 WebUI 实际由 `bun run dev` 启动、且与核心是 WS 连接而非进程内加载；已改为 `vite` 并补 `apiVersion`/`permissions`。

### 5.5 Priority Engine 接入过程中发现的三个 bug

接入优先级队列时暴露了三处**原本会让功能形同虚设**的缺陷，均已修复：

**① 同步排空让优先级排序失效**

`receiveMessage` 原先同步调用 `flush()`，第一条消息到达时队列里只有它，**立刻被取走处理**，
后续消息根本没机会参与排序。

更麻烦的是：即使改成「下一轮微任务再排空」也不够——WebSocket 每条消息是**独立的宏任务**，
微任务窗口对真实流量几乎没有批处理效果。端到端测试证实了这一点：P3 先发、P1 后发，
处理顺序仍是 `P3 → P1`。

最终方案：**有界批处理窗口（50ms）**。收到消息后等 50ms 再排空，
让同时到达的一批先一起入队，排序才有意义。50ms 人眼不可感知，
但足以让 `P1` 插到 `P3` 前面（实测处理顺序变为 `P1 → P3 → P3b`）。

**② `validateMessage` 的默认值短路了内容定级**

`validateMessage` 把 `priority` 兜底成 `"P3"`，导致 `getPriority()` 开头的
「客户端已显式指定就直接采用」判断命中——**P3 是合法值，于是永远返回 P3**，
按内容/来源定级的规则（私聊 → P1、管理指令 → PPP）从未生效。

`Message.priority` 本就是可选字段，已改为不兜底，留给 Priority Engine 定级。

**③ `handleInput` 内部的广播覆盖了 `processing` 状态**

`core.setStatus("processing")` 由外部调用时，会被 `handleInput` 内部几处
`broadcastState()` 读到的旧值覆盖，WebUI **永远看到 `idle`**，忙碌指示是假的。

改为由 `handleInput` 自己维护：处理 chat 前置 `processing`，`finally` 里恢复 `idle`
（异常路径也能恢复，否则一次报错会让 WebUI 永远停在处理中）。
实测广播序列为 `idle → processing → idle`。

### 5.6 Reflect Engine 数据供给修复过程中发现的问题

**① 传空数组反而覆盖了引擎自己的历史（根因）**

`core/index.ts` 原先调用 `reflectEngine.onMessageProcessed([], [])`。
这两个 `[]` 是**真值**，于是引擎内部的

```typescript
const recentEmotion = emotionHistory ?? _emotionHistory.slice(-5).map(e => e.pad)
```

`??` 从不触发——**调用方传的空数组把引擎自己积累的 30+ 条情绪历史顶掉了**。
也就是说，传 `undefined` 反而能工作，传 `[]` 才是坏的。

现已改为不传参数，并在这个函数的注释里写明这个坑。

**② 核心记忆永远筛不出来**

`triggerReflect` 用 `strength > 0.5` 筛选核心记忆，但长期记忆只在短期溢出时写入，
且强度**写死 0.3**——于是「核心记忆」集合恒为空集，自我审视永远只看得到情绪曲线。

改为按**当时的情绪强度**计算（PAD 模长映射到 `[0.3, 1.0]`）。

设计理由：人对一件事记得牢不牢，取决于它当时带来的情绪冲击；
平静时聊的内容很快淡忘，情绪剧烈时的一句话能记很久——这正是遗忘曲线里 `S` 的物理含义。
实测：平静时写入 0.300，剧烈时 0.930，真实链路下 0.96~0.98。

**③ 记忆混淆机制是死的**

`memory-engine.ts` 的 `applyInterference` 依赖一个 `longTermCache`，而该数组
**只 push 从不读取、也没有任何地方写入**，导致混淆检查永远比较不到对象，
同时还在无限增长。改为在**同一批检索结果内部**比较。

**④ 一次自己造成的返工（记录教训）**

修复过程中我用 PowerShell 的 `Set-Content` 去改源码，**默认编码把文件写成了
UTF-16/非 UTF-8，中文全部损坏**（`Unterminated string literal`），
且 `Get-Content | Out-File` 恢复时又写成了 UTF-16。

教训：**改源码一律使用 `edit` / `write` 工具，不要用 shell 重定向或 `Set-Content`**。
最终用 `[System.IO.File]::WriteAllText(..., UTF8Encoding($false))` 恢复。

过程中还有一个**误判**值得记录：验证脚本里我把「循环 10 次 `recordEmotion` +
1 次 `onMessageProcessed`」当成了 10 次计数，于是误以为是源码 bug，
花了时间在诊断上。**先确认测试断言本身是否正确，再怀疑被测代码**。


---

## 第六章：后续路线

按优先级排列。**未完成项明确标注，不含含糊表述。**

### P1 · Tool 返回结构化边界

**状态：未实施**

`sui.config.json` 的 `SuiTool` 接口仍返回 `string`（见 2.5）。改动会波及 `types/index.ts`、`ToolManager.execute()`、`message-handler` 的工具结果拼接路径。建议在第二个真实 Tool 落地时一并重构，避免只为一个 Tool 改接口。

### P2 · `bun run sui update` 升级命令

**状态：未实施**（锁文件的**记录**能力已完成，**消费**能力未做）

`tools/plugin-lock.ts` 已能记录与查询 commit，但还没有命令去：
- 查看当前锁定版本与远端最新版本的差异
- 按锁文件重建（`install --frozen`）
- 显式升级到指定 commit

建议在 `cli.ts` 的 `COMMANDS` 表加 `hand-update` / `tool-update`，并在 `command-handler.ts` 加对应 action。

### ~~P3 · Priority Engine 接入主流程~~ ✅ 已完成

**状态：已实施（见 5.2 第 18 项与 5.5 的三个 bug 修复）**

`HandManager` 现在持有优先级队列：chat 消息入队后按 `PRIORITY_ORDER` 有序排列，
`flush()` 一次只跑一条（半阻塞），同优先级内保持 FIFO。
`getPriority()` 按内容/来源定级（私聊 → P1、管理指令 → PPP），原先是被默认值短路的状态。

**仍有未完成的部分**：`SuiBot_FRAMEWORK.md` 2.2 节承诺的
**「PPP 级消息可打断当前轮次」尚未实现**。

当前语义是「PPP 排到队首，但不会中断正在处理的那一条」——
即**插队**已实现，**抢占**未实现。实现抢占需要 `AbortController` 中断 LLM 请求，
并处理被打断轮次的状态回滚，属于独立工作量。

### P4 · 配置热重载

**状态：未实施**

`core/llm.ts` 在模块加载时一次性读取 `config.json` 并缓存为常量，导致：

- WebUI Settings 页保存的 **API Key 必须重启 Core 才生效**
- `config_set` 修改的 `DECAY_LAMBDA` / `SENSITIVITY` / `SHORT_TERM_CAPACITY` **完全无效**（这些值在引擎里是写死的常量，与 `config.json` 无关联）

修复方向：改为惰性读取的 `getLLMConfig()`，并让引擎常量从配置读取。

### ~~P5 · Reflect Engine 的数据供给~~ ✅ 已完成

**状态：已实施（见 5.2 第 20-24 项与 5.6 的四项修复）**

原先 `core/index.ts` 传 `onMessageProcessed([], [])`，两个空数组是真值，
`??` 短路导致引擎自己的情绪历史被顶掉；同时长期记忆强度写死 0.3，
`strength > 0.5` 的核心记忆筛选恒为空集。

现已修复：

- **情绪曲线**：Core 改为不传参数，引擎使用自己积累的 50 条滚动历史
- **核心记忆**：长期记忆强度按写入时的 PAD 模长计算（`[0.3, 1.0]`），
  实测真实链路下为 0.96~0.98，能稳定进入核心记忆集合
- **每日定时审视**：补上了 `SuiBot_FRAMEWORK.md` 4.3 节承诺但从未实现的 23:00 触发

**仍待完善**：自我描述目前只进 System Prompt（`getSelfDescription()`），
没有写回 `character.current_state`。按框架文档 4.4 节的设计，
`current_state` 字段本应由 Self-Reflection Engine 写入，这一点尚未接上。

### P6 · 长期记忆真实持久化

**状态：部分缓解**

`state-store.ts` 已让长期记忆能落盘到 `data/state.json`，但这只是**内存存储的快照**。`engines/chromadb-store.ts` 的 `add()` / `search()` / `getAll()` **永远走内存分支**——`CHROMA_URL` 只影响启动时的心跳探测和一行日志，并没有真正接 ChromaDB 的 HTTP API。

同时 `memory-engine.ts` 的记忆混淆机制依赖 `longTermCache` 数组，而该数组**只 push 从不读取、也无处写入**，是死机制。

### P7 · `permissions` 强制化

**状态：未实施**（仅审计）

见 2.4。Hand 是任意代码，当前无任何沙箱。至少在插件启动时记录其声明权限与实际访问的差异。

---

## 附录：参考实现的取舍

> 本文档的插件契约部分参考了 DeepSeek Harness（DSH）的成熟做法。此处明确**借鉴了什么、以及为什么不照搬整体架构**，避免后续误读。

### 借鉴的

| 做法 | 在 SuiBot 的落地 |
|---|---|
| 注册返回精确 disposer | `ToolManager.register()` |
| 插件声明式元数据（name / Config / inject） | `sui.config.json` + `register.ts` 装配入口 |
| 契约版本显式校验、不兼容 fail loud | `apiVersion` + `checkContract()` |
| 事件总线作为主要扩展面（pre/around/post 钩子） | `core/hooks.ts` 的 4 个扩展点 |
| 工具返回规范化值、渲染由宿主负责 | 2.5 节的目标形态（未实施） |

### 不照搬的

| 做法 | 原因 |
|---|---|
| `next-turn` / `next-step` 的消息注入模型 | 它是为「用户中途插话打断」设计的；SuiBot 需要的是「半阻塞总线流 + 5 级优先级 + PPP 插队」，是另一套语义 |
| 上下文压缩（compaction） | 无差别裁剪省 token 与「选择性拟人化遗忘」语义冲突 |
| 单仓 monorepo（`packages/` 下数百个包） | 当前团队规模不需要这种复杂度；插件生态的价值来自**分发独立**，多仓已满足 |
| 把框架整体作为骨架 | 目标不同（见 1.1）；正确姿势是将来作为 **Hand** 接入 |

### 一个必须说清的边界

本文档借鉴的是**插件契约的工程形态**，不涉及任何业务逻辑、提示词或数据。SuiBot 的情绪模型、记忆机制、性格演化全部为自有设计（见 `SuiBot_FRAMEWORK.md` 第四章）。

---

*本文档随实现推进更新。未完成项以第六章为准，第五章只记录已验证的事实。*

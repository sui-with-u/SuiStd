# SuiBot 使用与验收指南

> 版本：v1.0 · 2026.10
> 适用代码：SuiBot `d5b5823`、SuiStd `50c738f`、H-SuiWeb `ff20a73`、H-SuiArena `89a8893`
>
> 本文档的每条命令都**实际跑过**，标注了预期输出。凡是「已实测」的结论都给出了具体数值，
> 未验证的部分明确标注为「未验证」。

---

## 一、最快上手（5 分钟）

### 前置

- **Bun ≥ 1.3**（`bun --version`）
- 一个 DeepSeek API Key（**可选**：没有 Key 也能跑，回复是模拟的，用于验证链路）

### 步骤

```bash
# 1. 主仓库
cd SuiBot
bun install
bun run start          # Core 监听 :8000
```

预期输出：

```
SuiBot v0.1.0
[ToolManager] Registered: get_current_time       ← 内置插件装配成功
[ChromaDB] Using in-memory vector store
[MemoryEngine] Long-term store ready (ChromaDB: false)
[StateStore] 定时快照已启动（每 60 秒一次）
SuiBot Core running
  WS: ws://0.0.0.0:8000/ws/<hand_name>
  Health: http://0.0.0.0:8000/health
```

```bash
# 2. 另开一个终端，验证 Core 活着
curl http://localhost:8000/health
# → {"status":"ok","emotion":{"pad":{"P":0.1,"A":-0.2,"D":0},"label":"平静中性"},"hands":[]}
```

```bash
# 3. 接上 WebUI（H-SuiWeb 是独立仓库，与 SuiBot 同级）
cd H-SuiWeb
bun install
bun run dev            # → http://localhost:5173
```

打开 `http://localhost:5173`，进入 `/chat` 发一条消息，应当收到回复。

> **更省事的方式**：在 `H-SuiWeb` 目录直接 `bun run dev:skydog`，
> 它会同时拉起 SuiBot Core 和 WebUI（要求两个仓库同级）。

### 配置 API Key（可选）

```bash
cd SuiBot
cp config.example.json config.json
# 编辑 config.json，填入 llm.api_key
```

> ⚠️ **保存后必须重启 Core 才生效**——`core/llm.ts` 在模块加载时读取一次并缓存。
> 这是已知限制（`SuiBot_ENGINEERING.md` 第六章），不是你的操作问题。
> 也可以用环境变量 `DEEPSEEK_API_KEY` 绕过这个限制。

---

## 二、我要怎么验收

按下面的顺序做，每条都有明确的通过标准。**建议至少做到验收 3**。

### 验收 1：Core 能起来、CLI 能通

```bash
cd SuiBot && bun run start          # 终端 1

# 终端 2
bun run sui hand-list
bun run sui tool-list
bun run sui list
bun run sui api-test
```

| 检查项 | 通过标准 |
|---|---|
| Core 启动 | 打印出上面那几行，无报错退出 |
| `hand-list` / `tool-list` / `list` | **在 10 秒内返回**（不是超时） |
| `api-test` | 输出「连接 / 模型 / Key / Token」四行，没配 Key 时显示「未配置」 |

> 这三条曾经**必然 10 秒超时**（Core 覆盖了客户端 `msg_id`，导致 reply_to 匹配不上）。
> 现在应当秒回。如果你看到超时，说明这个 bug 回归了。

### 验收 2：优先级队列与状态上报（自动化）

用 SDK 连 Core，连发两条消息：先发 `P3`，再发 `P1`。

| 检查项 | 通过标准 | 实测结果 |
|---|---|---|
| 优先级预取 | 回复顺序里 **P1 在 P3 之前** | `["ACC_P1","ACC_P3"]` ✅ |
| 消息不丢 | 两条都收到回复 | 2/2 ✅ |
| msg_id 回传 | 回复的 `reply_to` 等于你发的 `msg_id` | ✅ |
| 忙碌状态 | 收到 `core_state` 且 `status` 出现过 `processing` | `["processing","idle"]` ✅ |

> **注意**：`core_state` 只广播给 `type=webui` 的连接。
> 浏览器直连时 `hand_name` 用 `sui_web` 这类 `_web` 结尾的名字，Core 会自动推断类型；
> 用别的名字（如 `mybot`）收不到状态广播 —— 这是设计如此，不是 bug。

### 验收 3：心跳保活（最能说明问题的验收）

让一个连接挂 **2 分钟以上**，期间按协议应答心跳。

| 检查项 | 通过标准 | 实测结果 |
|---|---|---|
| 心跳应答 | 收到 `{"type":"ping"}` 后回 `{"type":"pong"}` | 130 秒内收到 5 次 ping ✅ |
| 连接存活 | **2 分钟后连接仍然 OPEN** | `readyState=1` ✅ |

**协议细节**（接入时最容易踩的坑）：

- Core 每 **30 秒**发 `{"type":"ping"}` —— 是 **JSON 对象**，不是裸字符串 `"ping"`
- 客户端须在 **10 秒内**回 `{"type":"pong"}`
- 未应答计一次失败，**连续 3 次约 2 分钟后强制断开**
- **发 `hand_state` 不算应答**，它只被 Core 打印，不会重置超时计时

> 这个 bug 修复前，所有 Hand 每 2 分钟被踢一次。现在连续 130 秒无断开。

### 验收 4：状态持久化（陪伴感的关键）

```bash
# 1. 启动 Core，通过 WebUI 或 SDK 聊几句
# 2. 等 60 秒以上（定时快照间隔），确认 SuiBot/data/state.json 生成
# 3. 杀掉 Core，重新 bun run start
# 4. 查 /health
```

| 检查项 | 通过标准 | 实测结果 |
|---|---|---|
| 快照落盘 | `SuiBot/data/state.json` 存在，含 `pad` / `shortTerm` | ✅ |
| 情绪还原 | 重启后 `/health` 的 PAD **等于**重启前的值 | `P=0.34853003950703026` 精确一致 ✅ |
| 记忆还原 | 短期记忆条数、中文内容完好 | 7 条 ✅ |
| 存档损坏容错 | 手动把 `state.json` 改成乱码后重启，**不崩**且按初始状态启动 | ✅ |

### 验收 5：插件体系（「一切皆插件」）

| 检查项 | 通过标准 | 实测结果 |
|---|---|---|
| 内置 Tool | 启动日志出现 `[ToolManager] Registered: get_current_time` | ✅ |
| 技能注入 | 配了 API Key 后问「现在几点」，穗穗应能调用该技能 | 未验证（需真实 Key） |
| 契约校验 | 把某个 `sui.config.json` 的 `apiVersion` 改成 `999`，安装/加载应**被拒绝并报错** | ✅ 单元级验证 |
| CLI 安装 | `bun run sui hand-install <名字>` 从 GitHub 拉取 | 未验证（需真实仓库） |

### 验收 6：WebUI 四个页面

| 路由 | 通过标准 |
|---|---|
| `/` 仪表盘 | 连接指示器显示「已连接」、出现 Session ID、Hand 列表、运行时长 |
| `/chat` | 发消息有回复；右侧情感侧边栏随回复变化 |
| `/emotion` | 显示 PAD 三维值与标签；拖滑块调用 `emotion_set` 生效 |
| `/settings` | 能读配置、填 API Key 并保存（**重启 Core 后生效**）、能测 API 连通性 |

---

## 三、本地开发：H-SuiWeb 的两种接法

`H-SuiWeb` 是**独立仓库**（顶层目录），而管理器的运行时落地目录是
`SuiBot/hands/hands/`（被 gitignore 排除）。两种接法：

### 方式 A：模拟真实用户（从 GitHub 拉取）

```bash
cd SuiBot
bun run sui hand-install H-SuiWeb
cd hands/hands/H-SuiWeb && bun install && bun run dev
```

### 方式 B：本地开发（推荐，改代码即时生效）

把顶层目录**联接**进落地目录，避免同时维护两份代码：

```bash
cd SuiBot
mkdir hands\hands
cmd /c mklink /J "hands\hands\H-SuiWeb" "%CD%\..\H-SuiWeb"
bun run sui hand-list          # 应当列出 H-SuiWeb
```

> ⚠️ **必须用绝对路径**。`mklink` 的相对路径是相对**当前工作目录**解析的，
> 不是相对链接所在位置；写相对路径会创建出指向 `C:\H-SuiWeb` 的坏联接
> （目录看着在，内容读不到）。这一点已实测踩过。
>
> 目录联接（`/J`）**不需要管理员权限**。

WebUI 侧直接 `cd H-SuiWeb && bun run dev` 即可。

---

## 四、已知限制（不是 bug，别浪费时间排查）

| 限制 | 影响 | 是否能绕过 |
|---|---|---|
| **改配置要重启 Core** | WebUI 保存的 API Key 不热生效 | 用 `DEEPSEEK_API_KEY` 环境变量 |
| **`config.json` 里 `core.decay_lambda` / `core.sensitivity` 无效** | 改了不影响情绪 | 否，常量写死在 `emotion-engine.ts` |
| **长期记忆不落 ChromaDB** | `CHROMA_URL` 只影响启动日志 | 记忆靠 `data/state.json` 快照存活 |
| **`Core = 一个角色`（设计约定，非限制）** | 一个 Core 进程只演一个角色 | **不是问题**：要 N 个人设就起 N 个 Core 实例（各配 `character.json` 与端口），平台按端点清单分别连接。智能体因此与平台解耦 |
| **PPP 只插队不抢占** | 不能打断正在处理的那一轮 | 否，未实现 |
| **自我描述不写回 `character.current_state`** | 只进 System Prompt | 可用 `character_update` 手动设 |
| **`permissions` 仅审计** | 无沙箱，Hand 是任意代码 | 否，未实现 |

---

## 五、常见问题

**Q：`bun run sui` 所有命令都超时**
检查 Core 是否在运行（`curl localhost:8000/health`）。
如果 Core 正常但仍超时，说明 `msg_id` 回传回归了 —— 见验收 1。

**Q：WebUI 每隔两分钟断一次、自动重连**
心跳没有应答。检查 `src/hooks/useWebSocket.ts` 是否处理了 `data.type === "ping"`。
注意是 JSON 的 `type` 字段，不是 `payload.action`。

**Q：`core_state` 收不到**
你的连接不是 `webui` 类型。`hand_name` 用 `_web` 结尾（如 `sui_web`），
或在 HELLO 里显式传 `hand_type: "webui"`。

**Q：端口 8000 被占用，Core 起不来**
```bash
netstat -ano | findstr :8000        # 找 PID
taskkill /PID <PID> /F
```
注意 `Get-NetTCPConnection` 在某些环境下不显示占用者，用 `netstat` 更可靠。

**Q：重启后穗穗「失忆」了**
检查 `SuiBot/data/state.json` 是否存在。若不存在，可能是启动后不到 60 秒就退出了
（定时快照间隔 60 秒；但**启动时会先存一次**，所以正常应有文件）。

**Q：`hand-list` 显示「(无)」但我明明把 Hand 放进去了**
- 目录必须含 `sui.config.json`
- 如果是用 `mklink /J` 链接进来的：需确认用的是**绝对路径**（相对路径会产生坏联接）

---

## 六、仓库与职责

| 仓库 | 职责 |
|---|---|
| `SuiBot` | Core + 四引擎 + 两个 Manager + 协议 + SDK + CLI（**大脑**） |
| `SuiStd` | 规范与文档唯一真源（`README.md` / `SuiBot_FRAMEWORK.md` / `SuiBot_ENGINEERING.md`） |
| `H-SuiWeb` | 管理面板 Hand（`type: webui`），纯前端直连 Core |
| `H-SuiArena` | 赛博图灵博弈平台 Hand（**未完工**，AI 侧待接入 Core） |
| `.github` | 组织主页 |

**文档阅读顺序**：

1. 本文档 —— 怎么用、怎么验收
2. `SuiStd/SuiBot_ENGINEERING.md` —— 工程规范、Hand 接入四条、执行记录、未完成清单
3. `SuiStd/SuiBot_FRAMEWORK.md` —— 框架全貌（**设计稿**，顶部有「已承诺/未实现」对照表）
4. `SuiStd/README.md` —— 编码规范

> ⚠️ `H-SuiWeb/HANDOVER_WEBUI.md` 与 `PROTOCOL.md` 是**平台独立时期**的存档件，
> 里面提到 `SuiCore`、Python SDK、REST API 等已失效内容（已在文件顶部标注）。
> 协议真源是 `SuiBot/sdk/sui-sdk.ts` 与 `SuiBot/README.md` 的「WebSocket 协议」一节。

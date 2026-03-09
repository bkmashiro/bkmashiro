# Botzone Judger 架构梳理

> 基于 GP-judger (原版) 和 BP-Judger (重构版) 代码 + Botzone 论文 (ITiCSE'18)

---

## 一、Botzone 系统全貌

```
前端 Web Server
   ├── 用户上传 Bot 代码
   ├── 发起 Match 请求 → 消息队列
   ├── 管理排行榜 / ELO 更新
   └── 提供外部 API（本地 Bot 接入）
          │
          ▼
    消息队列 (MQ)
          │
          ▼
后端 Judge Server (GP-judger / BP-Judger)
   ├── 接收 Task（含 Judge + Gamer 代码）
   ├── 编译各 Bot
   ├── 在 Sandbox 中运行裁判 + 选手程序
   ├── 管理回合通信（Judge ↔ Gamer via stdin/stdout）
   └── 回报结果到前端
```

---

## 二、核心概念

| 概念 | 说明 |
|------|------|
| **Bot** | 用户提交的 AI 程序，通过 stdin/stdout 与 Judge 通信 |
| **Judge** | 控制整局游戏流程的程序（也是 Bot 的一种），发 request/finish 指令 |
| **Gamer** | 参与对局的 Bot（可以是多个），响应 Judge 的 request |
| **Match** | 一局游戏，Judge + N 个 Gamer 共同参与 |
| **Sandbox** | 资源隔离环境（Docker / nsjail），防止 Bot 逃逸或占用过多资源 |

---

## 三、GP-judger（原版）运行流程

### 3.1 入口：HTTP 服务

```
POST /v1/judge
  body: { game: { judger: Code, player1: Code, ... }, callback: { update, finish } }
```

- `Code` = `{ language, source, limit: { time, memory } }`
- 校验来源 IP（trustIp）
- 通过 `Throttle` 控制并发数上限

### 3.2 God.start() — 主控流程

```
1. 编译阶段（status=1）
   for each agent in game:
     ExecutableAgent.init()     → 写源码文件到临时目录
     ExecutableAgent.compile()  → dockerSpawn 编译器，生成可执行文件
     检查 verdict（CE 则中止）

2. 运行阶段（status=2）
   for each agent:
     ExecutableAgent.exec()     → dockerSpawn 启动进程（暂停状态 SIGSTOP）
     绑定 ReadLine 到 stdout

3. 对局循环（Round Loop）
   round=1, toJudger={}
   loop:
     judger.cont()              → SIGCONT 恢复 Judge
     if round>1: 写 toJudger JSON 到 judger.stdin
     fromJudger = 读取 judger.stdout 一行 JSON
     judger.stop()              → SIGSTOP 暂停 Judge
     记录 JudgerRoundSummary
     
     if fromJudger.command == 'finish': break
     
     if fromJudger.command == 'request':
       for each key in fromJudger.content:
         gamer.cont()
         gamer.stdin.write(fromJudger.content[key] + '\n')
         raw = 读取 gamer.stdout 一行
         gamer.stop()
         toJudger[key] = { verdict, raw }
         记录 GamerRoundSummary

4. 清理
   for each process: process.clean()
   for each agent: agent.clean()

5. 回报结果（callback.finish）
   POST { result: JSON.stringify({ compile, round }) }
```

### 3.3 通信协议（stdin/stdout JSON）

**Judge → 系统（stdout）：**
```json
{ "command": "request", "display": "...", "content": { "player1": "game_state_str" } }
// 或
{ "command": "finish", "display": "...", "content": { "player1": 100, "player2": -100 } }
```

**系统 → Gamer（stdin）：**
```
game_state_str\n
```

**Gamer → 系统（stdout）：**
```
action_str\n
```

**系统 → Judge（stdin，第2轮起）：**
```json
{ "player1": { "verdict": "OK", "raw": "action_str" } }
```

### 3.4 进程控制（SIGSTOP/SIGCONT）

- 所有进程启动即 SIGSTOP（suspend）
- 需要它运行时 SIGCONT，读完输出再 SIGSTOP
- 超时检测：timeout 函数监控，到时 SIGKILL
- 资源测量：通过 `/proc/<pid>/status` 读取 memory；time via rusage

### 3.5 沙箱（dockerSpawn）

- 每个进程用独立 Docker 容器运行
- 限制：uid/gid、memory、pid 数量、文件大小
- 语言支持：C, C++, Java, Python2/3, JS(Node)

---

## 四、BP-Judger（重构版）架构变化

### 4.1 整体改变

| 维度 | GP-judger | BP-Judger |
|------|-----------|-----------|
| 框架 | Express 裸框架 | NestJS |
| 沙箱 | Docker | **nsjail**（更轻量，无需 Docker daemon）|
| 编译/执行 | 硬编码在代码里 | **Pipeline 系统**（JSON 配置驱动）|
| 游戏逻辑 | 耦合在 God 里 | 抽象为 `GameModule`/`PlayerModule` |
| 缓存 | 无 | 编译结果 **LRU 文件缓存**（fingerprint by MD5）|

### 4.2 Pipeline 系统（BKPileline）

核心理念：**将编译/执行步骤配置化**，每个步骤是一个 Job：

```
Pipeline = { jobs: Job[], constants?: {} }

Job 类型：
  { run: "command {{var}}" }       → 直接执行命令（支持模板变量）
  { use: "ModuleName", with: {} }  → 调用内置模块
```

内置 Module：
- `GameModule`：创建游戏实例
- `PlayerModule`：注册玩家
- `CompileModule`：编译代码（含缓存逻辑）
- `FileCacheModule`：LRU 文件缓存
- `POSTModule`：HTTP 回调

命令执行链（CoR 模式）：
```
PlainSystemCommandHandler → NSJailCommandHandler → NetnsCommandHandler
```

### 4.3 nsjail 沙箱

- 替代 Docker，直接用 Linux namespace 隔离
- 配置通过 `NsJailConfig` 对象传递
- 支持 netns（网络命名空间隔离）
- 更低延迟，不需要 daemon

### 4.4 编译缓存

```
code_fingerprint = MD5(source + lang + version)
  → 查 FileCache（LRU）
  → 命中：直接返回编译产物路径
  → 未命中：运行 Pipeline 编译，写入缓存
```

---

## 五、Judge ↔ Gamer 通信协议（不变）

两版都遵循 Botzone 标准协议，核心不变：

```
Round 1:
  Judge 收到空输入，输出游戏初始 request
  → Gamer 收到 request，回复 action
  → Judge 收到 action 集合，更新状态，再次 request 或 finish

结束：
  Judge 输出 finish + scores
  → 系统汇总 GameResult 回报前端
```

---

## 五·五、与官方 Botzone 协议的差异（⚠️ 重要）

通过 [Botzone Wiki](https://wiki.botzone.org.cn) 核查，我们的实现与官方协议存在根本性差异：

### 差异 1：Bot 生命周期模型（最大问题）

| | 官方 Botzone | GP-judger / BP-Judger |
|--|------|------|
| Bot 进程 | **每轮重启**，每次全新进程 | 持续运行（SIGSTOP/SIGCONT） |
| Bot 收到的输入 | **完整历史**：所有 requests + responses | 只有当前回合的 request |
| Bot 状态管理 | Bot 无状态，自己从历史重建局面 | Bot 自行维护内部状态（有状态） |

官方设计理念：**Bot 每次都看到所有历史，自己推演局面**。这样 Bot 更简单，不需要维护状态。

> 注：官方后来加了「长时运行」模式，可通过输出特定指令让 Bot 保持运行，减少冷启动开销——这类似 GP-judger 的 SIGSTOP 方式，但是**可选的**。

### 差异 2：官方 Bot 输入格式（JSON 交互）

官方输入是完整历史的 JSON：
```json
{
  "requests":  ["Turn1 req",  "Turn2 req"],
  "responses": ["Turn1 resp", "Turn2 resp"],
  "data":       "上回合 Bot 保存的数据（100KB 上限）",
  "globaldata": "跨对局持久化数据（100KB 上限）",
  "time_limit": 1000,
  "memory_limit": 262144
}
```

我们的实现：只传当前回合 request 字符串，**无历史、无 data/globaldata**。

### 差异 3：官方 Bot 输出格式

```json
{
  "response":   "本回合决策",
  "debug":      "调试信息（保留在 Log，1KB 上限）",
  "data":       "本回合保存数据（下回合恢复）",
  "globaldata": "全局持久化数据（跨对局保留）"
}
```

我们的实现：只读 raw string，**无 JSON 包装、无 data 持久化**。

### 差异 4：简化交互模式（我们未实现）

官方支持无 JSON 的简化交互：
```
n               ← 当前回合数
req_1           ← 第1回合 request
resp_1          ← 第1回合 response
...
req_n           ← 当前回合 request
data_line       ← 上回合保存的 data
```

### 差异 5：裁判输入格式

官方裁判收到的输入与 Bot 类似，包含完整历史 + 玩家 responses：
```json
{
  "log": [
    { "output": { "command": "request", "content": {...}, "display": "..." }, "verdict": "OK", ... },
    { "0": { "response": "...", "verdict": "OK", ... } },
    ...
  ],
  "initdata": "初始化数据（随机种子、地图等）"
}
```

裁判第一回合可以在输出中写 `initdata` 保存随机种子等，之后每轮收到该 `initdata`。

### 差异 6：裁判输出格式（基本一致，但细节有差）

官方裁判输出（我们基本实现了）：
```json
{ "command": "request", "content": { "0": "player0的request" }, "display": "..." }
// 或
{ "command": "finish", "content": { "0": 100, "1": -100 }, "display": "..." }
```

差异：content 的 key 是数字字符串（`"0"`, `"1"`），我们用的是 `"judger"`, `"player1"` 等任意 key。

---

### 总结：需要重新设计的核心问题

1. **Bot 生命周期**：官方是每轮启动新进程 + 传完整历史；我们是持续运行 + 只传当前 request
   - 建议：实现两种模式——默认模式（官方兼容）+ 长时运行模式（GP-judger 现有方式）
   
2. **Bot 输入/输出协议**：需要加 JSON 封装层，支持 `requests[]`, `responses[]`, `data`, `globaldata`

3. **裁判 initdata**：需要支持对局初始化参数传入

4. **data/globaldata 持久化**：Bot 跨回合/跨对局的数据存储机制

5. **简化交互**：对不会 JSON 的初学者的友好模式

---

## 六、重构目标分析

### 原版痛点
1. **编译/执行硬编码**：每加一门语言要改 TypeScript 代码
2. **Docker 依赖**：部署重，每局起容器延迟高
3. **God 类职责过重**：编译+运行+通信+回调全混在一起
4. **无缓存**：同一份代码重复提交会重复编译
5. **无 NestJS 依赖注入**：扩展性差

### BP-Judger 改进方向（已做）
- ✅ Pipeline JSON 配置化编译
- ✅ nsjail 替代 Docker
- ✅ LRU 编译缓存
- ✅ GameModule/PlayerModule 抽象

### 下一步重构建议（leverage 集成）
1. **统一 Task 入口**：参考 GP-judger 的 `POST /v1/judge`，设计 NestJS Controller
2. **Round Loop 重构**：从 God 的 while 循环 → Event-driven / async iterator
3. **多语言配置**：所有语言编译/执行命令移到 JSON 配置文件，不要硬编码
4. **资源统计**：cgroup v2 读取（Linux），macOS 开发环境 mock
5. **错误处理**：每种 Verdict（TLE/MLE/RE/CE/NJ）需要细化，前端展示友好
6. **测试覆盖**：每种语言的编译+执行用 Jest E2E 测试覆盖

---

## 七、数据结构速查

```typescript
// Task（输入）
interface Task {
  game: Record<string, Code>  // key: "judger" | "player1" | "player2" | ...
  callback: { update: string, finish: string }
}

interface Code {
  language: string   // "C" | "C++" | "Java" | "Python3" | ...
  source: string     // 源码字符串
  limit: { time: number, memory: number }
}

// GameResult（输出）
interface GameResult {
  compile: Record<string, CompileSingleResult>
  round: (JudgerRoundSummary | GamerRoundSummary)[]
}

// Verdict
type Verdict = "OK" | "TLE" | "MLE" | "NJ" | "RE" | "CE" | "SE" | "NR"
```

---

*生成时间：2026-03-09 | 基于 GP-judger@HEAD + BP-Judger@HEAD + ITiCSE'18 论文*

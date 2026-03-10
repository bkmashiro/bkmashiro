# Botzone Judger 重构设计方案

> 基于协议分析，设计一个官方兼容、可扩展的新版 Judger

---

## 一、设计原则

1. **官方协议兼容**：默认行为严格遵循 Botzone 官方 wiki 协议
2. **长时运行可选**：作为 `BotRunMode` 的一种策略，未来可插拔
3. **NestJS 模块化**：职责分离，易测试
4. **Pipeline 保留**：BP-Judger 的 Pipeline 思路保留用于编译阶段

---

## 二、核心架构

```
JudgerService (入口)
  │
  ├── CompileService          编译各 Bot（含 LRU 缓存）
  │     └── Pipeline          JSON 配置驱动的编译步骤
  │
  ├── MatchRunner             对局主控
  │     ├── JudgeProcess      裁判进程管理
  │     ├── PlayerProcess[]   玩家进程管理
  │     └── IBotRunStrategy   ← Bot 运行策略（可插拔）
  │           ├── RestartStrategy    官方默认：每轮重启，传完整历史
  │           └── LongRunStrategy   长时运行：SIGSTOP/SIGCONT（未来）
  │
  ├── DataStore               data/globaldata 持久化
  └── CallbackService         结果回报（update/finish）
```

---

## 三、Bot 运行策略接口

```typescript
/**
 * Bot 运行策略 - 决定 Bot 进程的生命周期和输入/输出方式
 */
export interface IBotRunStrategy {
  /**
   * 执行一个 Bot 回合
   * @param bot Bot 进程（或按需创建）
   * @param input BotInput 完整输入（含历史）
   * @returns BotOutput 包含 response/data/globaldata/debug
   */
  runRound(bot: BotContext, input: BotInput): Promise<BotOutput>;

  /**
   * 回合结束后的清理（重启策略：杀掉进程；长时策略：SIGSTOP）
   */
  afterRound(bot: BotContext): Promise<void>;

  /**
   * 对局结束后清理
   */
  cleanup(bot: BotContext): Promise<void>;
}
```

### 3.1 RestartStrategy（官方默认）

```typescript
export class RestartStrategy implements IBotRunStrategy {
  async runRound(bot: BotContext, input: BotInput): Promise<BotOutput> {
    // 1. 每轮启动全新进程
    const proc = await bot.spawn();

    // 2. 写入完整历史 JSON（官方格式）
    const payload = JSON.stringify({
      requests:    input.requests,
      responses:   input.responses,
      data:        input.data ?? '',
      globaldata:  input.globaldata ?? '',
      time_limit:  input.timeLimit,
      memory_limit: input.memoryLimit,
    }) + '\n';
    proc.stdin.write(payload);
    proc.stdin.end();

    // 3. 读取 Bot 输出（单行 JSON）
    const rawLine = await readLine(proc.stdout, input.timeLimit);
    const output = JSON.parse(rawLine) as BotOutput;

    // 4. 进程自然退出（或超时 kill）
    await proc.waitExit();
    return output;
  }

  async afterRound(_bot: BotContext) { /* 进程已退出，无需额外操作 */ }
  async cleanup(_bot: BotContext)     { /* 无持久进程 */ }
}
```

### 3.2 LongRunStrategy（未来实现）

```typescript
export class LongRunStrategy implements IBotRunStrategy {
  // Bot 在第一轮启动后持续存活（SIGSTOP/SIGCONT）
  // 输入只传当前回合 request（减少传输量）
  // Bot 输出特定指令（如 ">>>BOTZONE_LONGRUN"）表示进入长时运行模式
  // ... 详见官方文档"长时运行"章节
}
```

---

## 四、数据结构

### 4.1 Bot 输入（完整历史）

```typescript
export interface BotInput {
  requests:    string[];   // 历史所有 Judge request（含本轮）
  responses:   string[];   // 历史所有 Bot response（比 requests 少一条）
  data:        string;     // 上回合 data（最大 100KB）
  globaldata:  string;     // 跨对局全局数据（最大 100KB）
  timeLimit:   number;     // ms
  memoryLimit: number;     // bytes
}
```

### 4.2 Bot 输出

```typescript
export interface BotOutput {
  response:   string;        // 本回合决策（必填）
  debug?:     string;        // 调试信息（保留在 Log，最大 1KB）
  data?:      string;        // 本回合保存（下回合恢复）
  globaldata?: string;       // 全局持久化（跨对局）
}
```

### 4.3 裁判输入

```typescript
export interface JudgeInput {
  log: JudgeLogEntry[];      // 完整历史 log
  initdata: string | object; // 初始化数据（随机种子、地图等）
}

export type JudgeLogEntry =
  | { output: JudgeOutput; verdict: Verdict; memory: number; time: number }  // 裁判回合
  | Record<string, { response: string; verdict: Verdict; memory: number; time: number }>; // 玩家回合
```

### 4.4 裁判输出

```typescript
export interface JudgeOutput {
  command: 'request' | 'finish';
  content: Record<string, string | number>; // key 为数字字符串 "0","1"...
  display: string | object;
  initdata?: string | object;  // 仅第一回合有效，用于保存随机种子等
}
```

### 4.5 Verdict

```typescript
export type Verdict =
  | 'OK'   // 正常
  | 'TLE'  // 超时
  | 'MLE'  // 超内存
  | 'RE'   // 运行时错误
  | 'CE'   // 编译错误
  | 'NJ'   // Judge 输出非 JSON
  | 'SE'   // 系统错误
  | 'NR';  // 正常退出（无判决）
```

---

## 五、MatchRunner 主流程

```typescript
export class MatchRunner {
  async run(task: Task): Promise<GameResult> {
    const { game, callback, runMode = 'restart' } = task;
    const strategy = RunStrategyFactory.create(runMode); // 策略工厂

    // ── 1. 编译阶段 ──────────────────────────────────
    const compiled: Record<string, CompiledBot> = {};
    for (const [id, code] of Object.entries(game)) {
      compiled[id] = await this.compileService.compile(code);
      if (compiled[id].verdict !== 'OK') throw new CompileError(id, compiled[id]);
    }

    // ── 2. 初始化 Bot 上下文 ──────────────────────────
    const bots: Record<string, BotContext> = {};
    const history: BotInput[] = {};   // 每个 Bot 独立维护历史
    const data: Record<string, string> = {};
    const globaldata: Record<string, string> = {};
    const roundLog: LogEntry[] = [];

    for (const id of Object.keys(game)) {
      bots[id] = new BotContext(compiled[id], strategy);
      history[id] = { requests: [], responses: [], data: '', globaldata: '' };
    }

    // ── 3. 对局循环 ────────────────────────────────────
    let judgeInput: JudgeInput = { log: [], initdata: task.initdata ?? '' };
    let judgeOutput: JudgeOutput;

    for (let round = 1; ; round++) {
      // 3a. 运行裁判
      judgeOutput = await this.runJudge(bots['judger'], judgeInput);
      roundLog.push({ type: 'judge', output: judgeOutput, round });

      // 3b. 裁判保存 initdata（第一回合）
      if (round === 1 && judgeOutput.initdata !== undefined) {
        judgeInput.initdata = judgeOutput.initdata;
      }

      if (judgeOutput.command === 'finish') break;

      // 3c. 运行各玩家
      const playerOutputs: Record<string, BotOutput> = {};
      for (const [playerId, request] of Object.entries(judgeOutput.content)) {
        const bot = bots[playerId];
        const hist = history[playerId];

        hist.requests.push(request as string);
        const input: BotInput = {
          ...hist,
          data: data[playerId] ?? '',
          globaldata: globaldata[playerId] ?? '',
          timeLimit: game[playerId].limit.time,
          memoryLimit: game[playerId].limit.memory,
        };

        const output = await strategy.runRound(bot, input);
        playerOutputs[playerId] = output;

        // 更新历史和持久化数据
        hist.responses.push(output.response);
        data[playerId]       = output.data ?? '';
        globaldata[playerId] = output.globaldata ?? globaldata[playerId] ?? '';
        roundLog.push({ type: 'player', id: playerId, output, round });
      }

      // 3d. 更新裁判输入（加入玩家 responses）
      judgeInput.log = buildJudgeLog(roundLog);
    }

    // ── 4. 清理 ────────────────────────────────────────
    for (const bot of Object.values(bots)) {
      await strategy.cleanup(bot);
    }

    // ── 5. 回报 ────────────────────────────────────────
    const result: GameResult = {
      scores: judgeOutput.content as Record<string, number>,
      log:    roundLog,
    };
    await this.callbackService.finish(callback.finish, result);
    return result;
  }
}
```

---

## 六、API 设计

```typescript
// POST /v1/judge
export interface Task {
  game: Record<string, Code>;       // key: "judger" | "0" | "1" | ...
  callback: { update: string; finish: string };
  initdata?: string | object;       // 新增：对局初始化数据
  runMode?: 'restart' | 'longrun'; // 新增：Bot 运行模式（默认 restart）
}

export interface Code {
  language: string;
  source: string;
  limit: { time: number; memory: number };
}
```

---

## 六·五、OJ 评测策略（与 Botzone 并列实现）

OJ 比 Botzone 简单得多，完全复用同一套编译缓存 + nsjail 沙箱，只是交互方式不同。现在设计进去，不留技术债。

### OJ Task 格式

```typescript
export interface OJTask {
  type: 'oj';
  code: Code;                    // 用户提交的代码（一份）
  testcases: Testcase[];         // 测试点列表
  checker?: Code;                // special judge checker（可选）
  callback: Callback;
}

export interface Testcase {
  id: number;
  input: string;                 // stdin 内容
  expectedOutput?: string;       // 期望 stdout（普通题）
  timeLimit?: number;            // 可覆盖全局限制
  memoryLimit?: number;
}
```

### OJStrategy 实现思路

```
for each testcase:
  1. 启动已编译的用户程序（nsjail）
  2. 写 testcase.input 到 stdin
  3. 读 stdout（time/memory 限制内）
  4. 判断 verdict：
     - 无 checker：直接 diff（去除末尾空白）
     - 有 checker：把 input/output/expected 传给 checker，读 checker 输出
  5. 记录 verdict + 资源用量
```

### 通用入口

```typescript
// POST /v1/judge 统一入口
switch (task.type) {
  case 'botzone': return botzoneRunner.run(task as BotzoneTask);
  case 'oj':      return ojRunner.run(task as OJTask);
}
```

两者共享：`CompileService`、`NsjailService`、`DataStore`、`CallbackService`。

---

## 七、实现路线

### Phase 1 — 官方协议兼容（优先）
- [ ] `RestartStrategy`：每轮重启 + 完整历史 JSON 输入
- [ ] `BotOutput` JSON 解析（response/data/globaldata/debug）
- [ ] `data` 回合持久化（内存）
- [ ] `globaldata` 跨对局持久化（数据库/文件）
- [ ] 裁判 `initdata` 支持
- [ ] 玩家 ID 改为数字字符串（"0","1"...）

### Phase 2 — 简化交互模式
- [ ] 检测 Bot 是否使用简化交互（首行是否为数字）
- [ ] `SimplifiedInputFormatter`：生成 `n\nreq1\nresp1\n...\nreq_n\ndata\n`

### Phase 3 — 长时运行（性能优化）
- [ ] `LongRunStrategy`：检测 Bot 输出中的 `>>>BOTZONE_LONGRUN` 指令
- [ ] SIGSTOP/SIGCONT 进程复用
- [ ] 超时/崩溃时降级到 RestartStrategy

### Phase 4 — 完善
- [ ] ELO 排名更新接口
- [ ] 对局 Log 公开存储
- [ ] 编译缓存持久化（Redis/文件）

---

## 八、与现有代码的关系

| 模块 | 保留 | 重写 | 新增 |
|------|------|------|------|
| Pipeline (编译) | ✅ | — | — |
| nsjail 沙箱 | ✅ | — | — |
| LRU 文件缓存 | ✅ | — | — |
| God / MatchRunner | — | ✅ | — |
| Bot 生命周期 | — | ✅ | `IBotRunStrategy` |
| Bot 输入/输出协议 | — | ✅ | `BotInput`/`BotOutput` |
| data/globaldata | — | — | ✅ |
| initdata | — | — | ✅ |
| 简化交互 | — | — | ✅ |

---

*设计时间：2026-03-09*

# Worker 开发指南

新建一个高频定时任务 Worker 时，参照以下规范。

**参考项目：**
- `pendle-market-making`：完整的 DO + Alarm 链、http.ts、WorkerEnv/Env 双类型、extractErrorMessage
- `pendle-order-watch`：Pushover 通知、Effect.gen 业务逻辑、多市场 wrangler 部署

---

## 技术栈

- **运行时**：Cloudflare Workers
- **语言**：TypeScript（strict mode）
- **包管理 / 测试**：Bun（`bun install`、`bun test`）
- **核心库**：Effect（必须深度使用，见下文）、viem、decimal.js
- **部署**：Wrangler

---

## Worker 入口模式

每个 Worker 导出 `fetch` 和 `scheduled` 两个 handler。`scheduled()` 只负责 DO singleton lookup + ensureAlarm，不包含业务逻辑。

```ts
export default {
  async fetch(request: Request, env: WorkerEnv): Promise<Response> {
    // 可选：admin / health check 路由
    return new Response("ok");
  },

  scheduled(
    _event: ScheduledEvent,
    env: WorkerEnv,
    ctx: ExecutionContext,
  ) {
    const id = env.SCHEDULER.idFromName("singleton");
    const stub = env.SCHEDULER.get(id);
    ctx.waitUntil(
      stub.fetch(new Request("https://internal/ensure-alarm", { method: "POST" })),
    );
  },
} satisfies ExportedHandler<WorkerEnv>;
```

**要点：**
- `scheduled()` 中不 await，用 `ctx.waitUntil()` 保持异步
- DO stub 通过 `idFromName("singleton")` 获取，确保全局唯一
- 如需区域亲和性，使用 `env.SCHEDULER.get(id, { locationHint: "oc" })`

---

## Durable Object 完整结构

DO 类负责 alarm 生命周期管理，不包含业务逻辑。完整结构如下：

```ts
const ALARM_INTERVAL_MS = 15_000;
const MIN_INTERVAL_MS = 5_000;

export class SchedulerDO implements DurableObject {
  constructor(
    private ctx: DurableObjectState,
    private env: WorkerEnv,
  ) {}

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/ensure-alarm") {
      // pause flag 同时阻止 alarm 被重新拉起
      const paused = await this.ctx.storage.get<boolean>("paused");
      if (paused) {
        return new Response("paused, alarm not scheduled");
      }
      const currentAlarm = await this.ctx.storage.getAlarm();
      if (currentAlarm == null) {
        await this.ctx.storage.setAlarm(Date.now() + 1000);
      }
      return new Response("ok");
    }

    if (url.pathname === "/pause") {
      await this.ctx.storage.put("paused", true);
      await this.ctx.storage.deleteAlarm();
      return new Response("paused");
    }

    if (url.pathname === "/resume") {
      await this.ctx.storage.put("paused", false);
      await this.ctx.storage.setAlarm(Date.now() + 1000);
      return new Response("resumed");
    }

    // 其他管理路由（可选）：/status 等
    return new Response("not found", { status: 404 });
  }

  async alarm(): Promise<void> {
    const alarmStartedAt = Date.now();
    try {
      // 开头检查 pause flag，已暂停则跳过整个 tick 且不重设 alarm
      const paused = await this.ctx.storage.get<boolean>("paused");
      if (paused) {
        console.log("alarm: paused, skipping tick and not rescheduling");
        return;
      }

      const env = await resolveSecrets(this.env);

      // runTick 是 Effect 程序，通过 Effect.runPromise 桥接到 async
      const result = await Effect.runPromise(
        runTick.pipe(Effect.provide(buildLiveLayer(env))),
      );

      // 记录运行状态到 DO storage
      await this.ctx.storage.put("lastRunAt", new Date().toISOString());
      await this.ctx.storage.delete("lastError");
    } catch (error) {
      const msg = extractErrorMessage(error);
      try {
        await this.ctx.storage.put("lastError", msg);
      } catch (storageError) {
        console.error("alarm: failed to persist lastError:", storageError);
      }
      console.error("alarm: tick error:", msg);
    } finally {
      const elapsedMs = Date.now() - alarmStartedAt;
      console.log(`alarm: tick completed in ${elapsedMs}ms`);
      try {
        await this.scheduleNextAlarm(alarmStartedAt);
      } catch (scheduleError) {
        console.error("alarm: scheduleNextAlarm escaped:", scheduleError);
      }
    }
  }

  private async scheduleNextAlarm(alarmStartedAt: number): Promise<void> {
    try {
      // 再次检查 pause flag：如果在 tick 执行期间被暂停，不重设 alarm
      const paused = await this.ctx.storage.get<boolean>("paused");
      if (paused) {
        console.log("scheduleNextAlarm: paused during tick, not rescheduling");
        return;
      }

      const elapsed = Date.now() - alarmStartedAt;
      const delay = Math.max(MIN_INTERVAL_MS, ALARM_INTERVAL_MS - elapsed);
      await this.ctx.storage.setAlarm(Date.now() + delay);
    } catch (error) {
      console.error("failed to reschedule alarm:", error);
    }
  }
}
```

**要点：**
- `alarm()` 的第一个 `await` 也必须放进顶层 `try`，不要把 `paused` 检查留在 `try` 外面，否则它自己抛错时平台会直接记成 `Exception Thrown`
- `alarm()` 中 try/catch/finally，在 finally 中调度下一个 Alarm，确保链不断裂
- **`finally` 块中打印 tick 耗时**（`alarm: tick completed in ${elapsedMs}ms`），无论成功还是失败都要记录执行时间
- `resolveSecrets()` 在 alarm 开头调用一次，resolved env 向下传递
- 业务逻辑中用 `Effect.tapError` 做旁路通知，错误继续传播到 alarm 层的 catch，确保 `lastError` 正确记录
- `scheduleNextAlarm` 再次检查 `paused`：防止 tick 执行期间调 `/pause` 后仍重设 alarm
- `catch` 和 `finally` 里的 `await` 也不能裸奔；例如写 `lastError` 或重设 alarm 失败，会把原本已捕获的业务错误重新升级成平台层的 `Exception Thrown`
- `scheduleNextAlarm` 不只 `setAlarm` 要保护，前面的 `storage.get("paused")` 也必须包含在同一个 `try` 里

### Cloudflare 日志里的 `Exception Thrown` 是什么

在 Alarm 日志里，`Exception Thrown` 的含义是：这次 `alarm()` 调用最终以**未捕获异常**结束。它不等同于“业务任务失败”。

对 DO Alarm 链来说，最常见的根因不是 `runTick()` 本体，而是以下几类“收尾阶段”异常逃逸到了平台层：

- `try` 外的 `await` 抛错，例如 `alarm()` 开头的 `storage.get("paused")`
- `catch` 内部再抛错，例如写 `lastError` 到 DO storage 失败
- `finally` 内部再抛错，例如 `await scheduleNextAlarm(...)` 期间的存储读取失败

如果日志表现为“先打印 `alarm: tick completed in Xms`，随后平台显示 `Exception Thrown`”，优先怀疑 `finally` 或 `scheduleNextAlarm()` 路径，而不是业务采集逻辑本身。

---

## Effect 使用规范

所有业务逻辑必须使用 Effect 编写，充分利用其核心能力：

- **错误通道**：用 typed error 区分可恢复和不可恢复错误，不要 try/catch 吞掉错误类型
- **并发控制**：用 `Effect.all` 替代 `Promise.all`。默认使用有限并发（如 `{ concurrency: 5 }`），避免触发上游 429 / RPC 限流。仅在明确证明安全时才用 `"unbounded"`
- **重试**：用 `Effect.retry` + `Schedule` 实现退避策略，不要手写重试循环
- **超时**：Effect 管道层面用 `Effect.timeout`（可与 retry 组合）。`Effect.tryPromise` 内部的 `fetch()` 调用可用 `AbortSignal.timeout` 做底层请求取消——这不矛盾，两者作用在不同层次
- **依赖注入**：用 `Context` / `Layer` 管理服务依赖

### 硬规则：错误建模

以下 4 条是**硬规则**，所有项目必须遵循，无例外：

1. **`Effect.tryPromise` 必须显式提供 `catch`**——禁止使用只传单个函数的简写形式。不带 `catch` 会产生 `UnknownException`，丢失错误类型信息
2. **orchestration 层（`runTick` / `Effect.gen` 编排层）禁止直接写 `Effect.tryPromise`**——外部调用必须封装在 repository / service 层，orchestration 只组合已有 service 方法
3. **repository / service / runtime 边界必须把底层异常转换成 typed error**——底层 `UnknownException` 不得穿透到上层
4. **所有穿过 Effect 错误通道的应用内错误必须带 `readonly _tag`**——这样上层才能稳定使用 `Effect.catchTag` / `Effect.catchTags`，不退回 `catchAll` + 字符串匹配

#### 错误类型定义：使用 `Data.TaggedError`

优先使用 `Data.TaggedError` 定义 typed error，不要用裸 `class X extends Error`：

```ts
import { Data } from "effect";

// ✅ 正确：Data.TaggedError，自带 readonly _tag，与 Effect.catchTag 无缝配合
export class FetchOrderbookError extends Data.TaggedError("FetchOrderbookError")<{
  readonly message: string;
  readonly cause?: unknown;
}> {}

export class D1QueryError extends Data.TaggedError("D1QueryError")<{
  readonly message: string;
  readonly cause?: unknown;
  readonly query?: string;
}> {}
```

```ts
// ❌ 错误：裸 Error，没有 _tag，上层只能 catchAll + 字符串匹配
catch: (error) => new Error(`Failed to fetch orderbook: ${toErrorMessage(error)}`)
```

#### 边界层包装示例

```ts
// service 层：Effect.tryPromise + catch 转换为 typed error
const fetchOrderbook = (env: Env) =>
  Effect.tryPromise({
    try: async (): Promise<OrderbookResponse> => {
      const url = `${API_BASE}/orderbook?market=${env.MARKET_ADDRESS}`;
      console.log(`[orderbook] GET ${url}`);
      const start = Date.now();
      const response = await fetchFn(url);
      const elapsed = Date.now() - start;

      if (!response.ok) {
        const body = await response.text();
        const safeBody = body.length > 200 ? body.slice(0, 200) + "..." : body;
        console.log(`[orderbook] ${response.status} in ${elapsed}ms body=${safeBody}`);
        throw new Error(`Orderbook request failed with ${response.status}`);
      }

      const data = await response.json() as OrderbookResponse;
      console.log(`[orderbook] ${response.status} in ${elapsed}ms entries=${data.entries.length}`);
      return data;
    },
    catch: (error) =>
      new FetchOrderbookError({
        message: `Failed to fetch orderbook: ${toErrorMessage(error)}`,
        cause: error,
      }),
  });

// repository 层：D1 查询
const loadActiveTasks = (db: D1Database, sql: string) =>
  Effect.tryPromise({
    try: () => db.prepare(sql).all(),
    catch: (error) =>
      new D1QueryError({
        message: `Failed to load active tasks`,
        cause: error,
        query: sql,
      }),
  });
```

#### 上层精确匹配错误

```ts
// orchestration 层：用 catchTag 精确处理特定错误
program.pipe(
  Effect.catchTag("D1QueryError", (error) =>
    Console.error(`D1 failed: ${error.message}`),
  ),
  Effect.catchTag("FetchOrderbookError", (error) =>
    Console.error(`Orderbook fetch failed: ${error.message}`),
  ),
);
```

### 标准用法示例

#### 1. Effect.gen 编排业务流程

用 `Effect.gen` + generator 编写主业务逻辑。错误处理注意区分"旁路通知"和"错误吞没"：

```ts
const watchTask = (env: Env): Effect.Effect<void> =>
  Effect.gen(function* () {
    const data = yield* fetchData(env);

    if (!shouldProcess(data)) {
      yield* Console.log("skipped: condition not met");
      return;
    }

    yield* executeAction(env, data);
    yield* sendPushover(env, `Action completed for ${data.id}`).pipe(
      // 成功通知失败不影响业务结果
      Effect.catchAll((err) => Console.error(`notification failed: ${toErrorMessage(err)}`)),
    );
  }).pipe(
    // ✅ 正确：tapError 发通知后错误继续传播，alarm 层能捕获并记录 lastError
    Effect.tapError((error) =>
      sendPushover(env, `Task failed: ${toErrorMessage(error)}`).pipe(
        Effect.catchAll((pushoverError) =>
          Console.error(`Failed to send error notification: ${toErrorMessage(pushoverError)}`),
        ),
      ),
    ),
  );
```

**关键区别：**
- `Effect.tapError` — 旁路通知，错误继续传播到 alarm 层，`lastError` 正确记录 ✅
- `Effect.catchAll` — 吞掉错误，alarm 层看到的是成功，`lastError` 不会更新 ❌

只有当你明确要"降级为成功"时才用 `catchAll`（例如：通知发送失败不应阻断业务流程）。

#### 2. retry + timeout 组合

```ts
const fetchWithRetry = Effect.retry(
  Effect.tryPromise({
    try: () => fetchData(),
    catch: (error) => new FetchDataError({ message: toErrorMessage(error), cause: error }),
  }).pipe(
    Effect.timeout("10 seconds"),
  ),
  Schedule.exponential("1 second").pipe(Schedule.compose(Schedule.recurs(3))),
);
```

**反模式：**

```ts
// ❌ 错误：没有 catch，产生 UnknownException
const bad = Effect.tryPromise(() => fetchData());

// ❌ 错误：catch 返回裸 Error，没有 _tag
const alsoBad = Effect.tryPromise({
  try: () => fetchData(),
  catch: (error) => new Error(`failed: ${error}`),
});
```

---

## 运行时架构

```
Cron (每分钟) → Worker.scheduled()
                  ↓
            Durable Object (singleton)
                  ↓
            Alarm (每N秒) → runTick()
                  ↓
        ┌─────────┴─────────┐
   读取配置/外部数据      执行业务逻辑
```

Cloudflare Cron 最小粒度 1 分钟。需要更短间隔时，用 DO Alarm 链实现。Cron 仅作为 Alarm 的保活机制，每分钟调 `ensureAlarm()` 防止 Alarm 链意外中断。

---

## 必须遵循的规范

### 1. DO singleton + Alarm 链

- 用 `idFromName("singleton")` 确保全局唯一实例
- DO 保证 Alarm 串行执行，天然避免并发问题
- `alarm()` 中用 try/catch/finally，在 finally 中调度下一个 Alarm，确保链不断裂
- 完整 DO 类结构见上方 "Durable Object 完整结构" 一节

### 2. 停机与暂停

首选 DO 管理路由实现暂停/恢复：

- **DO `/pause` 路由**：写 DO storage `paused=true` + 删除当前 alarm。`/ensure-alarm`、`alarm()` 开头、`scheduleNextAlarm()` 三处检查 pause flag，Cron 也无法重新拉起 alarm 链。恢复时调 `/resume`
- **`wrangler delete`**：删除 Worker，最彻底的停止方式

可选方案：KV 软停机开关（适用于需要跨多个 Worker 统一控制的场景）：
- KV 中设置 `system_running` 键，值为 `"1"` 时运行
- 这是**软停机**：KV 有传播延迟（通常几秒，极端情况更长），切换后可能还会执行若干个 tick
- `alarm()` 开头读取一次，将结果向下传递，避免同一个 tick 内重复读取 KV
- 不要依赖 KV 做需要立即生效的停机

### 3. Alarm 间隔自适应

扣除 tick 已消耗的时间，保持稳定节奏。设置合理的最小间隔（建议 5 秒），防止慢请求导致间隔坍缩。

`scheduleNextAlarm()` 整体都可能抛错，必须把所有存储读取和 `setAlarm` 放在同一个 `try/catch` 里。只保护 `setAlarm` 还不够，因为前面的 `storage.get("paused")` 一旦失败，Cloudflare 仍会把这次 Alarm 记成 `Exception Thrown`。虽然 Cron 每分钟会通过 `ensureAlarm()` 保活，断裂最多持续到下一次 Cron 触发（最长 1 分钟），但对高频任务来说丢失一分钟的执行仍然不可接受：

```ts
private async scheduleNextAlarm(startedAt: number): Promise<void> {
  try {
    // 检查 pause flag
    const paused = await this.ctx.storage.get<boolean>("paused");
    if (paused) {
      console.log("scheduleNextAlarm: paused, not rescheduling");
      return;
    }

    const elapsed = Date.now() - startedAt;
    const delay = Math.max(MIN_INTERVAL_MS, ALARM_INTERVAL_MS - elapsed);
    await this.ctx.storage.setAlarm(Date.now() + delay);
  } catch (error) {
    console.error("failed to reschedule alarm:", error);
  }
}
```

### 4. 依赖注入

所有外部依赖（API 调用、链上读取、时间函数）通过 Effect `Context` / `Layer` 注入，使 `runTick` 可以在纯单元测试中运行。

`runTick` 是 Effect 程序，返回 `Effect.Effect<TickSummary, TickError, AppDeps>`。DO alarm 层通过 `Effect.runPromise` 桥接到 async 世界：

```ts
// runTick 是 Effect 程序本体
export const runTick = Effect.gen(function* () {
  const taskRepo = yield* TaskRepository;
  const tasks = yield* taskRepo.getActiveTasks();
  // ... 串行处理 tasks
  return summary;
});

// alarm() 中通过 Effect.runPromise 桥接
const result = await Effect.runPromise(
  runTick.pipe(Effect.provide(liveLayer)),
);
```

测试中替换为 fake layer：

```ts
const result = await Effect.runPromise(
  runTick.pipe(Effect.provide(testLayer)),
);
```

### 5. HTTP 代理层

上层代码统一调用 `fetchFn(url, init?)`，不关心底层是直连还是代理。代理切换由 `createSmartFetch` 自动管理。

#### 代理选择策略：直连优先，失败自动切代理（永久锁定）

```
直连 fetch()
  ↓ 403/429/5xx 或网络错误
  ↓ 失败计数 +1（按 host 维度，内存 Map）
  ↓ 连续失败 >= 3 次
锁定该 host → 永久走代理（持久化到 DO storage）
  ↓ 后续请求直接走代理，不再尝试直连
  ↓ relay 轮换：proxy_1 失败 → proxy_2 → ... → 全部失败则抛出错误
```

**核心行为：**
- 默认直连，延迟最低
- 按 host 维度跟踪失败，不同上游独立判断
- **失败计数**在 `createSmartFetch` 内部的内存 Map 中维护，不需要持久化（单个 tick 内即可完成判定）
- **锁定状态**通过 `RelayLock` 接口持久化到 **DO `ctx.storage`**，跨 DO 驱逐和重部署存活
- 连续失败 3 次后，**永久锁定该 host 走代理**，不再尝试直连恢复
- 直连成功时重置失败计数（防止偶发失败误触发锁定）
- 代理请求需携带 `X-Api-Key` header 进行认证，API key 从 Secrets Store 获取（`RELAY_API_KEY`）
- **Relay 轮换**：按 KV 中配置的顺序（`http_proxy_1` → `http_proxy_2` → ...）依次尝试，某个 relay 失败后自动尝试下一个，全部失败才抛出错误

**失败计数与 Effect retry 的协作：**
- 上层调用通过 `withRetryAndTimeout`（默认重试 2 次，共 3 次请求）包装 Effect
- 每次 smartFetch 直连失败都会递增内存计数，因此单个 tick 内 3 次重试即可触发锁定
- 这是预期行为：对于 Cloudflare Challenge（403）等不会自行恢复的错误，在一个 tick 内快速判定并切换代理，避免浪费后续 tick
- 失败计数不需要持久化，因为重试发生在同一个 tick 的同一个 `createSmartFetch` 实例内

#### 什么算"需要切代理的失败"

```ts
function shouldFallbackToRelay(response: Response): boolean {
  return !response.ok && (response.status === 403 || response.status === 429 || response.status >= 500);
}
```

此外，fetch 抛出的网络异常（连接超时、DNS 失败等）也计入失败。

#### RelayLock 接口

锁定状态通过 `RelayLock` 接口抽象，由 DO 端使用 `ctx.storage` 实现：

```ts
export interface RelayLock {
  isLocked(host: string): Promise<boolean>;  // 检查 host 是否已锁定走代理
  lock(host: string): Promise<void>;         // 永久锁定该 host 走代理
}
```

DO storage key 约定：
- `relay:{host}` → `true` 表示该 host 已锁定

#### 实现要点

- `createSmartFetch(config, relayApiKey, relayLock, baseFetch)` — 接收 KV（代理 URL）、API key、RelayLock、可选 baseFetch
- `fetchViaRelay()` — 通过单个代理发送请求，携带 `X-Api-Key` header
- `fetchViaRelayChain()` — 依次尝试所有代理，某个失败后自动尝试下一个，全部失败抛出最后一个错误
- `SchedulerDO` 构造函数中创建 `createStorageRelayLock(ctx.storage)` 并通过 `buildLiveRuntimeLayer` 传入

#### 调用约束

`createSmartFetch` 返回的函数只支持 `(url: string, init?: RequestInit)` 或 `(url: URL, init?: RequestInit)` 的调用方式。**不要传 `Request` 对象**——relay 实现只提取 URL 字符串，会丢失 `method`/`headers`/`body` 等属性。

```ts
// ✅ 正确
await fetchFn("https://api.example.com/data", { method: "POST", body });

// ❌ 错误：Request 对象的 method/headers/body 会丢失
await fetchFn(new Request("https://api.example.com/data", { method: "POST", body }));
```

#### KV 代理配置

代理 URL 从 KV namespace 获取（具体 namespace ID 在各项目 `wrangler.toml` 中配置）：

| Key | 说明 |
|-----|------|
| `http_proxy_1` | 第一优先代理 |
| `http_proxy_2` | 第二优先代理 |

可在 KV 中扩展更多 `http_proxy_N`。

代理的源代码在 /Users/dayucat/CloudflareWorkers/http-proxy-relay
代理使用指南在 /Users/dayucat/CloudflareWorkers/http-proxy-relay/README.md

### 6. 多市场部署

同一份代码通过多个 `wrangler.*.toml` 部署到不同市场。每个 toml 通过 `[vars]` 配置 chain、market 地址等参数。

命名规则：`wrangler.{token}.{expiry}.toml`，例如 `wrangler.susde.858d.toml`。

部署命令：`wrangler deploy -c wrangler.susde.858d.toml`

多 toml 之间的差异点：
- `name`：不同的 Worker 名称
- `[vars]`：不同的 chain ID、合约地址等业务参数
- secrets store bindings 和 DO bindings 通常保持一致

### 7. Secrets 管理

#### wrangler.toml 配置

私钥、RPC URL、API Key 使用 Cloudflare Secrets Store，不要放 `[vars]`。统一使用 store_id `04ac92bcff8644c7906c5b66c7246067`：

```toml
[[secrets_store_secrets]]
binding = "SECRET_KEY"
store_id = "04ac92bcff8644c7906c5b66c7246067"
secret_name = "SECRET_KEY_1"

[[secrets_store_secrets]]
binding = "ETH_NODE_URL"
store_id = "04ac92bcff8644c7906c5b66c7246067"
secret_name = "ETH_NODE_URL"
```

#### WorkerEnv → Env 双类型模式

`WorkerEnv` 是 Wrangler 注入的原始环境，secret 字段类型为 `SecretsStoreSecret`（handle，需要异步 `.get()` 才能获取值）。`Env` 是 resolved 后的环境，secret 字段类型为 `string`，供业务逻辑直接使用。

```ts
/** Worker binding env — secrets 是 SecretsStoreSecret handle */
export interface WorkerEnv {
  SCHEDULER: DurableObjectNamespace;
  HTTPPROXYCONFIG: KVNamespace;
  SECRET_KEY: SecretsStoreSecret;
  ETH_NODE_URL: SecretsStoreSecret;
  PUSHOVER_TOKEN: SecretsStoreSecret;
  PUSHOVER_USER: SecretsStoreSecret;
  // [vars] 中的值直接是 string
  MARKET_ADDRESS: string;
  CHAIN_ID: string;
}

/** Env with secrets resolved to plain strings — 业务逻辑使用这个 */
export interface Env {
  SCHEDULER: DurableObjectNamespace;
  HTTPPROXYCONFIG: KVNamespace;
  SECRET_KEY: string;
  ETH_NODE_URL: string;
  PUSHOVER_TOKEN: string;
  PUSHOVER_USER: string;
  MARKET_ADDRESS: string;
  CHAIN_ID: string;
}
```

#### resolveSecrets()

用 `Promise.all()` 并行解析所有 secret，返回 resolved 的 `Env`。在 DO `alarm()` 开头调用一次，向下传递：

```ts
export async function resolveSecrets(env: WorkerEnv): Promise<Env> {
  const [secretKey, ethNodeUrl, pushoverToken, pushoverUser] = await Promise.all([
    env.SECRET_KEY.get(),
    env.ETH_NODE_URL.get(),
    env.PUSHOVER_TOKEN.get(),
    env.PUSHOVER_USER.get(),
  ]);
  return {
    ...env,
    SECRET_KEY: secretKey,
    ETH_NODE_URL: ethNodeUrl,
    PUSHOVER_TOKEN: pushoverToken,
    PUSHOVER_USER: pushoverUser,
  };
}
```

**要点：**
- `resolveSecrets` 无内部 try/catch——任意 secret 解析失败会导致 `Promise.all` reject，异常传播到 `alarm()` 的 catch 块记录 `lastError`，整个 tick 跳过。下一个 alarm 会自动重试，这是预期行为
- 可选 secret 用 `env.OPTIONAL_SECRET?.get() ?? Promise.resolve(undefined)` 处理
- 每个 tick 只调用一次 `resolveSecrets()`，不要在业务逻辑中重复调用

### 8. 错误信息提取

使用 `extractErrorMessage` 递归提取嵌套错误信息。Effect 库内部的错误结构会通过 `cause`/`error`/`failure`/`left`/`right` 等键嵌套，需要递归遍历才能拿到真正的错误消息。

完整参考实现：

```ts
function extractErrorMessage(error: unknown): string {
  const visited = new Set<object>();

  const visit = (value: unknown): string | undefined => {
    if (typeof value === "object" && value !== null) {
      if (visited.has(value)) return undefined;
      visited.add(value);

      const record = value as Record<PropertyKey, unknown>;

      // 递归遍历 Effect 错误链的标准键
      for (const key of ["cause", "error", "failure", "left", "right"]) {
        const nested = visit(record[key]);
        if (nested) return nested;
      }

      // 遍历 Symbol 键（Effect 内部使用）
      for (const symbol of Object.getOwnPropertySymbols(value)) {
        const nested = visit(record[symbol]);
        if (nested) return nested;
      }

      if (typeof record.message === "string" && record.message.length > 0) {
        return record.message;
      }
    }
    return undefined;
  };

  return visit(error) ?? String(error);
}
```

简化版（适用于 Effect 已处理错误上下文的场景）：

```ts
const toErrorMessage = (error: unknown): string =>
  error instanceof Error ? error.message : String(error);
```

### 9. Pushover 通知

所有 Worker 使用 Pushover 发送运行时通知。标准实现如下：

```ts
export class PushoverError extends Data.TaggedError("PushoverError")<{
  readonly message: string;
  readonly cause?: unknown;
}> {}

const sendPushover = (env: Env, message: string, priority: string = "1") =>
  Effect.tryPromise({
    try: async () => {
      const pushoverUrl = "https://api.pushover.net/1/messages.json";
      console.log(`[pushover] POST message_length=${message.length}`);
      const start = Date.now();
      const response = await fetch(pushoverUrl, {
        method: "POST",
        signal: AbortSignal.timeout(10_000),
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          token: env.PUSHOVER_TOKEN,
          user: env.PUSHOVER_USER,
          message,
          priority,
        }),
      });
      const elapsed = Date.now() - start;
      console.log(`[pushover] ${response.status} in ${elapsed}ms`);
      if (!response.ok) {
        throw new Error(`Pushover request failed with ${response.status}`);
      }
    },
    catch: (error) =>
      new PushoverError({
        message: `Failed to send Pushover notification: ${toErrorMessage(error)}`,
        cause: error,
      }),
  });
```

**要点：**
- Pushover 不走代理，直接用 `fetch`
- 使用 `AbortSignal.timeout` 设置超时，避免通知卡住整个 tick
- credentials 通过 Secrets Store 管理：`PUSHOVER_TOKEN`、`PUSHOVER_USER`
- 通知的错误处理取决于调用场景：

```ts
// 成功通知：发送失败不影响业务结果，用 catchAll 降级
yield* sendPushover(env, successMessage).pipe(
  Effect.catchAll((err) => Console.error(`notification failed: ${toErrorMessage(err)}`)),
);

// 失败通知：用 tapError 旁路发送，不吞掉原始业务错误
const businessLogic = mainTask.pipe(
  Effect.tapError((error) =>
    sendPushover(env, `Task failed: ${toErrorMessage(error)}`).pipe(
      Effect.catchAll((err) => Console.error(`error notification failed: ${toErrorMessage(err)}`)),
    ),
  ),
);
```

---

## decimal.js 使用规范

涉及价格和数量的除法运算必须使用 `decimal.js`，避免 bigint 整除截断导致精度丢失。

### 舍入策略

所有金融计算转回整数时，必须显式指定舍入方向，不依赖 `toFixed()` 的默认行为：

- **计算借款/还款金额** → `ROUND_FLOOR`（向下取整）：偏小意味着少还一点，下次 tick 继续还；偏大可能超出可用额度导致交易 revert
- **计算手续费/保证金** → `ROUND_CEIL`（向上取整）：确保预留充足

### 必须用 decimal.js 的场景

```ts
import Decimal from "decimal.js";

// 价格减去偏移量
const effectivePrice = new Decimal(currentPrice.toString()).minus(PRICE_OFFSET);

// 除法运算（bigint 除法会截断小数部分）
const adjustedDebt = new Decimal(userDebt.toString())
  .mul(WAD.toString())
  .div(effectivePrice)
  .toFixed(0, Decimal.ROUND_FLOOR);  // ← 显式向下取整

const result = BigInt(adjustedDebt);
```

### 可以继续用 bigint 的场景

纯整数乘法和 decimal scaling 不涉及除法截断，继续用 bigint：

```ts
// token amount 的 decimal 位数转换
const scaled = amount * 10n ** BigInt(targetDecimals - sourceDecimals);

// 比较大小
const flashLoanAmount = adjustedDebt < amountIn ? adjustedDebt : amountIn;
```

**原则：乘法和比较用 bigint，除法和减法涉及精度时用 decimal.js。**

---

## 注意事项

### KV 读取优化

KV 读取有延迟且不保证强一致。避免每个 tick 大量读取 KV：
- 配置项用 DO storage 缓存，每分钟从 KV 同步一次
- 同一 tick 内不要重复读取同一个 key

### 及时清理废弃代码

策略变更后及时删除不再调用的函数。git 历史可以找回，不要在代码中保留"以防万一"的死代码。

### 日志规范

所有外部调用（HTTP、RPC）打印请求和响应日志，格式统一：

```ts
// 业务 API（URL 可打印）
console.log(`[kyberswap] GET ${url}`);
console.log(`[kyberswap] ${status} in ${elapsed}ms`);

// RPC 调用（URL 含 API key，禁止打印）
console.log(`[rpc] POST <redacted> method=${rpcMethod}`);
console.log(`[rpc] ${status} in ${elapsed}ms`);
```

**敏感信息 redaction 规则：**

以下内容**禁止**出现在日志中：
- 私钥、API Key、Pushover token 等凭据
- 完整的用户地址（可用 `0xabcd...ef12` 截断形式）
- **RPC endpoint URL**（如 Alchemy/Infura URL，含 API key），只打印 `[rpc] POST <redacted>`
- RPC 错误响应的完整 body（可能包含余额、nonce 等敏感信息），只打印 status code 和前 200 字符

**URL 打印规则：**
- 业务 API URL（KyberSwap、Pushover 等公开 API）：可以打印完整 URL，query 参数中的合约地址是公开链上数据
- RPC endpoint URL：**禁止打印**，因为 URL path 中通常包含 API key（如 `https://eth-mainnet.g.alchemy.com/v2/<api-key>`）

错误响应 body 的安全打印方式（注意：`try` 内部抛出的裸 `Error` 必须在 `catch` 回调中转换为 `Data.TaggedError`，参见第 4 条硬规则）：

```ts
// try 内部——抛原始错误，由 catch 回调统一转换
if (!response.ok) {
  const body = await response.text();
  const safeBody = body.length > 200 ? body.slice(0, 200) + "..." : body;
  console.log(`[service] ${response.status} in ${elapsed}ms body=${safeBody}`);
  throw new Error(`Request failed with ${response.status}`);
}

// catch 回调——必须转成 TaggedError，禁止裸 Error 穿过错误通道
// catch: (error) => new ServiceRequestError({ message: ..., cause: error })
```

Pushover 通知日志只打印消息长度，不打印完整内容：

```ts
console.log(`[pushover] POST message_length=${message.length}`);
```

---

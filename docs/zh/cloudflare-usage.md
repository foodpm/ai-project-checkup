# 边缘平台用量与账单检查（Cloudflare 为例）

> 检查项编号：CF-01 至 CF-03（检查项）、CF-V1 至 CF-V4（上线验证）
>
> 本文由同名运维 skill 转写为文档形态，只保留通用做法与可抄代码，不含任何具体账号、站点或密钥。示例以 Cloudflare Workers + D1 为例，思路可平移到其他 Serverless 平台。

## 1. 这个检查解决什么问题

Serverless 与按量计费平台有一个共同风险：**用量异常不会报错，只会静默烧钱**。典型场景是某个查询、某个爬虫或某个被刷的接口让请求量/数据库扫描行数在几小时内涨几十倍，等账单出来才发现。

本检查只做「查询 + 判定」，**不绑定任何通知渠道**——返回结构化结果，上报方式（飞书/Slack/邮件/日志）由使用方自行接。

## 2. 三类检查

| 编号 | 检查 | 数据源 | 判定条件 | 输出 |
|---|---|---|---|---|
| CF-01 | 突增异常 | GraphQL `workersInvocationsAdaptive` + `d1AnalyticsAdaptiveGroups` 按小时 buckets | 当前小时速率 > 绝对阈值 **且** ≥ 前 3 个完整小时均值 × 5 | `{ spike, level }` |
| CF-02 | 日累计 | 同上，查询窗口扩到业务时区当天零点 | 当日累计请求 ≥ 阈值（默认 100 万） | `{ todayReq }` |
| CF-03 | 昨日账单 | REST Billable Usage API | 昨日全账户成本 > $20（可调） | `{ totalUsd, items[] }` |

阈值存配置表（如 `app_settings`)或任意 KV/配置存储，设为 ≤0 表示关闭该项检查。

## 3. 前置条件

1. API Token 权限：**Account Analytics:Read**（GraphQL）+ **Billing Read**（Billable Usage）。
2. Account ID（24 位 hex）。
3. 若需周期性检查：已配置 cron triggers（如 `"*/5 * * * *"` 用量检查、`"10 6 * * *"` 账单检查）。

## 4. API 使用要点（踩过的坑）

1. GraphQL 端点：`POST https://api.cloudflare.com/client/v4/graphql`，Header `Authorization: Bearer <token>`。
2. **GraphQL Time 标量不接受毫秒**：`new Date(x).toISOString().slice(0, 19) + 'Z'`。
3. 聚合查询 `limit` 必须 ≤ 100；Workers 用 `workersInvocationsAdaptive`（`sum(requests, errors)`），D1 用 `d1AnalyticsAdaptiveGroups`（`sum(readQueries, writeQueries)`），均按 `datetimeHour` 聚合。
4. 业务时区当天零点（UTC ms），示例为 `Asia/Shanghai`（UTC+8）：

```ts
const cst = new Date(Date.now() + 8 * 3600000);
const dayStartMs = Date.UTC(cst.getUTCFullYear(), cst.getUTCMonth(), cst.getUTCDate()) - 8 * 3600000;
```

5. Billable Usage 昨日数据可能延迟更新：**无数据时视为“未更新”静默跳过，不要当成 $0 误报**。
6. 检查逻辑自身要便宜：禁止“每请求写数据库计数”式自监控——监控本身会烧数据库。

## 5. D1 计费避坑清单（检查时同步排查）

D1 按**底层扫描行数**（`rows_read`）计费，不是返回行数。用量暴增时按顺序排查：

1. 找出所有高频 cron（每分钟级）分支里的 SQL，逐条核对索引。
2. `OR` 组合条件必须建复合索引才能被 SQLite OR-optimization 利用（如 `(status, created_at)`）。
3. 项目若用 `SCHEMA_VERSION` 版本守卫跳过 DDL：**新增任何 DDL 必须递增版本号**，否则已有库永远不执行新迁移。
4. 无硬上限的轮询循环（`while` + 上游查询）必须加时间上限。
5. 禁止无 `WHERE` 的批量 UPDATE / 全表 UPDATE。

## 6. 参考实现

### 6.1 按小时拉取请求数与 D1 读写行数

```ts
type Env = Record<string, string>;

async function fetchHourlyBuckets(env: Env): Promise<Array<{ hour: string; req: number; d1: number }>> {
  // GraphQL Time 标量不接受毫秒，必须截断为 ...T05:47:00Z 格式
  const from = new Date(Date.now() - 6 * 3600000).toISOString().slice(0, 19) + 'Z';
  const to = new Date().toISOString().slice(0, 19) + 'Z';
  const query = `query { viewer { accounts(filter: { accountTag: $accountTag }) {
    workersInvocationsAdaptive(limit: 100, filter: { datetimeHour_geq: $from, datetimeHour_lt: $to }, orderBy: [datetimeHour_ASC]) {
      dimensions { datetimeHour } sum { requests errors } }
    d1AnalyticsAdaptiveGroups(limit: 100, filter: { datetimeHour_geq: $from, datetimeHour_lt: $to }, orderBy: [datetimeHour_ASC]) {
      dimensions { datetimeHour } sum { readQueries writeQueries } }
  } } }`;
  const res = await fetch('https://api.cloudflare.com/client/v4/graphql', {
    method: 'POST',
    headers: { authorization: `Bearer ${env.CF_API_TOKEN}`, 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables: { accountTag: env.CF_ACCOUNT_ID, from, to } }),
  });
  const data = await res.json();
  const req = data?.data?.viewer?.accounts?.[0]?.workersInvocationsAdaptive ?? [];
  const d1 = data?.data?.viewer?.accounts?.[0]?.d1AnalyticsAdaptiveGroups ?? [];
  const d1Map = new Map(d1.map((r: any) => [r.dimensions.datetimeHour, (r.sum.readQueries ?? 0) + (r.sum.writeQueries ?? 0)]));
  return req.map((r: any) => ({
    hour: r.dimensions.datetimeHour,
    req: r.sum.requests ?? 0,
    d1: d1Map.get(r.dimensions.datetimeHour) ?? 0,
  }));
}
```

### 6.2 突增与日累计判定

```ts
// 阈值从配置表/KV 读取，缺省时使用默认值；≤0 表示关闭
const DEFAULT_HOURLY_REQUESTS = 20000;
const DEFAULT_HOURLY_D1_ROWS = 200000000;
const MULTIPLIER = 5;

const last = buckets[buckets.length - 1];
const prev = buckets.slice(-4, -1);              // 前 3 个完整小时
const avgReq = prev.reduce((s, b) => s + b.req, 0) / prev.length;
const avgD1 = prev.reduce((s, b) => s + b.d1, 0) / prev.length;
const spikeReq = last.req >= DEFAULT_HOURLY_REQUESTS && last.req >= avgReq * MULTIPLIER;
const spikeD1 = last.d1 >= DEFAULT_HOURLY_D1_ROWS && last.d1 >= avgD1 * MULTIPLIER;
const levelOf = (v: number, t: number) => (v >= t * 10 ? 3 : v >= t * 5 ? 2 : v >= t * 2 ? 1 : 0);
```

判定要求：当前小时**同时**超过绝对阈值与环比倍数才报，避免低基数时误报。

### 6.3 昨日账单

```ts
const date = new Date(Date.now() - 86400000).toISOString().slice(0, 10);
const res = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${env.CF_ACCOUNT_ID}/billable-usage?date=${date}`,
  { headers: { authorization: `Bearer ${env.CF_API_TOKEN}` } },
);
const data = await res.json();
const items = (data?.result?.products ?? [])
  .map((p: any) => ({ product: p.productName, amountUsd: p.amountUsd ?? 0 }))
  .filter((x) => x.amountUsd > 0);
// items 为空 = 昨日数据尚未更新，返回 updated:false，调用方应跳过而不是当成 $0
```

## 7. 输出约定

`checkCfUsage` 返回：

```ts
{
  buckets: Array<{ hour: string; req: number; d1: number }>, // 按小时 UTC
  todayReq: number,        // 业务时区当日累计请求数
  spike: boolean,          // 突增判定
  spikeKind: 'requests' | 'd1' | 'both' | null,
  level: number,           // 0=1x 1=2x 2=5x 3=10x；spike=false 时为 -1
}
```

`checkCfBill` 返回：

```ts
{
  date: string,            // 昨日 YYYY-MM-DD
  totalUsd: number,
  items: Array<{ product: string; amountUsd: number }>, // 按成本降序
  updated: boolean,        // false = 昨日数据未更新（调用方应跳过，勿当 $0）
}
```

## 8. 接线与防重发

- cron 分支里拿到结果后自行决定上报渠道；`spike` 为真时按 `level` 分级。
- 防重发（同一天同级别只报一次）需要调用方持久化「日期 + 级别」，检查函数本身不代管状态。
- 账单检查失败（异常/未更新）不要静默吞掉，至少写一条日志，否则会形成“永远不告警”的盲区。

## 9. 验证步骤

```bash
npm run build && npx wrangler deploy   # 注意：deploy 不会自动 build，必须先 build
# 手动触发一次检查（curl 带 cron token 或临时路由），确认返回 JSON 正常
curl -s "https://<your-domain>/api/usage-check" | head -c 400
```

## 10. 检查清单

- [ ] **CF-V1** GraphQL 查询 `limit` ≤ 100、Time 无毫秒。
- [ ] **CF-V2** 阈值 ≤0 时整段 early-return（不产生 API 调用）。
- [ ] **CF-V3** 只返回判定结果，无通知副作用。
- [ ] **CF-V4** 高频 SQL 全部有索引，DDL 有版本递增；账单“未更新”与“$0”已区分。

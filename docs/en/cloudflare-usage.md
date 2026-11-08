# Edge Platform Usage and Billing Checks (Cloudflare as an example)

> Check IDs: CF-01 to CF-03 (checks), CF-V1 to CF-V4 (launch verification)
>
> This document was converted from a similarly named ops skill into documentation form. It keeps only generic practices and copyable code, with no accounts, sites, or keys. Examples use Cloudflare Workers + D1; the approach transfers to other serverless platforms.

## 1. What this check solves

Serverless and usage-based platforms share one risk: **usage anomalies do not throw errors — they quietly burn money.** A query, a crawler, or an abused endpoint can multiply request volume or database rows scanned within hours, and you only find out when the invoice arrives.

This check only **queries and evaluates**. It binds to no notification channel — it returns a structured result, and the caller decides how to report it (Feishu/Slack/email/logs).

## 2. Three checks

| ID | Check | Data source | Condition | Output |
|---|---|---|---|---|
| CF-01 | Spike detection | GraphQL `workersInvocationsAdaptive` + `d1AnalyticsAdaptiveGroups` hourly buckets | Current hour > absolute threshold **and** ≥ 5× the average of the previous 3 complete hours | `{ spike, level }` |
| CF-02 | Daily total | Same, window expanded to the business-timezone midnight | Today's cumulative requests ≥ threshold (default 1,000,000) | `{ todayReq }` |
| CF-03 | Yesterday's bill | REST Billable Usage API | Yesterday's account cost > $20 (adjustable) | `{ totalUsd, items[] }` |

Store thresholds in a settings table (e.g. `app_settings`) or any KV/config store. A value ≤ 0 disables that check.

## 3. Prerequisites

1. API token permissions: **Account Analytics:Read** (GraphQL) + **Billing Read** (Billable Usage).
2. Account ID (24-character hex).
3. For periodic checks, cron triggers configured (e.g. `"*/5 * * * *"` for usage, `"10 6 * * *"` for billing).

## 4. API notes (lessons learned)

1. GraphQL endpoint: `POST https://api.cloudflare.com/client/v4/graphql` with header `Authorization: Bearer <token>`.
2. **The GraphQL Time scalar does not accept milliseconds:** `new Date(x).toISOString().slice(0, 19) + 'Z'`.
3. Aggregate queries require `limit` ≤ 100; use `workersInvocationsAdaptive` (`sum(requests, errors)`) for Workers and `d1AnalyticsAdaptiveGroups` (`sum(readQueries, writeQueries)`) for D1, both aggregated by `datetimeHour`.
4. Business-timezone midnight in UTC ms (example: `Asia/Shanghai`, UTC+8):

```ts
const cst = new Date(Date.now() + 8 * 3600000);
const dayStartMs = Date.UTC(cst.getUTCFullYear(), cst.getUTCMonth(), cst.getUTCDate()) - 8 * 3600000;
```

5. Yesterday's Billable Usage may update late: **treat "no data" as "not updated" and skip silently — never report it as $0.**
6. The check itself must be cheap: never self-monitor by "writing a counter to the database on every request" — monitoring would then burn the database.

## 5. D1 billing pitfalls (check these at the same time)

D1 bills on **underlying rows scanned** (`rows_read`), not rows returned. When usage spikes, investigate in this order:

1. Find SQL inside every high-frequency cron branch (per-minute) and verify indexes query by query.
2. `OR` conditions require a composite index to benefit from SQLite OR-optimization (e.g. `(status, created_at)`).
3. If the project uses a `SCHEMA_VERSION` guard to skip DDL: **any new DDL must bump the version**, or existing databases never run the new migration.
4. Loops that poll upstream without a hard limit (`while` + upstream query) need a time cap.
5. Forbid batched UPDATE or full-table UPDATE without a `WHERE`.

## 6. Reference implementation

### 6.1 Fetch hourly request counts and D1 read/write rows

```ts
type Env = Record<string, string>;

async function fetchHourlyBuckets(env: Env): Promise<Array<{ hour: string; req: number; d1: number }>> {
  // The GraphQL Time scalar rejects milliseconds; truncate to ...T05:47:00Z
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

### 6.2 Spike and daily-total evaluation

```ts
// Read thresholds from the settings table/KV, falling back to defaults; ≤ 0 disables the check
const DEFAULT_HOURLY_REQUESTS = 20000;
const DEFAULT_HOURLY_D1_ROWS = 200000000;
const MULTIPLIER = 5;

const last = buckets[buckets.length - 1];
const prev = buckets.slice(-4, -1);              // the previous 3 complete hours
const avgReq = prev.reduce((s, b) => s + b.req, 0) / prev.length;
const avgD1 = prev.reduce((s, b) => s + b.d1, 0) / prev.length;
const spikeReq = last.req >= DEFAULT_HOURLY_REQUESTS && last.req >= avgReq * MULTIPLIER;
const spikeD1 = last.d1 >= DEFAULT_HOURLY_D1_ROWS && last.d1 >= avgD1 * MULTIPLIER;
const levelOf = (v: number, t: number) => (v >= t * 10 ? 3 : v >= t * 5 ? 2 : v >= t * 2 ? 1 : 0);
```

Requirement: report only when the current hour exceeds **both** the absolute threshold and the relative multiplier, so low baselines do not produce false positives.

### 6.3 Yesterday's bill

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
// Empty items = yesterday's data is not yet updated; return updated:false and let the caller skip it, not treat it as $0
```

## 7. Output contract

`checkCfUsage` returns:

```ts
{
  buckets: Array<{ hour: string; req: number; d1: number }>, // hourly, UTC
  todayReq: number,        // business-timezone cumulative requests today
  spike: boolean,          // spike verdict
  spikeKind: 'requests' | 'd1' | 'both' | null,
  level: number,           // 0=1x 1=2x 2=5x 3=10x; -1 when spike=false
}
```

`checkCfBill` returns:

```ts
{
  date: string,            // yesterday YYYY-MM-DD
  totalUsd: number,
  items: Array<{ product: string; amountUsd: number }>, // descending by cost
  updated: boolean,        // false = yesterday's data not updated (caller should skip, not read $0)
}
```

## 8. Wiring and duplicate suppression

- In a cron branch, decide the reporting channel after receiving the result; when `spike` is true, grade by `level`.
- Duplicate suppression (report once per date + level) requires the caller to persist "date + level"; the check function manages no state itself.
- Do not swallow billing check failures (exception or not-updated) silently — log at least one line, or you create a blind spot that never alerts.

## 9. Verification

```bash
npm run build && npx wrangler deploy   # deploy does not build automatically; build first
# Trigger the check manually (curl with a cron token or a temporary route) and confirm valid JSON
curl -s "https://<your-domain>/api/usage-check" | head -c 400
```

## 10. Checklist

- [ ] **CF-V1** GraphQL query `limit` ≤ 100; Time has no milliseconds.
- [ ] **CF-V2** A threshold ≤ 0 triggers an early return (no API call).
- [ ] **CF-V3** Returns only the verdict, with no notification side effects.
- [ ] **CF-V4** All high-frequency SQL is indexed, DDL bumps versions, and "not updated" is distinguished from "$0".

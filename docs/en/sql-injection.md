# AI Project Security Baseline: SQL Injection Defense and Testing Guide

> Check IDs: SQ-01 to SQ-10 (release gate), SQ-R1 to SQ-R7 (code review)
>
> Scope: new projects built with AI assistance (web apps, API services, full-stack products).
> Goal: build SQL injection defense into the **design phase** and test for injection **before launch**, not after a breach.
> This is a stack-agnostic baseline. Examples use JavaScript/TypeScript with SQLite/D1, but the principles transfer to any stack.

---

## 0. Five principles to remember

1. **Parameterize by default** — User input reaches SQL only through placeholders (`?` / `$1` / bind parameters). Concatenation is an exception, and exceptions must be explicitly marked and audited.
2. **Identify raw-SQL channels with the type system, never with string markers** (see §2.2 — the most commonly missed trap).
3. **Prefer format allowlists over escaping** — Validate what can be validated (dates, emails, enums); escape only genuinely free-form text.
4. **Do not leak errors** — SQL fragments, schema details, and database messages go to server logs only; clients receive generic copy.
5. **Run an injection test before launch** (static audit + black-box probes) and keep the cases in regression.

---

## 1. Why AI-generated code deserves extra scrutiny

Three recurring hazards in AI-generated data-access layers:

| Anti-pattern | Risk |
|---|---|
| SQL built with template strings | `` `select * from t where email = '${email}'` `` — classic injection |
| **A string marker implementing the "raw SQL" channel** | A magic string like `__RAW__` marks raw insertion, and the executor checks **every** interpolation for the prefix/suffix → user input can forge the marker at both ends and inject (see §2.2) |
| Errors passed straight through | `catch (e) { return error(String(e)) }` — leaks SQL templates and driver messages, helping the attacker map your schema |

---

## 2. Design phase: architectural defense

### 2.1 One and only one data-access entry point

- All SQL executes through a **single** data-access function/wrapper. Business code never writes queries against a raw driver connection.
- The wrapper binds every interpolation by default and exposes no bypass hatch.

```ts
// The single entry point (tagged template or query builder)
const rows = await sql`select id from users where email = ${email} limit 1`;
//        ↑ email becomes a ? placeholder + bind; no amount of malicious content is anything but a literal
```

### 2.2 Raw SQL channels: use class instances, not string markers (high-frequency trap)

Sometimes you genuinely need to splice SQL fragments (dynamic WHERE, allowlisted column expansion). How you mark "this is raw SQL" determines whether it is safe:

```ts
// ❌ Bad: string marker — user input can forge it
const RAW = '__RAW_SQL__';
const unsafe = (t: string) => RAW + t + RAW;
// Executor checks every interpolation: val.startsWith(RAW) && val.endsWith(RAW) → splice raw
// An attacker submits "__RAW_SQL__' or 1=1--__RAW_SQL__" and the check passes. Injection.

// ✅ Good: class-instance marker — impossible to forge via JSON or form data
class RawSqlFragment { constructor(public text: string) {} }
const unsafe = (t: string) => new RawSqlFragment(t);
// Executor: if (val instanceof RawSqlFragment) splice(val.text) else bind(val)
```

Test: **the attacker controls only string values. Any design where a value itself decides whether it takes the raw path is a hole.** The same applies in Python/Java/Go — use dedicated types/structs, not magic strings or agreed prefixes and suffixes.

### 2.3 Dynamic identifiers (table/column/sort field): allowlist mapping, never escaping

Identifiers cannot be parameterized, so allowlist them:

```ts
const SORTABLE = { created: 'created_at', price: 'price' } as const;
const col = SORTABLE[req.query.sort] ?? 'created_at';   // fall back to the default
```

Never splice user input into `order by`, table names, or column names — escaping quotes does nothing against identifier injection.

### 2.4 Special cases

| Case | Approach |
|---|---|
| `LIKE` search | Pass the value as a parameter; escape `%` and `_` in user input (`\%` `\_`) to prevent wildcard data dumping |
| `IN (...)` lists | Pass the array as one parameter and let the wrapper expand it into N `?` placeholders; never join it into a string |
| Date/time ranges | Validate the format first (e.g. `^\d{4}-\d{2}-\d{2}`); **the validator's fallback must not pass invalid input through** — return null or an error instead |
| Enum parameters (status/type) | `if (!['a','b'].includes(v)) reject`, then splicing is safe |
| Free-form text that must be spliced | Use the single centralized escape function (see §2.5) |

### 2.5 Exactly one escape function

- Doubling single quotes (`'` → `''`) is correct literal escaping in SQLite/MySQL; SQLite has **no** backslash escaping.
- Define one `q(s)`-style function project-wide and grep its call sites in review. Scattered `.replace(/'/g, "''")` calls are how leaks happen.
- Escaping is the fallback; parameterization is the norm. Every `q()` call should answer "why can't this be parameterized?"

### 2.6 Blast-radius control (defense in depth)

- Least-privilege database accounts: read-write for application traffic, read-only for reporting/analytics.
- Disable multi-statement execution (most drivers already do; confirm).
- Separate auth tiers for admin and public endpoints. Admin queries are still parameterized — **"there is a login wall" is not a reason to skip safety**; injection is the last line of defense if an admin session is stolen or a gate is misconfigured.
- Put unauthenticated endpoints (telemetry, forms, callbacks) at the top of the audit list: they are the only injection surface attackers can reach without credentials.

---

## 3. Code review checklist

- [ ] **SQ-R1** Search every `\$\{` inside SQL strings: placeholder or concatenation?
- [ ] **SQ-R2** Search for string addition into SQL such as `+ email` or `+ id` (common in legacy code)
- [ ] **SQ-R3** Is the raw-SQL channel implemented with a type/class instance or a string marker? String markers must be reworked
- [ ] **SQ-R4** Is there a naked path from user input to SQL with no case normalization or format validation? (trim/lowercase happening to break a payload is luck, not defense)
- [ ] **SQ-R5** Does any catch block return `error.message` to the client? Database errors must become generic copy
- [ ] **SQ-R6** Do date/enum validator fallback branches pass invalid input through?
- [ ] **SQ-R7** Do sort, filter, and pagination parameters go through an allowlist or a hard type conversion (`Number()` before splicing)?

---

## 4. Test methodology: pre-launch injection testing

### 4.1 Static audit (10 minutes, do this first)

Run these searches over the data-access wrapper and all business query files, then confirm each hit manually:

```text
\$\{             → interpolation inside SQL template strings
\.prepare\(      → direct driver calls that bypass the wrapper
unsafe\(|raw\(|RAW → every use of the raw-SQL channel
where \+=|whereStr \+= → dynamic WHERE concatenation
ilike|like '%   → LIKE concatenation
String\(err(or)?\) → error text flowing back into responses
```

### 4.2 Black-box probes

Send requests to a deployed site, one probe per case, judging by status code and response differences. **Only test projects you are authorized to test.**

**Batch 1: classic payloads (verify parameterization)** — expected to match a normal-input response:

```text
x'||(select 1)--@example.com
' or '1'='1
1; drop table x--
%"@example.com"@example.com
```

Judgment: login/form endpoints should return the same response as a "normally invalid input" (e.g. 401/400). A 500, a clear timing change, or a different body → concatenation exists. Stop immediately.

**Batch 2: framework-specific probes (verify raw-channel marking)** — craft forged input against your own "raw SQL" mechanism:

- With a string marker: put the marker at both ends of the input with a quote in between, e.g. `__MARKER__'__MARKER__`
- Judgment: **200 + a normal business response = safe** (the value was stored as a literal); **500 with SQL fragments in the error = confirmed injection**

**Batch 3: unauthenticated endpoints first** — telemetry, callbacks, public search, forms. These need no account, so any finding is automatically high severity.

### 4.3 Testing red lines (do not turn a probe into an attack)

- ❌ Do not run expensive payloads in production (e.g. a recursive CTE producing millions of rows: `with recursive c(x) as (select 1 union all select x+1 from c where x<5000000)`) — that is a CPU-burning DoS technique and will take your own service down.
- ❌ Do not run any injection test against a site you do not own or have authorization for.
- ✅ Choose **single-request, side-effect-free** probes: read endpoints first; for write endpoints, send only the minimal payload whose success/failure is observable.
- ✅ Send a normal baseline request first and judge by the **difference**, not the absolute value.

### 4.4 Verdict quick reference

| Observation | Conclusion |
|---|---|
| Response matches baseline (401/400/200 with normal copy) | Parameterization works — pass |
| 500 with SQL fragments/table names in the error | Concatenation hole plus error leakage |
| Noticeably slower response (time-based blind) | Possibly an injectable expression |
| Unstable results for the same payload | Investigate caching/retry logic before concluding |

### 4.5 Regression and continuous protection

- Turn the payloads you tested into automated cases (each entry point × each probe type) and run them in CI or as a post-deploy smoke test.
- Monitoring backstops: anomalous database scan rows/request volume (injection often causes full scans), 5xx spikes, and clustering of SQL errors in application logs.
- Re-run the static audit after dependency upgrades: the data-access wrapper is code too and can be broken.

---

## 5. Pre-launch checklist (one page)

- [ ] **SQ-01** Single data-access entry point, parameterized by default
- [ ] **SQ-02** Raw-SQL channel identified by type/class instance (not a string marker)
- [ ] **SQ-03** Identifiers/sort/enums handled by allowlists
- [ ] **SQ-04** Formatted parameters (dates, etc.) validated before use; fallbacks do not pass input through
- [ ] **SQ-05** Every free-text splice goes through the centralized `q()` escape
- [ ] **SQ-06** Error responses are generic; SQL details go to logs only
- [ ] **SQ-07** Unauthenticated endpoints have passed batch 2 and 3 probes
- [ ] **SQ-08** The six static-audit searches have zero unconfirmed hits
- [ ] **SQ-09** Probe cases are captured in regression
- [ ] **SQ-10** Database accounts are least-privilege and usage-anomaly alerts are in place

---

## Appendix: one sentence to remember

> **"User input decides whether the raw SQL channel is taken" is the common shape of every injection hole.**
> Parameterization stops 99% of injection. The remaining 1% hides in "the framework's own raw-SQL mechanism" and "naked concatenation in admin or unauthenticated endpoints" — probe those two places first.

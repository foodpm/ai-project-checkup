# AI Project Checkup · ai-project-checkup

[中文](README.md) | **English**

> A pre-launch checklist and methodology collection for AI-built projects. Six checkups: attack surface, SQL injection, backend performance, overseas compliance, edge-platform usage, and domain history.

Thirty minutes before launch is cheaper than three days of firefighting after.

---

## Table of contents

- [What this repository solves](#what-this-repository-solves)
- [Who it is for](#who-it-is-for)
- [What is included](#what-is-included)
- [How to use it](#how-to-use-it)
  - [Option 1: hand the prompt to an AI (recommended)](#option-1-hand-the-prompt-to-an-ai-recommended)
  - [Option 2: the compact prompt](#option-2-the-compact-prompt)
  - [Option 3: manual check](#option-3-manual-check)
- [How to record results](#how-to-record-results)
- [Repository layout](#repository-layout)
- [How the six checkups fit together](#how-the-six-checkups-fit-together)
- [Acknowledgements](#acknowledgements)
- [Disclaimer](#disclaimer)
- [License](#license)

---

## What this repository solves

AI coding tools ship features fast, but by default they do not consider:

- What happens when someone scans for `/.env`, probes backend endpoints, or forges OAuth credentials;
- Whether an AI-written SQL query is parameterized or string-concatenated;
- Whether an admin list page scans millions of rows and quietly inflates the bill;
- Whether age verification, unsubscribe, auto-renewal, and DMCA have hard failures for overseas users;
- How to detect abused serverless usage or a query that falls over before the invoice arrives;
- Whether a second-hand domain previously hosted gambling, phishing, or spam content — a reputation baggage that drags down SEO and ad reviews.

The common shape: **invisible during development, visible after launch as an incident or an invoice.** This repository turns those concerns into numbered, executable, evidence-producing checks, plus prompts you can hand directly to an AI assistant.

## Who it is for

- Independent developers building side projects, SaaS, overseas sites, or subscription products with AI;
- Small teams without dedicated security, compliance, or DBA staff;
- Anyone doing a pre-launch review or a periodic project checkup;
- Engineering leads who want an "launch gate" in their AI coding workflow.

## What is included

| Document | IDs | What it solves | Source |
|---|---|---|---|
| [Attack Surface Checklist](docs/en/attack-surface.md) | AS-01 – AS-21 | 21 techniques from file scanning and endpoint probing to supply-chain attacks, in four difficulty tiers | Jike post (see [Acknowledgements](#acknowledgements)) |
| [SQL Injection Defense and Testing](docs/en/sql-injection.md) | SQ-01 – SQ-10, SQ-R1 – SQ-R7 | The most common injection surface in AI-generated code: raw SQL channels, dynamic identifiers, error leakage | — |
| [Backend Performance Baseline](docs/en/backend-performance.md) | BP-01 – BP-13 | Indexes, snapshots, caching, admin queries, upsert, and data governance | — |
| [Overseas Compliance Checklist](docs/en/overseas-compliance.md) | CO-01 – CO-06, CO-P0/P1/P2 | Age verification, fonts, session replay, email unsubscribe, auto-renewal, DMCA | X post (see [Acknowledgements](#acknowledgements)) |
| [Edge Platform Usage Check](docs/en/cloudflare-usage.md) | CF-01 – CF-03, CF-V1 – CF-V4 | Request spikes, daily totals, yesterday's bill; D1 billing by rows scanned | — |
| [Domain History Checkup](docs/en/domain-history.md) | DH-01 – DH-03 | Investigate a second-hand domain's past with the Wayback Machine before handover: snapshot timeline, suspicious content scan, previous-site identification | — |

Chinese originals live in [docs/zh/](docs/zh/). IDs and structure map one-to-one.

## How to use it

### Option 1: hand the prompt to an AI (recommended)

Copy the whole block below (GitHub code blocks have a copy button in the top-right corner) and paste it into Cursor, Claude Code, Trae, or any AI assistant that can read your code. It will walk the checklist against your current project.

```text
You are my "AI Project Inspector". Run a pre-launch checkup on the project I currently have open, using the checklist in the ai-project-checkup repository.

Repository: https://github.com/foodpm/ai-project-checkup

Steps:
1. Read the repository README and the six documents under docs/en/:
   attack-surface.md, sql-injection.md, backend-performance.md,
   overseas-compliance.md, cloudflare-usage.md, domain-history.md.
2. First profile the project: stack, presence of frontend/backend/database/
   payments/email/user uploads/third-party scripts. Decide which documents
   apply and which do not.
3. Check every numbered item (AS-01…AS-21, SQ-01…SQ-10, BP-01…BP-13,
   CO-01…CO-06, CF-01…CF-03, DH-01…DH-03). Give exactly one verdict per item:
   Pass / Fail / Pending verification / Not applicable.
4. Every verdict needs evidence: file:line, a config key, or command output.
   If you cannot confirm it, write "Pending verification" — do not guess.
5. Do not change code yet. Produce a checkup report sorted by priority:
   P0 (must fix before launch), P1 (within 30 days after launch),
   P2 (ongoing governance). For each Fail, give the minimal fix and how to verify it.
6. End the report with a "10-minute manual checklist" containing only the items
   in this project that genuinely need a human to confirm.

If you cannot access the network, tell me and I will paste the six documents.
```

After receiving the report, clear the P0 items first. Then ask the AI to implement each "minimal fix" one at a time, running the verification after each change.

### Option 2: the compact prompt

Use this when time is short or you only need a "can we launch today?" answer:

```text
Using the checklist at https://github.com/foodpm/ai-project-checkup,
run a minimal pre-launch checkup on my current project. Check only three things:
1. Every item in the attack-surface document's "10-minute pre-launch checklist";
2. SQ-01 through SQ-10 in the SQL document;
3. The BP items directly related to this change in the performance document.
For each item: Pass / Fail / Pending verification / Not applicable,
plus evidence (file:line) and the minimal fix. Do not change code; report only.
```

### Option 3: manual check

For those who prefer not to hand code to an AI, or who need retained audit evidence.

1. Open the relevant document and start from its "Pre-launch checklist" or "Quick checklist" — no need to read everything first.
2. For each item, record the ID, verdict, and evidence (file:line / command output / screenshot).
3. Fix failures per the document's "Control" or "Rules", then re-verify with its "Acceptance tests".
4. Run a full pass quarterly or after a major release. Re-check compliance items on the P2 cadence in the document.

## How to record results

Keep one table per project (an AI can generate the Markdown for you):

| ID | Verdict | Evidence | Fix | Owner | Re-review |
|---|---|---|---|---|---|
| AS-01 | Pass | 6 paths all 404, screenshot attached | — | — | 2027-02-08 |
| SQ-02 | Fail | `src/db.ts:120` uses a string marker | Switch to a class instance | @dev | 2026-11-15 |
| CO-05 | Pending | First-screen screenshot pending | — | @pm | 2026-11-20 |

Use only four verdicts: Pass / Fail / Pending verification / Not applicable (with a reason). "It's in the privacy policy" and "the vendor dashboard has a toggle" do not count as a pass.

## Repository layout

```text
ai-project-checkup/
├── README.md                  # Chinese README
├── README.en.md               # English README (this file)
├── LICENSE                    # MIT
└── docs/
    ├── zh/
    │   ├── attack-surface.md
    │   ├── sql-injection.md
    │   ├── backend-performance.md
    │   ├── overseas-compliance.md
    │   ├── cloudflare-usage.md
    │   └── domain-history.md
    └── en/
        ├── attack-surface.md
        ├── sql-injection.md
        ├── backend-performance.md
        ├── overseas-compliance.md
        ├── cloudflare-usage.md
        └── domain-history.md
```

## How the six checkups fit together

| Stage | Documents | Purpose |
|---|---|---|
| Taking over a second-hand domain | Domain history | Learn the domain's past and avoid inheriting a reputation baggage |
| Before writing code | Backend performance, SQL injection | Design in indexes, parameterization, and snapshots to avoid rework |
| Feature-complete, pre-launch | Attack surface, SQL injection, overseas compliance | Clear the security and compliance gate; P0 must be zero |
| First week after launch | Edge platform usage, performance | Confirm no usage anomaly, slow query, or bill shock |
| Every quarter | All six | Full review; refresh evidence and verdicts |

## Acknowledgements

- The attack surface checklist is inspired by this Jike post: <https://m.okjike.com/originalPosts/6aec2ba5b8cb8d00dd8d4e7a?s=ewoidSI6ICI1OTE3YTE3Y2M5ZjUzNDAwMTE1YzQzNGUiCn0=>
- The overseas compliance checklist is inspired by this X post: <https://x.com/liangzhu_ai/status/2113894763221176515?a=_PQxdz11i6zyXb8fjTg&s=46>
- The remaining documents cite no external source; they are consolidated general engineering practice, and no attribution is invented.

## Disclaimer

This repository is a checklist and methodology collection. It is **not a security guarantee and not legal advice.** It helps surface common problems but does not replace penetration testing, code audit, or qualified legal review. Have a human verify critical conclusions before launch. For high-risk data processing, children's products, healthcare, finance, UGC platforms, or significant cross-border operations, consult a professional in the relevant jurisdiction.

## License

[MIT](LICENSE)

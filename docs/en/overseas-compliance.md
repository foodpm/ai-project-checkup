# Overseas Website Compliance Audit and Launch Checklist

> Check IDs: CO-01 to CO-06 (six high-risk areas), CO-P0-01 to CO-P2-06 (launch gates)
>
> Version 1.0 · Updated 2026-11-08
>
> Idea source: this X post <https://x.com/liangzhu_ai/status/2113894763221176515?a=_PQxdz11i6zyXb8fjTg&s=46>. Rewritten as a generic template; it contains no specific operating entity, domain, or account details.
>
> Audience: sites, web apps, SaaS, subscription products, and platforms with user content serving overseas users
> Use: source audit, remediation acceptance, launch gate, periodic review

## 0. How to use and how to record verdicts

This is a **generic template** aimed at no particular entity. Ticking boxes does not prove compliance, and you cannot claim the six risks are fixed from this checklist alone. You must verify against your project's actual source, configuration, pages, and backend records.

Each item takes exactly one status:

| Status | Meaning |
|---|---|
| Pass | Code, configuration, screenshot, or backend record proves the control is in effect |
| Fail | A gap was found; remediation is required |
| Pending verification | Not enough evidence yet |
| Not applicable | A documented reason proves the rule does not apply |

Never treat "a statement is on the page", "the privacy policy mentions it", or "the vendor dashboard has a toggle" as a pass on its own. Verify the actual data flow, server behavior, user interface, and retained evidence.

## 1. The six legal risks

| ID | Risk | Default severity | Primary rules |
|---|---|---:|---|
| CO-01 | No age verification at registration | P0 | COPPA, GDPR, UK GDPR |
| CO-02 | Fonts hosted remotely by third parties | P1 | GDPR/ePrivacy, CCPA/CPRA, contract and privacy disclosure |
| CO-03 | Session replay without consent or capturing sensitive input | P0 | GDPR/ePrivacy, CCPA/CPRA, HIPAA and similar context rules |
| CO-04 | Marketing email missing unsubscribe link or postal address | P0 | CAN-SPAM, ePrivacy, UK PECR |
| CO-05 | Subscription terms not disclosed next to the button | P0 | ROSCA, FTC Act, state auto-renewal laws, EU consumer law |
| CO-06 | No DMCA designated agent registered | P1 | 17 U.S.C. § 512, 37 C.F.R. § 201.38 |

## 2. Remediation specs and acceptance

### CO-01 Age verification at registration

#### Required

1. Determine age before creating an account, collecting email, name, phone, marketing consent, or any other personal data.
2. Age input must be neutral: never default to adult, and never use leading copy that encourages misreporting.
3. Age determination must happen server-side. Client validation only improves UX; it is not a security boundary.
4. Users below the minimum age in the target market must not create an account or enter further personal-data collection.
5. If children are allowed, obtain verifiable parental consent (VPC) under COPPA and implement data minimization, parental access, and deletion.
6. Retain the minimum age information needed for the purpose. Prefer "age band / eligibility" plus a timestamp over storing a full date of birth long-term.
7. Cover every registration entry: email, Google/Apple/other SSO, invite, mobile, and API registration.
8. Existing accounts confirmed to belong to underage users need a suspension, deletion, and appeal process.

#### Market thresholds

| Market | Default rule | Implementation |
|---|---|---|
| United States | COPPA focuses on children under 13 | For child-directed, mixed-audience, or actual-knowledge collection, give notice and obtain VPC before collection |
| EU/EEA | GDPR consent age defaults to 16 | Member states may lower it to no less than 13; configure per user country and verify parental consent |
| United Kingdom | 13 is typical for consent | Also run an ICO children's code risk assessment for services possibly accessed by under-18s |
| Other markets | Confirm market by market | Never substitute one global threshold for local legal analysis |

#### Suggested copy

> Please confirm your date of birth. We use it to determine whether you can create an account and retain only the necessary age result. Users below the applicable age cannot create an account.

#### Acceptance tests

- Test dates below, exactly at, and above the threshold; server results are correct.
- A below-threshold request creates no user, session, subscription, or marketing-contact record.
- Calling the registration API directly, bypassing the front end, is still blocked.
- All SSO and invite entries enforce the same rule.
- Logs, analytics events, and error reporting contain no full date of birth.
- If a children's path is enabled, VPC, parental rights, and deletion flows are demonstrably runnable.

---

### CO-02 Self-host fonts

#### Required

1. Inventory font requests across pages, stylesheets, email templates, component libraries, and third-party pages.
2. Remove remote font dependencies: Google Fonts, Adobe Fonts, Typekit, font CDNs.
3. Convert or download legitimately obtained font files to local WOFF2 on your own domain or a controlled static-asset domain.
4. Retain the font license, scope of authorization, source, version, and redistribution proof.
5. Update CSP to allow only your own font source:

```http
Content-Security-Policy: font-src 'self';
```

6. Email clients rarely support custom web fonts reliably; marketing email should use safe fallbacks so fonts do not introduce remote tracking assets.
7. If the privacy policy disclosed a font vendor, update it; re-check vendor DPAs and cross-border transfer inventories as well.

#### Acceptance tests

- Page loads request nothing from `fonts.googleapis.com`, `fonts.gstatic.com`, or other remote font domains.
- All remote font `@import`, `preconnect`, `preload`, and CSS references are gone.
- Fonts still load offline and when third-party domains are blocked.
- CSP reports show no font violations and no new remote font fallbacks.
- A font license inventory exists and production font files match the licensed version.

---

### CO-03 Session replay

#### Default

**Disable session replay by default.** It records interactions, inputs, click paths, and potentially sensitive information, and must not be enabled without a clear business need and legal basis.

#### If it must be enabled

1. Obtain explicit, revocable consent before the replay SDK loads; do not show a prompt after recording has begun.
2. Mask all inputs by default; do not rely on per-field configuration.
3. Fully exclude payment, password, login, CAPTCHA, health, identity-document, children's-data, direct-message, and account-settings pages.
4. Never capture keystrokes, clipboard, file-upload contents, or raw form text before submission.
5. Use the vendor's strictest masking policy and add business-level `data-sensitive` markers as a second layer.
6. Define retention, authorized access, audit logs, deletion, and vendor subprocessors.
7. Sign a DPA; confirm data storage region, cross-border transfer mechanism, and law-enforcement request handling.
8. Stop recording immediately on consent withdrawal and delete or anonymize existing recordings per policy.
9. Never use replay data for ad profiling or join it with marketing data.

#### Configuration sketch

Adjust every parameter to the actual SDK documentation; do not copy blindly:

```js
// Illustrative only: mask all inputs by default and exclude sensitive regions.
sessionReplay.init({
  maskAllInputs: true,
  maskTextSelector: "*",
  blockSelector: "[data-sensitive], .payment, .auth, .health",
});
```

#### Acceptance tests

- Before consent, the network panel shows no replay SDK requests or recording uploads.
- A canary test with a test card number, password, email, and CAPTCHA code is invisible in recordings.
- Payment, login, and health pages produce no recording at all.
- No new recordings after consent withdrawal.
- Only authorized staff can access recordings, and access is logged.
- Data is actually deleted when the retention period ends.

---

### CO-04 Marketing email unsubscribe and postal address

#### Every commercial marketing email must have

1. Clear sender identity and a statement of the email's nature.
2. A clear, prominent unsubscribe entry.
3. A valid physical postal address.
4. A working `List-Unsubscribe` header.
5. `List-Unsubscribe-Post: List-Unsubscribe=One-Click` when one-click unsubscribe is supported.
6. Unsubscribe requires no login, payment, phone call, extra personal information, or reason.
7. Unsubscribe requests enter a unified suppression list synced to every email vendor and business system.
8. CAN-SPAM's statutory maximum handling time is 10 business days; target immediate or within 24 hours.
9. Check email templates, promotional content inside transactional email, automated marketing flows, and re-engagement sequences.
10. EU/UK markets cannot rely on CAN-SPAM alone; electronic marketing generally also requires local consent rules.

#### Recommended footer

> You are receiving this because you subscribed to updates from [Brand].
> [Unsubscribe from all marketing email] · [Manage preferences]
> [Legal company name], [valid physical postal address]
> [Privacy policy] · [Contact]

#### Acceptance tests

- Send a test of every marketing template; links and headers all work.
- After unsubscribing, no further marketing email of that category arrives.
- Unsubscribe status is consistent across CRM, email platform, and automation flows.
- The unsubscribe page asks for no extra personal data.
- The postal address is real, deliverable, and matches the sending entity.
- Tracking pixels and link tracking in email are covered by privacy consent and disclosure.

---

### CO-05 Subscription renewal terms

#### Must appear next to the subscribe button

Before the user enters payment information, display directly near the purchase button:

- Current charge amount and currency.
- Billing period, e.g. monthly, yearly, or every 4 weeks.
- Whether it renews automatically.
- First charge date and next renewal date.
- Price and charge date after any trial ends.
- How to cancel and the latest cancellation time.
- Price-change rules.

Example:

> US$120 per year, charged today. Then US$120 automatically every 12 months until canceled. Cancel anytime online at "Account > Billing > Cancel subscription" — no phone call required.

#### Consent and cancellation

1. Auto-renewal consent must not be pre-checked and must not be buried in the privacy policy or general terms.
2. Charge only after explicit informed consent.
3. Retain the terms version, display time, button copy, user action, and consent record.
4. Provide an online cancellation path requiring no phone call, email approval, or retention offer.
5. Send a confirmation after cancellation stating when service ends, whether access continues, and whether further charges occur.
6. A reminder before a trial converts is recommended, but it does not replace clear disclosure at purchase.
7. Refund rights, the EU right of withdrawal, and subscription cancellation are distinct concepts; never collapse them into one "cancel" flow.

#### Legal note

On 2025-07-08 the U.S. Eighth Circuit vacated the FTC's federal Negative Option Rule. That does not mean auto-renewal is unregulated: ROSCA, the FTC Act, and state auto-renewal laws may still apply. Do not remove clear disclosure, explicit consent, and simple cancellation on the grounds that "the federal rule was vacated."

#### Acceptance tests

- Check the desktop and mobile first screen: renewal terms are visible without expanding, scrolling to the privacy policy, or clicking a link.
- Auto-renewal can be declined without selecting extra options; if the business only offers auto-renewal, say so explicitly.
- The payment API cannot charge by bypassing the front-end consent state.
- Cancellation is self-service from account settings.
- A confirmation arrives after cancellation and no new charge occurs.
- Trials, promotional pricing, price increases, pauses, and resumptions all have test cases.

---

### CO-06 DMCA designated agent registration

#### Clarification

A DMCA designated agent is not a copyright registration. It designates the agent who receives infringement notices. Registering an agent does not automatically grant § 512 safe harbor; notice-and-takedown, counter-notice, and repeat-infringer policies are also required.

#### Preparation

| Item | Requirement |
|---|---|
| Legal name of the service provider | Must match the legal entity |
| Physical street address | Generally no P.O. Box; exceptions require prior approval |
| Public aliases | DBA, brand, domain, app name, common search names |
| Agent name | Individual, title, department, or third-party service |
| Agent address | Street address or P.O. Box |
| Agent phone | Reachable |
| Agent email | Dedicated `dmca@` address recommended |

One service provider may designate only one agent; separate legal entities register separately.

#### Steps

1. Identify which site features involve user uploads, storage, or information location services.
2. Determine the legal name and street address of the actual operating entity.
3. Publish `/dmca` or `/copyright` showing legal name, address, aliases, and agent contact.
4. Create a dedicated `dmca@` mailbox and ticket queue with a primary and backup owner.
5. Go to the U.S. Copyright Office DMCA designated agent system: <https://dmca.copyright.gov/osp/login.html>.
6. Create an account and select or add the service provider.
7. Enter legal name, street address, all public aliases, agent name/organization, address, phone, and email.
8. Verify the public page matches the system entry exactly.
9. Pay the current fee (check the Copyright Office page; historically US$6) and submit.
10. Save the confirmation page, receipt, submission date, and directory search screenshot.
11. Search the Copyright Office public directory and confirm the entry is live.
12. Add the renewal date to a calendar: amend or resubmit before the three-year expiry.
13. Set internal reminders at 90, 60, 30, and 7 days before expiry; do not rely on official reminder email.

#### After registration

- Build a process for receiving, reviewing, takedown, and replying to infringement notices.
- Retain notices, takedown times, evidence, and communications.
- Build a counter-notice process with a 10–14 business day restoration rule.
- Adopt and reasonably enforce a repeat-infringer termination policy.
- Tell users in your terms about the repeat-infringer policy and how to file copyright complaints.
- Update the registration promptly on agent departure, company renaming, domain change, or address change.

#### Acceptance tests

- The public page is accessible without login.
- Page information matches the Copyright Office directory exactly.
- A test notice to the agent mailbox creates a ticket with an owner.
- Registration records, receipt, and the next renewal date are archived.
- At least one simulated takedown, counter-notice, and restoration has been run.

## 3. Common compliance pitfalls for overseas sites

| Area | Common mistake | Minimum control |
|---|---|---|
| Privacy policy | Template company name, address, email, vendors do not match reality | Automatically verify entity, contact details, data flows, and terms version at every launch |
| Cookies and analytics | Analytics, ad pixels, heatmaps, and replay load before consent | Deny non-essential scripts by default, load after consent, provide an equally prominent reject button |
| Cross-border transfer | Using U.S. cloud services without SCCs, DPF, or a transfer assessment | Maintain an inventory of vendors, data categories, storage regions, transfer mechanisms, and DPAs |
| Data subject requests | Email only, no identity verification, deadline, or deletion linkage | Build a ticket flow for access, correction, deletion, export, restriction, and objection |
| Data retention | Logs, backups, marketing lists, and recordings kept forever | Define retention, automatic deletion, and backup deletion per data category |
| CCPA/CPRA | "Selling personal information" with no opt-out link and no GPC support | Provide Do Not Sell/Share, support GPC, and give notice at collection |
| Children and teens | Adult-only terms, no checks on SSO, invites, or public content entries | Unified age threshold, high privacy by default, limits on ad profiling and public profiles |
| Dark patterns | Pre-checked marketing, hidden fees, fake countdowns, hard-to-find cancel | Reject and accept equally prominent; cancel and purchase paths symmetric |
| Trials and refunds | Only "cancelable" with no charge date or method | Trial end date, conversion price, cancellation deadline, and refund policy near the CTA |
| Accessibility | Color-only errors, keyboard traps, dialogs without focus management | Automated and manual WCAG 2.2 AA testing |
| User content | No process for copyright complaints, trademarks, defamation, hate, or CSAM | Build reporting, review, takedown, appeal, law-enforcement retention, and emergency escalation |
| Asset licensing | Images, music, icons, code, and AI-generated content of unknown origin | Maintain an asset ledger with licenses, sources, generation records, and scope |
| Email and SMS | Purchased lists, pre-checked consent, unsynced unsubscribes, SMS without STOP | Record consent source and time, unify the suppression list, support STOP/HELP |
| Tax | Global sales without assessing VAT, GST, sales tax, or invoicing duties | Assess registration by customer location, digital service type, and revenue threshold |
| Payment compliance | Restricted business, high decline rate, no 3DS/SCA | Review processor policy, billing descriptor, refunds, disputes, and strong customer authentication |
| Security | No MFA, sessions never expire, cookies missing Secure/HttpOnly, no rate limits | Threat modeling, dependency scanning, key management, log redaction, penetration testing |
| Data breach | No response playbook, unaware of the 72-hour notification duty | Build discovery, containment, assessment, notification, forensics, and postmortem flows |
| Terms versioning | Cannot prove which version a user agreed to after an update | Version terms and retain display time, version, and consent evidence |
| Localization | English-only terms, missing Germany's Impressum and similar local mandates | Check language, company information, contact, and local consumer rights per market |
| EU right of withdrawal | Treating subscription cancellation as withdrawal, mishandling digital-content exceptions | Distinguish withdrawal, cancellation, and refund; disclose before checkout and capture confirmation |
| Advertising claims | Fake discounts, undisclosed paid reviews, fabricated testimonials | Retain substantiation, disclose affiliate and paid relationships |
| Promotions | Sweepstakes, rebates, referral rewards without rules or tax treatment | Publish full rules, eligibility, deadlines, privacy, and tax notes |
| Sanctions and export | No screening of sanctioned regions, restricted parties, or restricted products | Assess payment, IP, account, and customer screening requirements |
| Platform content rules | UGC platforms handle only DMCA and ignore other illegal content | Build notice-and-action, transparency, complaints, and repeat-violation handling |

## 4. Launch gates

### P0: must be complete before launch

- [ ] **CO-P0-01** Age determination is enforced server-side; below-threshold users cannot create accounts.
- [ ] **CO-P0-02** Session replay is off by default; if enabled, consent, input masking, and sensitive-page exclusion are verified.
- [ ] **CO-P0-03** Every marketing email has a working unsubscribe link and a valid postal address.
- [ ] **CO-P0-04** Unsubscribes enter a unified suppression list and stop sending within 10 business days.
- [ ] **CO-P0-05** The subscription CTA directly shows price, period, auto-renewal, charge dates, and cancellation method.
- [ ] **CO-P0-06** Auto-renewal consent is not pre-checked and informed consent is recorded before charging.
- [ ] **CO-P0-07** Non-essential cookies, analytics, ad pixels, and session replay do not load before consent.
- [ ] **CO-P0-08** Entity, address, contact details, vendors, and data flows in the privacy policy match reality.
- [ ] **CO-P0-09** Working channels exist for data deletion, access, and unsubscribe requests.
- [ ] **CO-P0-10** Passwords, payment data, identity documents, and children's data are never written to ordinary logs or analytics events.

### P1: within 30 days after launch

- [ ] **CO-P1-01** All fonts self-hosted; licenses and CSP verified.
- [ ] **CO-P1-02** DMCA designated agent published and registered with the Copyright Office.
- [ ] **CO-P1-03** DMCA notice, counter-notice, and repeat-infringer flows rehearsed.
- [ ] **CO-P1-04** Vendor DPAs, cross-border transfers, and subprocessor inventory complete.
- [ ] **CO-P1-05** Data retention periods and automatic deletion jobs live.
- [ ] **CO-P1-06** CCPA/CPRA Do Not Sell/Share and GPC tested.
- [ ] **CO-P1-07** WCAG 2.2 AA critical-path review complete.
- [ ] **CO-P1-08** Data breach response contacts and the 72-hour process established.

### P2: ongoing governance

- [ ] **CO-P2-01** Quarterly review of cookies, pixels, SDKs, and font domains.
- [ ] **CO-P2-02** Quarterly review of the marketing suppression list and unsubscribe automation.
- [ ] **CO-P2-03** Semi-annual review of subscription pricing, trials, cancellation, and refunds.
- [ ] **CO-P2-04** Annual review of target markets, tax, processor policies, and accessibility requirements.
- [ ] **CO-P2-05** Maintain the DMCA registration through its three-year cycle and amend immediately on changes.
- [ ] **CO-P2-06** Keep version and change records for all terms, policies, and consent screens.

## 5. Market overlay rules

| Market | Key rules | Easily missed |
|---|---|---|
| United States | COPPA, CAN-SPAM, ROSCA, FTC Act, state privacy laws, DMCA, ADA | State-law differences, auto-renewal, GPC, accessibility, UGC infringement |
| EU/EEA | GDPR, ePrivacy, consumer rights, EAA, DSA, VAT/OSS | Cookie consent, right of withdrawal, accessibility, digital services taxes, platform content duties |
| United Kingdom | UK GDPR, PECR, consumer law, children's design code, Online Safety Act | Services children may access, marketing consent, UGC risk, local enforcement |
| Canada | PIPEDA, CASL, provincial privacy laws | CASL consent and unsubscribe rules, Quebec French and privacy requirements |
| Australia | Privacy Act, Spam Act, Australian Consumer Law | Marketing consent, unsubscribe, misleading pricing, consumer guarantees |
| Germany | GDPR, TTDSG/TDDDG, Impressum, consumer law | Operator information, cookie consent, right of withdrawal, price transparency |

## 6. Audit evidence template

Keep at least the following for every control:

| Field | Content |
|---|---|
| Control ID | e.g. CO-01, CO-03, CO-05 |
| Applicable markets | US, EU, UK, etc. |
| Implementation location | File, endpoint, config key, backend page |
| Evidence | Commit, screenshot, test report, log sample, contract |
| Test date | YYYY-MM-DD |
| Tester | Name or team |
| Result | Pass / Fail / Pending verification / Not applicable |
| Defect ID | Linked ticket |
| Re-review date | YYYY-MM-DD |

### Suggested source-scanning commands

```bash
rg -n -i "fonts\.googleapis|fonts\.gstatic|@import[^;]*https?://|preconnect" .
rg -n -i "session.?replay|fullstory|hotjar|clarity|smartlook|logrocket" .
rg -n -i "googletagmanager|gtag\(|facebook.*pixel|tiktok.*pixel|linkedin.*insight" .
rg -n -i "List-Unsubscribe|unsubscribe|postal|address" .
rg -n -i "auto.?renew|renewal|trial|subscription|cancel" .
rg -n -i "dmca|copyright agent|takedown|counter.?notice" .
```

Scanning only finds leads. It does not replace runtime network capture, server-side testing, or backend configuration review.

## 7. Pre-release sign-off

| Role | Confirms | Signature / date |
|---|---|---|
| Product owner | User flows, pricing, renewal, cancellation copy |  |
| Engineering owner | Age, session replay, fonts, logging, endpoint controls |  |
| Growth/email owner | Marketing consent, unsubscribe, postal address, suppression list |  |
| Finance/payment owner | Pricing, tax, charges, refunds, disputes |  |
| Legal/privacy owner | Target markets, privacy policy, DPAs, DMCA, contracts |  |
| Security owner | Sensitive data, access control, logging, breach response |  |

## 8. Official references

These links are for verification and are not legal advice. Rules change; re-check the target market and the latest version before launch.

### United States

- DMCA designated agent directory: <https://www.copyright.gov/dmca-directory/>
- DMCA registration system: <https://dmca.copyright.gov/osp/login.html>
- DMCA FAQ: <https://www.copyright.gov/dmca-directory/faq.html>
- 37 C.F.R. § 201.38: <https://www.ecfr.gov/current/title-37/section-201.38>
- 17 U.S.C. § 512: <https://www.law.cornell.edu/uscode/text/17/512>
- Copyright Office fee schedule: <https://www.copyright.gov/about/fees.html>
- COPPA rule: <https://www.ecfr.gov/current/title-16/part-312>
- COPPA statute: <https://www.law.cornell.edu/uscode/text/15/6502>
- CAN-SPAM statute: <https://www.law.cornell.edu/uscode/text/15/7704>
- CAN-SPAM rule: <https://www.ecfr.gov/current/title-16/part-316>
- ROSCA: <https://www.law.cornell.edu/uscode/text/15/8403>
- 2025 Eighth Circuit decision vacating the Negative Option Rule: <https://ecf.ca8.uscourts.gov/opndir/25/07/243137P.pdf>
- California CCPA: <https://oag.ca.gov/privacy/ccpa>

### EU and UK

- GDPR Article 7: <https://gdpr-info.eu/art-7-gdpr/>
- GDPR Article 8: <https://gdpr-info.eu/art-8-gdpr/>
- GDPR Article 13: <https://gdpr-info.eu/art-13-gdpr/>
- GDPR Article 33: <https://gdpr-info.eu/art-33-gdpr/>
- EDPB consent guidelines: <https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-052020-consent-under-regulation-2016679_en>
- European Accessibility Act: <https://commission.europa.eu/strategy-and-policy/policies/justice-and-fundamental-rights/disability/european-accessibility-act-eaa_en>
- ICO children's code: <https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/>

### Other

- 18 U.S.C. § 2258A, child sexual exploitation reporting duty: <https://www.law.cornell.edu/uscode/text/18/2258A>
- WCAG 2.2: <https://www.w3.org/TR/WCAG22/>

## 9. Disclaimer

This document is an engineering and operational compliance checklist template, not legal advice for any specific entity. Applicability depends on the operating entity, user locations, service type, data processing, revenue scale, child audience, and contractual arrangements. For high-risk data processing, children's products, healthcare, finance, UGC platforms, or significant cross-border operations, have a lawyer qualified in the relevant jurisdiction review it.

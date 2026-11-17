# SEO Checkup (Optional, Website Projects)

> Check IDs: SEO-01 to SEO-33
>
> **This is an optional check**: it applies only to projects with public pages that want organic search traffic (marketing sites, blogs, e-commerce, content sites, SaaS landing pages). Pure back-office systems, API services, internal tools, and apps can mark the whole document "Not applicable".
>
> Inspired by *SEO Praxis* (《SEO实战密码》 by Zac/昝辉), chapters 3–5, 7, 8, 10, 13, distilled into an executable checklist. Core ideas: search engines exist to satisfy searchers; the three pillars of ranking are **relevance, authority, and usability**; getting the basics right solves 95% of the problem — fine-tuning contributes at most 5%.

## 0. How to use and verdict rules

Every item gets exactly one verdict: **Pass / Fail / Pending verification / Not applicable** (with a reason). Every verdict needs evidence: file:line, an HTML snippet, HTTP status output, or a webmaster-tools screenshot.

**Suggested order of work** (after receiving the report):

1. Run SEO-01 / SEO-02 (index ratio + webmaster tools) to locate big problems;
2. Check SEO-03 spider traps and SEO-04/05 robots (fatal tier);
3. Handle SEO-08/09 canonicalization and duplicate content (consolidate weight);
4. Fix SEO-07/11/12 (URLs, internal links, 404) + add sitemaps;
5. Apply Part C page by page (Title > H1/body > ALT > description > the rest);
6. If the site already has traffic and rankings, add Part D/E (risk and penalty detection) — stacked manipulation signals should be cleaned up ("de-optimized") before adding new optimization.

## A. Site-wide indexing and quick diagnosis

### SEO-01 Indexed-page ratio

- **What to check**: indexed pages / actual total pages.
- **How**: count real pages first (CMS/database); then query Google/Bing webmaster tools or a `site:` command.
- **Pass**: ratio ≥ 60%.
- **Fail**: 20–30% indicates serious structural problems — check spider traps, robots mis-rules, and internal-link structure per SEO-03/04/11.

### SEO-02 Webmaster-tools full checkup

- **What to check**: hack/penalty messages, crawl errors and crawl rate, HTML suggestions (duplicate titles/descriptions), backlinks and anchor text, content themes, internal-link counts.
- **How**: register and review Google Search Console / Bing Webmaster Tools panels.
- **Pass**: no critical messages; crawl rate matches site size; top content keywords match the site theme (template noise like Reply/says means template text outweighs content).
- **Fail**: mass 404s, too-low crawl rate, duplicate titles/descriptions — fix per the checks below.

## B. Site structure

### SEO-03 Spider traps

- **What to check**: all-Flash sites, Session IDs in URLs, non-301 redirects (302/JS/Meta Refresh), frames, long-parameter dynamic URLs, JS/Flash-generated navigation, login-required content, forced cookies.
- **How**: inspect source of the homepage and key pages — links must be plain HTML `<a>` tags; use "fetch as crawler" to compare what the bot sees vs. the user; check for sid/session parameters.
- **Pass**: site is primarily HTML text; navigation and body links are standard HTML; no Session IDs, frames, or non-301 redirects.
- **Fail**: any single trap fails the item. Fix: add HTML link entries alongside Flash; switch Session IDs to cookies; unify redirects to 301; render navigation with CSS; give up on indexing members-only content.

### SEO-04 robots.txt

- **What to check**: correctness of root robots.txt; accidentally disallowed directories; Sitemap declared.
- **How**: fetch `/robots.txt`; use the webmaster tools robots tester on specific URLs.
- **Pass**: file exists (even an empty one — avoid the server returning a 200 error page as robots); each Disallow on its own line; a `Sitemap:` line present; print/sort-page duplicates disallowed.
- **Fail**: missing robots with typos that block whole directories; multiple Disallows on one line. Note: robots-blocked URLs can still appear as bare URLs in results — use meta robots (SEO-05) to fully exclude.

### SEO-05 meta robots tags

- **What to check**: pages that must be excluded (paid content, test pages, duplicates) carry `noindex`.
- **How**: inspect `<meta name="robots" content="...">` in `<head>`.
- **Pass**: excluded pages have `noindex` (omit `nofollow` to keep weight flowing); normal pages carry no meta robots at all (`index,follow` is redundant).
- **Fail**: pages expected to be excluded rely only on "no internal links" (does not guarantee exclusion).

### SEO-06 nofollow and weight distribution

- **What to check**: low-value pages (contact, privacy, login/register, help, sort links) nofollowed; handling of repeated links to the same URL.
- **How**: check templates for `rel="nofollow"`; count per-page links to the homepage (logo, nav, footer).
- **Pass**: low-search-value pages nofollowed or shown only on the homepage; the first homepage link (logo image) has ALT with the homepage's target keyword; core content pages are not mistakenly nofollowed; no footer keyword-stuffed links.
- **Fail**: site-wide weight flowing to privacy-policy-type pages; footer exact-match keyword links (a common penalty trigger in recent years). Note: nofollow is a secondary factor — the real fix is the architecture in SEO-11.

### SEO-07 URL staticization and URL design

- **What to check**: static/rewritten URLs; parameter count; directory depth; length; descriptive naming; case; separators.
- **How**: sample URLs across levels.
- **Pass**: shorter is better; ≤2–3 parameters; minimal directory depth; descriptive names (`/news/finance/`, not `/cat-01/`); all lowercase; words separated by hyphens `-` (not underscores); one consistent form (directory or file).
- **Fail**: many `?a=1&b=2` URLs; mixed case (404s on case-sensitive servers); underscores. If full rewriting is not feasible short-term, at least reduce parameters.

### SEO-08 URL canonicalization (301 / canonical)

- **What to check**: same content reachable via multiple URLs (www/non-www, with/without index.html, trailing slash, http/https, port, parameter order).
- **How**: fetch each variant and check status codes with `curl -sI`; verify the homepage link URL is unique site-wide.
- **Pass**: one canonical URL chosen; all variants 301 to it; internal links and XML sitemaps use canonical URLs; duplicate/sort pages use canonical tags (absolute URLs recommended).
- **Fail**: variants all reachable (10–30% of web URLs have this), splitting weight and consuming crawl budget. Fix: ① CMS produces only canonical URLs; ② unify internal links; ③ 301 redirects; ④ canonical tags. Note: Google takes 1–3 months to consolidate 301 signals.

### SEO-09 Duplicate content

- **What to check**: same/similar content at multiple URLs — print versions, sort pages, session-ID pages, RSS-generated pages, thin pages (body shorter than nav/footer boilerplate), URLs that return 200 with arbitrary extra characters.
- **How**: webmaster tools HTML suggestions (duplicate titles/descriptions is the easiest signal); search a random body sentence in quotes to find duplicates.
- **Pass**: one article = one URL; duplicates robots-blocked or noindexed; bodies have sufficient unique content.
- **Fail**: crawlable duplicates exist. Fix: allow one version, block the rest via robots/noindex; use canonical for variant pages (e.g. same product, different color); for syndication add copyright + original-source links.

### SEO-10 Sitemaps

- **What to check**: HTML sitemap present; XML sitemap present and submitted/declared in robots.
- **How**: fetch `/sitemap.html` and `/sitemap.xml`; check the robots.txt Sitemap line and webmaster tools.
- **Pass**: HTML map lists the whole site (small sites) or main 1st/2nd-level categories (large sites); XML map is UTF-8, ≤50k URLs/10MB with index files beyond (up to 1000); submitted via webmaster tools.
- **Fail**: no sitemap, or a map identical to the navigation expected to save indexing. Note: XML sitemaps are an aid, not a substitute for good structure.

### SEO-11 Internal links and weight distribution

- **What to check**: sane tree structure (home → category → content; categories cross-link; content links back to home and its category); key pages linked directly from the home; pagination depth; navigation anchor text.
- **How**: count clicks from home to inner pages; review the internal-links panel in webmaster tools.
- **Pass**: every page ≤4–5 clicks from home; key inner pages linked directly from home/sidebar; big categories get more weight; pagination re-subdivided or jump-style (1,2,3,4,5,10,20,30); related-content links have randomness ("related recommendations", tag clusters — not just prev/next); navigation anchor text is 2–4 character category keywords, with synonym variants (PC/computer).
- **Fail**: deep pages too weak to index; old content decaying with no entry points. Fix: direct home links to key pages; support big categories via dropdown menus; subdivide pagination; add a few external links to deep pages.

### SEO-12 404 pages

- **What to check**: status code returned for nonexistent pages; custom 404 page design.
- **How**: `curl -sI https://example.com/definitely-not-exist`; visit a gibberish URL.
- **Pass**: real 404 status code; the 404 page keeps the site template, explains the error, and links to home, key categories, the sitemap, plus site search.
- **Fail**: nonexistent pages return 200 (judged as mass duplicate content); 404s 301/JS/Meta-Refresh to the home page (judged as a duplicate of home).

### SEO-13 Navigation, subdomains, breadcrumbs

- **What to check**: HTML text navigation, breadcrumbs, current-category highlighting; subdomain vs. directory choice; footer link stuffing.
- **How**: inspect navigation source and footer link areas; inventory subdomains.
- **Pass**: text navigation (styled by CSS); breadcrumbs on medium/large sites; no footer keyword stuffing; directories preferred over subdomains unless content volume or brand lines genuinely justify them (a subdomain is a separate site and splits weight).
- **Fail**: JS/Flash/image navigation; footer link stuffing; subdomains opened casually.

## C. Page optimization

### SEO-14 Page titles (Title)

- **What to check**: uniqueness, accuracy, length, stuffing, keyword position, brand placement.
- **How**: webmaster tools HTML suggestions for duplicates; sample `<title>` in source (should immediately follow `<head>`); search for default titles like "Untitled Document".
- **Pass**: unique per page; ≤30 Chinese characters (15–20 recommended); target keyword first, format like "Product - Category - Site Name"; concise and natural, not stuffed; at most 3–4 keywords per page, combined into readable phrases; brand/company name last; paginated titles prefixed ("Page 2 -") to stay unique; no "Welcome to ..." filler.
- **Fail**: same title site-wide; overlong and truncated; unnatural stuffing ("kids clothes, kids clothes wholesale, kids clothes retail" lists).

### SEO-15 Meta description

- **What to check**: present? duplicates the title? keyword lists? sentence quality? length?
- **How**: sample meta descriptions; check whether SERP snippets come from them.
- **Pass**: one or two readable, accurate, click-attracting sentences including the target keyword; ≤70 Chinese characters; unique per page.
- **Fail**: duplicating the title or pure keyword lists (search engines then take over the snippet dynamically). If you can't write a good one, omit it.

### SEO-16 Keywords meta tag

- **What to check**: whether anyone is still maintaining it.
- **Verdict**: no major search engine uses it; writing it is pointless. Skip it; the CMS should allow removing the tag. Mark "Not applicable/Pass".

### SEO-17 Keywords in body text

- **What to check**: frequency, first 50–100 words, variants, semantically related terms, category-page intro text.
- **How**: read the body and verify keyword placement.
- **Pass**: natural writing — 2–3 occurrences in short pieces, 4–6 in long ones; keyword in the opening paragraph; synonyms and semantically related terms used; keyword phrases kept intact and in order; category/channel pages have at least 2–3 paragraphs of hand-written unique text; do not obsess over keyword density.
- **Fail**: keyword stuffing; bodies so short that template boilerplate dominates; category pages that are bare item lists.

### SEO-18 H tags

- **What to check**: single H1 with the most important keyword; hierarchy matching article structure.
- **How**: inspect H1–H6 distribution in source.
- **Pass**: article title is the only H1 and contains the most important keyword; subheads are H2 with supporting keywords; category-page H1 = category name, content-page H1 = content name (not the site name).
- **Fail**: the CMS puts the site name in H1 site-wide; H1/H2 abused everywhere.

### SEO-19 ALT text

- **What to check**: image ALT briefly describes content and includes the keyword once; logo ALT.
- **How**: inspect img alt attributes.
- **Pass**: every image has descriptive ALT with the keyword once; the top-left logo's ALT contains the homepage target keyword (it is usually the first link to home, so ALT acts as anchor text); image links' ALT equals anchor text.
- **Fail**: missing ALT or multi-keyword stuffing.

### SEO-20 Lean code

- **What to check**: CSS/JS externalized; comments; nested tables; file size; link count.
- **How**: inspect source size and structure.
- **Pass**: styles defined once in CSS without inline duplication; CSS/JS in external files; comments trimmed; HTML as small as practical (Google once suggested <100KB, <100 links); main content near the top of the file.
- **Fail**: large CSS/JS blocks in head; formatting code diluting the text-to-noise ratio.

### SEO-21 Internal links and anchor text (page level)

- **What to check**: natural variety of body-link anchor text; distribution of positions.
- **How**: sample body internal links.
- **Pass**: a mix of exact-match keyword anchors and natural varied ones; spread through the body (learn from Wikipedia).
- **Fail**: all exact-match (over-optimization); exact-match keyword links piled in the footer.

### SEO-22 Outbound links

- **What to check**: whether the site never links out ("hoarding link equity").
- **Pass**: relevant, high-quality outbound links with relevant anchor text — they clarify the target's topic and support this page's own relevance.
- **Fail**: zero outbound links site-wide. Fix: cite authoritative external resources naturally in the body.

### SEO-23 Other page elements

- **What to check**: W3C validation, bold/italic usage, content updates.
- **Pass**: no serious HTML errors (warnings ignorable; W3C pass/fail has no evident ranking effect); moderate bold on keywords; time-sensitive content updated regularly to invite recrawls.
- **Fail**: serious HTML errors preventing text extraction; bold/italic overuse.

## D. Health and manipulation signals (risk review)

> Principle: penalties are **points-based** — one suspect technique rarely triggers a penalty alone; stacked signals reaching a threshold do. Tolerance differs by site size (large sites get more slack — do not copy big sites). Not caught today ≠ safe forever. Black hat vs. white hat is fundamentally a **risk judgment**, not a moral one. This part suits sites that already have rankings and traffic, checked periodically.

### SEO-24 Page-level manipulation signals

- **What to check**: hidden text (same/near-color text, 1px text, CSS `display:none` / off-screen positioning), hidden links, keyword stuffing (title/ALT/anchors/HTML comments), doorway pages, cloaking (different content to bots vs. users), non-301 redirects.
- **How**: select-all (Ctrl+A) on the page to reveal hidden text; inspect CSS for hidden elements and color tricks (background #FFFFFF, text #FFFFFD); spoof the bot User-Agent and compare; compare cached snippets with current pages; sweep for non-301 redirects.
- **Pass**: none of the above; titles and body read naturally.
- **Fail**: remove on sight — prefer "de-optimization" over stacking signals.

### SEO-25 Link spam and paid links

- **What to check**: blog/forum comment spam (off-topic comments, target-keyword anchor text, blast patterns); bought links (topic-irrelevant sources, sudden appearance/disappearance, footer/sponsor placements); sponsored templates/paid posts landing on topic-chaotic blogs.
- **How**: audit backlink sources (webmaster tools + Ahrefs/Majestic); watch the backlink curve for spikes and cliffs.
- **Pass**: topical, naturally growing links.
- **Fail**: cleanup is long-term work; do not expect removal to be instant — removing all paid links at once is itself a signal.

### SEO-26 Link farms, bad neighborhoods, site networks

- **What to check**: link-exchange pages that are pure link lists; linking out to penalized sites (**a bad neighbor linking to you is fine — you linking to it is dangerous**); excessive cross-linking among sites you control; spam-heavy same-IP neighbors.
- **How**: before exchanging links, check the partner's `site:` indexing and whether its unique name ranks first; inventory same-IP sites.
- **Pass**: none of the above.
- **Fail**: remove links to bad neighbors; move hosts if same-IP neighbors are mass spamming.

### SEO-27 Over-concentrated anchor text

- **What to check**: backlinks massively repeating one target keyword; over-optimized internal anchors; keyword-rich footer internal links.
- **How**: analyze the anchor-text distribution of backlinks.
- **Pass**: a healthy profile includes brand names and natural anchors like "click here" in significant proportion (practitioner experience: 85% brand anchors is normal); anchors spread through bodies, not concentrated in footers.
- **Fail**: diversify anchors; when rankings for all keywords drop, de-optimize first.

### SEO-28 Site-wide technical risks

- **What to check**: robots.txt/noindex accidentally deleted or added (first check when the whole site vanishes); pages that return 404/no-response to crawlers while looking fine to users; recent registrant changes or expired-then-rebought domains.
- **How**: read robots manually + verify in webmaster tools; simulate crawler requests with a header-checking tool; check whois history.
- **Pass**: no anomalies.

## E. Penalty detection

### SEO-29 Confirming "are we penalized"

- **What to check**: distinguishing normal fluctuation from penalties when rankings drop hard.
- **How, in order**:
  1. `site:` the domain — zero results = severe penalty removal (or robots/server trouble);
  2. Search the site's unique name/brand — if the first result is not your site, usually penalized;
  3. Search unique site text (phone, email, registration number) — if the first result is not yours, some form of penalty;
  4. Check whether crawler visit frequency dropped sharply (with site size and update rate unchanged);
  5. Only a **site-wide, steep** drop (page one to page tens) suggests a penalty; mixed moves are normal; page one → page two is usually an algorithm shift or stronger competitors.
- **Response order**: after a drop, wait about a month before acting (algorithm shifts often revert); if still down after 1–2 months, work through Part D: robots → server neighbors → redirects → meta → thorough de-optimization → suspicious-link cleanup → duplicate content. Recovery rates from severe penalties are low; sometimes re-registering a domain is the pragmatic choice.

## F. Monitoring and prioritization

### SEO-30 Baseline and site goals

- **What to check**: was a monitoring baseline recorded before SEO work started; is the site goal specific, singular, measurable, actionable.
- **Pass**: the goal is "the one concrete thing you most want visitors to do" (order/subscribe/download/contact), not outlooks like "grow traffic 10%"; competitor data is recorded on the same cadence (competitors optimize too — your data alone misleads).
- **Note**: SEO is a long-term strategy — at least 6–12 months for normal cycles; search traffic above ~70% of a commercial site's total is itself a risk signal; diversify traffic sources.

### SEO-31 Non-traffic metric set

- **What to check**: total indexed pages (`site:`); featured-page indexing (home + typical category + typical content pages, checked by URL); keyword rankings (tens to hundreds, recorded on a fixed schedule); backlinks (total links/linking domains — watch trends, not daily numbers); conversions and sales.
- **Pass**: one query method chosen and stuck to (numbers are not comparable across methods); weekly/monthly cadence; focus on trends and causal links to ranking/traffic changes.

### SEO-32 Traffic analysis

- **What to check**: visits and bounce rate (~60% on content/e-commerce sites is a warning — locate the high-bounce pages); traffic-source mix (direct/referral/search); whether top entry pages match the optimization plan; keyword-level traffic breakdown (which terms drive traffic/engagement/conversions, including unexpected high-quality terms); long-tail effect.
- **Pass**: on medium/large sites the top 10 keywords contribute under 50% of search traffic — more indicates serious internal-link/weight-distribution problems.
- **How to use**: expand unexpected traffic terms into sections/topics; contact unfamiliar referring sites to build link relationships.

### SEO-33 Prioritization

- **What to check**: whether improvements follow the right priority order.
- **Principles**:
  1. **Basics first (95/5 rule)**: confirm keyword research, indexing/structure/internal links, Title/H1, body keywords, originality, and one or two mastered link-building methods; fine-tuning contributes ≤5%;
  2. **Data-driven ROI**: prioritize keywords ranked "top of page 2" and "6–10 on page 1" — small nudges reach page one and multiply traffic; terms on page 7–8 are not worth investment;
  3. **Risk removal before polish**: stacked Part-D signals get cleaned first; when all keywords drop, de-optimize first;
  4. **Whole-site over single keywords**: the long tail brings the volume; head terms rise on site-wide strength;
  5. **Balance over extremes**: no optimization factor should look anomalous; prefer steady and continuous over sprints.

## What this check cannot cover

- Keyword research and competitive analysis (SEO planning, not a launch checkup);
- Quantitative backlink profiling (needs paid tools like Ahrefs/Majestic);
- Mobile experience and Core Web Vitals (see Lighthouse; run as a separate check);
- Structured data (Schema.org) and modern search features — this list is based on classic methodology and does not cover the AI-search/answers era.

## Disclaimer

This document distills the methodology of *SEO Praxis* for engineering self-checks; it is not a ranking promise. Search algorithms change constantly — specific thresholds (60% index ratio, 30-character titles) are practitioner reference values, not official standards.

# Domain History Checkup (Wayback Machine)

> Check IDs: DH-01 to DH-03
>
> For anyone taking over a second-hand domain, or auditing the "past lives" of an existing project's domains. This document keeps only generic practices and directly copyable commands, with no specific domains or sites. The data source is the Internet Archive's Wayback Machine (CDX API) — read-only queries only, no API key required.

## 1. What this check solves

The biggest hidden risk of a second-hand domain is not SEO — it is **historical reputation**: a previous owner may have hosted gambling, adult, phishing, or spam content, and search engines, security vendors, and ad platforms remember long after the content is gone. Typical symptoms:

- Google Search Console keeps showing a "spam violation" penalty that no appeal can clear;
- Ad campaigns and payment-provider reviews get rejected as "domain history suspicious";
- The backlink profile contains hundreds of casino links, so SEO starts from below zero.

None of this is visible at handover — the domain looks clean now, while the dirt is all in the past. The Wayback Machine is the only free, public, traceable "domain archive". Ten minutes before taking over a domain is far cheaper than discovering a blacklist entry after launch and having to switch domains.

## 2. Checks at a glance

| ID | Check | Data source | What to look at |
|---|---|---|---|
| DH-01 | Snapshot volume and timeline | CDX API (matchType=domain) | Any history at all, first year, gap years |
| DH-02 | Suspicious content scan | Same, per-URL | Typical paths and keywords of gambling/adult/phishing/spam sites |
| DH-03 | Identify the previous site | Fetch archived snapshot content | What kind of business the old site was, whether the industry is gray |

## 3. Prerequisites

1. Access to `web.archive.org` (a proxy may be needed on some networks).
2. The list of domains to check (root domains suffice — `matchType=domain` automatically covers subdomains and www).
3. python3 recommended for batch analysis of the returned JSON.

## 4. API notes (lessons learned)

1. CDX endpoint: `https://web.archive.org/cdx/search/cdx`. **Use https** — plain http hangs and times out.
2. A single page holds up to 150k rows. Probe volume with `showNumPages=true` first, then decide whether to paginate.
3. `collapse=urlkey` deduplicates by URL: you get "unique URLs", not "snapshot counts" — good for content analysis. Drop it when you care about time density.
4. `matchType=domain` matches the whole domain tree (including subdomains); for just root + www use `url=a.com|www.a.com`.
5. The endpoint 504s under peak load: add `--retry 2 --retry-delay 5`, run domains serially, and let one failure not block the rest.
6. Archived old sites may use GBK/GB2312 encoding. Parsing as UTF-8 raises `UnicodeDecodeError` — fall back to `decode('gbk', 'ignore')`.
7. Wayback crawls are sampled: **a small archive ≠ a clean domain — it only means nobody was watching.** Zero snapshots can mean a brand-new domain or a shady one that was never crawled. Always attach this caveat to your verdict.

## 5. Copyable commands

### 5.1 Fetch all unique-URL records (DH-01 / DH-02)

```bash
for d in example.com example.org; do
  curl -s --max-time 300 --retry 2 --retry-delay 5 \
    "https://web.archive.org/cdx/search/cdx?url=${d}&matchType=domain&output=json&fl=timestamp,original,statuscode&collapse=urlkey&limit=30000" \
    -A "Mozilla/5.0" -o /tmp/wb-${d}.json
  echo "$d: $(wc -c < /tmp/wb-${d}.json) bytes"
done
```

### 5.2 Batch analysis (DH-02)

```python
import json, re, collections

SUS = re.compile(
    r'casino|porn|sex|viagra|pharma|loan|bet|poker|xxx|hack|crack'
    r'|escort|drug|replica|gambl|slot|backlink|seo', re.I)

rows = json.load(open('/tmp/wb-example.com.json'))[1:]
if not rows:
    print('No snapshots (note: ≠ clean domain; it may simply never have been crawled)')
else:
    years = collections.Counter(r[0][:4] for r in rows)          # year distribution, spot gaps
    codes = collections.Counter(r[2] for r in rows)              # status code distribution
    hits = [r for r in rows if SUS.search(r[1])]                 # suspicious URLs
    print(f'{len(rows)} unique URLs | years {dict(sorted(years.items()))} | codes {dict(codes)}')
    print(f'earliest {rows[0]} | latest {rows[-1]} | suspicious hits: {len(hits)}')
```

### 5.3 Identify the previous site (DH-03)

For the earliest 200-status snapshot, fetch the original content and read title / keywords / body:

```bash
curl -sL "https://web.archive.org/web/<timestamp>/<original-url>" -A "Mozilla/5.0" -o /tmp/old.html
```

```python
import re, html
raw = open('/tmp/old.html', 'rb').read()
t = raw.decode('utf-8', 'ignore') if b'charset=utf-8' in raw[:2048].lower() else raw.decode('gbk', 'ignore')
m = re.search(r'<title[^>]*>(.*?)</title>', t, re.S | re.I)
print(html.unescape(m.group(1)).strip() if m else 'no title')
```

Verdict reference:

| Finding | Verdict |
|---|---|
| Zero snapshots | New domain or never crawled — risk unknown; cross-check other reputation sources |
| Snapshots are only the site's own assets | Only you have used the domain, ✅ clean |
| Old snapshots show a normal business site / blog | ✅ Basically clean; also check for brand-name collisions |
| Old snapshots contain gambling/adult/phishing paths, or mass 302 redirects to third-party sites | ⚠️ High risk — consider a different domain or start a reputation appeal |
| Long gap years (e.g. no snapshots 2018–2024) | Likely expired, dropped, and re-registered; focus on the state right before the gap |

## 6. A real checkup, as an example

Four project domains checked with the workflow above (the largest archive held only 43 unique URLs; the whole run took 10 minutes):

| Domain type | Archive history | Verdict |
|---|---|---|
| Primary domain A | One snapshot only, showing a default nginx 404 page | Empty site before handover, ✅ |
| Domain B (same brand as A) | 43 snapshots all from a single day: 1 × 200 + 42 × 404 | Previously a legitimate education company's site (an old ASP CMS); no malicious content, ✅ |
| Domains C / D | Snapshots begin only after this project launched | Brand-new registrations with zero history, ✅ |

Two details worth remembering:

1. **42 × 404 does not indicate a problem** — those are dead links from the old-site era hit by the archive crawler. They actually show the site was half-abandoned at the time, not that it was malicious.
2. **A same-brand domain can be a lucky homophone pickup**: the previous owner's company name happened to sound identical to the project's brand. That coincidence is serendipity, not risk — but worth recording.

## 7. What this check cannot cover

Wayback only sees "crawled pages". These risks need other sources (out of scope here):

- Un-archived black-hat history: Google Safe Browsing, VirusTotal domain reputation;
- Backlink profiles (to judge negative SEO assets): Google Search Console + Ahrefs/Majestic;
- Registration and drop history: whois history, DomainTools;
- Email reputation (did the previous owner send spam from it): MXToolbox blacklist lookup.

## Disclaimer

This document is a summary of technical experience, not advice on any domain transaction or investment. Wayback Machine data is copyrighted by the Internet Archive; when using its API, comply with the official terms and rate limits.

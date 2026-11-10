# 域名历史体检（Wayback Machine）

> 检查项编号：DH-01 至 DH-03
>
> 适用于接手二手域名、或给现有项目域名做一次「前世调查」。本文只保留通用做法与可直接抄的命令，不含任何具体域名或站点信息。数据源为 Internet Archive 的 Wayback Machine（CDX API），全程只读查询，不需要任何 API Key。

## 1. 这个检查解决什么问题

二手域名最大的隐性风险不是 SEO，而是**历史信誉**：前任持有者可能在上面挂过博彩、色情、钓鱼、垃圾站，这些记录会被搜索引擎、安全厂商和广告平台长期记住。表现为：

- 域名在 Google Search Console 里一直「因垃圾内容被处置」，怎么申诉都没用；
- 广告投放、支付通道审核被拒，理由是「域名历史可疑」；
- 外链档案里有几百条赌博站外链，SEO 从负数起步。

这类问题在接手时看不出来——域名现在是干净的，坏事全在过去。Wayback Machine 是唯一免费、公开、可追溯的「域名前世档案」，接手前花 10 分钟查一遍，比上线后发现被拉黑再换域名便宜得多。

## 2. 检查项总览

| 编号 | 检查 | 数据源 | 看什么 |
|---|---|---|---|
| DH-01 | 快照体量与时间线 | CDX API（matchType=domain） | 有没有历史、从哪年开始、哪年断档 |
| DH-02 | 可疑内容扫描 | 同上，逐条 URL | 博彩/色情/钓鱼/垃圾站典型路径与关键词 |
| DH-03 | 前身内容识别 | 抓取归档快照原文 | 旧站是什么类型的业务、是否灰色行业 |

## 3. 前置条件

1. 能访问 `web.archive.org`（部分网络环境需要代理）。
2. 待检域名列表（根域名即可，`matchType=domain` 会自动覆盖全部子域与 www）。
3. 建议准备 python3，用于批量分析返回的 JSON。

## 4. API 使用要点（踩过的坑）

1. CDX 端点：`https://web.archive.org/cdx/search/cdx`。**必须用 https**，http 会无响应超时。
2. 单页上限 15 万条，先用 `showNumPages=true` 探体量，再决定是否分页拉取。
3. `collapse=urlkey` 按 URL 去重，拿到的是「唯一 URL」而非「快照次数」，适合内容分析；查时间密度时去掉该参数。
4. `matchType=domain` 匹配整个域名树（含子域）；只想查根域加 www 用 `url=a.com|www.a.com`。
5. 该接口高峰期容易 504：加 `--retry 2 --retry-delay 5`，单域名失败不影响其他域名，串行跑。
6. 归档快照的旧站可能是 GBK/GB2312 编码，直接按 UTF-8 解析会报 `UnicodeDecodeError`，用 `decode('gbk', 'ignore')` 兜底。
7. Wayback 是抽样爬取：**归档量小 ≠ 域名干净，只代表无人关注**。零快照的域名既可能是全新的，也可能是有历史但没被爬过——结论里必须带上这个限定语。

## 5. 可抄命令

### 5.1 拉取全量唯一 URL 记录（DH-01 / DH-02）

```bash
for d in example.com example.org; do
  curl -s --max-time 300 --retry 2 --retry-delay 5 \
    "https://web.archive.org/cdx/search/cdx?url=${d}&matchType=domain&output=json&fl=timestamp,original,statuscode&collapse=urlkey&limit=30000" \
    -A "Mozilla/5.0" -o /tmp/wb-${d}.json
  echo "$d: $(wc -c < /tmp/wb-${d}.json) bytes"
done
```

### 5.2 批量分析（DH-02）

```python
import json, re, collections

SUS = re.compile(
    r'casino|porn|sex|viagra|pharma|loan|bet|poker|xxx|hack|crack'
    r'|escort|drug|replica|gambl|slot|backlink|seo'
    r'|博彩|色情|彩票|代孕|办证|私服', re.I)

rows = json.load(open('/tmp/wb-example.com.json'))[1:]
if not rows:
    print('无任何快照（注意：≠ 域名干净，可能只是没被爬过）')
else:
    years = collections.Counter(r[0][:4] for r in rows)          # 年份分布，看断档
    codes = collections.Counter(r[2] for r in rows)              # 状态码分布
    hits = [r for r in rows if SUS.search(r[1])]                 # 可疑 URL
    print(f'唯一URL {len(rows)} | 年份 {dict(sorted(years.items()))} | 状态码 {dict(codes)}')
    print(f'最早 {rows[0]} | 最新 {rows[-1]} | 可疑命中 {len(hits)} 条')
```

### 5.3 识别前身站点（DH-03）

对最早一条 200 状态的快照，抓原文看 title / keywords / 正文：

```bash
curl -sL "https://web.archive.org/web/<时间戳>/<原URL>" -A "Mozilla/5.0" -o /tmp/old.html
```

```python
import re, html
raw = open('/tmp/old.html', 'rb').read()
t = raw.decode('utf-8', 'ignore') if b'charset=utf-8' in raw[:2048].lower() else raw.decode('gbk', 'ignore')
m = re.search(r'<title[^>]*>(.*?)</title>', t, re.S | re.I)
print(html.unescape(m.group(1)).strip() if m else '无标题')
```

判定参考：

| 发现 | 判定 |
|---|---|
| 零快照 | 全新域名或从未被爬，风险未知，需配合其他信誉源 |
| 快照全是本站自身资源 | 域名只被你用过，✅ 干净 |
| 旧快照是正常企业站/博客 | ✅ 基本干净，顺带确认品牌词是否撞车 |
| 旧快照含博彩/色情/钓鱼路径，或大量 302 跳外站 | ⚠️ 高风险，考虑换域名或做信誉申诉 |
| 长年断档（如 2018–2024 无快照） | 大概率弃用到期后重新注册，重点查断档前最后状态 |

## 6. 一次真实体检的结论样例

某项目四个域名按上述流程体检（体量最大的也只有 43 条唯一 URL，10 分钟跑完）：

| 域名类型 | 归档历史 | 结论 |
|---|---|---|
| 主域名 A | 仅 1 条快照，内容是 nginx 默认 404 | 接手前是空站，✅ |
| 域名 B（与 A 同品牌） | 43 条快照集中在某年同一天，1 条 200 + 42 条 404 | 前身是一家正规教育公司官网（老式 ASP CMS），无恶意内容，✅ |
| 域名 C / D | 快照全部从本项目上线后开始 | 零历史的新注册域名，✅ |

两个值得记住的细节：

1. **42 条 404 不代表有问题**——那是归档爬虫在抓旧站时代留下的死链，恰恰说明站点当时已半荒废，不是恶意站特征。
2. **同品牌域名可能是「捡到的谐音梗」**：域名 B 的前身公司名恰好与项目品牌同音（比如原主叫「早图」而项目叫「造图」），这种巧合是缘分不是风险，但值得记录下来。

## 7. 本检查覆盖不了什么

Wayback 只看得到「被爬过的页面」，以下风险需要另外的源（本文不展开）：

- 未被归档的黑产历史：查 Google Safe Browsing、VirusTotal 域名信誉；
- 外链档案（判断 SEO 负资产）：Google Search Console + Ahrefs/Majestic；
- 域名注册历史与掉落记录：whois 历史、DomainTools；
- 邮件信誉（前任是否拿它发过垃圾邮件）：MXToolbox 黑名单查询。

## 免责声明

本文仅为技术经验总结，不构成任何域名交易或投资建议。Wayback Machine 数据版权归 Internet Archive 所有，使用其 API 请遵守官方条款与限速要求。

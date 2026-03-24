---
name: cross-platform-research
description: >
  Cross-platform search and research across Chinese social platforms.
  Search WeChat, Weibo, XiaoHongShu, Douban, Zhihu in parallel.
  Deep-read posts with comments. Generate Markdown research reports.
  Triggers: "调研", "研究", "跨平台搜", "帮我调研", "全网搜", "帮我搜",
  "搜一搜", "了解一下", "research", "cross-platform search"
---

# Cross-Platform Research Skill

## Workflow Overview

This skill performs cross-platform research in 3 phases:

1. **Phase 1 — Parallel Search**: Fire off searches across 5 Chinese platforms simultaneously, collect raw results.
2. **Phase 2 — Structured Summary**: Merge results into a globally-numbered table, present to user, ask whether to deep-read.
3. **Phase 3 — Deep Read + Report**: Fetch full content + comments for selected posts, analyze, and save Markdown report to `/tmp/`.

---

## Phase 1: Parallel Search

### Prerequisites

Before running searches, verify that the required tools are available:

1. **mcporter** — used for Weibo, XiaoHongShu, Douban, Zhihu searches. Run `which mcporter` to confirm.
2. **miku_ai** — Python package used for WeChat article search. Run `python3 -c "import miku_ai"` to confirm.
   - If missing, install with: `https_proxy= http_proxy= pip3 install miku-ai` (clear proxy env vars to avoid issues).

### Platform Commands

Replace `QUERY` with the user's search query.

**Important: WeChat (miku_ai) works best with short keywords (2-4 Chinese characters).** If the user's query is long (e.g., "靠谱容易上手的副业"), extract the core keyword (e.g., "副业推荐") for the WeChat search. Other platforms handle long queries fine.

**Important: Always use the CURRENT YEAR from the system date when constructing search queries.** Do NOT default to your training data cutoff year. For example, if today is 2026-03-24, use "2026" in queries like "2026 副业推荐", not "2025".

**WeChat (微信公众号):**
```bash
python3 -c "
import asyncio, json
from miku_ai import get_wexin_article
async def s():
    results = await get_wexin_article('QUERY', top_num=10)
    print(json.dumps(results, ensure_ascii=False, indent=2))
asyncio.run(s())
"
```

**Weibo (微博):**
```bash
mcporter call 'weibo.search_content(keyword: "QUERY", limit: 10)'
```

**XiaoHongShu (小红书):**
```bash
mcporter call 'xiaohongshu.search_feeds(keyword: "QUERY")'
```

**Douban (豆瓣):**
```bash
mcporter call 'exa.web_search_exa(query: "QUERY site:douban.com", numResults: 10)'
```

**Zhihu (知乎):**
```bash
mcporter call 'exa.web_search_exa(query: "QUERY site:zhihu.com", numResults: 10)'
```

### Execution Rules

1. **All 5 searches MUST run as 5 simultaneous Bash tool calls in a single response.** This is how Claude Code achieves parallelism — do NOT run them sequentially. Each platform gets its own Bash tool call, all issued in the same message. **If one parallel call fails/errors and causes other calls to be cancelled, immediately retry the cancelled calls** — do not skip them just because they were collateral damage.

2. **Default behavior**: Search all 5 platforms. The user can specify a subset:
   - "只搜微博和知乎" → only Weibo + Zhihu
   - "排除豆瓣" → all except Douban
   - Respect any platform inclusion/exclusion instructions from the user.

3. **Default count**: 10 results per platform. XiaoHongShu returns the API default (no `limit` param available).
   - User can override count, e.g. "每个平台搜5条" → set `top_num=5`, `limit: 5`, `numResults: 5` accordingly.
   - XiaoHongShu count cannot be overridden.

4. **Auto-retry on empty results**: If a platform returns empty results (e.g., WeChat returns `[]`), immediately retry with a shorter/broader keyword (e.g., extract the core 2-4 character keyword from the user's query). Do NOT ask the user — just retry autonomously. If 2 retries still return empty, then skip and note it.

5. After all searches complete, briefly summarize result counts per platform before proceeding to Phase 2.

---

## Phase 2: Structured Summary

Merge all platform results into a single globally-numbered table. Numbers are continuous across platforms: WeChat results are 1–N, Weibo results are N+1–M, XiaoHongShu results are M+1–P, and so on.

Present each platform's results using the format templates below. Only include platforms that returned results.

**WeChat (微信公众号):**

| # | 标题 | 来源 | 链接 |
|---|------|------|------|

**Weibo (微博):**

| # | 内容摘要 | 作者 | 转评赞 | 链接 |
|---|----------|------|--------|------|

转评赞 format: "转发数/评论数/点赞数"

**XiaoHongShu (小红书):**

| # | 标题 | 作者 | 点赞/收藏/评论 | 链接 |
|---|------|------|----------------|------|

**Douban (豆瓣):**

| # | 标题 | 来源 | 链接 |
|---|------|------|------|

Douban via Exa: no engagement metrics available, just title + source.

**Zhihu (知乎):**

| # | 标题 | 来源 | 链接 |
|---|------|------|------|

Zhihu via Exa: no engagement metrics available, just title + source.

After presenting the table, ask the user:

```
共 N 条结果。输入编号深入阅读（如 "3,15,27"），或输入 "auto" 让我自动挑选，或 "skip" 跳过直接结束。
```

**"auto" selection criteria:** Pick 3–5 posts prioritizing:
1. **Filter out ads/promotional content first** — skip posts that are obviously selling courses, recruiting agents, or contain "加微信/点卡片/扣1" type CTAs. Also skip posts from corporate accounts promoting their own products. Note which posts were excluded as ads so the user knows.
2. Diversity across platforms (at least one from each platform that returned results)
3. Highest engagement metrics (but be wary of inflated metrics — e.g., comment count >> like count may indicate bot activity)
4. Recency

If user says "skip", end the skill. Otherwise proceed to Phase 3.

---

## Phase 3: Deep Read + Report

This phase fetches full content and comments for the selected posts, then generates a structured research report. Each platform has its own deep-read method.

### WeChat (微信公众号) Deep Read

Read the full article:
```bash
cd ~/.agent-reach/tools/wechat-article-for-ai && python3 main.py "URL"
```

Comments: NOT available (WeChat has no public comment API).

### Weibo (微博) Deep Read

Full text: already available from Phase 1 search results.

Comments:
```bash
mcporter call 'weibo.get_comments(feed_id: ID)'
```

- The `feed_id` is the `id` field from the search result.
- Pagination via `page` parameter (e.g., `page: 2`) if needed for more comments.
- Extract top 10 comments with author name and text.

### XiaoHongShu (小红书) Deep Read

Read full post + comments in one call:
```bash
mcporter call 'xiaohongshu.get_feed_detail(feed_id: "ID", xsec_token: "TOKEN", load_all_comments: true)'
```

Field mapping from Phase 1 search results: `feed_id` = result `id` field, `xsec_token` = result `xsecToken` field.

### Douban (豆瓣) Deep Read

Two paths based on URL:

**Default: use Camoufox for ALL douban.com pages** (Jina Reader frequently fails even on public pages like /note/, returning empty "载入中" content). Use cookie injection if cookie file exists, otherwise run without cookies:

**If `~/.agent-reach/cookies/douban.json` exists** (login-gated and public pages):

Use Camoufox with cookie injection:
```python
import json
from camoufox.sync_api import Camoufox

with open('/Users/kg1898/.agent-reach/cookies/douban.json') as f:
    raw_cookies = json.load(f)

with Camoufox(headless=True) as browser:
    page = browser.new_page()
    page.goto('https://www.douban.com/', timeout=15000, wait_until='domcontentloaded')
    for c in raw_cookies:
        cookie = {'name': c['name'], 'value': c['value'], 'domain': c['domain'], 'path': c.get('path', '/')}
        if not c.get('session') and 'expirationDate' in c:
            cookie['expires'] = c['expirationDate']
        if c.get('secure'):
            cookie['secure'] = True
        page.context.add_cookies([cookie])
    page.goto('TARGET_URL', timeout=30000, wait_until='domcontentloaded')
    page.wait_for_timeout(3000)
    # Post body: page.query_selector('.topic-richtext') or page.query_selector('.topic-content')
    # Comments: page.query_selector_all('.reply-content')
```

If cookie file (`~/.agent-reach/cookies/douban.json`) is missing or expired, fall back to Jina Reader for public pages, skip login-gated pages, and warn user to refresh cookie.

### Zhihu (知乎) Deep Read

Jina Reader is BLOCKED by Zhihu (returns 403). Use Camoufox instead:
```python
from camoufox.sync_api import Camoufox

with Camoufox(headless=True) as browser:
    page = browser.new_page()
    page.goto('ZHIHU_URL', timeout=30000, wait_until='domcontentloaded')
    page.wait_for_timeout(5000)
    # Extract text content from the page
    # Zhihu Q&A pages include top answers natively
    # No login required
```

### Report Generation

After deep-reading all selected posts, generate a research report and save to `~/Documents/claude/research/research-<topic>-<YYYYMMDD-HHmm>.md` (create the directory if it doesn't exist).

Report structure:
```markdown
# 调研报告: <topic>
日期: YYYY-MM-DD

## 概述
<1-2 paragraph summary of key findings across platforms>

## 搜索结果汇总

Include the Phase 2 structured summary tables here (all platforms), preserving the original table format with numbering, engagement metrics, and links. Mark ads/promotional content with ⚠️. Bold the posts selected for deep reading. This section serves as a permanent index of all search results.

## 各平台详情（深读）

### 微信公众号
#### <Article Title>
- 来源: <source>
- 链接: <url>
- 摘要: <summary>

### 微博
#### <Post snippet>
- 作者: <author> | 转发: X | 评论: Y | 点赞: Z
- 摘要: <summary>
- 热门评论:
  1. <comment>
  2. <comment>

### 小红书
#### <Post Title>
- 作者: <author> | 点赞: X | 收藏: Y | 评论: Z
- 摘要: <summary>
- 热门评论:
  1. <comment>
  2. <comment>

### 豆瓣
#### <Post Title>
- 来源: <source>
- 链接: <url>
- 摘要: <summary>
- 热门评论:
  1. <comment>
  2. <comment>

### 知乎
#### <Question/Answer Title>
- 链接: <url>
- 摘要: <summary>

## 跨平台分析
<Cross-platform comparison: sentiment differences, unique perspectives per platform, consensus vs disagreement>

## 关键发现
- <finding 1>
- <finding 2>
- <finding 3>
```

After saving, tell the user: "调研报告已保存到 `~/Documents/claude/research/research-<topic>-<timestamp>.md`"

---

## Error Handling

**Core principle: exhaust all self-service recovery before asking the user.** Only ask the user when you truly cannot proceed (e.g., need them to scan a QR code or unlock a physical device).

- **Platform search fails**: Log warning, continue with other platforms. In Phase 2 summary, note which platforms failed.
- **Search returns empty**: Auto-retry with shorter/broader keywords (up to 2 retries). Do NOT ask the user.
- **XiaoHongShu offline (ECONNREFUSED on port 18060)**: This is a Docker-based MCP service. Self-recover in order:
  1. Check Docker status: `docker ps -a | grep xiaohongshu`
  2. If Docker Desktop is paused → ask user to unpause (this requires GUI interaction)
  3. If container exists but stopped → `docker start xiaohongshu-mcp`
  4. Wait 3s, verify with `mcporter list xiaohongshu`
  5. If container doesn't exist → inform user the container needs to be recreated
- **Deep read timeout**: Skip that post, note it in the report as "内容获取失败".
- **Douban cookie expired/missing**: Fall back to Jina Reader for public pages, skip login-gated pages, warn user: "豆瓣 cookie 已过期，请重新导出 cookie 到 ~/.agent-reach/cookies/douban.json"
- **miku_ai not installed**: Auto-install with `https_proxy= http_proxy= pip3 install miku-ai`
- **Exa rate limiting**: Douban and Zhihu both use Exa for search. If rate-limited, run them sequentially instead of in parallel, or reduce `numResults`.

---

## Extending to More Platforms

To add a new platform:
1. Add a search command to the Phase 1 routing table.
2. Add a deep read method to the Phase 3 section.
3. Add a table format template to Phase 2.

Candidate platforms for future addition: Twitter/X (bird CLI), Bilibili (yt-dlp + B站 API), V2EX (public API), Reddit (JSON API).

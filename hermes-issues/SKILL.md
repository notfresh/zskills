---
name: hermes-issues
description: 查 NousResearch/hermes-agent 本地 issue/PR 库的事实与方法. 触发词：加载 hermes issues、hermes issue、hermes-agent issue、hermes 库、hermes 同步。本地库在 /root/projects/github-repo-crawler/service/NousResearch--hermes-agent/data/issues.db，cron c1393d8c3ece 每日 20:00 自动增量。覆盖：库位置/大小/时间范围、PR vs issue 分桶、5 套查询方法（query.py / search.py / SQLite / Search API / upstream 元数据）、insert vs 真新创建等口径。
---

# Hermes-Agent Issue 本地库（观测用）

**目的**：每天观测 NousResearch/hermes-agent 的 issue/PR 更新了多少、库里全量多少、按条件怎么查。

## 关键事实（必记）

| 项 | 值 |
|---|---|
| **服务根目录** | `/root/projects/github-repo-crawler/service/` |
| **仓库目录** | `NousResearch--hermes-agent/`（**双连字符**，保留大小写） |
| **库文件** | `NousResearch--hermes-agent/data/issues.db`（358 MB） |
| **增量指针** | `NousResearch--hermes-agent/last_crawl.txt` = `2026-09-30T12:00:08Z` |
| **维护 cron** | `c1393d8c3ece`（每日 20:00 跑 `run-hermes-daily-sync.sh` → `sync.py -r hermes`） |
| **库内总数** | **32267** 条 = issue 10216 + PR 22051 |
| **open / closed** | 17384 / 14883 |
| **库内时间起点** | **2026-03-05**（**不是全量**，缺 2025-11 ~ 2026-03 早期历史） |
| **库内时间终点** | 2026-09-30T11:58:02Z |
| **upstream 实时** | stars 250,429 / open_issues_count **47642** |

## 观测：每次 cron 跑批 + 真新创建

cron `c1393d8c3ece` 每日 20:00 跑 `sync.py -r hermes`。stdout 含三件事：

| 字段 | 含义 |
|---|---|
| `本次: insert=N` | 本次真插进本地库的条数（含"老 issue 之前没库、昨晚被更新带回"的补全历史） |
| `本次: update=N` | 库里已有条目被改动（评论、改标签、状态切换） |
| `本次: skip=N` | `updated_at` 没变，跳过 |

**基线（9-30 跑批实测）**：insert=1889 / update=1357 / 耗时 47 分钟。日均真新 insert ≈ **1400-1900 条**。

**要看"今天真新建了多少"**：不能看 `insert`，要在 SQLite 里按本次抓取窗口算 `created_at`：

```sql
-- 替换 since 和 fetched_at 为本次 cron 报告的两个时间戳
SELECT is_pr, COUNT(*) FROM issues
WHERE created_at >= '<since>' AND created_at < '<fetched_at>'
GROUP BY is_pr;
-- is_pr=0 → 新 issue 数；is_pr=1 → 新 PR 数
```

## 查询方法

### 1. query.py（推荐，零配额，中文友好）

```bash
cd /root/projects/github-repo-crawler/service
python3 query.py -r hermes                          # 交互模式
python3 query.py -r hermes --title <关键词>         # 标题模糊
python3 query.py -r hermes -n <真实 issue 号码> -d   # 单条详情
python3 query.py -r hermes --type issue --state open # 组合过滤
python3 query.py -r hermes --since 2026-09-25 --until 2026-09-30
```

### 2. search.py（语义搜，需先建索引）

```bash
python3 search.py "你的问题描述" -r hermes --top 10
python3 search.py "..." --type issue --state open
```

### 3. 直查 SQLite（统计/观测）

```python
import sqlite3
db = sqlite3.connect('/root/projects/github-repo-crawler/service/NousResearch--hermes-agent/data/issues.db')
db.row_factory = sqlite3.Row

# 常用 SQL 模式：
# - PR/issue 分桶：is_pr=0 / is_pr=1
# - 状态分桶：state = 'open' / 'closed'
# - 零评论 open（无人认领区）：is_pr=0 AND state='open' AND comments=0
# - 最近活动：ORDER BY datetime(updated_at) DESC
# - 最近新建：ORDER BY datetime(created_at) DESC
```

### 4. live GitHub Search API（查早期缺失数据）

```bash
curl "https://api.github.com/search/issues?q=repo:NousResearch/hermes-agent+<关键词>&per_page=10"
```

### 5. upstream 元数据

```bash
curl https://api.github.com/repos/NousResearch/hermes-agent
# stargazers_count / open_issues_count / pushed_at
```

### 6. 找 7 天热点 issue / PR

**判据**: 7 天内 `created_at` 或 `updated_at` 有动静 + `comments >= 10`(阈值 5 膨胀到 2452 条,阈值 10 ≈ 132 条,阈值 15 ≈ 46 条)。

```sql
SELECT number, is_pr, state, title, comments, created_at, updated_at
FROM issues
WHERE (created_at >= :since OR updated_at >= :since)
  AND created_at < :until
  AND comments >= 10
ORDER BY comments DESC;
```

展示纪律:取一次数据后**单张表按评论数排,每行带 [状态/类型/创建/更新]**,别按"复活/合并/关闭"等角度分组循环取 Top(同一条 PR 会反复出现,看起来信息量大其实是凑数)。

## 口径警示（交付必带）

1. **`insert=N` ≠ 真新创建 N 条**：含补全历史。看真新创建用 `created_at` 窗口 SQL。
2. **`since=` 按 `updated_at` 过滤**：老 issue 被评论/改标签也会被拉回 → update。
3. **`/issues` 端点 PR 与 issue 混合**：分桶用 `is_pr`。
4. **库从 2026-03 起**：缺早期 ~30000 条 open issue/PR，查早期用 Search API。
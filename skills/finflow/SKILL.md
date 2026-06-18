---
name: finflow
description: Query China/HK/US financial market DATA with the finflow CLI. Use for ANY real-time market lookup — a stock quote, K-line/order-book, resolving a company name (茅台/腾讯/NVDA/宁德时代) to its code, listing the member stocks of an industry (A股申万 / 美股 GICS / 港股恒生 — 半导体/白酒/医药…) or a concept (AI/华为/国企改革) or a board (创业板/科创板/沪深/美股/港股/中概股), 龙虎榜 (dragon-tiger abnormal movers), capital flow (主力/北向资金), fundamentals & financial statements, flash news, market sentiment and rankings, macro/economic calendar, futures. Trigger whenever the user names a stock/company/ticker, OR names a sector/industry/concept EVEN WITHOUT the word "板块" (so "半导体怎么样" / "白酒今天如何" count), asks how 大盘 is doing, wants 今日涨跌/异动/龙虎榜/板块排行/北向资金, or any live market number — even when they never say "finflow". When a stock is named without a code, resolve it with `quote search` first. Not for crypto, conceptual/educational finance questions (e.g. 什么是市盈率), investment recommendations, or coding/backtesting tasks.
---

# Finflow — Chinese Financial Data CLI

Finflow (`@openduo/finflow`) aggregates real-time data from four Chinese platforms — CLS (财联社), Jin10 (金十), Xueqiu (雪球), Gelonghui (格隆汇) — with Yahoo Finance as a fallback for markets they don't cover (Japan/Korea/HK/US). Cover A-shares, HK, US stocks, futures, macro & flash news.

## Core principles

1. **Run via Bash**: `finflow <command>` — install first if missing: `npm install -g @openduo/finflow`.
2. **Stock codes are flexible**: `sh600519`, `SH600519`, `600519` all work — the CLI normalizes internally.
3. **Markets**: A-shares (`SH600519`), HK (`00700`), US (`AAPL`, `NVDA`, `.DJI`). JP/KR and Yahoo-only symbols fall back to Yahoo.
4. **Output is always one compact JSON line** — `{"data": {...}, "meta": {"source", "timestamp", "command"}}`. Parse it (pipe to `jq` or python).
5. **Percent fields are percentage numbers, not ratios**: `changePercent: -4.34` means **-4.34%** (NOT -0.0434 or -434%). This holds for every percent-named field across every command — `changePercent`, `percent`, `turnoverRate`, `amplitude`, `pe`, `pb`, `dividendYield`, etc. Display them verbatim with a `%`; never ×/÷ 100. (`change` is the absolute price delta — a different field.)
6. **Name → code first**: if a company is named without a code (茅台/宁德时代/腾讯), run `quote search <名字>` to get the code before quoting. Never guess a code from a name.

## The golden rule: classify the question, then pick ONE command

Almost every market question maps to a single finflow command that pulls the whole picture at once. **The #1 source of bad answers is reconstructing data from memory** — listing "半导体/AI/白酒" tickers by hand, guessing an index code, etc. Before running anything, classify what the user is actually asking (which kind of 板块? per-stock or market-wide flow? snapshot or multi-period?), then pick from the tables below. The data is one command away.

### ❌→✅ Anti-patterns (do not do these)

| ❌ Wrong instinct | ✅ Right command |
|---|---|
| "半导体/白酒/AI 里都有啥股票" → enumerate tickers from memory, quote each | Classify 板块 type (table below), pull it in one go: `quote industry` / `quote sector stocks` / `quote board` |
| "大盘怎么样 / 上证多少点" → guess `sh000001` and `quote` it | `market` returns all major indices + 沪深 capital flow directly (use `market index` for indices only) |
| "AI / 华为 / 国企改革 概念股" → `quote industry` (industries, not themes) | Concepts are themes, not industries: reverse-lookup the cls plate code via `quote sector plate <一只已知成分股>` → `quote sector stocks <码>`（`sector rank -t concept` 只列**当日热门**概念，通常不含你要的那个） |
| Turn a name "茅台" into a code via `info search` (that searches news) | Name→code uses `quote search`; news/article search uses `info search` |
| Want "latest fundamentals" → `quote finance` (multi-period statements, not a fresh snapshot) | Latest single-period snapshot (PE/PB/ROE/市值) → `quote f10 indicator`; multi-period reports → `quote finance -t …` |
| A stock spiked, want its **same-industry** peers → `quote related <code>` (one cmd) | Same-**concept-plate** members → `sector plate <code>` then `sector stocks <plate>` (two steps) |
| "北向资金 / 主力资金流" → `quote flow <某只股票>` (that's per-stock) | Market north/south capital → `quote flow market` (or `market flow`); per-stock main fund → `quote flow <code>` |

### "板块 / 行业 / 概念 / 市场" — four meanings, four commands

Chinese "板块" is overloaded. The single word that causes the most misuse. Match the user's word to its real meaning first:

| 用户说的词 | 实际含义 | 怎么判断 | 命令 |
|---|---|---|---|
| **行业**: 半导体 / 白酒 / 医药 / 银行 / 证券 / 新能源车 / 钢铁 | 申万 / GICS / 恒生 **标准行业** | 一个真实的业务部门，有官方分类体系 | 成分股: `quote industry <ind_code> --market cn\|us\|hk`<br>整体排行: `quote sector rank -t industry` |
| **概念 / 题材**: AI / 华为 / 国企改革 / 元宇宙 / 碳中和 / 网红经济 | 财联社 **概念板块** | 一个跨行业的主题叙事 | 找码: `quote sector plate <已知成分股>` 反查 cls 码（`sector rank -t concept` 只有当日热门，不全）→ 成分股: `quote sector stocks <plate_code>` |
| **上市板 / 市场**: 创业板 / 科创板 / 沪市主板 / 深市主板 / 北交所 / 新三板 / 美股 / 港股 / 中概股 | **市场板块** | 按上市地划分的交易板 | `quote board <type>`（`cyb`/`kcb`/`us`/`hk`…） |

> 拿不准是行业还是概念？**先试 `quote industry`**（申万覆盖了绝大多数真实行业）；如果明显是一个题材叙事/政策主题，就走概念。想看"整个板块今天涨跌排第几"用 `quote sector rank`。**永远不要因为分不清就退回去凭记忆列股票**——直接拉，拉不到再换下一种。

## Intent → command (pick by what the user wants)

### A. Single stock (个股)

| 用户想要 | 命令 |
|---|---|
| 只知道名字（茅台/宁德时代），要代码或行情 | 先 `quote search <名字>`，再 `quote <code>` |
| 某只股票实时行情 | `quote <code>` |
| 多只一起看 | `quote batch <codes...>` |
| K线 | `quote kline <code> -p <period> -n <num>`（分钟级走雪球，日/周/月/年走 cls；美港加 `--src xq`） |
| 五档盘口 / 分时 / 逐笔 | `quote depth\|timeline\|ticks <code>` |

### B. Sector / industry / concept / board (板块)

| 用户想要 | 命令 |
|---|---|
| 某行业成分股（半导体/医药…，A/US/HK） | `quote industry <ind_code> --market cn\|us\|hk`（先 `quote industry --market <m>` 列全部行业找码） |
| 某概念/题材成分股（下钻） | 反查码: `quote sector plate <已知成分股>`（`rank -t concept` 只有当日热门）→ `quote sector stocks <plate_code>` |
| 某上市板/市场股票排行（创业板/科创板/沪深/美股/港股/中概股） | `quote board <type>` |
| 板块（行业/概念）涨跌排行 | `quote sector rank -t industry\|concept` |
| **某股异动 → 找同行业联动股**（最常用） | `quote related <code>`（同行业 peer，一条命令） |
| 某股异动 → 找**同概念板块**成员 | `sector plate <code>` → `sector stocks <plate>`（两步） |

### C. Fundamentals & statements (基本面/财报) — `f10` ≠ `finance`

| 用户想要 | 命令 |
|---|---|
| **最新一期快照**（PE/PB/ROE/EPS/市值/股息率/负债率） | `quote f10 indicator <code>`（单期，最新） |
| 多期报表（利润/资产负债/现金流/指标） | `quote finance <code> -t income\|balance\|cashflow\|indicator` |
| 十大股东 | `quote f10 holders <code>` |
| 分红送股 | `quote f10 bonus <code>` |

### D. Market overview (大盘/市场) — don't guess index codes

| 用户想要 | 命令 |
|---|---|
| 主要指数 + 沪深资金（默认大盘总览） | `market` |
| 纯大盘指数 | `market index` |
| 市场情绪（涨停/跌停/封板率/涨跌分布） | `market emotion` |
| 全市场排行（涨跌幅/成交量/成交额/换手率/振幅） | `market rank [type]` |
| 港股排行 | `market hk [sort]` |
| 今天哪些股票异动 / 龙虎榜（带上榜原因） | `market longhu [date]` |
| 全球央行利率 | `market rates` |

> 三种"异动/排行"别混：官方当日异常 = `market longhu`；当日涨幅榜 = `market rank change`；某股的同**行业**联动 = `quote related`（同**概念板块**成员用 `sector plate` → `sector stocks`）。

### E. Capital flow (资金面) — per-stock vs market-wide

| 用户想要 | 命令 |
|---|---|
| 个股主力资金（主力/超大/大/中/小单净额） | `quote flow <code>` |
| 北向 / 南向资金（沪港通/深港通） | `quote flow market`（或 `market flow`） |
| 个股历史资金流（多日 + 3/5/10/20日净额） | `quote flow history <code> -c <num>` |
| 融资融券 | `quote flow margin <code>` |

### F. News & info (资讯) — `quote search` ≠ `info search`

| 用户想要 | 命令 |
|---|---|
| 名字 → 股票代码（茅台/腾讯） | `quote search <名字>`（搜股票，**不是**搜新闻） |
| 实时快讯/滚动电报 | `info flash`（默认 cls；`-s jin10`/`-s glh`） |
| 新闻/深度文章 | `info news`（`news depth <channel>` 深度；`news stock <code>` 个股） |
| 话题/题材热点 | `info topic`（`topic hot` 财联社热门题材） |
| 搜资讯/新闻/电报/文章 | `info search <关键词>`（**不是**搜股票代码；用单个词或整体短语） |

### G. Macro / calendar / futures

| 用户想要 | 命令 |
|---|---|
| 宏观数据 / 大事 / 全球假期 | `calendar macro\|event\|holiday [date]` |
| A股 / 港股 / 美股 数据日历 | `calendar ashare\|hk\|us [date]` |
| A股投资日历 | `calendar invest [date]` |
| 期货快讯 / 头条 | `future`（`-c <频道>` 按品种；`future headline`） |
| 期现基差 | `future basis [date]`（`-g <板块>`） |

## Global options

| Flag | Default | Description |
|------|---------|-------------|
| `-n <num>` | varies | Item count. `10` for most; `kline` 120; `industry`/`board` 30. (`flow history` uses `-c` instead.) |
| `-h` | — | Help |

## Name → code lookup

To turn a company name into a tradable code, run **`finflow quote search <keyword>`** (Xueqiu, live). It is the single source of truth — it covers **all three markets in one call** (A-shares + HK + US), which a per-market offline list cannot, and it stays current as stocks list/rename. No login, cookie, or local index needed.

Each hit is `{code, name, type}`. **`type` is always `"stock"` — don't filter on it; read the market/instrument from the `code` prefix instead:**

| `code` shape | market / instrument | example |
|---|---|---|
| `SH` / `SZ` + 6 digits | A-share | `SH600519` 贵州茅台 |
| bare 5 digits | HK stock | `00700` 腾讯控股 |
| plain letters | US ticker | `NVDA`, `AAPL`, `BYDDY` (US ADR) |
| `BK****` | CLS plate / board | `BK0088` 白酒 |
| `CSI*` / `SH000***` / `SZ399***` | index / ETF | `SZ399997` 中证白酒 |

A name that is also listed in HK/US returns all of them at once (e.g. `比亚迪` → `SZ002594` A, `01211` H, `BYDDY` US ADR) — pick by prefix.

Two boundaries to expect:
- **Concept/industry words return plates, indices, and ETFs — not a member-stock list.** `quote search 半导体` gives you `BK0021 半导体` and SOXX/SOXL, not the constituent stocks. For *industry → its stocks*, use `quote industry <ind_code>`; for *concept → its members*, use `quote sector stocks <plate_code>`.
- **Search matches the official security name (substring/prefix), not slang.** `茅台` works (it's a substring of 贵州茅台); pure market slang like `宁王` returns nothing — normalize it to the official name/短称 before searching.

## Command Reference (detailed syntax)

### `quote` — securities

**`quote <code>`** — Single quote (CLS, auto-fallback xq → Yahoo).

**`quote batch <codes...>`** — Batch (Xueqiu, **single-source, no fallback**). Auto-prefixes `SH`/`SZ` for 6-digit A-share codes; supports HK/US.
```
quote batch sh600519 sz000858        # A股
quote batch 00700 09988               # 港股
quote batch AAPL NVDA TSLA .DJI       # 美股
```

**`quote search <keyword>`** — Search by name/code across A/HK/US (Xueqiu, live). This is the **name → code** entry point — see [Name → code lookup](#name--code-lookup) above for the `code`-prefix conventions and the two boundaries (concept words return plates not members; slang isn't matched). Note: `-n` has no effect here — it always returns the top ~10 hits.
```
quote search 比亚迪          # → SZ002594 (A) · 01211 (H) · BYDDY (US ADR)
```

**`quote related <code>`** — Same-industry peers (Xueqiu). The one-command 以点扩面 (茅台 → 五粮液/泸州老窖/洋河/古井贡酒).

**`quote industry [ind_code] [--market cn|us|hk]`** — Industry → member stocks (Xueqiu). `ind_code` is a **行业代码 (板块ID) — NOT a stock code**. Each market uses a different classification:

| `--market` | classification | example `ind_code` |
|---|---|---|
| `cn` (default) | 申万 (Shenwan) | `S2701` = 半导体 |
| `us` | GICS | `453010` = 半导体 |
| `hk` | 恒生 (Hang Seng) | `7030` = 工业工程 |

- No `ind_code` → lists all industries of that market `{code, name, pinyin}` (run first to find the code).
- With `ind_code` → that industry's stocks. Pass the matching `--market` (codes differ per market).
> This is *industry → stocks*. It is **not** "which industry does stock X belong to" (don't pass a stock code).

**`quote board <type>`** — Stocks by board/market (Xueqiu). Market is derived from `<type>`, no flag:

| type | board | | type | board |
|------|-------|-|------|-------|
| `sh_sz` | 沪深一览 | | `shb` | 沪市B股 |
| `sha` | 沪市A股 | | `szb` | 深市B股 |
| `sza` | 深市A股 | | `zxb` | 中小板 |
| `cyb` | 创业板 | | `xsbno` | 新三板协议 |
| `kcb` | 科创板 | | `xsbin` | 新三板做市 |
| `us` | 美股一览 | | `us_china` | 中概股 |
| `hk` | 港股一览 | | | |

`industry` & `board` share these options:

| Flag | Default | Description |
|------|---------|-------------|
| `-n <num>` | `30` | Number of stocks (paginated, 90/page) |
| `--order-by <key>` | `percent` | Sortable field key (below) |
| `--order` | `desc` | `asc` / `desc` |

`--order-by` keys (only these are confirmed sortable; unknown key → empty): `current`(当前价) `chg`(涨跌额) `percent`(涨跌幅) `current_year_percent`(年初至今) `volume`(成交量) `amount`(成交额) `turnover_rate`(换手率) `pe_ttm`(市盈率TTM) `dividend_yield`(股息率) `market_capital`(市值).

Each returned stock: `{code, name, price, changePercent, amount, volume, marketCap, pe, pb, turnoverRate}`.
```
quote industry                       # list 申万 (A股) industries
quote industry S2701 -n 20           # A股 半导体, top 20 by 涨跌幅
quote industry 453010 --market us -n 10   # 美股 半导体
quote board cyb                      # 创业板 涨跌幅 ranking
quote board kcb --order-by market_capital -n 5   # 科创板 市值前5
quote board us_china                 # 中概股一览
```

**`quote kline <code>`** — K-line.

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--period` | `-p` | `d` | `d` `w` `m` `y` `5d` `1m` `5m` `15m` `30m` `60m` `120m` | Period |
| `--number` | `-n` | `120` | number | Candle count |
| `--src` | — | auto | `cls` `xq` | Source (US/HK → `xq`) |

Minute periods (`1m`…`120m`) auto-use xueqiu; daily+ default to cls. US/HK → always `--src xq`.

**`quote depth <code>`** — 5-level order book. **`quote ticks <code>`** — tick trades. **`quote timeline <code>`** — intraday line. All take `--src cls|xq` (US/HK → `xq`).

**`quote sector`** — CLS plate data.

| Subcommand | Args | Options | Description |
|---|---|---|---|
| `sector rank` | — | `-t <type>` (default `industry`) | Plate ranking. Type: `industry` `concept`. Each plate has a `code`. |
| `sector plate <code>` | code | — | A stock's plates (each with `code`). |
| `sector stocks <plate_code>` | plate_code | `--way change\|last_px` | Members of one plate/concept — the drill-down (each with `reason`, `isCore`). |
| `sector industry <code>` | code | — | Industry info (xueqiu F10). |

`plate_code` (cls plate id, e.g. `cls80405`=华为产业链) — get it by **reverse-lookup**: `quote sector plate <stock>` where <stock> is ANY stock you know is in the concept (e.g. 润和软件 `sz300339` → 华为产业链 `cls80405`); the result lists that stock's plates, each with a `code`. Then `quote sector stocks <plate_code>`. For same-**industry** peers use `quote related` instead. (`sector rank -t concept` only lists today's ~6 **hot** concepts — don't rely on it to find a specific concept's code.)

Default (no subcommand): with code → `plate`, without → `rank -t industry`.

### `quote flow` — capital flow

| Subcommand | Args | Options | Source | Description |
|---|---|---|---|---|
| (default) | code | — | cls | Per-stock fund flow (main/super/large/medium/small) |
| `flow market` | — | — | glh | Northbound/southbound capital |
| `flow history <code>` | code | `-c <num>` (default 20) | xq | Historical flow + 3/5/10/20d net |
| `flow margin <code>` | code | — | xq | Margin trading |
| `flow intraday <code>` | code | — | xq | Intraday capital flow |
| `flow xq <code>` | code | — | xq | Xueqiu capital flow |

Default without code → market flow (same as `flow market`).

### `quote f10` / `quote finance` — fundamentals (Xueqiu)

`quote f10 <code>` with no subcommand → `indicator`.

| Subcommand | Description |
|---|---|
| `f10 indicator` (default) | **Latest single-period snapshot**: PE/PB/ROE/EPS/市值/股息率/负债率 |
| `f10 holders` | Top 10 shareholders |
| `f10 bonus` | Dividend history (≤15) |

`quote finance <code>` — multi-period statements:

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--type` | `-t` | `indicator` | `indicator` `cashflow` `income` `balance` | Report type |

| Type | Content |
|---|---|
| `indicator` | ROE, EPS, gross margin, revenue YoY |
| `income` | Revenue, net profit, attributable net profit |
| `balance` | Total assets, liabilities, debt ratio, equity |
| `cashflow` | Operating, investing, financing, net increase |

### `info` — news & flash

**`info flash`** — flash/breaking news.

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--src` | `-s` | `cls` | `cls` `jin10` `glh` | Source |
| `--category` | `-c` | varies | varies by source | Channel filter |

CLS categories: `all` `red` `announcement` `watch` `hk_us` `fund` `remind`. GLH categories: `all` `popular` `AStock` `HKStock` `USStock` `fundLive` `ai` `international` `chineseMainland` `hongkongMacaoTaiwan` `virtualAssets` `exchangeCommodity` `house` `debenture`.

| Subcommand | Args | Notes |
|---|---|---|
| `flash important` | — | Important news (glh) |
| `flash popular` | — | Popular flash (glh) |
| `flash detail <id>` | id | Flash detail (jin10, tries cookie) |
| `flash date <date>` | date | Historical flash (jin10, **requires login**) |

**`info news`** — articles.

| Subcommand | Args | Description |
|---|---|---|
| (default) | — | CLS headline news |
| `news depth [channel]` | channel | CLS deep articles. Channels: `headline` `ashare` `world` `company` `realestate` `finance_ch` `hkstock` `fund` `broker` `auto` `tech` `pinjian` |
| `news slide` | — | Important events slideshow (jin10) |
| `news feature` | — | Featured articles (jin10) |
| `news hot` | — | Hot articles ranking (cls) |
| `news stock <code>` | code | Per-stock news (cls) |

**`info topic`** — topics.

| Subcommand | Args | Description |
|---|---|---|
| (default) | — | List all topics (jin10) |
| `topic search <keyword>` | keyword | Search topics |
| `topic groups` | — | Topic category groups |
| `topic hot` | — | Hot CLS subjects |
| `topic show <id>` | id | Articles in topic (jin10, tries cookie) |

**`info search`** — full-site search (news/articles/stocks). **Not** for resolving stock codes — use `quote search` for that.

| Subcommand | Args/Options | Description |
|---|---|---|
| `search <keyword>` | `-s cls\|jin10`, `-t <type>` | Search informational content only (CLS keeps 股票/板块/电报/要闻/深度/文章; paywalled/noise excluded). Stock hits include `code`/`price`/`percent`. Use a **single keyword or solid phrase** (`美联储加息`, not `美联储 加息`) — both backends treat spaces as loose OR. |
| `search detail <id>` | id (from a `search` result) | Full body of a CLS telegraph (电报) by `id` — `brief` in results is truncated. |

### `market` — overview

| Subcommand | Args/Options | Source | Description |
|---|---|---|---|
| (default) | — | glh | Major indices + Shanghai/Shenzhen capital flow |
| `market emotion` | — | cls | Sentiment: limit-up/down, seal rate, turnover, distribution |
| `market rank [type]` | `change` `volume` `amount` `turnover` `amplitude` (default `change`) | cls | Stock ranking |
| `market index` | — | glh | Major indices only |
| `market hk [sort]` | `netChange` `turnoverValue` `turnVolume` (default `netChange`). `-o asc\|desc` (default `desc`) | glh | HK ranking |
| `market flow` | — | glh | Northbound/southbound capital |
| `market rates` | — | jin10 | Global central bank rates |
| `market longhu [date]` | `YYYY-MM-DD` (default today) | xq | 龙虎榜 — abnormal traders with `reasons`. Empty before close / non-trading days. |

### `calendar` — macro

All accept optional `<date>` (default today, `YYYY-MM-DD`).

| Subcommand | Source | Description |
|---|---|---|
| `calendar invest [date]` | cls | A-share investment calendar |
| `calendar macro [date]` | jin10 | Global macro data releases |
| `calendar event [date]` | jin10 | Macro events (speeches, decisions) |
| `calendar holiday [date]` | jin10 | Global holidays |
| `calendar ashare` | jin10 (WS) | A-share major events |
| `calendar hk [date]` | jin10 (WS) | HK data calendar |
| `calendar us [date]` | jin10 (WS) | US data calendar |
| `calendar future-event [date]` | jin10 | Futures calendar events |
| `calendar future-holiday [date]` | jin10 | Futures calendar holidays |

Default (no subcommand) → `calendar invest` today.

### `future` — futures

Top-level `--channel`/`-c` applies to all: `all`(默认) `oil` `steel` `coal` `gas` `metal` `nonferrous` `chemical` `sugar` `grain` `pig` `macro`.

| Subcommand | Description |
|---|---|
| (default) | Futures flash news (paginated) |
| `future headline` | Futures headlines |
| `future basis [date]` | Spot-futures basis. `-g <group>`: `黑色系` `金属` `农产品` `能源化工` `金融` |
| `future calendar [date]` | Futures data calendar |
| `future event [date]` | Futures calendar events |
| `future holiday [date]` | Futures calendar holidays |

## Auth (zero-config)

All commands work with no setup. **Xueqiu** (F10, finance, industry, board, longhu, flow, margin, search) auto-fetches an anonymous session (full /hq cookie set incl. `acw_tc` WAF cookie) on first use, cached at `~/.xueqiu-anon-cookie`; no login needed. **Jin10** (`flash detail`, `flash date`, `topic show`, `calendar macro/event/holiday`) tries a browser cookie and works without it (less data); `flash date` needs login. **CLS** signs every request (no login). **Gelonghui** needs no auth.

## Interpretation tips & known limits

- **`meta.source`** = which platform served the data (freshness context). Fallback markers use the space form (`xq fallback`, `yahoo fallback`).
- **`market emotion`** shows limit-up/down counts, seal rates — key sentiment.
- **`quote flow market`** / **`market flow`** show northbound (沪港通/深港通) capital — key for A-share trend.
- **Non-trading hours**: `emotion`, `timeline` return empty; `longhu` empty before close (~17:30).
- **`market rank` quirks**: `--order`/`--order-by` are NOT supported here (only `quote board`/`industry` support them); the `turnover`/`amplitude` rank types are often empty intraday (populate after close).
- **`-n` defaults vary**: most 10; `kline` 120; `industry`/`board` 30; `flow history` uses `-c`.
- **Xueqiu single-source commands** (`search`/`industry`/`board`/`f10`/`finance`/`flow history`) have **no cls/Yahoo fallback** — if xueqiu's WAF blocks them they error. Only `quote <code>` and `kline`/`depth`/`ticks`/`timeline` have fallback chains.
- **CLS flash with jin10 source** auto-paginates up to 50 pages (slow for large `-n`).
- **`info flash -c <栏目>`** (non-`all`) is cls single-source with no cross-source fallback. The category is filtered **server-side** by cls, so results are accurate for `watch`/`hk_us`/`fund`/`red`/`announcement`. Use default (`all`) or `-s jin10` for cross-source results.
- **Time formats are mostly ISO**, but `timeline` (时分), `ticks` (时分秒), and some `calendar` commands are exceptions — parse carefully.

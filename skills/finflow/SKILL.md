---
name: finflow
description: Query China/HK/US financial market data with the finflow CLI. Use for ANY market question — a stock quote, searching a stock by company name, listing the stocks in an industry (A股申万 / 美股 GICS / 港股恒生) or a board (创业板/科创板/沪深/美股/港股/中概股), 龙虎榜 (dragon-tiger list of abnormal movers), K-line and order book, capital flow (主力/北向资金), fundamentals & financial statements, flash news, market sentiment and rankings, macro/economic calendar, futures. Trigger whenever the user mentions a company by name (茅台/腾讯/NVDA) or ticker, asks how the market/大盘 is doing, wants industry/board members, 今日涨跌/异动/龙虎榜, 板块排行, 北向资金, or any real-time financial data — even when they don't say "finflow". When a stock is named without a code, resolve it with `quote search` first.
---

# Finflow — Chinese Financial Data CLI

Finflow (`@openduo/finflow`) aggregates real-time data from four Chinese financial platforms: CLS (财联社), Jin10 (金十), Xueqiu (雪球), and Gelonghui (格隆汇).

## Core Principles

1. **Run via Bash**: `finflow <command>` — if not found, install first with `npm install -g @openduo/finflow`
2. **Stock code formats are flexible**: `sh600519`, `SH600519`, `600519` all work — the CLI normalizes internally
3. **Global markets supported**: A-shares (`SH600519`), HK stocks (`00700`, `09988`), US stocks (`AAPL`, `NVDA`, `.DJI`)
4. **Output is always JSON**: every command prints a single compact line `{"data": {...}, "meta": {"source": ..., "timestamp": ..., "command": ...}}`. There is no text/raw mode and no format flag — just pipe/parse the JSON.
5. **Percent fields are percentage numbers, not ratios**: `changePercent: -4.34` means **-4.34%** (NOT -0.0434 or -434%). This applies to every percent-named field across all commands — `changePercent`, `percent`, `turnoverRate`, `amplitude`, `pb`, `pe`, etc. Never divide/multiply by 100 when consuming them; display them verbatim with a `%`. (`change` is the absolute price delta in yuan/cents, a different field — keep the two distinct.)
6. **Name → code first**: if the user names a stock by company name (e.g. 茅台, 宁德时代, 腾讯) without a code, run `quote search <名字>` to get the code before quoting it. Don't guess a code from a name.

## Choosing the right command

Pick by what the user actually wants — not by command family. (Details for each are in the Command Reference below.)

| 用户想要 | 命令 |
|---|---|
| 只知道名字（茅台/宁德时代），要代码或行情 | 先 `quote search <名字>`，再 `quote <code>` |
| 某只股票实时行情 | `quote <code>` |
| 多只股票一起看 | `quote batch <codes...>` |
| **某行业有哪些股票**（半导体/医药…，A/US/HK） | `quote industry <ind_code> --market cn\|us\|hk`（先 `quote industry --market <m>` 列出该市场全部行业找代码） |
| **某板块/市场股票排行**（创业板/科创板/沪深/美股/港股/中概股） | `quote board <type>`（type: A股 cyb/kcb…，美股 `us`/`us_china`，港股 `hk`） |
| **今天哪些股票异动 / 龙虎榜** | `market longhu [date]` |
| 大盘指数 / 市场情绪 | `market` / `market emotion` |
| 涨跌幅·成交量·成交额·换手率·振幅排行 | `market rank [type]` |
| 板块（行业/概念）涨跌排行 | `quote sector rank -t industry\|concept` |
| **某票异动 → 找同板块联动股**（最常用） | `quote related <code>`（同行业 peers，一条命令） |
| **某概念/板块的成分股**（下钻） | `quote sector stocks <plate_code>`（plate_code 见下方 sector） |
| 个股资金流 / 北向南向资金 | `quote flow <code>` / `quote flow market` |
| K线·五档盘口·分时·逐笔 | `quote kline\|depth\|timeline\|ticks <code>` |
| 基本面·财报·十大股东·分红 | `quote f10 indicator\|holders\|bonus <code>` / `quote finance <code>` |
| 财经快讯·深度文章 | `info flash` / `info news` |
| 宏观日历·经济数据·假期 | `calendar [macro\|event\|holiday]` |
| 期货快讯·期现基差·期货日历 | `future [...]` |

## Global Options (apply to all commands)

| Flag | Default | Description |
|------|---------|-------------|
| `-n <num>` | varies | Number of items. Default `10` for most commands; `kline` defaults to `120`, `industry`/`board` to `30`. (`flow history` uses `-c` instead.) |
| `-h` | — | Show help |

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

## Command Reference

### Stock Quotes — `quote`

**`quote <code>`** — Single stock quote (CLS)

**`quote batch <codes...>`** — Batch quotes (Xueqiu). Auto-prefixes `SH`/`SZ` for 6-digit A-share codes. Supports HK/US stocks.
```
quote batch sh600519 sz000858        # A股
quote batch 00700 09988               # 港股（纯数字代码）
quote batch AAPL NVDA TSLA .DJI     # 美股
```

**`quote search <keyword>`** — Search by name/code across A/HK/US (Xueqiu, live). This is the **name → code** entry point — see [Name → code lookup](#name--code-lookup) above for the `code`-prefix conventions and the two boundaries (concept words return plates not members; slang isn't matched). Note: `-n` has no effect here — it always returns the top ~10 hits.
```
quote search 比亚迪          # → SZ002594 (A) · 01211 (H) · BYDDY (US ADR)
```

**`quote related <code>`** — Same-industry peer stocks (Xueqiu). The simplest **以点扩面**: one stock → its industry peers in one command (e.g. 茅台 → 五粮液/泸州老窖/洋河/古井贡酒). Use this for "某票异动，找同板块联动股".

**`quote industry [ind_code] [--market cn|us|hk]`** — Industry → its member stocks (Xueqiu). `ind_code` is a **行业代码 (板块ID) — NOT a stock code**. Each market uses a *different* industry classification with its own codes:

| `--market` | classification | example `ind_code` |
|---|---|---|
| `cn` (default) | 申万 (Shenwan) | `S2701` = 半导体 |
| `us` | GICS | `453010` = 半导体 |
| `hk` | 恒生 (Hang Seng) | `7030` = 工业工程 |

- No `ind_code` → lists all industries of that market `{code, name, pinyin}` (run this first to find the code you need).
- With `ind_code` → that industry's stocks (remember to pass the matching `--market`, since codes differ per market).
> This is *industry → stocks*. It is **not** "which industry does stock X belong to" (don't pass a stock code here).

**`quote board <type>`** — Stocks by board / market overview (Xueqiu). The `market` (CN/US/HK) is **derived from `<type>`** — no flag needed:

| type | board / market | | type | board / market |
|------|-------|-|------|-------|
| `sh_sz` | 沪深一览 | | `shb` | 沪市B股 |
| `sha` | 沪市A股 | | `szb` | 深市B股 |
| `sza` | 深市A股 | | `zxb` | 中小板 |
| `cyb` | 创业板 | | `xsbno` | 新三板协议 |
| `kcb` | 科创板 | | `xsbin` | 新三板做市 |
| `us` | **美股一览** | | `us_china` | **中概股** |
| `hk` | **港股一览** | | | |

Both commands share these options:

| Flag | Default | Description |
|------|---------|-------------|
| `-n <num>` | `30` | Number of stocks (paginated, 90/page) |
| `--order-by <key>` | `percent` | Any 成分股 field key — see below |
| `--order` | `desc` | `asc` or `desc` |

`--order-by` sorts by a 成分股 field key. These are the columns xueqiu's page actually exposes for sorting (中文含义) — use one of them:

| key | 含义 | key | 含义 |
|----|------|----|------|
| `current` | 当前价 | `chg` | 涨跌额 |
| `percent` | 涨跌幅 | `current_year_percent` | 年初至今 |
| `volume` | 成交量 | `amount` | 成交额 |
| `turnover_rate` | 换手率 | `pe_ttm` | 市盈率TTM |
| `dividend_yield` | 股息率 | `market_capital` | 市值 |

The API returns more fields, but only these are confirmed sortable. An unknown key silently returns empty.

Each returned stock: `{code, name, price, changePercent, amount, volume, marketCap, pe, pb, turnoverRate}`.
```
quote industry                       # list 申万 (A股) industries to find a code
quote industry S2701 -n 20           # A股 半导体, top 20 by 涨跌幅
quote industry --market us           # list 美股 GICS industries
quote industry 453010 --market us -n 10   # 美股 半导体 members
quote industry --market hk           # list 港股 恒生 industries
quote board cyb                      # 创业板 today's 涨跌幅 ranking (default)
quote board cyb -n 10 --order-by turnover_rate    # 创业板 top 10 by 换手率
quote board kcb --order-by market_capital -n 5    # 科创板 largest by 总市值
quote board us                       # 美股一览 by 涨跌幅
quote board us_china                 # 中概股一览
quote board hk --order-by market_capital -n 10    # 港股 top 10 by 总市值
```

**`quote kline <code>`** — K-line data

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--period` | `-p` | `d` | `d` `w` `m` `y` `5d` `1m` `5m` `15m` `30m` `60m` `120m` | Period |
| `--number` | `-n` | `120` | number | Number of candles |
| `--src` | — | auto | `cls` `xq` | Data source |

Source auto-selection: minute periods (`1m` `5m` `15m` `30m` `60m` `120m`) auto-use xueqiu; daily+ default to cls. For US/HK stocks, always use `--src xq`.
```
quote kline sh600519 -p d -n 120     # 日K 120根 (cls)
quote kline sh600519 -p 5m -n 48     # 5分钟K线 (xueqiu, auto)
quote kline sh600519 -p w            # 周K (cls)
quote kline sh600519 -p d --src xq   # 强制用雪球
```

**`quote depth <code>`** — 5-level order book

| Flag | Default | Values | Description |
|------|---------|--------|-------------|
| `--src` | `cls` | `cls` `xq` | Data source |

**`quote ticks <code>`** — Tick-by-tick trades. Options same as `depth`.

**`quote timeline <code>`** — Intraday timeline. Options same as `depth`.

**`quote sector`** — Sector/plate data

| Subcommand | Args | Options | Description |
|---|---|---|---|
| `sector rank` | — | `-t <type>` (default: `industry`) | Plate ranking. Type: `industry` `concept`. Each plate carries a `code`. |
| `sector plate <code>` | code | — | A stock's plates (each with `code`). |
| `sector stocks <plate_code>` | plate_code | `--way change\|last_px` | Members of one plate/concept — the 以点扩面 drill-down (each with `reason`, `isCore`). |
| `sector industry <code>` | code | — | Industry info (xueqiu F10). |

**`plate_code`** (cls plate id, e.g. `cls80218`=国企改革) — get it from:
- `quote sector plate <stock>` → that stock's plates (each with a `code`), or
- `quote sector rank -t concept` → browse all concepts with codes.
Then `quote sector stocks <plate_code>` expands to that plate's members. For same-**industry** peers (the common case) use `quote related <code>` instead — one command, no code needed.

Default (no subcommand): with code → `plate`, without → `rank -t industry`.

### Capital Flow — `quote flow`

| Subcommand | Args | Options | Source | Description |
|---|---|---|---|---|
| (default) | code | — | cls | Per-stock fund flow (main in/out, super/large/medium/small) |
| `flow market` | — | — | glh | Northbound/southbound capital |
| `flow history <code>` | code | `-c <num>` (default: 20) | xq | Historical flow + 3d/5d/10d/20d net |
| `flow margin <code>` | code | — | xq | Margin trading data |
| `flow intraday <code>` | code | — | xq | Intraday capital flow |
| `flow xq <code>` | code | — | xq | Xueqiu capital flow |

Default without code → shows market flow (same as `flow market`).

### Fundamentals — `quote f10` / `quote finance`

**`quote f10`** subcommands (xueqiu). `quote f10 <code>` with no subcommand defaults to `indicator`:

| Subcommand | Description |
|---|---|
| `f10 <code>` (default) | Same as `f10 indicator` |
| `f10 indicator <code>` | Key indicators: ROE, EPS, gross margin, revenue growth |
| `f10 holders <code>` | Top 10 shareholders |
| `f10 bonus <code>` | Dividend history (up to 15 items) |

**`quote finance <code>`** — Financial statements (xueqiu):

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--type` | `-t` | `indicator` | `indicator` `cashflow` `income` `balance` | Report type |

| Type | Content |
|---|---|
| `indicator` | ROE, EPS, gross margin, revenue YoY |
| `income` | Revenue, net profit, attributable net profit |
| `balance` | Total assets, liabilities, debt ratio, equity |
| `cashflow` | Operating, investing, financing, net increase |

### News & Flash — `info`

**`info flash`** — Flash/breaking news

| Flag | Alias | Default | Values | Description |
|------|-------|---------|--------|-------------|
| `--src` | `-s` | `cls` | `cls` `jin10` `glh` | Data source |
| `--category` | `-c` | varies | varies by source | Category/channel filter |

CLS categories (`-s cls`): `all` `red` `announcement` `watch` `hk_us` `fund` `remind`
GLH categories (`-s glh`): `all` `popular` `AStock` `HKStock` `USStock` `fundLive` `ai` `international` `chineseMainland` `hongkongMacaoTaiwan` `virtualAssets` `exchangeCommodity` `house` `debenture`

| Subcommand | Args | Extra Options | Notes |
|---|---|---|---|
| `flash important` | — | `--channel <glh_channel>` | Important news (glh) |
| `flash popular` | — | — | Popular flash (glh) |
| `flash detail <id>` | id | — | Flash detail (jin10, tries browser cookie) |
| `flash date <date>` | date | — | Historical flash (jin10, **requires login**) |

**`info news`** — Articles

| Subcommand | Args | Description |
|---|---|---|
| (default) | — | CLS headline news |
| `news depth [channel]` | channel | CLS deep articles. Channels: `headline` `ashare` `world` `company` `realestate` `finance_ch` `hkstock` `fund` `broker` `auto` `tech` `pinjian` |
| `news slide` | — | Important events slideshow (jin10) |
| `news feature` | — | Featured articles (jin10) |
| `news hot` | — | Hot articles ranking (cls) |
| `news stock <code>` | code | Per-stock news (cls) |


**`info topic`** — Topics

| Subcommand | Args | Description |
|---|---|---|
| (default) | — | List all topics (jin10) |
| `topic search <keyword>` | keyword | Search topics |
| `topic groups` | — | Topic category groups |
| `topic hot` | — | Hot CLS subjects |
| `topic show <id>` | id | Articles in topic (jin10, tries browser cookie) |

**`info search`** — Full-site search (news, articles, stocks)

| Subcommand | Args/Options | Source | Description |
|---|---|---|---|
| `search <keyword>` | keyword, `-s cls\|jin10`, `-t <type>` | cls (default) / jin10 | Search across informational content only (CLS keeps 股票/板块/电报/要闻/深度/文章; paywalled/noise groups like invest_pro/fm_video/video are excluded). Stock hits include `code`/`price`/`percent`. `-t` restricts the CLS category. **Use a single keyword or a solid phrase** (e.g. `美联储加息`, not `美联储 加息`) — both backends treat space-separated words as loose OR matches. |
| `search detail <id>` | id (from a `search` result item) | cls | Fetch the full body of a CLS telegraph (电报) by its `id` — the `brief` in search results is truncated; use this for the complete text. |

### Market Overview — `market`

| Subcommand | Args/Options | Source | Description |
|---|---|---|---|
| (default) | — | glh | Major indices + Shanghai/Shenzhen capital flow |
| `market emotion` | — | cls | Market sentiment: limit-up/down, seal rate, turnover, distribution |
| `market rank [type]` | type: `change` `volume` `amount` `turnover` `amplitude` (default: `change`) | cls | Stock ranking |
| `market index` | — | glh | Major market indices |
| `market hk [sort]` | sort: `netChange` `turnoverValue` `turnVolume` (default: `netChange`). `-o asc\|desc` (default: `desc`) | glh | HK stock ranking |
| `market flow` | — | glh | Northbound/southbound capital flow |
| `market rates` | — | jin10 | Global central bank interest rates |
| `market longhu [date]` | date: `YYYY-MM-DD` (default: today) | xq | 龙虎榜 — stocks with abnormal trading that day, each with `reasons` (上榜原因). Empty for non-trading days / before close. |

### Macro Calendar — `calendar`

All subcommands accept an optional `<date>` argument (defaults to today).

| Subcommand | Source | Description |
|---|---|---|
| `calendar invest [date]` | cls | A-share investment calendar |
| `calendar macro [date]` | jin10 | Global macro data releases |
| `calendar event [date]` | jin10 | Macro events (speeches, decisions) |
| `calendar holiday [date]` | jin10 | Global holidays |
| `calendar ashare` | jin10 (WS) | A-share major events |
| `calendar hk [date]` | jin10 (WS) | HK stock data calendar |
| `calendar us [date]` | jin10 (WS) | US stock data calendar |
| `calendar future-event [date]` | jin10 | Futures calendar events |
| `calendar future-holiday [date]` | jin10 | Futures calendar holidays |

All dates default to today, format: `YYYY-MM-DD`.

Default (no subcommand): same as `calendar invest` with today.

### Futures — `future`

Top-level `--channel` / `-c` applies to all subcommands:

| Channel | Description |
|---|---|
| `all` | 全部 (default) |
| `oil` | 原油 |
| `steel` | 钢铁 |
| `coal` | 煤炭 |
| `gas` | 天然气 |
| `metal` | 金属 |
| `nonferrous` | 有色金属 |
| `chemical` | 化工 |
| `sugar` | 白糖 |
| `grain` | 粮食 |
| `pig` | 生猪 |
| `macro` | 宏观 |

| Subcommand | Description |
|---|---|
| (default) | Futures flash news (paginated) |
| `future headline` | Futures headlines |
| `future basis [date]` | Spot-futures basis. `-g <group>`: `黑色系` `金属` `农产品` `能源化工` `金融` |
| `future calendar [date]` | Futures data calendar |
| `future event [date]` | Futures calendar events |
| `future holiday [date]` | Futures calendar holidays |

## Auth (zero-config)

**All commands work with no setup.** Xueqiu (F10, finance, industry, board, longhu, flow, margin, search) auto-fetches an anonymous session (full /hq cookie set incl. the `acw_tc` WAF cookie) on first use and caches it at `~/.xueqiu-anon-cookie`; no login or cookie needed.

Jin10 commands (`info flash detail`, `info flash date`, `info topic show`, `calendar macro/event/holiday`) try a browser cookie automatically and work without it (with less data). `info flash date` needs jin10 login.

## Data Interpretation Tips

- **`meta.source`** tells you which platform the data came from — useful context for freshness
- **CLS flash** is fastest for A-share breaking news
- **`market emotion`** shows limit-up/down counts, seal rates — key sentiment indicators
- **`quote flow market`** shows northbound (沪港通/深港通) capital — important for A-share trend
- **Non-trading hours**: some commands return empty data (emotion, timeline)
- **CLS flash with jin10 source** auto-paginates up to 50 pages (slow for large `-n`)

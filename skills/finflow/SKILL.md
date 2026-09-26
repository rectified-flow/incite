---
name: finflow
description: "Financial-data CLI (`finflow`) for agents: one JSON `{data,meta}` per command. Data: 行情/K线/盘口, 基本面/财报/估值, 资金, 板块/行业/概念, 大盘/热力图/全场热度, 涨停/龙虎榜/异动, 快讯, 宏观日历, 基金/ETF — 股票, 期货, 外汇, 商品, 全球指数, 国债收益率. Trigger on any finance question (行情, 板块, 大盘, 领涨领跌, 涨跌家数, 全场热度, 龙虎榜, 涨停池, 北向资金, 汇率, 国债收益率, 黄金, 原油, 指数, 财报, 快讯, 基金), even without saying finflow. `quote` 股票及外汇对/国债收益率等符号；`price` 期货/现货/外汇/指数 (`CODE.EXCHANGE`). 无代码先 `quote search` 或 `price search`. 涨停池/复权日K/基金需 `finflow key set ths`."
---

# Finflow — financial data CLI (stocks + forex/commodity/index/futures)

Two engines — pick by **what the instrument is**:

- **`quote`** — 股票（A/港/美等），以及外汇对、国债收益率等符号（如 `EURUSD=X` `^TNX`）。美股快照带盘前盘后 / 52周 / PE；全景详情用 `quote detail`。
- **`price`** — 期货 / 现货商品 / 外汇 / 全球指数，代码 `CODE.EXCHANGE`（`XAUUSD.GOODS` `i9888.DCE` `SPX.INDEX`）。

`CODE.EXCHANGE` 的指数/外汇/商品/期货用 `price`，不要 `quote KS11`（空）。概念/行业指数 `88xxxx.TI` 走 `quote`。

## Core principles

1. **Run via Bash**: `finflow <command>` — install first if missing: `npm install -g @openduo/finflow`.
2. **Stock codes are flexible**: `sh600519`, `SH600519`, `600519`, `AAPL`, `NVDA` all work — the CLI normalizes internally.
3. **Two engines — classify the instrument first** (this is the #1 routing decision, more important than which subcommand):
   - **`quote`** — 股票代码（`SH600519` / `00700` / `AAPL`），以及外汇对、国债收益率等符号（`EURUSD=X` `^TNX`）。美股快照带盘前盘后/52周/PE；全景用 `quote detail`。
   - **`price`** — 期货 · 现货 · 外汇 · 全球指数，代码 `CODE.EXCHANGE`（`XAUUSD.GOODS` `i9888.DCE` `SPX.INDEX`）。
   - **`CODE.EXCHANGE` 指数/外汇/商品/期货不要走 `quote`** — `quote KS11` / `quote .DJI` 为空。韩国KOSPI / 日经 / 标普 / 恒生 / 上证用 `price *.INDEX`。**例外：概念/行业指数 `88xxxx.TI` 是 `quote`**（见 Intent 表 A；成分股见表 C）。
4. **Output is compact JSON** — `{"data": {...}, "meta": {"source", "timestamp", "command"}}`. One line per command. **Exception: `price sub` is NDJSON** (one `{data, meta}` line per tick) and does not exit unless `-s <秒>` or SIGINT / pipe close — a one-shot snapshot is `price`, not `price sub`.
5. **Percent fields are percentage numbers, not ratios**: `changePercent: -4.34` means **-4.34%** (NOT -0.0434 or -434%). This holds for every percent-named field across every command — `changePercent`, `percent`, `turnoverRate`, `amplitude`, `pe`, `pb`, `dividendYield`, etc. Display them verbatim with a `%`; never ×/÷ 100. (`change` is the absolute price delta — a different field.)
6. **Name → code first**: if a company is named without a code (茅台/宁德时代/腾讯), run `quote search <名字>` to get the code before quoting. Never guess a code from a name.
7. **部分命令需一次 API key** — 涨停池/连板/热榜/异动/竞价、`fund`、`quote valuation|indicators|adjust|auction`、`quote kline --adjust`、`sector search`、`calendar trading`。去 https://fuyao.aicubes.cn/ 申请 `sk-fuyao-…`，再 `finflow key set ths <key>`（或环境变量 `THS_API_KEY`）。用前 `finflow key test ths`（items[0].ok=true 即就绪）。报未配置/需要 key 时引导用户去该站申请（不要编造 key），再 `key set` + `key test`。429 时等 ~70s，不要密集重试。其余命令不需要 key。

## How to navigate this skill

Jump to the section that matches the task instead of reading end-to-end:

| You want to… | Go to |
|---|---|
| Pick WHICH command fits a question | "Intent → command" tables (A–J) + the 板块 disambiguation above |
| Quote a **non-stock** (forex / commodity / global index / futures) | Intent table **B** (the `price` engine — indices/forex/commodities, NOT `quote`) |
| 涨停池/连板/热榜/龙虎榜明细/异动原因/竞价 | Intent tables **E + I** (`market limitup` 等, 需 key) |
| 基金/ETF 净值、持仓、经理、业绩 | Intent table **J** (`fund`, 需 key) |
| 复权K线/估值/三大报表/复权因子 | Intent tables **A + D** (`quote kline --adjust` / `quote valuation` / `quote finance --src ths`) |
| See every command at a glance (purpose + key flag) | "Command index" below |
| Get exact flags / category lists / syntax for one command | [`references/commands.md`](references/commands.md) |
| Build a `price` code / 品种分类与可订阅范围 / 交易所产品码表 | [`references/price-codes.md`](references/price-codes.md)（9 大分类 + 构造规则兜底；拿代码首选 `price search`） |
| Chain commands into a workflow | "Common workflows" below |
| Turn a company name into a code | "Name → code lookup" below |
| 需 key 的命令 / 去哪申请 | Core principle 7（https://fuyao.aicubes.cn/ 申请 → `finflow key set ths`） |

**Mental model:** classify the question (one row in Intent → command) → run ONE command → open a `references/` file only if you need exact flags or a code table. Most questions never leave the Intent tables.

## The golden rule: classify the question, then pick ONE command

Almost every market question maps to a single finflow command that pulls the whole picture at once. **The #1 source of bad answers is reconstructing data from memory** — listing "半导体/AI/白酒" tickers by hand, guessing an index code, etc. Before running anything, classify what the user is actually asking (which kind of 板块? per-stock or market-wide flow? snapshot or multi-period?), then pick from the tables below. The data is one command away.

### ❌→✅ Anti-patterns (do not do these)

| ❌ Wrong instinct | ✅ Right command |
|---|---|
| "半导体/白酒/AI 里都有啥股票" → enumerate tickers from memory, quote each | Classify 板块 type (table below), pull it in one go: `quote industry` / `sector stocks` / `quote board` |
| "大盘怎么样 / 上证多少点" → guess `sh000001` and `quote` it | `market` returns all major indices + 沪深 capital flow directly (use `market index` for indices only) |
| "今天谁涨最多 / 涨跌家数 / 全场热度" → 凭记忆 quote 一串，或只用 `market emotion` | 名单+家数：`market heatmap`（`summary` 是家数，`items` 是领涨领跌）；只要涨停家数/封板率：`market emotion` |
| "AI / 华为 / 国企改革 概念股" → `quote industry` (industries, not themes) | Concepts are themes, not industries: `sector plate <一只已知成分股>` → `sector <码>`（`sector rank -t concept` 只列**当日热门**概念，通常不含你要的那个；带行情的概念/行业指数用 `sector search <名称>` 找 `88xxxx.TI`） |
| Turn a name "茅台" into a code via `info search` (that searches news) | Name→code uses `quote search`; news/article search uses `info search` |
| Want "latest fundamentals" → `quote finance` (multi-period statements, not a fresh snapshot) | Latest single-period snapshot (PE/PB/ROE/市值) → `quote f10 indicator`; multi-period reports → `quote finance -t …` |
| A stock spiked, want its **same-industry** peers → `quote related <code>` (one cmd) | Same-**concept-plate** members → `sector plate <code>` then `sector <plate>` (one step via smart default) |
| "北向资金 / 主力资金流" → `quote flow <某只股票>` (that's per-stock) | Market north/south capital → `quote flow market` (or `market flow`); per-stock main fund → `quote flow <code>` |
| "涨停池 / 连板梯队 / 炸板" → `market emotion` (only aggregate counts) | Per-stock pools with 连板数/封单额/涨停原因 → `market limitup` / `market ladder` / `market limitbreak` (Intent 表 I) |
| "个股为什么异动/涨停原因" → guess from news | `market anomaly <code>` / `market limitup` 的 `reason` 字段 (官方异动原因) |
| 查韩国KOSPI / 日经 / 标普 / 黄金 / 美元指数 → `quote KS11` / `quote .DJI`（空） | **指数/外汇/商品/期货走 `price`**（命令示例见 Intent 表 B）；外汇对/国债收益率也可用 `quote EURUSD=X` / `quote ^TNX` |
| 基金/ETF 净值、持仓 → `quote <基金代码>`（只能拿到场内价） | `fund nav/perf/holdings <code>`（Intent 表 J）；ETF 场内行情可用 `fund etf` |

### "板块 / 行业 / 概念 / 市场" — four meanings, four commands

Chinese "板块" is overloaded. The single word that causes the most misuse. Match the user's word to its real meaning first:

| 用户说的词 | 实际含义 | 怎么判断 | 命令 |
|---|---|---|---|
| **行业**: 半导体 / 白酒 / 医药 / 银行 / 证券 / 新能源车 / 钢铁 | 申万 / GICS / 恒生 **标准行业** | 一个真实的业务部门，有官方分类体系 | 成分股: `quote industry <ind_code> --market cn\|us\|hk`<br>整体排行: `sector rank -t industry` |
| **概念 / 题材**: AI / 华为 / 国企改革 / 元宇宙 / 碳中和 / 网红经济 | **概念板块** 或 **概念/行业指数 `88xxxx.TI`** | 一个跨行业的主题叙事 | 板块码: `sector plate <已知成分股>` 反查 → `sector <plate_code>`；带行情/K线: `sector search <名称>`（如 `sector search 人工智能` → 885728.TI）→ `sector stocks <码.TI>` / `quote <码.TI>` |
| **上市板 / 市场**: 创业板 / 科创板 / 沪市主板 / 深市主板 / 北交所 / 新三板 / 美股 / 港股 / 中概股 | **市场板块** | 按上市地划分的交易板 | `quote board <type>`（`cyb`/`kcb`/`us`/`hk`…） |

> 拿不准是行业还是概念？**先试 `quote industry`**（申万覆盖了绝大多数真实行业）；如果明显是一个题材叙事/政策主题，就走概念。想看"整个板块今天涨跌排第几"用 `sector rank`。**要板块指数本身的点位/涨跌幅/K线：`sector search <名称>` 找 `88xxxx.TI` → `quote <码.TI>`**。**永远不要因为分不清就退回去凭记忆列股票**——直接拉，拉不到再换下一种。

## Intent → command (pick by what the user wants)

### A. Single stock (个股)

| 用户想要 | 命令 |
|---|---|
| 只知道名字（茅台/宁德时代/英伟达），要代码或行情 | 先 `quote search <名字>`，再 `quote <code>` |
| 某只股票实时行情 | `quote <code>`（美股快照带盘前盘后/52周/PE） |
| **美股/全球股票全景深度详情**（行情+盘前盘后+公司高管业务+核心财务估值+分析师目标价/评级+财报与季度EPS历史） | `quote detail <code>`（如 `quote detail NVDA` / `quote detail AAPL`） |
| 多只一起看 | `quote <code1> <code2> ...` 或 `quote batch <code...>`（空格分隔即批量；`^TNX`/`^TYX`/`^VIX` 可进批量） |
| K线 | `quote kline <code> -p <period> -n <num>` |
| **前/后复权日K**（回测/收益率计算，输出带 `adjust`） | `quote kline <code> --adjust forward\|backward -n <num>`（需 key） |
| **概念/行业指数行情或K线**（`88xxxx.TI`，概念 885/886 段、行业 881 段） | `quote 885728.TI`（快照）/ `quote kline 885728.TI -n 60`（日K）——这类指数是 `quote` 不是 `price` |
| 五档盘口 / 分时 / 逐笔 | `quote depth\|timeline\|ticks <code>` |

### B. Non-equity real-time quotes — forex / commodity / index / futures (外汇/商品/指数/期货)

期货 / 现货 / 外汇 / 指数用 `price`，代码 `CODE.EXCHANGE`，空格分隔可批量。股票用 `quote`（但 Yahoo 式外汇/期货符号如 `EURUSD=X` / `GC=F` 两个引擎里 `price` 已原生归一化）。

**Codes are forgiving — don't agonize over the suffix**: the CLI normalizes input automatically. **Yahoo Finance symbols work as-is** (`GC=F`→`GCc1.CMX`, `CLZ26.NYM`→`NYMEXCL2612.NYM`, `EURUSD=X`, `JPY=X`→`USDJPY`, `BTC-USD`, `^GSPC`; no jin10 equivalent exists for `ES`/`NQ`/`ZN`/`6E`/`KC=F` — those pass through as missing). Common aliases work as-is (`GOLD`/`SP500`/`^GSPC`/`US30`/`WTI`/`DXY`/`KOSPI`/`XAU/USD` → canonical code), bare codes infer the exchange (`XAUUSD` → `XAUUSD.GOODS`), and case is fixed (`i9888.dce` → `i9888.DCE`). **If unsure of a code at all, run `price search <关键词>`** — 中文或代码子串都行，返回的 `code` 字段直接用于 `price`：`price search 黄金` → `GCc1.CMX`(黄金期货连续)/`XAUUSD.GOODS`(现货黄金)；`price search 铁矿石主力` → `i9888.DCE`；`price search kospi` → `KS11.INDEX`. Categories & construction rules: [`price-codes.md`](references/price-codes.md).

| 用户想要 | 命令 |
|---|---|
| 不确定代码（名称→代码） | `price search <关键词>`（中文/代码子串；`[-n 20]` 截 items；`total` 为全库命中数。排序：期货/外汇/现货/指数 → 股票/基金 → 期权；主力/连续/现货优先于月份合约。结果自带 exchange/exchangeName，直接挑要的变体） |
| 外汇（美元指数 / 离岸人民币 / 欧元美元 / 美元日元 / 美元瑞郎） | `price DXY.NYF USDCNH.FXCM EURUSD.FXCM USDJPY.FXCM` |
| 现货商品（黄金 / 白银 / 原油 / 铜） | `price XAUUSD.GOODS USOIL.GOODS UKOIL.GOODS COPPER.GOODS`（USOIL=WTI，UKOIL=布伦特，别拿错） |
| **全球指数（标普 / 道琼斯 / 恒生 / 日经 / 韩国KOSPI / 上证）— 不是 `quote`** | `price SPX.INDEX DJI.INDEX KS11.INDEX N225.INDEX HSI.INDEX` |
| 国内期货主力（铁矿石 / 螺纹钢 / 纯碱 / 沪铜 / 碳酸锂） | `price i9888.DCE rb888.SHF SA888.CZC cu888.SHF lc888.GFEX`（主力码规则见 price-codes.md） |
| 这些品种的 K线历史 | `price kline <code> -p <period> -n <num>`；时间窗口 `--from/--to YYYY-MM-DD`（只给窗口=该区间全部；与 `-n` 同时用=窗口末尾 N 根）。分钟线近实时，日线最新 1–2 根可能未落盘。⚠️ 并非所有品种有 K 线——`rb888.SHF` 全周期 0 根；返回 0 根时换相关品种或降级 `price <code>` 快照，**勿换周期重试** |
| **持续订阅实时行情（NDJSON，可 pipe）** | `price sub [codes...] [-s 秒]`（每笔推送一行，字段同 `price`；无参数则 stdin 读代码，或把 `price search` 的输出管过来。**不设 `-s` 不会退出**；一次性快照用 `price`） |
| **订单流（逐笔买卖盘价量）** | `price orderflow <code> [-s 10] [--out <path>]`（如 `XAUUSD.BANK` `GC.FFE` `au888.SHF`；采集 N 秒，**全数据写 JSON 文件**，stdout 只报 file/records/时长——自己读文件分析，CLI 不代劳聚合） |
| **期货品种基本面 F10**（仅国内期货，供给/需求/库存指标） | `price f10 <code>`（品种档案+指标清单；裸码/别名自动归一化，如 `price f10 rb888`）→ `--tag <id>` 出该指标历史序列 |
| 不带代码 = 主流看盘列表 | `price`（12 个品种：国内期货主力 + 金油 + DXY/EURUSD/USDCNH + SPX/HSI） |

> 不确定一个品种是股票还是非股票？**有股票代码（`SH600519`/`00700`/`AAPL`）→ `quote`；带 `.` 交易所后缀（`.INDEX`/`.FXCM`/`.GOODS`/`.DCE`）或是个指数/外汇/商品/期货 → `price`。** 全球指数（含韩国KOSPI/日经）走 `price` 不走 `quote`——这是最常见的误路由。代码拼不准时先 `price search`，别反复盲试。

### C. Sector / industry / concept / board (板块)

| 用户想要 | 命令 |
|---|---|
| 某行业成分股（半导体/医药…，A/US/HK） | `quote industry <ind_code> --market cn\|us\|hk`（先 `quote industry --market <m>` 列全部行业找码） |
| 某概念/题材成分股（下钻） | 反查码: `sector plate <已知成分股>`（`rank -t concept` 只有当日热门）→ `sector <plate_code>`（智能默认直接出成分股） |
| 某上市板/市场股票排行（创业板/科创板/沪深/美股/港股/中概股） | `quote board <type>` |
| 板块（行业/概念）涨跌排行 | `sector rank -t industry\|concept`（**仅当日热门 ~6~10 条**，非全量；带涨幅数值。找某个具体概念用 `sector plate` 反查板块码） |
| **今日盘中板块/个股异动**（板块轮动全景） | `sector anchor [date]`（时间轴，**无涨幅数值只有 up/down 方向**；板块项 `cls*` 码 → `sector <code>` 下钻成分股） |
| **某股异动 → 找同行业联动股**（最常用） | `quote related <code>`（同行业 peer，一条命令） |
| 某股异动 → 找**同概念板块**成员 | `sector plate <code>` → `sector <plate>`（一步下钻） |

> 大板块（如人工智能 1000+ 只）`sector stocks` 全量倾倒——**只问家数读 JSON 尾部 `total` 字段，勿用 head 截断**。`quote <88xxxx.TI>` 快照的 `name` 为空（上游不给），名称从 `sector search` 结果对应。

### D. Fundamentals & statements (基本面/财报) — `f10` ≠ `finance`

| 用户想要 | 命令 |
|---|---|
| **最新一期快照**（PE/PB/ROE/EPS/市值/股息率/负债率） | `quote f10 indicator <code>`（单期，最新） |
| **历史 PE/PB/PS/PCF 估值分位数分析**（当前分位%、高低值、中位数、机会/危险值、估值状态评级） | `quote valuation <code> [-y 1\|3\|5\|10]`（默认 5 年） |
| **批量估值对比**（PE_TTM/PE_MRQ/PB/PS/PCF，≤100 只） | `quote valuation <code1> <code2> ...`（需 key） |
| **三大报表多期**（年报/季报口径切换） | `quote finance <code> -t income\|balance\|cashflow --src ths --period annual\|quarterly`（需 key） |
| **五维财务指标**（成长/盈利/偿债/营运/现金流，单报告期） | `quote indicators <code> [2025-4]`（report=年-季，1~4；默认上一年年报） |
| **复权因子/分红送转事件流** | `quote adjust <code>`（每股分红+送转比例，按除权日降序） |
| 十大股东 | `quote f10 holders <code>` |
| 分红送股 | `quote f10 bonus <code>` |
| 个股公告（最新公告列表） | `quote announcement <code>` |
| 互动易问答（投资者提问与公司回复） | `quote interaction <code>` |

### E. Market overview (大盘/市场) — don't guess index codes

| 用户想要 | 命令 |
|---|---|
| 主要指数 + 沪深资金（默认大盘总览） | `market` |
| 纯大盘指数 | `market index` |
| 市场情绪（涨停/跌停/封板率/涨跌分布） | `market emotion` |
| 港股排行 | `market hk [sort]` |
| 期现基差（现货 vs 期货价差） | `market basis [date]`（`-g <板块>`：黑色系/金属/农产品/能源化工/金融） |
| 官方当日龙虎榜（带上榜原因） | `market longhu [date]` |
| 全球央行利率 | `market rates` |
| 全市场领涨/领跌名单、涨跌家数、按行业看谁在涨、全场热度（热力图） | `market heatmap`（免 key。`--sort up\|down`；`--plate` 名只取输出 `plates[]`，不是申万/概念；外汇 `-t forex`。`summary`=家数，`items`=Top N。`--metric`/`--compare` 见 commands.md） |

> 两种"异动"别混：盘中实时板块轮动/拉升跳水时间轴 = `sector anchor`（板块项的 `cls*` 码直接 `sector <code>` 下钻成分股）；收盘后龙虎榜（异常波动上榜） = `market longhu`。某股异动后想找同**行业**联动股用 `quote related`（同**概念板块**成员用 `sector plate` → `sector <plate>`）。
>
> **`market index` 只覆盖 A 股主要指数**（上证/深证/创业板等）。**全球指数（标普/道琼斯/恒生/日经/韩国KOSPI）走 `price CODE.INDEX`，不是 `market`、更不是 `quote`** —— 见 Intent 表 B。

### F. Capital flow (资金面) — per-stock vs market-wide

| 用户想要 | 命令 |
|---|---|
| 个股主力资金（主力/超大/大/中/小单净额） | `quote flow <code>` |
| 北向 / 南向资金（沪港通/深港通） | `quote flow market`（或 `market flow`） |
| 个股历史资金流（多日 + 3/5/10/20日净额） | `quote flow history <code> -c <num>` |
| 融资融券 | `quote flow margin <code>` |

### G. News & info (资讯) — `quote search` ≠ `info search`

| 用户想要 | 命令 |
|---|---|
| 名字 → 股票代码（茅台/腾讯） | `quote search <名字>`（搜股票，**不是**搜新闻） |
| 实时快讯/滚动电报 | `info flash`（`-s cls\|jin10\|glh` 选频道风格） |
| **某品种相关快讯**（黄金/铁矿有什么新闻） | `info flash symbol <CODE.EXCHANGE>`（裸码/别名自动归一化，如 `info flash symbol XAUUSD`） |
| 新闻/深度文章 | `info news`（`news depth <channel>` 深度；`news stock <code>` 个股；`news futures` 期货头条） |
| 话题/题材热点 | `info topic`（`topic hot` 热门题材） |
| 搜资讯/新闻/电报/文章 | `info search <关键词>`（**不是**搜股票代码；用单个词或整体短语） |

### H. Macro / calendar (宏观/日历)

| 用户想要 | 命令 |
|---|---|
| 宏观数据 / 大事 / 全球假期 | `calendar macro\|event\|holiday [date]` |
| A股 / 港股 / 美股 数据日历 | `calendar ashare\|hk\|us [date]` |
| **A股交易日历**（近一年哪些天开市） | `calendar trading`（需 key） |
| **期货产业数据日历**（马棕油出口/检验量等） | `calendar futures [date]` |
| 期货日历事件 / 假期 | `calendar future-event\|future-holiday [date]` |

### I. 盘面特色数据（涨停池/连板/热榜/异动/竞价，需 key）

都挂在 `market`（盘面域）与 `quote`（个股域）下，与既有命令同体系：

| 用户想要 | 命令 |
|---|---|
| 今日/某日**涨停池**（连板数、封单额、涨停原因、涨停时间） | `market limitup [date]`（默认按连板数降序；`--sort seal_money\|limit_up_time`） |
| 跌停池 / 炸板池 | `market limitdown [date]` / `market limitbreak [date]`（开板次数） |
| **连板天梯**（近30交易日梯队矩阵，次日晋级） | `market ladder` |
| **热股榜 / 人气飙升榜** Top30 | `market hot [date]` / `market skyrocket`（`--hour` 小时级；`market hot <date>` 历史） |
| 个股热度排名走势 | `market hot-trend <code> --days 30` |
| **个股异动原因**（大涨/大跌/快速拉升为什么） | `market anomaly <codes...>`（按股查）；`market anomaly [LIMIT_UP SHARP_FALL ...]`（按标签过滤）；不带参数=全盘 |
| **龙虎榜**（净买/买卖额/机构游资拆分/上榜原因，一年内） | `market longhu [date] [--board org\|hot_money]`（需 key 才有机构/游资拆分与历史；无 key 仅当日榜） |
| **集合竞价**（竞价价格/量比/未匹配/换手，个股盘前） | `quote auction <codes...> [--live]` |
| 短线风向标（高开/放量标签池） | `market benchmark [date]` |

> 与 `market emotion`（只有涨停/跌停家数等聚合指标）区分：要**逐股明细+原因**用上表命令。涨停池条目的 `reason` 就是官方涨停原因，写复盘直接引用。
> `[date]` 为 `YYYY-MM-DD`（池类/龙虎榜一年内；竞价基准当日）。

### J. 基金 / ETF（需 key）

| 用户想要 | 命令 |
|---|---|
| 基金档案（经理/规模/费率；输出 manager_id / company_id 供下钻） | `fund profile <code>` |
| 净值序列（单位+复权） | `fund nav <code> --range week\|month\|...\|fyear`（省略=最新一条） |
| 区间收益 + 同类平均 + 同类排名 | `fund perf <code>` |
| 最大回撤（各周期） | `fund drawdown <code>` |
| 重仓持仓（股/债/基 + 汇总占比） | `fund holdings <code>`；历史明细 `fund report-dates <code>` → `fund holdings-history <code> --end <日期> [--bond]` |
| 资产/行业配置 | `fund allocation <code>` / `fund industry <code>` |
| 持有人结构 / 前十大 | `fund holders <code> [--top]` |
| 基金经理 / 公司 | `fund manager <code> [--range year]` / `fund company <company_id>` |
| 财务/分红/诊断/新发/资讯 | `fund financial\|dividend\|diagnose\|issue\|news ...` |
| **ETF 实时行情 / 历史日K** | `fund etf <code>` / `fund etf-kline <code> --days 120`（仅 ETF；LOF/场外不支持） |

> 代码格式：场外基金带 `.OF` 后缀（如 `025480.OF`），ETF 用 6 位或带 `.SH/.SZ`（如 `510300`）。搜基金代码用 `quote search <名称或代码>`。

## Common workflows (commands chain naturally)

Real questions are usually a 2–3 command chain, not one command. These are the recurring patterns — reach for them instead of improvising:

- **Name → quote → peers**: `quote search 茅台` → `quote SH600519` → `quote related SH600519` (same-industry peers)
- **Intraday sector rotation (top-down)**: `sector anchor` → grab a plate's `cls*` code → `sector <code>` (成分股) → `quote <stock>`
- **One stock → its concept plate (bottom-up)**: `sector plate <stock>` (which plates it belongs to) → `sector <plate>` (sibling stocks)
- **Full single-stock picture**: `quote <code>` (price) + `quote f10 indicator <code>` (PE/PB/ROE snapshot) + `quote flow <code>` (fund flow)
- **Market + money flow**: `market heatmap` (广度+领涨领跌) → `market` (indices + 沪深资金) or `market index`, then `quote flow market` (northbound)
- **Name → live ticks**: `price search 黄金 -n 5 \| price sub -s 10`（search 的输出直接管给 sub，它会抽出 code；每 tick 一行 JSON）
- **Cross-asset K-line**: `quote kline <stock>` (equity) + `price kline XAUUSD.GOODS` (commodity) — same `-p` / `-n` flags
- **短线情绪复盘**（需 key）: `market heatmap` (广度+领涨) → `market limitup` (谁涨停+几连板+原因) → `market ladder` (梯队与晋级) → `market longhu` (资金榜) → `market hot` (人气)
- **异动归因**（需 key）: `sector anchor` / `market anomaly` 发现异动 → `market anomaly <code>` 原因 → `quote related` / `sector plate` 找联动标的
- **量化回测取数**（需 key）: `quote kline <code> --adjust forward -n 250` (复权日K) + `quote adjust <code>` (分红事件核对) + `quote valuation <codes...>` (估值截面)
- **基金诊断**（需 key）: `fund profile <code>` (拿 manager_id/company_id) → `fund perf` + `fund drawdown` (业绩风险) → `fund holdings` (持仓) → `fund manager <code>` (经理)

## Command index

One line per command — purpose + the flag that matters. For full flags, category lists, and examples, open [`references/commands.md`](references/commands.md); for `price` code tables, [`references/price-codes.md`](references/price-codes.md).

| Command | Does | Key flag(s) |
|---|---|---|
| `quote <code...>` | 证券行情（股票；也支持外汇对/国债收益率等符号）。2+ 空格分隔 = 批量 | `quote batch` 显式批量 |
| `quote search <name>` | Name → code, A/HK/US in one call (the name→code entry point) | returns ~10 |
| `quote related <code>` | Same-industry peers | — |
| `quote industry [ind_code]` | Industry → member stocks | `--market cn\|us\|hk`, `-n` |
| `quote board <type>` | Stocks by board (`cyb`/`kcb`/`us`/`hk`/`us_china`…) | `--order-by`, `-n` |
| `quote kline <code>` | K线；`--adjust` 复权日K | `-p`, `-n`, `--src`, `--adjust` |
| `quote <88xxxx.TI>` | 概念/行业指数快照（批量；码从 `sector search <名称>` 查） | 需 key |
| `quote valuation <codes...>` | 批量估值 PE/PB/PS/PCF（≤100） | 需 key |
| `quote indicators <code> [report]` | 五维财务指标（成长/盈利/偿债/营运/现金流） | `2025-4` |
| `quote adjust <code>` | 复权因子事件流（每股分红/送转） | `--from/--to` |
| `quote finance <code>` | 多期报表（`--src ths` 需 key） | `-t`, `--src`, `--period` |
| `quote depth \| ticks \| timeline <code>` | 5-level orderbook / tick trades / intraday line | `--src` |
| `quote flow [code \| market \| history \| margin \| intraday \| xq]` | 资金流向（个股或北向/南向） | `-c` (history) |
| `quote f10 [indicator \| holders \| bonus]` | Fundamentals snapshot (latest single period) | — |
| `quote announcement <code>` | 公司公告 | `-p` |
| `quote interaction <code>` | 投资者问答 / 互动易 | `-p`, `-s`, `-k` |
| `quote industry-profile <code>` | Which industry a stock belongs to (F10) | — |
| **`price [codes...]`** | **Non-equity engine** — forex / commodity / **global indices (any country, incl. KOSPI/日经/标普/恒生)** / 期货主力. `CODE.EXCHANGE`, batch. NOT `quote` | code tables → price-codes.md |
| **`price sub [codes...]`** | 实时行情订阅，stdout 每 tick 一行 JSON（可 pipe）；无参数读 stdin | `-s` 秒（省略不退出） |
| **`price search <关键词>`** | 品种搜索：中文/代码子串 → 精确 `CODE.EXCHANGE`（例子见 Intent 表 B） | `-n` |
| **`price orderflow <code>`** | 订单流：逐笔买卖盘价量采集，**全数据写 JSON 文件**，stdout 报 file/records | `-s 秒`, `--out` |
| **`price f10 <code>`** | 期货品种基本面 F10（供给/需求/库存指标，仅国内期货） | `--tag <id>` |
| **`price kline <code>`** | 非股票 K 线历史 | `-p`, `-n`, `--from`, `--to` |
| `sector [code \| anchor \| rank \| plate \| stocks \| search]` | 板块/概念工作流（`cls*` 板块码与 `88xxxx.TI` 指数自动路由） | `-t`, `--way` |
| `fund [profile \| nav \| perf \| drawdown \| holdings \| ... \| etf \| etf-kline]` | **基金全量数据**（17 个子命令，需 key） | `--range`, `--type`, `--days` |
| `key [set\|show\|clear\|test] <source>` | API key 管理（`key set ths <key>`） | — |
| `info [codes \| flash \| news \| topic \| search]` | News, flash, topics, full-site search, 代码表；`flash futures` 期货快讯 / `flash symbol <code>` 品种快讯（`--symbol` 旗标亦可） / `news futures` 期货头条 | `-s`, `-c` |
| `market [index \| emotion \| basis \| hk \| flow \| rates \| heatmap \| longhu \| limitup \| limitdown \| limitbreak \| ladder \| hot \| skyrocket \| hot-trend \| anomaly \| benchmark]` | 大盘 / 情绪 / 期现基差 / 热力图领涨领跌 / 涨跌停池/连板天梯/热榜/异动/龙虎榜（后十项需 key） | `[date]`, `--sort`, `--board`, `-g`, `--plate` |
| `calendar [trading \| macro \| event \| holiday \| futures \| future-event \| future-holiday \| ashare \| hk \| us]` | Trading-day calendar / macro & econ calendar / 期货产业数据日历 | `[date]` |

## Global options

| Flag | Default | Description |
|------|---------|-------------|
| `-n <num>` | varies | Item count. `10` for most; `kline` 120; `industry`/`board` 30. (`flow history` uses `-c` instead.) |
| `-h` | — | Help |

## Name → code lookup

To turn a company name into a tradable code, run **`finflow quote search <keyword>`**. It covers **A-shares + HK + US in one call** and stays current as stocks list/rename.

Each hit is `{code, name, type}`. **`type` is always `"stock"` — don't filter on it; read the market/instrument from the `code` prefix instead:**

| `code` shape | market / instrument | example |
|---|---|---|
| `SH` / `SZ` + 6 digits | A-share | `SH600519` 贵州茅台 |
| bare 5 digits | HK stock | `00700` 腾讯控股 |
| plain letters | US ticker | `NVDA`, `AAPL`, `BYDDY` (US ADR) |
| `BK****` | 板块 / 上市板 | `BK0088` 白酒 |
| `CSI*` / `SH000***` / `SZ399***` | index / ETF | `SZ399997` 中证白酒 |

A name that is also listed in HK/US returns all of them at once (e.g. `比亚迪` → `SZ002594` A, `01211` H, `BYDDY` US ADR) — pick by prefix.

Two boundaries to expect:
- **Concept/industry words return plates, indices, and ETFs — not a member-stock list.** `quote search 半导体` gives you `BK0021 半导体` and SOXX/SOXL, not the constituent stocks. For *industry → its stocks*, use `quote industry <ind_code>`; for *concept → its members*, use `sector <plate_code>`.
- **Search matches the official security name (substring/prefix), not slang.** `茅台` works (it's a substring of 贵州茅台); pure market slang like `宁王` returns nothing — normalize it to the official name/短称 before searching.

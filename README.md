# Polymarket Crypto Up/Down — Market Data

Tick-level market data for Polymarket crypto Up/Down markets, collected 24/7
directly from the source feeds. This repository hosts **free samples and
documentation**; the full dataset is sold by subscription or by range.

本仓库提供 Polymarket 加密 Up/Down 市场数据的**免费样例与文档**；完整数据集
按订阅或按区间出售。

**Full dataset:** [outcometick.com](https://outcometick.com) — subscribe to the
newest 30 days, or buy any date range once (priced per archived day). A key is
issued at checkout and works with the [HTTP API](https://outcometick.com/docs/api).
Strategy code can also run against the archive in a hosted sandbox at
[/backtest](https://outcometick.com/backtest), nothing to download first.
**Free:** the sample day below, and `ot run` replays it locally on the same
engine — no key needed.

**完整数据集：** [outcometick.com](https://outcometick.com/zh)——订阅最近 30 天，
或一次性买断任意日期区间（按归档天计价）。付款后即发 key，配合
[HTTP API](https://outcometick.com/zh/docs/api) 使用。策略代码也可以直接在
[/backtest](https://outcometick.com/zh/backtest) 的托管沙箱里跑归档数据，不用先下载。
**免费：** 下面的样例日，加上 `ot run` 本地重放，同一个引擎，不需要 key。

> Also available: **[Predict.fun data](https://github.com/outcometick/predict.fun-data-samples)**
> — the prediction-market venue behind the Binance Wallet front end, collected
> and sold separately (different venue, different settlement source).
>
> 另有 **[Predict.fun 数据](https://github.com/outcometick/predict.fun-data-samples)**
> （币安钱包前端接入的预测市场），独立采集、独立出售。

## What the full dataset contains / 完整数据集内容

| dataset | description |
|---|---|
| `prices` | The instantaneous Chainlink Data Streams feed, tick-by-tick (~1Hz per symbol), the relay's full-precision fixed-point value whenever the relay publishes it, three independent timestamps per tick. **Settled the markets through 2026-08-06**; still the underlying price line |
| `twap` | The **Chainlink TWAP streams that settle the markets since 2026-08-07** — the 60s-lookback stream settles both 5-minute and 15-minute markets today; the 30s stream no longer decides any outcome and was collected and shipped through 2026-10-19 (~1Hz, same columns as `prices`) |
| `book` | Full-depth CLOB order-book snapshots, up to 1/sec per token |
| `price_change` | Order-book deltas with best bid/ask. Kept at most 1 per market per 500ms — **since 2026-08-25, per 20ms for BTC and 100ms for ETH**, and **since 2026-10-18, per 20ms for every asset** (up to a 25x increase in resolution) |
| `best_bid_ask` | **Unthrottled top of book** — every top-of-book move the venue pushed, prices only (no sizes). The same quote `price_change` carries, at full resolution instead of the capture cadence. From 2026-09-02 |
| `last_trade_price` | Trade prints as the venue's WebSocket broadcast them — stored unthrottled, never sampled; not reconciled against on-chain fills |
| `markets` | Per-market metadata with **settlement outcome** (who won) and **strike** (the official priceToBeat) |

- Assets: BTC, ETH, SOL, DOGE, XRP, BNB, HYPE × intervals 5m / 15m
- History from 2026-06-06, growing daily; TWAP streams from 2026-08-08
  (first complete UTC day 2026-08-09)
- Every file comes with its size and SHA-256, so you can verify what you
  received; every price row carries its own timestamps, so gaps in the feed are
  measurable directly from the data
- How the order-book, top-of-book and trade streams are captured, and what each
  one can and cannot reconstruct: [Polymarket order book data](https://outcometick.com/polymarket-order-book-data)
  / [Polymarket 盘口数据说明](https://outcometick.com/zh/polymarket-order-book-data)

### Rebuilding an order book / 重建盘口

`book` and `price_change` are kept at the cadence above, so frames that arrived
between two kept ones are not in the archive. Each `price_change` item sets a
level to an absolute size (size `0` removes it), so a removal that fell between
kept frames leaves that level in a rebuilt ladder until the next `book` line
for that token replaces the whole ladder. To rebuild: start each token from its
latest `book` line, apply `price_change` items on top, then drop any bid above
or ask below the newest best bid/ask — from `best_bid_ask` (unthrottled) where
the day has it, otherwise from the `best_bid`/`best_ask` on the `price_change`
item itself. That removes every level the best prices have moved past, but it
cannot restore what the dropped frames added or resized: until the next
snapshot a level, at the top too, can be missing or carry an old size, because
`best_bid_ask` has prices only, no sizes. For the latest best bid/ask
themselves, read `best_bid_ask`.
Field-by-field schema: [docs/schemas](https://outcometick.com/docs/schemas).

`book` 与 `price_change` 按上表的频率保留，两次被保留的帧之间到达的帧不在归档里。
`price_change` 的每一项是把某个价位设成绝对挂单量（size 为 `0` 表示删除），所以
如果删除恰好落在两次保留之间，重建出的档位会一直留着这个价位，直到该 token 的下一行
`book` 整体替换它。重建方法：每个 token 从最近一行 `book` 开始，应用 `price_change`，
再删掉高于最新最优买价的买档、低于最新最优卖价的卖档——最优价取自不限流的
`best_bid_ask`（当天有这条流时），否则取 `price_change` 那一项自带的
`best_bid`/`best_ask`。这样能删掉所有已被最优价越过的档位，但补不回被丢掉的帧里
新增或改量的档位：在下一次快照前，某个价位可能缺失或挂单量是旧的，盘口顶部也一样，
因为 `best_bid_ask` 只有价格、没有挂单量。要看最新的最优买卖价本身，直接读
`best_bid_ask`。逐字段说明见
[docs/schemas](https://outcometick.com/zh/docs/schemas)。

### Settlement source change on 2026-08-07 / 结算源切换（2026-08-07）

From 2026-08-07 00:00 UTC, Polymarket settles its crypto Up/Down markets on
Chainlink **TWAP** (time-weighted average price) streams instead of the
instantaneous feed: the market resolves Up when the TWAP value at the close is
greater than or equal to the TWAP value at the open. **Which stream governs a
given market is written on the market itself**, in
`raw.cryptoMarketConfig.twapLookbackSeconds` — read it per market rather than
inferring it from the date. It started as 30s for 5-minute markets and 60s for
15-minute ones; upstream migrated the 5-minute markets to the 60s stream over
2026-08-13/14, so **since 2026-08-15 every market settles on the 60s stream**.
The 30s stream no longer decides any outcome; it was collected and shipped
through 2026-10-19 (UTC). **To recompute outcomes, use the `twap` files for markets from
2026-08-07 onward and the `prices` files for earlier markets.**
The TWAP streams cannot be reconstructed exactly from the ~1Hz instantaneous
ticks — Chainlink computes them from its internal higher-frequency data — which
is why the dataset carries both lines. 2026-08-07 itself predates our TWAP
collection; for that single day the official outcome labels in the `markets`
files are the settlement authority. The settlement rule, and how to recompute
an outcome yourself, are written up in
[how Polymarket Up/Down markets settle](https://outcometick.com/docs/settlement); the settlement
price lines themselves are described under
[Chainlink settlement price data](https://outcometick.com/chainlink-settlement-data).

2026-08-07 00:00 UTC 起，Polymarket 加密 Up/Down 市场改用 Chainlink **TWAP**
（时间加权均价）流结算：收盘 TWAP 值 ≥ 开盘 TWAP 值判 Up。**用哪条流结算写在市场
自己身上**，即 `raw.cryptoMarketConfig.twapLookbackSeconds`——请逐个市场读该字段，
不要按日期推断。最初 5 分钟市场用 30 秒回看、15 分钟用 60 秒；上游已在
2026-08-13/14 把 5 分钟市场迁到 60 秒流，**2026-08-15 起所有市场都用 60 秒流结算**。
30 秒流已不再决定任何输赢，采集与出货至 2026-10-19（UTC）为止。**重算输赢：
2026-08-07 起的市场用 `twap` 文件，此前的市场用 `prices` 文件。** TWAP 流无法从
约 1Hz 的瞬时 tick 精确重建（Chainlink 用其内部更高频数据计算），因此数据集同时
提供两条线。2026-08-07 当天早于我们的 TWAP 采集起点，该天以 `markets` 文件中的
官方结算标签为准。结算规则与自行重算的方法见
[Polymarket 涨跌市场怎么判定输赢](https://outcometick.com/zh/docs/settlement)；结算价流本身的说明见
[Chainlink 结算价数据](https://outcometick.com/zh/chainlink-settlement-data)。

## Samples / 样例

**[Download the sample bundle](https://github.com/outcometick/polymarket-tick-data-samples/releases/latest/download/polymarket-data-samples.tar.gz)**
— one real, unmodified UTC day (**2026-10-18**) of the BTC 5-minute series plus all
three BTC settlement price lines.

The same sample day, together with the Predict.fun one, is also on
[Kaggle](https://www.kaggle.com/datasets/ligengxin96/polymarket-predict-fun-tick-data)
(unpacked) and
[Hugging Face](https://huggingface.co/datasets/Ligengxin96/polymarket-predict-fun-tick-data)
(original `.gz` files), under CC BY 4.0.

同一个样例日（连同 Predict.fun 那份）也发布在
[Kaggle](https://www.kaggle.com/datasets/ligengxin96/polymarket-predict-fun-tick-data)（解压版）
与 [Hugging Face](https://huggingface.co/datasets/Ligengxin96/polymarket-predict-fun-tick-data)
（原样 `.gz`），协议 CC BY 4.0。

> **The data is in the Release, not in the git tree.** Clicking *Code → Download
> ZIP* gets you the documentation and nothing else. This is deliberate: a
> half-populated archive would replay without errors and give wrong answers, so
> the repository ships either the whole day or none of it.
>
> **数据在 Release 里，不在 git 仓库里。** 点 *Code → Download ZIP* 只会拿到文档。
> 这是有意的：残缺的归档跑起来不报错却会给出错误结果，所以要么给完整一天，要么不给。

```bash
curl -L https://github.com/outcometick/polymarket-tick-data-samples/releases/latest/download/polymarket-data-samples.tar.gz | tar xz
```

It unpacks to `polymarket-data-samples/`, laid out exactly like the paid
archive, so it replays directly:

```bash
ot run . --data ./polymarket-data-samples
```

| file (inside the bundle) | rows | what |
|---|---|---|
| `data/chainlink-twap-60s/…/BTCUSD-twap60s-prices-2026-10-18.csv.gz` | 85,339 | **TWAP 60s stream — settles every market today** |
| `data/chainlink-twap-30s/…/BTCUSD-twap30s-prices-2026-10-18.csv.gz` | 84,644 | TWAP 30s stream — archived through 2026-10-19, no longer settles anything |
| `data/chainlink/…/BTCUSD-prices-2026-10-18.csv.gz` | 85,339 | instantaneous Chainlink feed |
| `data/polymarket/…/book/BTC-5m/BTC-5m-book-2026-10-18.jsonl.gz` | 132,930 | order-book snapshots |
| `data/polymarket/…/best_bid_ask/BTC-5m/BTC-5m-best_bid_ask-2026-10-18.jsonl.gz` | 1,140,802 | **unthrottled top of book** |
| `data/polymarket/…/price_change/BTC-5m/BTC-5m-price_change-2026-10-18.jsonl.gz` | 3,798,184 | order-book deltas (best bid/ask), at the 20ms resolution |
| `data/polymarket/…/last_trade_price/BTC-5m/BTC-5m-last_trade_price-2026-10-18.jsonl.gz` | 386,998 | trade prints, as broadcast |
| `data/polymarket/…/markets/BTC-5m/BTC-5m-markets-2026-10-18.jsonl.gz` | 292 | markets + outcomes + strikes |

**What the unthrottled stream is worth, measured on this day.** A "move" is a
change in the pair (best bid, best ask) for one token, counted the same way on
both files. `best_bid_ask` records **488,616** of them. The throttled deltas
recover **306,828 — 62.8%**. And that is at the finest throttle we run: on this
day deltas were kept at one per 20ms for every asset. Before 2026-10-18 only BTC
ran at 20ms (ETH at 100ms, everything else at 500ms), so on those days other
assets keep far less of the top of book. That is the entire reason this stream
exists.

**未限流的那条流值多少钱，用本样例日实测。**「一次变动」＝某个 token 的
(最优买价, 最优卖价) 发生变化，两个文件用同一口径计数。`best_bid_ask` 记录了
**488,616 次**；限流的增量流只能还原 **306,828 次，即 62.8%**。而这已经是我们
**最细的限流档位**——样例日当天所有币种的增量流都是每 20ms 保留一条。2026-10-18 之前
只有 BTC 是 20ms（ETH 100ms、其余币种 500ms），那些日子里其它币种能留下的顶部变动
远少于此。这就是这条流存在的全部理由。

Each file is also attached to the Release individually, for anyone who only
wants one of them.

Row counts are data rows (CSV headers excluded); every file carries its SHA-256
in [`samples/manifest.json`](samples/manifest.json). Those checksums are the
archive's own, so a sample verifies byte for byte against a delivered bundle.

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文). The same column definitions online:
[dataset schema reference](https://outcometick.com/docs/schemas) / [字段规格](https://outcometick.com/zh/docs/schemas).

## Capture latency / 采集延迟

Timestamps are only worth what the capture path is worth, so here is ours,
measured on the sample day. `recv_ms` is our receive time; the reference is the
timestamp the upstream itself put on the message.

| stream | p50 | p95 |
|---|---|---|
| CLOB order book (`recv_ms − event_ts_ms`) | **9 ms** | 41 ms |
| Top of book, unthrottled (`recv_ms − payload.timestamp`) | **8 ms** | 51 ms |
| Chainlink TWAP 60s stream (`recv_ms − server_ts_ms`) | **246 ms** | 398 ms |
| Chainlink TWAP 30s stream (`recv_ms − server_ts_ms`) | **226 ms** | 341 ms |
| Chainlink instantaneous feed (`recv_ms − server_ts_ms`) | **271 ms** | 406 ms |

Collection runs next to the venues' own infrastructure (`eu-west-1`). Every
settlement-price tick through 2026-10-19 carries all three timestamps (event,
relay server, our receive), and every order-book and trade row carries the venue's event time and
our receive time, so you can verify the capture path yourself rather than take
our word for it.

采集点部署在 `eu-west-1`，紧邻场方基础设施。2026-10-19 及以前每条结算价 tick 都带三个时间戳（事件、
relay 服务端、我方接收），每条盘口与成交行带场方事件时间与我方接收时间，延迟链路
可自行核验。

## Verify it yourself / 自行验证

The point of a settlement feed is that you can re-derive the outcome from it.
Here is that check, run against these exact files — nothing but this bundle is
needed to reproduce it.

Every market in this sample carries `raw.cryptoMarketConfig.twapEnabled = true`
with a **60-second lookback**, so the governing rule is the current one: **Up
wins when the TWAP value at the close is greater than or equal to the TWAP value
at the open.** Both values are read from
`BTCUSD-twap60s-prices-2026-10-18.csv.gz` at the exact boundary seconds, using
the full-precision integer column (`full_accuracy_value`; every row in this
sample carries the relay's own fixed-point integer) — no floating point anywhere
in the comparison.

On 2026-10-18 the BTC 5-minute series had **288 markets settling inside the
day** (the file's other 4 rows close after midnight and settle in the next
day's file):

| | count | result |
|---|---|---|
| we hold the exact TWAP reports at both boundary seconds | 271 | **271 of 271 reproduce the official outcome** |
| a boundary second is not present in this file | 17 | not independently provable — reported as undetermined |

We call a settlement reproducible only when we hold the exact boundary-second
reports on **both** ends. A neighbouring tick is not proof of where the boundary
actually landed, so those markets are reported as undetermined rather than
counted as agreement — and of the 271 where we do hold both ends, every single
one matches Polymarket.

本样例中每个市场的 `raw.cryptoMarketConfig.twapEnabled` 均为 `true`、**回看 60 秒**，
因此适用当前规则：**收盘时刻 TWAP 值 ≥ 开盘时刻 TWAP 值则 Up 赢**。两端取值均来自
`BTCUSD-twap60s-prices-2026-10-18.csv.gz` 中精确边界秒的全精度整数列
（`full_accuracy_value`；本样例每一行都是 relay 自己发布的定点整数），比较过程全程
不使用浮点。

2026-10-18 当天 BTC 5 分钟局共 **288 个在本日内结算的市场**（文件中另外 4 行收盘在
午夜之后，于次日文件结算）：两端都持有精确边界秒 TWAP 报价的 **271 个，271/271
全部与官方结果一致**；另有 17 个因某一端边界秒不在本文件内，按不可独立证明处理、
不作判定。

只有两端都持有精确边界秒报价，我们才称其可复现——邻近 tick 并不能证明边界究竟落在
哪一侧，因此这类市场记为未判定而非算作一致。而这 271 个全部对得上。

### Run the audit's own code / 用审计自己的代码跑一遍

The `verify/` folder holds the code that produces outcometick.com's public
settlement audit — the same files, copied verbatim. It settles every market this
sample can settle on its own, compares each verdict with the audit published for
that day, and with `--official` asks Polymarket's own API for a sample of the
outcomes. Node 18+, no dependencies.

`verify/` 目录里就是生成 outcometick.com 公开结算审计的代码（原样复制）。它把本样例
能独立判定的每个市场重算一遍，逐个与当天公开的审计结论比对；加 `--official` 时再抽样
向 Polymarket 自己的接口核对官方结果。需要 Node 18+，无依赖。

```bash
git clone https://github.com/outcometick/polymarket-tick-data-samples
curl -L https://github.com/outcometick/polymarket-tick-data-samples/releases/latest/download/polymarket-data-samples.tar.gz | tar xz
node polymarket-tick-data-samples/verify/verify-settlement.mjs --data ./polymarket-data-samples --official 30
```

The published audit: https://outcometick.com/polymarket-settlement-audit

## Gaps in the feed / 断档

The feed does have gaps. The archive records the Chainlink reports as Polymarket's
public real-time feed relays them, while the official outcome is resolved from
Chainlink Data Streams directly — so a pause in that relay can leave the venue
with a report the archive lacks. A missing second can also come from a capture
loss; the files do not tell these apart. Every price row carries its own
timestamps, so you can locate and size the gaps straight from the files before
you backtest, rather than take a completeness claim on trust.

feed 确实存在断档。归档记录的是 Polymarket 公开实时通道转发的 Chainlink 报价，而官方
结果直接按 Chainlink Data Streams 判定——通道暂停时，官方仍有那一秒的报价，归档里
没有；缺的那一秒也可能来自采集丢失，文件本身分辨不出是哪一种。每条价格行都带自己的
时间戳，回测前可直接从文件里定位并评估断档，不必对完整性声明照单全收。

## Buy / 购买

**[outcometick.com](https://outcometick.com)** — subscribe or buy a date range,
then pull the data with your API key. It is the same archive these samples come
from, scoped to exactly the assets, data types and dates you purchase.
Weighing it against other sources? See [what to compare when choosing Polymarket
data](https://outcometick.com/compare).

Questions, or something not working:
**[Telegram group](https://t.me/+TcK_hJOzOz5mZTk1)** ·
**[Discord](https://discord.gg/ZDMqkptwm5)** (English and 中文) ·
**support@outcometick.com**

**[outcometick.com](https://outcometick.com)** —— 按订阅或按日期区间购买，用 API
key 自助取数。样例即取自同一份归档，购买后按你选择的币种、数据类型与日期范围开放。
在和其他来源比较？见[选数据时该比什么](https://outcometick.com/zh/compare)。

有疑问或遇到问题：**[Telegram 群](https://t.me/+TcK_hJOzOz5mZTk1)** ·
**[Discord 交流群](https://discord.gg/ZDMqkptwm5)**（中英文分区）·
**support@outcometick.com**

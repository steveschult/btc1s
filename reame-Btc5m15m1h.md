# BTC 5m/15m/1h Kline Export

这个目录保存 BTCUSDT 的 5m、15m、1h K 线月度 JSONL 导出，供 AI 回测和策略分析读取。

## 目录结构

```text
btc1s/
  btc5m/
    index.json
    2026-07.jsonl
  btc15m/
    index.json
    2026-07.jsonl
  btc1h/
    index.json
    2026-07.jsonl
```

周期对应关系：

```text
btc5m   BTCUSDT 5 分钟 K线
btc15m  BTCUSDT 15 分钟 K线
btc1h   BTCUSDT 1 小时 K线
```

## JSONL 数据格式

每个 `.jsonl` 文件是一行一根 K 线。示例：

```json
{"t":1783790400000,"ct":1783790699999,"o":64239.88,"h":64239.88,"l":64192,"c":64209.99,"v":16.89478,"qv":1084761.156408,"n":4972,"tb":4.98412,"tq":319988.1478293}
```

字段含义：

```text
t   K线开始时间，毫秒时间戳，对应数据库 _id / Binance openTime
ct  K线结束时间，毫秒时间戳，对应 Binance closeTime
o   open，开盘价
h   high，最高价
l   low，最低价
c   close，收盘价
v   volume，成交量，单位 BTC
qv  quoteAssetVolume，成交额，单位 USDT
n   trades，成交笔数
tb  takerBuyBaseVolume，主动买入成交量，单位 BTC
tq  takerBuyQuoteVolume，主动买入成交额，单位 USDT
```

## AI 推荐读取方式
这些文件是按月保存的。AI 不应该扫描所有月份文件，而应该先把 5m 轮次时间转成毫秒，再根据这个时间戳计算它在 `Asia/Shanghai` 时区属于哪个月份，直接定位该月份 JSONL 文件。

`poly-bot` 的 5m round 数据里 `timestamp` 通常是秒级轮次开始时间：

```json
{"timestamp":"1784626500"}
```

先转成毫秒：

```ts
const marketStartMs = Number(round.timestamp) * 1000;
```

根据 `marketStartMs` 计算月份。注意这里要用 `Asia/Shanghai`，因为导出文件的月边界也是按 `Asia/Shanghai` 计算：

```ts
const month = new Intl.DateTimeFormat("en-CA", {
  timeZone: "Asia/Shanghai",
  year: "numeric",
  month: "2-digit",
}).format(new Date(marketStartMs));
```

选择要读取的周期目录：

```text
5m   btc1s/btc5m
15m  btc1s/btc15m
1h   btc1s/btc1h
```

读取该目录的 `index.json`，检查这个月份文件是否存在并覆盖 `marketStartMs`：

```ts
const fileName = `${month}.jsonl`;
const entry = index.entries.find((item) => item.month === month && item.file === fileName);

if (!entry) {
  throw new Error(`missing kline month file: ${fileName}`);
}

if (entry.startMs > marketStartMs || entry.endMs < marketStartMs) {
  throw new Error(`kline month file does not cover marketStartMs=${marketStartMs}`);
}

const filePath = `${periodDir}/${entry.file}`;
```

例子：

```text
round.timestamp = "1784626500"
marketStartMs   = 1784626500000
month           = 2026-07
periodDir       = btc1s/btc5m
index           = btc1s/btc5m/index.json
entry.file      = 2026-07.jsonl
filePath        = btc1s/btc5m/2026-07.jsonl
```

读取月度 JSONL 后，只使用该 5m 轮次开始前已经完整收盘的 K 线：

```ts
const rows = jsonlRows
  .filter((row) => row.t < marketStartMs)
  .sort((a, b) => a.t - b.t);
```

获取上一根完整 K 线：

```ts
const prev = rows.at(-1);
```

判断上一根 K 线是否正好是“上一根”，需要按周期对齐：

```ts
const intervalMs = 5 * 60 * 1000;
const expectedPrevStart = Math.floor((marketStartMs - intervalMs) / intervalMs) * intervalMs;
const isAccurate = prev?.t === expectedPrevStart;
```

15m 和 1h 只替换周期毫秒数：

```text
5m   300000
15m  900000
1h   3600000
```

完整流程：

```text
1. 从 round.timestamp 取得 5m 轮次开始时间，单位通常是秒
2. marketStartMs = Number(round.timestamp) * 1000，转成毫秒
3. 按 Asia/Shanghai 从 marketStartMs 算出月份，例如 2026-07
4. 选择周期目录：btc5m / btc15m / btc1h
5. 拼出月文件名：`${month}.jsonl`
6. 读取该目录 index.json，确认 entries 中存在该 month/file，且 startMs/endMs 覆盖 marketStartMs
7. 读取 entry.file，例如 2026-07.jsonl
8. 过滤 row.t < marketStartMs，得到轮次开始前已收盘的历史 K 线
9. 按 row.t 升序排序，再取最近 N 根用于趋势判断
```

趋势判断建议读取：

```text
5m   marketStartMs 前已经完整收盘的最近 12-50 根
15m  marketStartMs 前已经完整收盘的最近 8-50 根
1h   marketStartMs 前已经完整收盘的最近 6-50 根
```

注意：真实策略代码中 5m、15m、1h K 线来自 Redis/数据库，不应在运行时用 1m 或 5m 临时聚合 1h。回测缺少某周期数据时才可以离线聚合补齐。

## index.json 格式

每个周期目录都有独立的 `index.json`。示例：

```json
{
  "interval": "5m",
  "symbol": "BTCUSDT",
  "timezone": "Asia/Shanghai",
  "lastTimestamp": "1784626200000",
  "updatedAt": "2026/07/21 17:35:35",
  "entries": [
    {
      "interval": "5m",
      "symbol": "BTCUSDT",
      "month": "2026-07",
      "timezone": "Asia/Shanghai",
      "file": "2026-07.jsonl",
      "startMs": 1782835200000,
      "endMs": 1785513599999,
      "rows": 2786,
      "addedRows": 1,
      "generatedAt": "2026-07-21T09:35:35.492Z"
    }
  ]
}
```

字段含义：

```text
interval       K线周期：5m / 15m / 1h
symbol         交易对，当前为 BTCUSDT
timezone       导出月份边界使用的时区，当前为 Asia/Shanghai
lastTimestamp  当前周期已导出的最大 K线开始时间，毫秒时间戳
updatedAt      index.json 最近更新时间，北京时间字符串
entries        月度导出文件列表
month          月份，格式 YYYY-MM
file           对应 JSONL 文件名
startMs        该月起始毫秒时间戳，按 Asia/Shanghai 月边界计算
endMs          该月结束毫秒时间戳，按 Asia/Shanghai 月边界计算
rows           当前 JSONL 文件总行数
addedRows      最近一次执行导出时追加的行数
generatedAt    最近一次写入该 entry 的 UTC ISO 时间
```


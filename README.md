# BTC 1s Kline Export

这个目录存放 BTCUSDT 的 1 秒 K 线导出文件。数据不放在 `poly-bot` 仓库内，避免大文件进入 Git。

## 目录结构

```text
btc1s/
  README.md
  index.json
  2026-07-18.jsonl
  2026-07-19.jsonl
```

## 导出命令

在 `dataAnalysis/crypto` 目录运行：

```bash
npm run exportBtc1s
npm run exportBtc1s -- --date=2026-07-18
npm run exportBtc1s -- 2026-07-18
```

不传日期时，默认导出 `Asia/Shanghai` 当天。

## 文件格式

每天一个 `.jsonl` 文件，一行代表一根 BTCUSDT 1 秒 K 线。

示例：

```json
{"t":1784419838000,"o":64843.91,"h":64843.91,"l":64843.91,"c":64843.91,"v":0.01079,"tb":0}
```

字段含义：

```text
t   K线开始时间，毫秒时间戳，对应数据库 _id / Binance k.t
o   open，开盘价
h   high，最高价
l   low，最低价
c   close，收盘价
v   volume，成交量，单位 BTC
tb  takerBuyBaseVolume，主动买入成交量，单位 BTC
```

## index.json

`index.json` 用于让脚本或 AI 根据 timestamp 快速定位日期文件。

示例：

```json
[
  {
    "date": "2026-07-18",
    "timezone": "Asia/Shanghai",
    "file": "2026-07-18.jsonl",
    "startMs": 1784304000000,
    "endMs": 1784390399999,
    "rows": 86400,
    "generatedAt": "2026-07-19T03:00:00.000Z"
  }
]
```

## 和 5m 轮次关联

`poly-bot` 的 5m 轮次文件里有：

```text
timestamp // 秒
```

查询该轮次对应的 1s K 线：

```ts
const startMs = Number(round.timestamp) * 1000;
const endMs = startMs + 5 * 60 * 1000 - 1000;
```

然后在 `index.json` 中找到满足下面条件的文件：

```ts
entry.startMs <= endMs && entry.endMs >= startMs
```

读取对应 `.jsonl`，过滤：

```ts
row.t >= startMs && row.t <= endMs
```

这样可以避免把 `1sKlineRange` 直接嵌入轮次 JSON，减少主文件体积。

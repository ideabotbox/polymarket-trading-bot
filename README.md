# polymarket-trading-bot (Ladder Bot)
polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot polymarket trading bot

## A Python liquidity-maker bot for Polymarket 5m / 15m / 1h / 4h / 1d crypto Up/Down markets (Polymarket Trading Ladder bot).
This bot does not speculate on market direction.
Instead, it captures spread by selling both YES and NO outcome tokens at prices whose combined value exceeds $1.
The strategy focuses on market making, not directional trading.
<img width="1098" height="727" alt="image" src="https://github.com/user-attachments/assets/69f7c1d7-20c6-4b30-928b-5df382795c8a" />

## Overview

Polymarket trading bot is designed to provide automated liquidity in Polymarket prediction markets.

## Bot Workflow

## 🔄 Polymarket Bot Workflow

<table>
<tr>
<td>

**1. Split collateral**

Divide USDC into YES/NO outcome tokens.

**2. Pair selling**

Sell both YES and NO tokens simultaneously.

**3. Market resolution**

Wait until the prediction market resolves.

**4. Redeem**

Redeem the winning tokens for collateral.

</td>

<td>

<img src="https://github.com/user-attachments/assets/565fd7a8-db16-4799-ae8f-77d9fc6226d4" width="500">

</td>
</tr>
</table>
Profit is generated from spread capture, where the bot sells both outcomes at prices that sum to greater than $1.

## Strategy Concept

When a market is created on Polymarket:

- YES token = pays $1 if event happens

- NO token = pays $1 if event does not happen

### The bot:

Converts USDC.e into YES + NO tokens

Sells both tokens

Ensures the sum of prices ≥ target spread

Example:
| Token | Sell Price |
| ----- | ---------- |
| YES   | 0.54       |
| NO    | 0.49       |


Total received:

```
0.54 + 0.49 = 1.03
```

Cost to create pair:

```
1.00
```

Profit before fees:

```
1.03 - 1.00 = 0.03 per pair
```
Typical range:

```
Total received ≈ 1.03 – 1.10
```

After the market resolves:

* One token becomes **$1**
* The other becomes **$0**

The bot redeems the winning token.

---

# Strategy Flow

The bot runs a **state machine** cycle:

```
IDLE
  ↓
SPLIT
  ↓
PAIR_SELL
  ↓
MONITOR
  ↓
REDEEM
  ↓
IDLE
```

### Phase Explanation

| Phase     | Description                                |
| --------- | ------------------------------------------ |
| SPLIT     | Convert USDC.e into YES and NO tokens      |
| PAIR_SELL | Sell YES and NO tokens in sequential pairs |
| MONITOR   | Wait for market resolution                 |
| REDEEM    | Redeem winning tokens for USDC.e           |

---

# Pair Selling Logic

The bot sells tokens **in pairs** rather than all at once.


<table>
<tr>
<td>
### For every pair:
  
**1. Fetch orderbook**

**2. Read best bid prices**

**3. Compute optimal sell prices**

**4. Place YES and NO limit orders**

**5. Wait until both fill**

**6. Retry if timeout occurs**

</td>

<td>

<img width="1096" height="718" alt="image" src="https://github.com/user-attachments/assets/12bb8ba0-cf26-422e-942b-b6e9e1961565" />

</td>
</tr>
</table>

---

## Dynamic Pricing

Sell prices are calculated using:

```
yes_price = best_bid_yes + spread
no_price  = best_bid_no  + spread
```

Then the bot enforces:

```
yes_price + no_price ≥ min_pair_sum
```

Example configuration:

```
best_bid_yes = 0.49
best_bid_no  = 0.50
spread       = 0.01
```

Calculated:

```
yes_price = 0.50
no_price  = 0.51
sum = 1.01
```

If `min_pair_sum = 1.03`, the bot increases both prices proportionally.
### If running continuously
| Markets       | Cycles per hour | Hourly Profit |
| ------------- | --------------- | ------------- |
| 1 market (5m) | 12              | ~$36          |
| 2 markets     | 12              | ~$72          |
| 4 markets     | 12              | ~$144         |




```
polymarket-trading-ladder-bot
│
├── config
│   ├── default.yaml
│   └── paper.yaml
│
├── src/polybot5m
│
│   ├── cli.py
│   ├── config.py
│   ├── constants.py
│   ├── engine.py
│
│   ├── data
│   │   ├── gamma.py
│   │   └── slug_builder.py
│
│   └── execution
│       ├── clob_client.py
│       ├── executor.py
│       ├── split.py
│       └── redeem.py
│
└── exports
    └── liquidity_maker_activity.json
```

---

# Configuration

Main configuration file:

```
config/default.yaml
```

Example:

```yaml
liquidity_maker:
  portfolio_allocation_usdc: 100

  pair_size: 5
  min_pair_sum: 1.03

  price_follow_spread: 0.01

  pair_sell_timeout_seconds: 300
  order_check_interval_seconds: 5

  redeem_delay_seconds: 10
  stagger_delay_seconds: 2
```

---

# Important Parameters

| Parameter                    | Description                 |
| ---------------------------- | --------------------------- |
| portfolio_allocation_usdc    | USDC used per market cycle  |
| pair_size                    | Shares per YES/NO pair      |
| min_pair_sum                 | Minimum combined sell price |
| price_follow_spread          | Price offset above best bid |
| pair_sell_timeout_seconds    | Cancel orders if not filled |
| order_check_interval_seconds | Poll order status           |

---

# Supported Markets

Designed for **short-term Polymarket crypto markets**:


| Asset | 5m | 15m | 1h | 4h | 1d |
|------|------|------|------|------|------|
| BTC | ✅ | ✅ | ✅ | ✅ | ✅ |
| ETH | ✅ | ✅ | ✅ | ✅ | ✅ |
| SOL | ✅ | ✅ | ✅ | ✅ | ✅ |
| XRP | ✅ | ✅ | ✅ | ✅ | ✅ |

Slug format:

```
{symbol}-updown-{interval}-{epoch_timestamp}
```

Example:

```
btc-updown-5m-1709916600
```

---

# Run Modes

## Paper Trading (Recommended First)

Uses **real orderbook data** but **no real trades**.

```
polybot5m -c config/paper.yaml run
```

or

```
polybot5m run --dry-run
```

---

## Live Trading

Example:

```
polybot5m run --cycles 1 --allocation 10
```

Meaning:

* run one cycle
* allocate **$10 per market**

---

## Custom Run

```
polybot5m run \
  --allocation 500 \
  --cycles 5
```

---

# Environment Variables

Required for **live trading**.

```
PRIVATE_KEY=
FUNDER=

API_KEY=
API_SECRET=
API_PASSPHRASE=
```

Optional builder relayer keys:

```
BUILDER_API_KEY_1=
BUILDER_API_SECRET_1=
BUILDER_API_PASSPHRASE_1=
```

Multiple keys can be used to avoid rate limits.

---

# Activity Logs

Trading activity is exported to:

```
exports/liquidity_maker_activity.json
```

Example record:

```json
{
  "market": "btc-updown-5m",
  "action": "pair_sell",
  "yes_price": 0.52,
  "no_price": 0.51,
  "size": 5,
  "timestamp": 1709916600
}
```

---

# Summary

| Feature       | Description                |
| ------------- | -------------------------- |
| Strategy      | Liquidity market making | Ladder Sell Logic   |
| Profit source | Spread capture             |
| Market type   | Polymarket 5m / 15m / 1h / 4h / 1d crypto |
| Language      | Python / Rust              |
| APIs          | Gamma API + CLOB API       |
| Execution     | Builder Relayer            |

---

# BNBOT Complete Guide - Standard & PRO Edition

## Table of Contents

1. [Overview](#overview)
2. [Standard vs PRO Comparison](#standard-vs-pro-comparison)
3. [Standard Bot Features](#standard-bot-features)
4. [PRO Bot Features](#pro-bot-features)
   - [PRO v1 Features (Core MM)](#pro-v1-features-core-mm)
   - [PRO v2 Features (Advanced MM)](#pro-v2-features-advanced-mm)
   - [PRO v2.5+ Features (Inventory Flush)](#pro-v25-features-inventory-flush)
5. [Complete Configuration Guide](#complete-configuration-guide)
   - [[app] Section](#app-section)
   - [[profiles] Section](#profilessafe--profileshigh-section)
   - [[flush] Section (PRO v2.5+)](#flush-section-pro-v25-only)
6. [Advanced Tuning Guide](#advanced-tuning-guide)
   - [Inventory Flush Tuning](#inventory-flush-tuning-pro-v25)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
   - [Flush-Specific Issues](#flush-specific-issues-pro-v25)

---

## Overview

BNBOT is a grid trading bot for Binance Futures (USD-M) with two editions:

- **Standard Edition**: Basic grid bot with trend protection and risk controls
- **PRO Edition (v2.9)**: Advanced Market Maker (MM) features with dynamic spread, size, inventory skew, toxicity filters, and **Inventory Flush**

### Bot Modes

Both editions support:
- **Hedge Mode**: Separate LONG and SHORT positions (requires Binance Futures Hedge Mode account)
- **One-Way Mode**: Single position direction (standard Futures account)

### Current Versions

- **Standard**: v0.1.2
- **PRO**: v0.2.9-pro-v2

---

## Standard vs PRO Comparison

| Feature | Standard | PRO |
|---------|----------|-----|
| **Grid Strategy** | Fixed step, fixed size | Dynamic step (ATR-based), dynamic size (volatility-based) |
| **Spread Management** | Symmetric bid/ask | Inventory-skewed pricing |
| **Order Type** | LIMIT GTC | LIMIT GTX (Post-Only) |
| **Maker Ratio** | ❌ | ✅ Tracked |
| **Adverse Selection** | ❌ | ✅ Toxicity filter |
| **Rebuild Cooldown** | ❌ | ✅ Prevents churn |
| **Trend Debounce** | ❌ | ✅ Warmup + debounce |
| **WAIT Mode** | ❌ | ✅ Re-entry hysteresis |
| **Resume Cooldown** | ❌ | ✅ Prevents flip-flop |
| **Inventory Flush** | ❌ | ✅ Profit lock & risk-off (v2.5+) |
| **Flush Metrics** | ❌ | ✅ PnL, slippage, fees (v2.7+) |

### When to Use Standard

- Simple grid trading
- Low capital (< $1000)
- Learning/testing
- Don't need advanced MM features

### When to Use PRO

- Professional market making
- Larger capital (> $5000)
- Want to avoid adverse selection
- Need inventory management
- Want maker fee guarantee

---

## Standard Bot Features

### Core Features

1. **Grid Trading**
   - Fixed step size (`step_bps`)
   - Fixed order size (`base_order_usdt`)
   - Symmetric bid/ask spreads

2. **Trend Protection**
   - EMA-based trend detection
   - Partial position close on trend
   - Grid pause/resume

3. **Risk Controls**
   - Equity drawdown stop
   - Price deviation kill switch
   - ATR-based volatility pause
   - Position size limits

4. **Grid Management**
   - Automatic re-anchoring
   - Fill handling and regrid
   - Starter entry (optional)

### Standard Bot Flow

```
Start → Set leverage/margin → Compute regime → Starter entry (optional) → Place grid
  ↓
Main Loop:
  - Check kill switches
  - Detect trend/volatility
  - Handle fills → Regrid
  - Re-anchor if needed
  - Log metrics
```

---

## PRO Bot Features

### PRO v1 Features (Core MM)

#### 1. Dynamic Step (Spread)

**What it does:**
- Automatically widens spread when volatility (ATR%) increases
- Formula: `dyn_step_bps = base_step_bps + (atr_step_k * atr_pct)`

**Example:**
- ATR% = 0.1% (0.001) → step = 10 + (1400 * 0.001) = 11.4 bps
- ATR% = 0.3% (0.003) → step = 10 + (1400 * 0.003) = 14.2 bps

**Why it matters:**
- Avoids adverse selection during high volatility
- Captures more profit during low volatility

**Config parameters:**
- `base_step_bps`: Minimum step (default: 10 bps)
- `atr_step_k`: ATR scaling factor (default: 1400.0)
- `min_step_bps`: Floor to prevent too-tight spreads (default: 8 bps)

#### 2. Dynamic Size

**What it does:**
- Automatically reduces order size when volatility increases
- Formula: `scale = 1 / (1 + vol_size_k * atr_pct)`

**Example:**
- ATR% = 0.1% → scale = 1/(1+30*0.001) = 0.97 → 3% smaller orders
- ATR% = 0.3% → scale = 1/(1+30*0.003) = 0.91 → 9% smaller orders

**Why it matters:**
- Reduces exposure during volatile periods
- Maintains position size during calm markets

**Config parameters:**
- `base_order_usdt`: Base order notional (default: 160 USDT)
- `min_order_usdt`: Minimum order (must cover minQty) (default: 140 USDT)
- `vol_size_k`: Volatility scaling factor (default: 30.0)

#### 3. Inventory Skew

**What it does:**
- Asymmetric bid/ask spreads based on current inventory
- When inventory high → wider bid (slow buying), tighter ask (faster selling)
- When inventory low → tighter bid (faster buying), wider ask (slow selling)

**Example:**
- Inventory pressure = 0.45 (45% of max)
- Target = 0.35 (35%)
- Skew: bid_step = 12.3 bps, ask_step = 8.7 bps

**Why it matters:**
- Manages inventory risk automatically
- Prevents inventory from building too high/low

**Config parameters:**
- `inv_skew_k`: Skew strength in bps (default: 40.0)
- `inv_target_frac`: Target inventory as fraction of max_position (default: 0.35)
- `max_skew_bps`: Cap max skew to prevent extreme asymmetry (default: 8.0)

### PRO v2 Features (Advanced MM)

#### 4. Post-Only Orders (GTX)

**What it does:**
- Uses `timeInForce="GTX"` to guarantee maker fees
- Rejects order if it would immediately match (become taker)
- Auto-reprices if rejected (moves 1 tick away)

**Why it matters:**
- Guarantees maker fees (rebate instead of fee)
- Prevents accidental taker fills

**Config parameters:**
- `use_post_only`: Enable GTX orders (default: true)

**Log output:**
```
[ORDER][GTX] LIMIT BUY LONG qty=0.002 price=87599.90 id=12345
[REPRICE] GTX rejected (1/3), adjusting BUY to 87599.80
```

#### 5. Maker Ratio Tracking

**What it does:**
- Tracks percentage of fills that are maker vs taker
- Displays in periodic `[MM]` logs

**Why it matters:**
- Monitor if bot is getting maker fees
- Detect if spreads are too tight (causing taker fills)

**Log output:**
```
[MM] maker_ratio=95.2% (20/21) | toxicity=2.3bps | cancels=5
[MM] maker_ratio=N/A (0 fills)  # When no fills yet
```

#### 6. Adverse Selection Tracking (Toxicity)

**What it does:**
- Measures post-fill price movements against the bot
- BUY fill → price going DOWN is adverse
- SELL fill → price going UP is adverse
- Calculates rolling average toxicity score

**Why it matters:**
- Detects when bot is being "picked off"
- Triggers spread widening to protect against toxic flow

**Config parameters:**
- `adverse_threshold_bps`: Threshold to trigger (default: 8.0 bps)
- `toxicity_widen_bps`: Extra spread when toxic (default: 5.0 bps)
- `toxicity_lookback`: Number of fills to analyze (default: 10)

**Log output:**
```
[TOXIC] Widening spread by 5.0 bps due to adverse selection
[SKEW] book=LONG inv_pressure=0.30 bid_step_bps=12.3 ask_step_bps=8.7 toxicity=9.2bps
```

#### 7. Rebuild Cooldown

**What it does:**
- Prevents excessive grid cancellation/repositioning
- Minimum seconds between grid rebuilds

**Why it matters:**
- Reduces rate limit issues
- Prevents churn (cancel → place → cancel loop)
- Saves opportunity cost

**Config parameters:**
- `rebuild_cooldown_sec`: Min seconds between rebuilds (default: 5.0s)

#### 8. Trend Debounce & Warmup

**What it does:**
- Requires N consecutive trend hits before TREND ON
- Skips trend check during warmup period after rebuild
- Prevents premature trend activation

**Why it matters:**
- Avoids false trend signals right after grid placement
- Reduces unnecessary grid cancellations

**Config parameters:**
- `trend_debounce_hits`: Require N consecutive hits (default: 3)
- `warmup_after_rebuild_sec`: Skip trend check for N seconds (default: 20.0s)
- `starter_check_trend`: Only starter entry when trend is calm (default: true)

#### 9. WAIT Mode with Re-entry Hysteresis

**What it does:**
- If market is too trendy at startup, bot enters WAIT mode
- Requires N consecutive "calm" cycles before resuming
- Prevents jumping in/out of trades

**Why it matters:**
- Avoids entering trades in bad conditions
- Prevents flip-flop between WAIT and trading

**Config parameters:**
- `trend_debounce_hits`: Also used for re-entry (default: 3)

**Log output:**
```
[SKIP] starter entry + grid: trend_score=4.92 > trend_off=1.3 (too trendy, waiting)
[WAIT] trend=4.92 atr%=0.08% calm_hits=0/3
[RESUME] Market calm (trend=1.15, atr%=0.15%), starting grid...
```

#### 10. Resume Cooldown

**What it does:**
- Prevents TREND ON from triggering within 30s of RESUME
- Only triggers if trend is very strong (>1.5x threshold)

**Why it matters:**
- Prevents flip-flop between RESUME and TREND ON
- Reduces unnecessary grid cancellations

### PRO v2.5+ Features (Inventory Flush)

#### 11. Inventory Flush System

**What it does:**
- Optional feature to lock profits and reduce risk in bad conditions
- NOT a take-profit - it's risk-off + profit lock
- Multiple trigger conditions with hysteresis
- Maker-first execution (tries LIMIT GTX before MARKET)

**Why it matters:**
- Protects gains when equity reaches high-watermark
- Closes stuck inventory when market stays trendy
- Emergency circuit breaker for extreme volatility
- Reduces exposure during adverse conditions

**State Machine:**
```
FLUSH_OFF → FLUSH_ARMED → FLUSHING_MAKER → FLUSHING_MARKET → FLUSH_COOLDOWN → FLUSH_OFF
                ↓                                                    ↑
         (disarm if                                           (failed flush)
          trigger gone)                                             ↓
                                                              FLUSH_OFF
```

**Log output:**
```
[FLUSH] ARMED: PROFIT_CUSHION (0.52%) inv_pressure=0.25
[FLUSH] FLUSHING_MAKER: PROFIT_CUSHION (0.52%) inv=0.012 BTC ref_price=87500.00
[FLUSH] Maker order filled at 87504.20
[FLUSH] COOLDOWN: flush #1 complete | pnl=2.35 USDT | fee_est=0.17 USDT | avg_slippage=0.48bps
[FLUSH] Cumulative: total_pnl=2.35 USDT | total_fees=0.17 USDT
```

#### 12. Flush Triggers

**12a. Profit Cushion Flush (Equity HWM)**

**What it does:**
- Triggers when daily equity gain reaches threshold
- Uses High-Watermark (HWM) not just realized PnL
- Hysteresis: triggers at `profit_cushion_pct`, disarms at `profit_cushion_hysteresis_pct`
- Can be limited to once per day

**Config parameters:**
- `profit_cushion_on`: Enable this trigger (default: true)
- `profit_cushion_pct`: Trigger at X% daily gain (default: 0.50%)
- `profit_cushion_hysteresis_pct`: Disarm below X% (default: 0.25%)
- `profit_requires_risk_signal`: Only flush if risk signal present (default: true)
- `profit_cushion_once_per_day`: Limit to one flush per day (default: true)

**12b. WAIT Timeout Flush (Stuck Inventory)**

**What it does:**
- Triggers when stuck in WAIT mode with inventory that's not decreasing
- Uses "stuck" detection: inventory pressure not changing
- Can require loss condition before flushing

**Config parameters:**
- `wait_timeout_on`: Enable this trigger (default: true)
- `wait_timeout_sec`: WAIT > X seconds (default: 1800 = 30 min)
- `wait_stuck_sec`: Check stuck over X seconds (default: 600 = 10 min)
- `wait_stuck_delta`: Stuck = inv_pressure change < X (default: 0.05)
- `wait_requires_loss`: Only flush if losing (default: false)
- `wait_requires_risk_signal`: Only flush if risk signal present (default: true)

**12c. Trend Prolonged Flush**

**What it does:**
- Triggers when TREND ON lasts too long with significant inventory
- Time-based (not cycle-based) for consistency

**Config parameters:**
- `trend_prolonged_on`: Enable this trigger (default: false)
- `trend_prolonged_sec`: X seconds of TREND ON (default: 300 = 5 min)
- `trend_prolonged_requires_inventory_pressure`: Only if inv_pressure > X (default: 0.50)

**12d. Volatility Flush (Circuit Breaker)**

**What it does:**
- Emergency trigger for extreme volatility
- Hysteresis: triggers at `vol_atr_pct_on`, disarms at `vol_atr_pct_off`

**Config parameters:**
- `vol_flush_on`: Enable this trigger (default: false)
- `vol_atr_pct_on`: Trigger when ATR% > X (default: 0.018 = 1.8%)
- `vol_atr_pct_off`: Disarm when ATR% < X (default: 0.014 = 1.4%)
- `vol_requires_toxicity`: Also require toxicity high (default: false)

#### 13. Flush Execution

**Maker-First Strategy:**
1. Cancel grid orders
2. Place LIMIT GTX order to close inventory
3. Re-quote every `maker_requote_sec` if not filled
4. After `maker_first_timeout_sec`, switch to MARKET (if risky)
5. Total timeout: `max_flush_time_sec`

**Config parameters:**
- `maker_first`: Use maker orders first (default: true)
- `maker_first_timeout_sec`: Try maker for X seconds (default: 45)
- `maker_requote_sec`: Re-quote every X seconds (default: 5)
- `max_flush_time_sec`: Total time limit (default: 90)
- `market_only_if_risky`: Only use market if risk signal active (default: true)
- `close_offset_bps`: Offset from mark price for maker close (default: 4.0 bps)

#### 14. Flush Safety & Gating

**Gating conditions (all must pass):**
- `enabled = true` (master switch)
- `mode = "AUTO"` (not "OFF")
- `total_fills >= min_fills_before_enable` (enough data)
- `inv_pressure >= min_inventory_pressure` (significant inventory)
- Not in warmup (if `require_not_in_warmup = true`)
- Not in resume cooldown (if `require_not_in_resume_cooldown = true`)
- Not in flush cooldown (after successful/failed flush)

**Cooldowns:**
- `cooldown_after_flush_sec`: Wait after successful flush (default: 1800 = 30 min)
- `cooldown_after_failed_flush_sec`: Wait after failed flush (default: 600 = 10 min)

**Confirmation:**
- `flush_confirm_hits`: Need X consecutive cycles to confirm (default: 2)

#### 15. Flush Metrics (v2.7)

**Tracked metrics:**
- `flush_total_realized_pnl`: Cumulative PnL from all flushes
- `flush_total_fees`: Estimated fees (0.02% taker, 0.01% maker)
- `flush_slippage_history`: Slippage per flush (bps)
- `flush_count`: Number of successful flushes

**Log output:**
```
[FLUSH] COOLDOWN: flush #1 complete | pnl=2.35 USDT | fee_est=0.17 USDT | avg_slippage=0.48bps
[FLUSH] Cumulative: total_pnl=2.35 USDT | total_fees=0.17 USDT
[FLUSH] state=OFF hwm%=0.45 inv_p=0.22 flushes=1
```

---

## Complete Configuration Guide

### [app] Section

#### `symbol`
- **Type**: String
- **Default**: `"BTCUSDT"`
- **Description**: Trading pair symbol (e.g., BTCUSDT, ETHUSDT, SOLUSDT)
- **Tuning**: Choose based on your capital and risk tolerance

#### `use_testnet`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Use Binance Futures testnet (true) or mainnet (false)
- **Important**: 
  - Testnet keys only work with `use_testnet = true`
  - Real keys only work with `use_testnet = false`
  - Endpoint switches automatically: `demo-fapi.binance.com` (testnet) vs `fapi.binance.com` (real)

#### `mode`
- **Type**: String
- **Options**: `"SAFE"` or `"HIGH"`
- **Default**: `"SAFE"`
- **Description**: Selects which profile (`[profiles.SAFE]` or `[profiles.HIGH]`) to use
- **Tuning**: 
  - SAFE: Lower leverage, wider spreads, smaller positions
  - HIGH: Higher leverage, tighter spreads, larger positions

#### `poll_seconds`
- **Type**: Float
- **Default**: `2.0`
- **Description**: Main loop polling interval (seconds)
- **Tuning**: 
  - Lower = faster reaction but more API calls
  - Higher = slower reaction but fewer API calls
  - Recommended: 1.0-3.0 seconds

#### `dry_run`
- **Type**: Boolean
- **Default**: `false`
- **Description**: If true, bot logs orders but doesn't place them
- **Use case**: Testing config without risking capital

#### `initial_capital_usdt`
- **Type**: Float
- **Default**: `5000.0`
- **Description**: For reporting/sizing guidance (bot doesn't deposit this)
- **Tuning**: Set to your actual capital for accurate sizing

#### `starter_entry`
- **Type**: Boolean
- **Default**: `true` (Standard), `true` (PRO with `starter_check_trend`)
- **Description**: Place a market entry at startup before grid
- **PRO Note**: If `starter_check_trend = true`, starter is skipped if market is trendy

#### `starter_entry_usdt`
- **Type**: Float
- **Default**: `0` (uses `base_order_usdt`)
- **Description**: Custom starter entry size (0 = use `base_order_usdt`)

#### `grid_book` (Hedge Mode Only)
- **Type**: String
- **Options**: `"AUTO"`, `"LONG"`, `"SHORT"`
- **Default**: `"AUTO"`
- **Description**: 
  - `AUTO`: Follow EMA bias (fast > slow = LONG, slow > fast = SHORT)
  - `LONG`: Always trade LONG side
  - `SHORT`: Always trade SHORT side

### [profiles.SAFE] / [profiles.HIGH] Section

#### Basic Settings

##### `leverage`
- **Type**: Integer
- **Default**: `2` (SAFE), `10` (HIGH)
- **Description**: Futures leverage (1-125 on Binance)
- **Tuning**: 
  - Lower = safer but less profit
  - Higher = more profit but more risk
  - Recommended: 2-5 for SAFE, 5-10 for HIGH

##### `margin_type`
- **Type**: String
- **Options**: `"ISOLATED"` or `"CROSSED"`
- **Default**: `"ISOLATED"`
- **Description**: 
  - ISOLATED: Margin isolated per position (safer)
  - CROSSED: Margin shared across positions (more efficient but riskier)

##### `kline_interval`
- **Type**: String
- **Default**: `"1m"` (SAFE), `"1m"` (HIGH)
- **Description**: Kline interval for ATR/EMA calculation
- **Options**: `"1m"`, `"5m"`, `"15m"`, `"1h"`, etc.

##### `atr_len`
- **Type**: Integer
- **Default**: `14`
- **Description**: ATR period length (number of bars)
- **Tuning**: 
  - Lower = more reactive to recent volatility
  - Higher = smoother, less reactive

##### `ema_fast` / `ema_slow`
- **Type**: Integer
- **Default**: `20/60` (SAFE), `14/40` (HIGH)
- **Description**: EMA lengths for trend detection
- **Tuning**: 
  - Closer together = more sensitive
  - Further apart = less sensitive
  - Formula: `trend_score = abs(ema_fast - ema_slow) / atr`

#### Grid Structure

##### `levels_each_side`
- **Type**: Integer
- **Default**: `10` (SAFE), `18` (HIGH)
- **Description**: Number of grid levels on each side of anchor
- **Tuning**: 
  - More levels = more orders, more capital needed
  - Fewer levels = less orders, less capital

##### `step_bps` (Standard) / `base_step_bps` (PRO)
- **Type**: Float
- **Default**: `12` (Standard SAFE), `10` (PRO SAFE)
- **Description**: Grid step size in basis points (1 bps = 0.01%)
- **Tuning**: 
  - Smaller = more fills but tighter spreads
  - Larger = fewer fills but wider spreads
  - PRO: This is the base step, actual step is dynamic (ATR-adjusted)

##### `band_atr_mult`
- **Type**: Float
- **Default**: `2.5` (SAFE), `3.2` (HIGH)
- **Description**: Grid band half-width = ATR × band_atr_mult
- **Tuning**: 
  - Larger = wider grid, more capital needed
  - Smaller = tighter grid, less capital

##### `size_ramp`
- **Type**: Float
- **Default**: `0.05` (SAFE), `0.18` (HIGH)
- **Description**: Size increase per level (0.0 = fixed, 0.1 = +10% per level)
- **Tuning**: 
  - Higher = larger orders further from anchor
  - Lower = more uniform order sizes

#### Order Size

##### `base_order_usdt`
- **Type**: Float
- **Default**: `120` (Standard SAFE), `160` (PRO SAFE)
- **Description**: Base order notional in USDT
- **Important**: Must be >= `minQty × price` and >= 100 USDT (Binance requirement)
- **Tuning**: 
  - Check `[MIN_ORDER]` log to see actual minQty/minNotional
  - Example: BTC at $60k, minQty=0.002 → minNotional = $120
  - Set `base_order_usdt` >= max(minQty×price, 100) × 1.1 (buffer)

##### `min_order_usdt` (PRO Only)
- **Type**: Float
- **Default**: `140` (PRO SAFE)
- **Description**: Minimum order size (after dynamic size scaling)
- **Important**: Must cover minQty even when size is reduced by volatility

#### PRO: Dynamic Step Parameters

##### `atr_step_k` (PRO Only)
- **Type**: Float
- **Default**: `1400.0` (SAFE), `1800.0` (HIGH)
- **Description**: ATR scaling factor for dynamic step
- **Formula**: `dyn_step_bps = base_step_bps + (atr_step_k × atr_pct)`
- **Tuning**: 
  - Higher = more reactive to volatility
  - Lower = less reactive
  - Recommended: 1000-2000

##### `min_step_bps` (PRO Only)
- **Type**: Float
- **Default**: `8` (SAFE), `7` (HIGH)
- **Description**: Floor to prevent too-tight spreads
- **Tuning**: 
  - Should be < `base_step_bps`
  - Prevents spreads from getting too tight in low volatility

#### PRO: Dynamic Size Parameters

##### `vol_size_k` (PRO Only)
- **Type**: Float
- **Default**: `30.0` (SAFE), `45.0` (HIGH)
- **Description**: Volatility scaling factor for dynamic size
- **Formula**: `scale = 1 / (1 + vol_size_k × atr_pct)`
- **Tuning**: 
  - Higher = more aggressive size reduction
  - Lower = less size reduction
  - Recommended: 20-50

#### PRO: Inventory Skew Parameters

##### `inv_skew_k` (PRO Only)
- **Type**: Float
- **Default**: `40.0` (SAFE), `70.0` (HIGH)
- **Description**: Skew strength in basis points
- **Tuning**: 
  - Higher = more asymmetric pricing
  - Lower = less skew
  - Set to 0 to disable skew (symmetric pricing)

##### `inv_target_frac` (PRO Only)
- **Type**: Float
- **Default**: `0.35` (SAFE), `0.30` (HIGH)
- **Description**: Target inventory as fraction of max_position
- **Tuning**: 
  - Lower = faster inventory reduction
  - Higher = slower inventory reduction
  - Recommended: 0.25-0.40

##### `max_skew_bps` (PRO Only)
- **Type**: Float
- **Default**: `8.0` (SAFE), `12.0` (HIGH)
- **Description**: Cap max skew to prevent extreme asymmetry
- **Tuning**: 
  - Prevents bid/ask steps from deviating too far from base
  - Recommended: 6-12 bps

#### PRO v2: Maker & Toxicity Parameters

##### `use_post_only` (PRO v2 Only)
- **Type**: Boolean
- **Default**: `true`
- **Description**: Use GTX (Post-Only) orders
- **Tuning**: 
  - `true`: Guarantee maker fees (recommended)
  - `false`: Use GTC orders (may become taker)

##### `rebuild_cooldown_sec` (PRO v2 Only)
- **Type**: Float
- **Default**: `5.0` (SAFE), `3.0` (HIGH)
- **Description**: Min seconds between grid rebuilds
- **Tuning**: 
  - Prevents excessive cancel/replace churn
  - Recommended: 3-10 seconds

##### `adverse_threshold_bps` (PRO v2 Only)
- **Type**: Float
- **Default**: `8.0` (SAFE), `10.0` (HIGH)
- **Description**: Toxicity threshold to trigger spread widening (bps)
- **Tuning**: 
  - Lower = more sensitive to adverse selection
  - Higher = less sensitive
  - Recommended: 5-12 bps

##### `toxicity_widen_bps` (PRO v2 Only)
- **Type**: Float
- **Default**: `5.0` (SAFE), `8.0` (HIGH)
- **Description**: Extra spread when toxic (bps)
- **Tuning**: 
  - Higher = wider spread when toxic
  - Lower = less widening
  - Recommended: 3-10 bps

##### `toxicity_lookback` (PRO v2 Only)
- **Type**: Integer
- **Default**: `10` (SAFE), `15` (HIGH)
- **Description**: Number of fills to analyze for toxicity
- **Tuning**: 
  - More fills = smoother average but slower reaction
  - Fewer fills = faster reaction but more noise

#### PRO v2: Trend Debounce Parameters

##### `trend_debounce_hits` (PRO v2 Only)
- **Type**: Integer
- **Default**: `3`
- **Description**: Require N consecutive trend hits before TREND ON
- **Tuning**: 
  - Higher = less sensitive to trend
  - Lower = more sensitive
  - Recommended: 2-5

##### `warmup_after_rebuild_sec` (PRO v2 Only)
- **Type**: Float
- **Default**: `20.0` (SAFE), `15.0` (HIGH)
- **Description**: Skip trend check for N seconds after rebuild
- **Tuning**: 
  - Prevents false trend signals right after grid placement
  - Recommended: 15-30 seconds

##### `starter_check_trend` (PRO v2 Only)
- **Type**: Boolean
- **Default**: `true`
- **Description**: Only starter entry when trend is calm
- **Tuning**: 
  - `true`: Skip starter if market is trendy (recommended)
  - `false`: Always place starter entry

#### Risk Controls

##### `max_position`
- **Type**: Float
- **Default**: `0.02` (SAFE), `0.04` (HIGH) for BTCUSDT
- **Description**: Max position in base asset (e.g., BTC for BTCUSDT)
- **Important**: Must be >= `minQty` to allow any trades
- **Tuning**: 
  - Check minQty from `[MIN_ORDER]` log
  - Example: SOLUSDT minQty=1 SOL → `max_position` must be >= 1.0
  - Recommended: 2-5x minQty minimum

##### `max_open_orders`
- **Type**: Integer
- **Default**: `60` (SAFE), `120` (HIGH)
- **Description**: Max number of tracked open orders (local safety)
- **Tuning**: 
  - Should be >= `levels_each_side × 2`
  - Higher = more orders allowed

##### `trend_on` / `trend_off`
- **Type**: Float
- **Default**: `2.0/1.3` (SAFE), `2.6/1.8` (HIGH)
- **Description**: Trend score thresholds
- **Formula**: `trend_score = abs(ema_fast - ema_slow) / atr`
- **Tuning**: 
  - `trend_on` > `trend_off` (hysteresis)
  - Lower = more sensitive to trend
  - Higher = less sensitive
  - Recommended: trend_on = 1.5-3.0, trend_off = 1.0-2.0

##### `reduce_pct_on_trend`
- **Type**: Float
- **Default**: `0.35` (SAFE), `0.30` (HIGH)
- **Description**: Reduce position % when trend detected
- **Tuning**: 
  - Higher = close more position on trend
  - Lower = close less position
  - Recommended: 0.25-0.50

##### `equity_dd_stop`
- **Type**: Float
- **Default**: `0.12` (SAFE), `0.20` (HIGH)
- **Description**: Stop if equity drawdown > X% (0.12 = 12%)
- **Tuning**: 
  - Lower = more conservative
  - Higher = more aggressive
  - Recommended: 0.10-0.25

##### `reanchor_dev`
- **Type**: Float
- **Default**: `0.02` (SAFE), `0.015` (HIGH)
- **Description**: Reanchor if price moves > X% from anchor (0.02 = 2%)
- **Tuning**: 
  - Lower = more frequent reanchoring
  - Higher = less frequent
  - Recommended: 0.01-0.03

##### `hard_kill_dev`
- **Type**: Float
- **Default**: `0.08` (SAFE), `0.055` (HIGH)
- **Description**: Kill switch if price moves > X% from anchor (0.08 = 8%)
- **Tuning**: 
  - Lower = more conservative
  - Higher = more aggressive
  - Recommended: 0.05-0.10

##### `atr_pct_threshold`
- **Type**: Float
- **Default**: `0.020` (SAFE), `0.030` (HIGH)
- **Description**: Pause if ATR% > X% (0.020 = 2.0%)
- **Tuning**: 
  - Lower = pause earlier in volatility
  - Higher = pause later
  - Recommended: 0.015-0.035

### [flush] Section (PRO v2.5+ Only)

> **Important**: Keep `enabled = false` until you have 50+ fills and understand how your bot behaves. Flush should complement trend_protect, not replace it.

#### Master Control

##### `enabled`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Master ON/OFF switch for flush system
- **Tuning**: Start with false, enable after 50+ fills

##### `mode`
- **Type**: String
- **Options**: `"AUTO"`, `"OFF"`, `"FORCE"`
- **Default**: `"AUTO"`
- **Description**: 
  - `AUTO`: Normal trigger-based operation
  - `OFF`: Disable flush (same as enabled=false)
  - `FORCE`: Force flush when inv_pressure >= min (testing only)

#### Safety / Gating

##### `cooldown_after_flush_sec`
- **Type**: Float
- **Default**: `1800` (30 min)
- **Description**: Wait time after successful flush
- **Tuning**: Lower = more frequent flushes (aggressive)

##### `cooldown_after_failed_flush_sec`
- **Type**: Float
- **Default**: `600` (10 min)
- **Description**: Wait time after failed flush
- **Tuning**: Shorter than successful cooldown

##### `min_fills_before_enable`
- **Type**: Integer
- **Default**: `50`
- **Description**: Need X fills before flush can trigger
- **Tuning**: Higher = more data required (safer)

##### `flush_confirm_hits`
- **Type**: Integer
- **Default**: `2`
- **Description**: Need X consecutive cycles to confirm flush
- **Tuning**: Higher = more confirmation required

#### Execution Style

##### `maker_first`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Try maker orders before market
- **Tuning**: true = lower fees, false = faster execution

##### `maker_first_timeout_sec`
- **Type**: Float
- **Default**: `45`
- **Description**: Try maker for X seconds before market
- **Tuning**: Lower = faster switch to market

##### `maker_requote_sec`
- **Type**: Float
- **Default**: `5`
- **Description**: Re-quote maker order every X seconds
- **Tuning**: Lower = more aggressive re-quoting

##### `max_flush_time_sec`
- **Type**: Float
- **Default**: `90`
- **Description**: Total time limit for flush
- **Tuning**: After this, flush is considered failed

##### `close_offset_bps`
- **Type**: Float
- **Default**: `4.0`
- **Description**: Offset from mark price for maker close (bps)
- **Tuning**: 
  - Higher = more likely to fill as maker
  - Lower = better price but may not fill
  - Recommended: 2-6 bps

#### Profit Cushion Trigger

##### `profit_cushion_on`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Enable profit cushion trigger
- **Tuning**: Most important trigger for profit protection

##### `profit_cushion_pct`
- **Type**: Float
- **Default**: `0.50` (0.5%)
- **Description**: Trigger at X% daily equity gain
- **Tuning**: Lower = trigger earlier, Higher = wait for more profit

##### `profit_cushion_hysteresis_pct`
- **Type**: Float
- **Default**: `0.25` (0.25%)
- **Description**: Disarm when equity gain drops below X%
- **Tuning**: Should be < profit_cushion_pct

##### `profit_requires_risk_signal`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Only flush if risk signal (trend/toxic/vol) present
- **Tuning**: true = safer, false = flush even in calm markets

##### `profit_cushion_once_per_day`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Limit to one profit flush per day
- **Tuning**: true = prevent over-flushing

#### WAIT Timeout Trigger

##### `wait_timeout_on`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Enable WAIT timeout trigger
- **Tuning**: Useful for stuck inventory in WAIT mode

##### `wait_timeout_sec`
- **Type**: Float
- **Default**: `1800` (30 min)
- **Description**: WAIT > X seconds before considering flush
- **Tuning**: Lower = flush sooner when stuck

##### `wait_stuck_sec`
- **Type**: Float
- **Default**: `600` (10 min)
- **Description**: Check stuck over X seconds
- **Tuning**: Period for stuck detection

##### `wait_stuck_delta`
- **Type**: Float
- **Default**: `0.05`
- **Description**: Stuck = inv_pressure change < X
- **Tuning**: Lower = stricter stuck detection

##### `wait_requires_loss`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Only flush if unrealized PnL is negative
- **Tuning**: true = only flush losing positions

#### Trend Prolonged Trigger

##### `trend_prolonged_on`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Enable trend prolonged trigger
- **Tuning**: false = safer (let trend protection handle it)

##### `trend_prolonged_sec`
- **Type**: Float
- **Default**: `300` (5 min)
- **Description**: X seconds of TREND ON before flush
- **Tuning**: Lower = flush sooner in trends

##### `trend_prolonged_requires_inventory_pressure`
- **Type**: Float
- **Default**: `0.50`
- **Description**: Only if inv_pressure > X
- **Tuning**: Higher = only flush large positions

#### Volatility Trigger

##### `vol_flush_on`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Enable volatility trigger (circuit breaker)
- **Tuning**: false = safer, use only for extreme volatility

##### `vol_atr_pct_on`
- **Type**: Float
- **Default**: `0.018` (1.8%)
- **Description**: Trigger when ATR% > X
- **Tuning**: Should be high (extreme volatility only)

##### `vol_atr_pct_off`
- **Type**: Float
- **Default**: `0.014` (1.4%)
- **Description**: Disarm when ATR% < X
- **Tuning**: Should be < vol_atr_pct_on (hysteresis)

#### Position Gating

##### `min_inventory_pressure`
- **Type**: Float
- **Default**: `0.15`
- **Description**: Only flush if inv_pressure > X (15% of cap)
- **Tuning**: Higher = only flush larger positions

---

## Advanced Tuning Guide

### Tuning for Different Market Conditions

#### Low Volatility (ATR% < 0.1%)

**Standard:**
- Increase `step_bps` (wider spreads)
- Increase `base_order_usdt` (larger orders)

**PRO:**
- `base_step_bps` is already optimal
- `atr_step_k` will reduce step automatically
- Consider increasing `base_order_usdt` slightly

#### High Volatility (ATR% > 0.3%)

**Standard:**
- Consider pausing (ATR threshold)
- Reduce `base_order_usdt` manually

**PRO:**
- Dynamic step/size handle this automatically
- May want to increase `atr_step_k` for more reactivity
- May want to increase `vol_size_k` for more size reduction

#### Trending Markets

**Both:**
- `trend_on` / `trend_off` control sensitivity
- Lower thresholds = pause earlier
- Higher thresholds = trade longer in trend

**PRO:**
- `trend_debounce_hits` prevents false signals
- `warmup_after_rebuild_sec` prevents premature activation
- `starter_check_trend` skips entry in trendy markets

#### Range Markets

**Both:**
- Lower `trend_on` / `trend_off` to avoid false pauses
- Increase `base_order_usdt` for more profit
- Increase `levels_each_side` for more opportunities

**PRO:**
- Inventory skew helps manage inventory automatically
- Dynamic step/size optimize for current volatility

### Tuning for Different Symbols

#### BTCUSDT
- **minQty**: Usually 0.001 BTC
- **minNotional**: 100 USDT (Binance requirement)
- **Recommended `base_order_usdt`**: 150-200 USDT
- **Recommended `max_position`**: 0.02-0.05 BTC

#### ETHUSDT
- **minQty**: Usually 0.01 ETH
- **minNotional**: 100 USDT
- **Recommended `base_order_usdt`**: 120-180 USDT
- **Recommended `max_position`**: 0.2-0.5 ETH

#### SOLUSDT
- **minQty**: Usually 1 SOL
- **minNotional**: 100 USDT
- **Recommended `base_order_usdt`**: 150-200 USDT
- **Recommended `max_position`**: 5-10 SOL (must be >= 1.0)

**Important**: Always check `[MIN_ORDER]` log for actual minQty/minNotional!

### Tuning PRO Parameters

#### Dynamic Step Tuning

**If getting adverse selection:**
- Increase `atr_step_k` (more reactive to volatility)
- Increase `base_step_bps` (wider base spread)

**If too few fills:**
- Decrease `base_step_bps` (tighter base spread)
- Decrease `min_step_bps` (allow tighter spreads)

#### Dynamic Size Tuning

**If position too large in volatility:**
- Increase `vol_size_k` (more aggressive size reduction)

**If position too small:**
- Decrease `vol_size_k` (less size reduction)
- Increase `base_order_usdt`

#### Inventory Skew Tuning

**If inventory builds too high:**
- Decrease `inv_target_frac` (faster reduction)
- Increase `inv_skew_k` (stronger skew)

**If inventory too low:**
- Increase `inv_target_frac` (slower reduction)
- Decrease `inv_skew_k` (weaker skew)

**If skew too extreme:**
- Decrease `max_skew_bps` (cap the skew)

#### Toxicity Filter Tuning

**If getting picked off:**
- Decrease `adverse_threshold_bps` (more sensitive)
- Increase `toxicity_widen_bps` (wider spread when toxic)

**If too many false positives:**
- Increase `adverse_threshold_bps` (less sensitive)
- Increase `toxicity_lookback` (smoother average)

#### Inventory Flush Tuning (PRO v2.5+)

**Getting Started:**
1. Keep `enabled = false` initially
2. Run bot until you have 50+ fills
3. Review maker_ratio, toxicity, cancel metrics
4. Enable with conservative settings:
   ```toml
   [flush]
   enabled = true
   profit_cushion_on = true
   profit_requires_risk_signal = true
   wait_timeout_on = true
   trend_prolonged_on = false  # Keep off initially
   vol_flush_on = false        # Keep off initially
   ```

**If flush triggering too often:**
- Increase `profit_cushion_pct` (require more profit)
- Increase `min_fills_before_enable` (more data required)
- Increase `cooldown_after_flush_sec` (longer cooldown)
- Increase `flush_confirm_hits` (more confirmation)
- Set `profit_requires_risk_signal = true`

**If flush not triggering when needed:**
- Decrease `profit_cushion_pct` (lower threshold)
- Decrease `min_inventory_pressure` (flush smaller positions)
- Set `profit_requires_risk_signal = false`
- Enable additional triggers (trend_prolonged, vol_flush)

**If flush execution too slow:**
- Decrease `maker_first_timeout_sec` (faster switch to market)
- Decrease `maker_requote_sec` (more frequent re-quotes)
- Increase `close_offset_bps` (more attractive price)
- Set `market_only_if_risky = false` (always allow market)

**If slippage too high:**
- Increase `maker_first_timeout_sec` (longer maker attempt)
- Decrease `close_offset_bps` (better price)
- Set `market_only_if_risky = true` (avoid market in calm)

**Recommended Progression:**

**Week 1-2 (Learning):**
```toml
enabled = false  # Monitor only
```

**Week 3-4 (Testing):**
```toml
enabled = true
profit_cushion_on = true
profit_cushion_pct = 0.75  # Higher threshold
profit_requires_risk_signal = true
wait_timeout_on = true
trend_prolonged_on = false
vol_flush_on = false
min_fills_before_enable = 100  # More data
```

**After 100+ fills (Production):**
```toml
enabled = true
profit_cushion_on = true
profit_cushion_pct = 0.50
profit_requires_risk_signal = true
wait_timeout_on = true
wait_timeout_sec = 1800
trend_prolonged_on = false  # Enable if needed
vol_flush_on = false        # Enable only for emergencies
```

---

## Best Practices

### Pre-Launch Checklist

1. **Test on Testnet First**
   - Run for 24-48 hours on testnet
   - Verify all features work correctly
   - Check logs for errors

2. **Verify Config**
   - `base_order_usdt` >= minNotional (check `[MIN_ORDER]` log)
   - `max_position` >= minQty (check `[MIN_ORDER]` log)
   - `use_testnet` matches your API keys

3. **Start Small**
   - Use SAFE profile initially
   - Start with small capital
   - Monitor for first few days

4. **Monitor Key Metrics**
   - Maker ratio (should be > 80% for PRO)
   - Toxicity score (should be < threshold)
   - Cancel count (should be reasonable)
   - PnL (realized + unrealized)

### Runtime Monitoring

**Key Logs to Watch:**

```
[MIN_ORDER] minQty=0.001 BTC, minNotional=87.45 USDT
[CONFIG] base_order_usdt=160.0 USDT, max_position=0.02 BTC
[SKEW] book=LONG inv_pressure=0.30 bid_step_bps=12.3 ask_step_bps=8.7 toxicity=2.3bps
[MM] maker_ratio=95.2% (20/21) | toxicity=2.3bps | cancels=5
[REGIME] TREND ON score=2.15 atr%=0.12%
[PNL] Realized: +5.2 USDT | Unrealized: -1.3 USDT | Total: +3.9 USDT
```

**Red Flags:**
- `maker_ratio < 80%` → Spreads too tight
- `toxicity > adverse_threshold_bps` → Getting picked off
- `cancels` very high → Too much churn
- `[WARN] base_order_usdt < min_notional` → Config issue

### When to Stop Bot

1. **Equity Drawdown Stop**
   - Bot stops automatically if `equity_dd_stop` exceeded
   - Check logs for `[KILL] equity drawdown`

2. **Price Kill Switch**
   - Bot stops automatically if `hard_kill_dev` exceeded
   - Check logs for `[KILL] price deviation`

3. **Manual Stop**
   - Stop via tray app
   - Or create `STOP_FILE` (if configured)

4. **Market Conditions**
   - Major news events
   - Extreme volatility
   - Exchange issues

### Recovery Procedures

**After Stop:**
1. Check logs for reason
2. Verify positions on exchange
3. Adjust config if needed
4. Restart when conditions improve

**After Trend Protect:**
1. Bot automatically resumes when trend calms
2. Monitor for re-entry
3. Check inventory levels

---

## Troubleshooting

### Common Issues

#### "Order's notional must be no smaller than 100"
- **Cause**: `base_order_usdt` too small
- **Fix**: Increase `base_order_usdt` to >= 100 USDT (with buffer)
- **Check**: `[MIN_ORDER]` log for actual minNotional

#### "SKIP starter entry: blocked by max_position cap"
- **Cause**: Current position already at/above `max_position`
- **Fix**: 
  - Flatten position on exchange
  - Or increase `max_position` (if intentional)

#### "maker_ratio < 80%"
- **Cause**: Spreads too tight, orders crossing
- **Fix**: 
  - Increase `base_step_bps` (PRO)
  - Increase `step_bps` (Standard)
  - Check if `use_post_only = true` (PRO)

#### "toxicity > threshold"
- **Cause**: Getting picked off by toxic flow
- **Fix**: 
  - Increase `adverse_threshold_bps` (less sensitive)
  - Increase `toxicity_widen_bps` (wider spread)
  - Check if spreads are appropriate for volatility

#### "TREND ON immediately after startup"
- **Cause**: Trend detector too sensitive (Standard) or warmup not working (PRO)
- **Fix**: 
  - Standard: Increase `trend_on` / `trend_off`
  - PRO: Check `warmup_after_rebuild_sec` and `trend_debounce_hits`

#### "Bot not placing orders"
- **Cause**: Multiple possible
- **Check**:
  1. `[MIN_ORDER]` log - is `base_order_usdt` sufficient?
  2. `[REGIME]` log - is bot in TREND ON or ATR pause?
  3. `[WAIT]` log - is bot waiting for calm?
  4. `[ORDERS][LIVE]` - are orders actually placed?

#### "Config changes not applied"
- **Cause**: Bot needs restart to load new config
- **Fix**: Stop and restart bot via tray app

### PRO-Specific Issues

#### "GTX rejected" spam
- **Cause**: Spreads too tight, orders would cross
- **Fix**: 
  - Increase `base_step_bps`
  - Check `close_offset_bps` if using flush

#### "Inventory skew too extreme"
- **Cause**: `max_skew_bps` too high or `inv_skew_k` too high
- **Fix**: 
  - Decrease `max_skew_bps`
  - Decrease `inv_skew_k`

#### "WAIT mode too long"
- **Cause**: Market staying trendy
- **Fix**: 
  - This is by design (safety feature)
  - Wait for market to calm
  - Or adjust `trend_off` threshold (less conservative)
  - Consider enabling `wait_timeout_on` to flush stuck inventory

### Flush-Specific Issues (PRO v2.5+)

#### "Flush not triggering"
- **Cause**: Gating conditions not met
- **Check**:
  1. `enabled = true` in [flush] section
  2. `mode = "AUTO"` (not "OFF")
  3. `total_fills >= min_fills_before_enable`
  4. `inv_pressure >= min_inventory_pressure`
  5. Not in warmup/resume cooldown
  6. Risk signal present (if `profit_requires_risk_signal = true`)
- **Fix**: Adjust gating thresholds or wait for more fills

#### "Flush triggering too often"
- **Cause**: Thresholds too low
- **Fix**:
  - Increase `profit_cushion_pct`
  - Increase `cooldown_after_flush_sec`
  - Set `profit_requires_risk_signal = true`
  - Increase `min_fills_before_enable`

#### "FLUSH FAILED" appearing
- **Cause**: Could not complete flush within timeout
- **Fix**:
  - Increase `max_flush_time_sec`
  - Increase `close_offset_bps` (more attractive maker price)
  - Set `market_only_if_risky = false` (allow market sooner)
- **Note**: Failed flush has shorter cooldown (`cooldown_after_failed_flush_sec`)

#### "Flush slippage too high"
- **Cause**: Market execution with high slippage
- **Fix**:
  - Increase `maker_first_timeout_sec` (longer maker attempt)
  - Decrease `close_offset_bps` (better price)
  - Set `market_only_if_risky = true` (avoid market in calm)

#### "Flush maker order not filling"
- **Cause**: Price moving away from order
- **Fix**:
  - Increase `close_offset_bps` (more attractive price)
  - Decrease `maker_requote_sec` (faster re-quotes)
  - Decrease `maker_first_timeout_sec` (switch to market sooner)

#### "Flush PnL negative"
- **Cause**: Position closed at loss
- **Note**: Flush is risk-off, not take-profit
- **Fix**: 
  - Set `profit_requires_risk_signal = true` (only flush in bad conditions)
  - Set `wait_requires_loss = false` (don't flush profitable WAIT positions)
  - Increase `profit_cushion_pct` (require more cushion before flush)

---

## Version History

### Standard Bot
- **v0.1.2**: Current stable version
  - Basic grid trading
  - Trend protection
  - Risk controls

### PRO Bot
- **v0.2.0**: Initial PRO release
  - Dynamic step, size, inventory skew
  - Post-only orders
  - Maker ratio tracking
  - Toxicity filter
  - Rebuild cooldown

- **v0.2.1**: Trend debounce & warmup
  - Trend debounce hits
  - Warmup period
  - Max skew cap
  - Starter entry trend check

- **v0.2.2**: WAIT mode improvements
  - Skip grid when trendy at startup
  - WAIT mode with re-entry hysteresis
  - Consistent counter resets

- **v0.2.3**: Robust WAIT handling
  - Risk switches active in WAIT
  - Metrics logging in WAIT
  - Inventory management

- **v0.2.4**: Final stability fixes
  - Resume cooldown (30s)
  - Counter resets on state changes
  - Post-only retry limits
  - Toxicity with insufficient data check

- **v0.2.5**: Inventory Flush Foundation
  - State machine: OFF → ARMED → FLUSHING_MAKER → FLUSHING_MARKET → COOLDOWN
  - Daily equity HWM tracking
  - WAIT/Trend duration tracking
  - Inventory pressure history (stuck detection)
  - [flush] TOML config section
  - Risk signal detection (trend/toxic/vol/churn)
  - Basic flush state logging

- **v0.2.6**: Flush Triggers
  - Profit Cushion trigger (equity HWM-based)
  - Profit cushion hysteresis
  - Once-per-day enforcement
  - WAIT Timeout trigger (stuck inventory)
  - Stuck inventory detection
  - Loss requirement check
  - Trend Prolonged trigger (time-based)
  - Volatility trigger with hysteresis
  - Trigger confirmation (flush_confirm_hits)

- **v0.2.7**: Flush Execution & Metrics
  - Mark price support for flush pricing
  - Maker-first execution with re-quote loop
  - Market emergency fallback
  - Failed flush handling
  - Slippage tracking
  - Fee estimation (maker/taker)
  - Cumulative flush PnL tracking
  - Enhanced flush logging

- **v0.2.8**: MM Review Fixes
  - UTC day boundary for `once_per_day` flush (fixes timezone issues)
  - Reordered trigger priority: VOL > TREND > WAIT > PROFIT (tail risk first)
  - Trend protect interaction: 60s cooldown before flush after `trend_protect` reduce
  - Added `last_trend_reduce_ts` tracking to avoid double-reduce

- **v0.2.9**: Accurate Maker Detection
  - Fixed maker/taker detection using `get_account_trades` API
  - `query_order` doesn't return `maker` field, only `userTrades` does
  - Now queries actual trade records to determine maker status
  - Metrics (maker_ratio, toxicity) now based on accurate data

---

## Additional Resources

- **BOT_PRO_GUIDE.md**: Detailed PRO features guide
- **INVENTORY_FLUSH_DESIGN.md**: Flush feature design document
- **SETUP_GUIDE.md**: Initial setup instructions
- **BOT_CONFIG_GUIDE.md**: Config structure details

---

## Support

For issues or questions:
1. Check logs for error messages
2. Review this guide's troubleshooting section
3. Verify config against examples
4. Test on testnet first

---

**Last Updated**: 2026-01-28 (PRO v2.9)

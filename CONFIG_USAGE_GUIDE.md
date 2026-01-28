# Hướng Dẫn Cài Đặt Config - BNBOT

## Mục Lục

1. [Tổng Quan File Config](#tổng-quan-file-config)
2. [Section [app] - Cài Đặt Cơ Bản](#section-app---cài-đặt-cơ-bản)
3. [Section [profiles] - Cài Đặt Trading](#section-profiles---cài-đặt-trading)
4. [PRO: Dynamic Step & Size](#pro-dynamic-step--size)
5. [PRO: Inventory Skew](#pro-inventory-skew)
6. [PRO: Maker & Toxicity](#pro-maker--toxicity)
7. [PRO: Trend Debounce](#pro-trend-debounce)
8. [Các Preset Theo Symbol](#các-preset-theo-symbol)
9. [Troubleshooting Config](#troubleshooting-config)

---

## Tổng Quan File Config

### Vị Trí File

```
instances/
├── BTCUSDT/
│   └── bot_config.toml    ← Config cho BTC
├── ETHUSDT/
│   └── bot_config.toml    ← Config cho ETH
└── SOLUSDT/
    └── bot_config.toml    ← Config cho SOL
```

### Cấu Trúc File TOML

```toml
[app]                    # Cài đặt chung
symbol = "BTCUSDT"
...

[profiles.SAFE]          # Profile an toàn
leverage = 2
...

[profiles.HIGH]          # Profile aggressive
leverage = 10
...

[flush]                  # PRO only: Inventory flush
enabled = false
...
```

### Standard vs PRO

| Config | Standard | PRO |
|--------|----------|-----|
| `[app]` | ✅ | ✅ |
| `[profiles]` | ✅ (basic) | ✅ (extended) |
| `[flush]` | ❌ | ✅ |

---

## Section [app] - Cài Đặt Cơ Bản

### symbol

```toml
symbol = "BTCUSDT"
```

| Symbol | minQty | Khuyến nghị base_order |
|--------|--------|------------------------|
| BTCUSDT | 0.001 BTC | 150-200 USDT |
| ETHUSDT | 0.01 ETH | 120-180 USDT |
| SOLUSDT | 1 SOL | 150-200 USDT |

### use_testnet

```toml
use_testnet = true   # Testnet (demo)
use_testnet = false  # Real (mainnet)
```

⚠️ **QUAN TRỌNG:**
- Testnet key chỉ hoạt động với `use_testnet = true`
- Real key chỉ hoạt động với `use_testnet = false`
- KHÔNG dùng lẫn!

### mode

```toml
mode = "SAFE"   # Profile an toàn
mode = "HIGH"   # Profile aggressive
```

**Khi nào dùng gì:**
| Mode | Vốn | Kinh nghiệm | Volatility |
|------|-----|-------------|------------|
| SAFE | < $5000 | Mới | Bình thường |
| HIGH | > $5000 | Có kinh nghiệm | Thấp |

### grid_book (Hedge Mode Only)

```toml
grid_book = "AUTO"   # Theo EMA bias
grid_book = "LONG"   # Chỉ trade LONG
grid_book = "SHORT"  # Chỉ trade SHORT
```

**Khi nào dùng gì:**
- `AUTO`: Phổ biến nhất, tự chuyển theo trend
- `LONG`: Khi tin BTC sẽ tăng
- `SHORT`: Khi tin BTC sẽ giảm

### starter_entry

```toml
starter_entry = true    # Mở market entry khi start
starter_entry = false   # Chỉ đặt grid, không entry
```

**Khuyến nghị:**
- `true`: Muốn bot bắt đầu trade ngay
- `false`: Chỉ muốn đặt grid chờ fill

### poll_seconds

```toml
poll_seconds = 2.0   # Check mỗi 2 giây
```

| Giá trị | Ưu điểm | Nhược điểm |
|---------|---------|------------|
| 1.0 | Phản ứng nhanh | Nhiều API calls |
| 2.0 | Cân bằng | - |
| 5.0 | Ít API calls | Chậm phản ứng |

---

## Section [profiles] - Cài Đặt Trading

### leverage

```toml
leverage = 2    # SAFE: x2
leverage = 10   # HIGH: x10
```

**Khuyến nghị theo vốn:**
| Vốn | Leverage Max |
|-----|--------------|
| < $1000 | 2-3x |
| $1000-5000 | 3-5x |
| $5000-20000 | 5-10x |
| > $20000 | 10-20x |

### base_order_usdt

```toml
base_order_usdt = 160   # Mỗi order ~ 160 USDT
```

⚠️ **PHẢI >= 100 USDT** (Binance minNotional)

**Cách tính:**
```
base_order_usdt >= max(minQty × price × 1.1, 110)

Ví dụ BTC @ $90,000:
- minQty = 0.001 BTC
- minNotional = 0.001 × 90000 = $90
- base_order_usdt >= max(90 × 1.1, 110) = $110
- Khuyến nghị: 150-200 USDT
```

### max_position

```toml
max_position = 0.02   # Max 0.02 BTC
```

**Cách tính:**
```
max_position = (vốn × leverage × %rủi_ro) / giá

Ví dụ: Vốn $5000, leverage 2x, rủi ro 20%, BTC @ $90,000
max_position = (5000 × 2 × 0.20) / 90000 = 0.022 BTC
```

⚠️ **PHẢI >= minQty** của symbol!

### levels_each_side

```toml
levels_each_side = 10   # 10 lệnh mỗi bên (tổng 20)
```

| Levels | Vốn cần | Grid coverage |
|--------|---------|---------------|
| 5 | Thấp | Hẹp |
| 10 | Trung bình | Vừa |
| 18 | Cao | Rộng |

### step_bps (Standard) / base_step_bps (PRO)

```toml
step_bps = 12        # Standard: 0.12% mỗi level
base_step_bps = 10   # PRO: Base 0.10%, dynamic theo ATR
```

**Spread thực tế (round-trip):**
```
spread = step_bps × 2

Ví dụ step_bps = 12:
- Spread = 24 bps = 0.24%
- Maker fee ~2 bps/leg = 4 bps total
- Net profit/trade ≈ 20 bps = 0.20%
```

### size_ramp

```toml
size_ramp = 0.05   # Tăng 5% size mỗi level
```

**Ví dụ với base_order = 160, size_ramp = 0.05:**
```
Level 1: 160 × (1 + 0.05×0) = 160 USDT
Level 2: 160 × (1 + 0.05×1) = 168 USDT
Level 3: 160 × (1 + 0.05×2) = 176 USDT
...
Level 10: 160 × (1 + 0.05×9) = 232 USDT
```

### Risk Controls

#### trend_on / trend_off

```toml
trend_on = 2.0    # Pause grid khi trend_score > 2.0
trend_off = 1.3   # Resume grid khi trend_score < 1.3
```

**Trend score:**
```
trend_score = |EMA_fast - EMA_slow| / ATR

Ví dụ: EMA_20 = 90000, EMA_60 = 89500, ATR = 200
trend_score = |90000 - 89500| / 200 = 2.5 → TREND ON
```

#### reduce_pct_on_trend

```toml
reduce_pct_on_trend = 0.35   # Đóng 35% vị thế khi trend
```

**Ví dụ:**
```
Position = 0.02 BTC, trend_on triggers
→ Đóng 0.02 × 0.35 = 0.007 BTC
→ Còn lại 0.013 BTC
```

#### equity_dd_stop

```toml
equity_dd_stop = 0.12   # Stop nếu equity -12%
```

**Cách tính:**
```
equity_start = 5000
equity_current = 4400
drawdown = (5000 - 4400) / 5000 = 12%
→ BOT STOP!
```

#### hard_kill_dev

```toml
hard_kill_dev = 0.08   # Stop nếu giá move 8% từ anchor
```

**Ví dụ:**
```
anchor = 90000
price_current = 82000
deviation = |90000 - 82000| / 90000 = 8.9%
→ BOT STOP!
```

---

## PRO: Dynamic Step & Size

### Công Thức Dynamic Step

```toml
base_step_bps = 10      # Base spread
atr_step_k = 1400       # ATR multiplier
min_step_bps = 8        # Floor
```

```
dyn_step = max(min_step, base_step + atr_step_k × atr_pct)

Ví dụ ATR% = 0.10% (0.001):
dyn_step = max(8, 10 + 1400 × 0.001) = max(8, 11.4) = 11.4 bps
```

### Bảng Tham Khảo Dynamic Step

| ATR% | dyn_step (k=1400) |
|------|-------------------|
| 0.05% | 10.7 bps |
| 0.10% | 11.4 bps |
| 0.20% | 12.8 bps |
| 0.30% | 14.2 bps |
| 0.50% | 17.0 bps |

### Công Thức Dynamic Size

```toml
base_order_usdt = 160   # Base size
min_order_usdt = 140    # Floor
vol_size_k = 30         # Vol multiplier
```

```
scale = 1 / (1 + vol_size_k × atr_pct)
dyn_size = max(min_order, base_order × scale)

Ví dụ ATR% = 0.10%:
scale = 1 / (1 + 30 × 0.001) = 0.97
dyn_size = max(140, 160 × 0.97) = max(140, 155.2) = 155.2 USDT
```

### Bảng Tham Khảo Dynamic Size

| ATR% | scale (k=30) | size (base=160) |
|------|--------------|-----------------|
| 0.05% | 0.985 | 157.6 USDT |
| 0.10% | 0.971 | 155.3 USDT |
| 0.20% | 0.943 | 150.9 USDT |
| 0.30% | 0.917 | 146.7 USDT |
| 0.50% | 0.870 | 140.0 USDT (min) |

---

## PRO: Inventory Skew

### Công Thức

```toml
inv_skew_k = 40         # Skew strength (bps)
inv_target_frac = 0.35  # Target inventory
max_skew_bps = 8        # Max skew cap
```

```
inv_pressure = current_inventory / max_position
deviation = inv_pressure - inv_target_frac
raw_skew = deviation × inv_skew_k
skew = clamp(raw_skew, -max_skew, +max_skew)

bid_step = dyn_step + skew
ask_step = dyn_step - skew
```

### Ví Dụ Inventory Skew

**Scenario: Inventory cao (cần bán)**
```
inv_pressure = 0.50 (50% cap)
inv_target = 0.35
deviation = 0.50 - 0.35 = 0.15
raw_skew = 0.15 × 40 = 6.0 bps

dyn_step = 11 bps
bid_step = 11 + 6 = 17 bps (mua khó hơn)
ask_step = 11 - 6 = 5 bps → clamped to 8 bps (bán dễ hơn)
```

**Scenario: Inventory thấp (cần mua)**
```
inv_pressure = 0.10 (10% cap)
inv_target = 0.35
deviation = 0.10 - 0.35 = -0.25
raw_skew = -0.25 × 40 = -10.0 bps → clamped to -8 bps

dyn_step = 11 bps
bid_step = 11 - 8 = 3 bps → clamped to 8 bps (mua dễ hơn)
ask_step = 11 + 8 = 19 bps (bán khó hơn)
```

### Log Giải Thích

```
[SKEW] book=LONG inv=0.012 pressure=0.60 bid_step_bps=17.0 ask_step_bps=8.0
```

- `book=LONG`: Đang trade LONG side
- `inv=0.012`: Inventory hiện tại 0.012 BTC
- `pressure=0.60`: 60% của max_position
- `bid_step=17.0`: Mua ở xa hơn (giảm mua)
- `ask_step=8.0`: Bán ở gần hơn (tăng bán)

---

## PRO: Maker & Toxicity

### Post-Only (GTX)

```toml
use_post_only = true   # Dùng GTX orders
```

**GTX = Guarantee Maker:**
- Order bị reject nếu sẽ match ngay (taker)
- Bot tự reprice và retry
- Đảm bảo luôn là maker

**Log:**
```
[ORDER][GTX] LIMIT BUY LONG qty=0.002 price=87599.90 id=12345
[REPRICE] GTX rejected (1/3), adjusting BUY to 87599.80
```

### Toxicity Filter

```toml
adverse_threshold_bps = 8.0   # Ngưỡng toxicity
toxicity_widen_bps = 5.0      # Widen spread khi toxic
toxicity_lookback = 10        # Số fills để tính
```

**Adverse selection:**
```
BUY fill → giá đi xuống = adverse
SELL fill → giá đi lên = adverse

toxicity = trung bình adverse move (bps)
```

**Log:**
```
[MM] maker_ratio=95.2% (20/21) | toxicity=2.3bps | cancels=5
```

**Giải thích:**
- `maker_ratio=95.2%`: 95.2% fills là maker (tốt!)
- `toxicity=2.3bps`: Trung bình adverse move (thấp = tốt)
- `cancels=5`: Số lần cancel (không quá cao)

### Rebuild Cooldown

```toml
rebuild_cooldown_sec = 5.0   # Min 5s giữa các rebuild
```

**Mục đích:** Tránh cancel/replace liên tục (churn)

---

## PRO: Trend Debounce

### Warmup Period

```toml
warmup_after_rebuild_sec = 20.0   # Skip trend check 20s sau rebuild
```

**Mục đích:** Tránh TREND ON ngay sau khi đặt grid

### Trend Debounce

```toml
trend_debounce_hits = 3   # Cần 3 lần liên tiếp
```

**Mục đích:** Tránh false positive trend signal

**Log:**
```
[REGIME] TREND ON  score=2.15  atr%=0.12% (debounced)
```

### Starter Check Trend

```toml
starter_check_trend = true   # Skip starter nếu trendy
```

**Log:**
```
[SKIP] starter entry + grid: trend_score=4.92 > trend_off=1.3 (too trendy, waiting)
[WAIT] Waiting for trend to subside (need 3 calm hits)...
```

---

## Các Preset Theo Symbol

### BTCUSDT (Ổn định nhất)

```toml
[app]
symbol = "BTCUSDT"

[profiles.SAFE]
leverage = 2
base_order_usdt = 160
max_position = 0.02
levels_each_side = 10
base_step_bps = 10
atr_step_k = 1400
min_step_bps = 8
inv_skew_k = 40
trend_on = 2.0
trend_off = 1.3
```

### ETHUSDT (Vol cao hơn)

```toml
[app]
symbol = "ETHUSDT"

[profiles.SAFE]
leverage = 2
base_order_usdt = 150
max_position = 0.15
levels_each_side = 10
base_step_bps = 12          # Wider step
atr_step_k = 1600           # More reactive
min_step_bps = 10
inv_skew_k = 50             # Stronger skew
trend_on = 1.8              # More sensitive
trend_off = 1.2
```

### SOLUSDT (Vol cao nhất)

```toml
[app]
symbol = "SOLUSDT"

[profiles.SAFE]
leverage = 2
base_order_usdt = 180       # Lớn hơn vì minQty = 1 SOL
max_position = 5.0          # minQty = 1, nên >= 2-3
levels_each_side = 8
base_step_bps = 15          # Wider step
atr_step_k = 1800           # Very reactive
min_step_bps = 12
inv_skew_k = 60             # Strong skew
trend_on = 1.6              # Very sensitive
trend_off = 1.0
atr_pct_threshold = 0.025   # Higher vol threshold
```

---

## Troubleshooting Config

### "Order's notional must be no smaller than 100"

**Nguyên nhân:** `base_order_usdt` quá thấp

**Fix:**
```toml
base_order_usdt = 160   # Tăng lên >= 110
```

### "SKIP starter entry: blocked by max_position cap"

**Nguyên nhân:** Đã có position >= max_position

**Fix:**
1. Đóng position trên exchange, HOẶC
2. Tăng `max_position`

### "maker_ratio < 80%"

**Nguyên nhân:** Spread quá hẹp, orders cross

**Fix:**
```toml
base_step_bps = 12      # Tăng từ 10
use_post_only = true    # Bật GTX
```

### "TREND ON immediately after startup"

**Nguyên nhân:** Trend detector quá nhạy

**Fix (PRO):**
```toml
warmup_after_rebuild_sec = 30   # Tăng warmup
trend_debounce_hits = 4          # Tăng debounce
trend_on = 2.5                   # Tăng ngưỡng
```

### "Bot không đặt orders"

**Kiểm tra theo thứ tự:**
1. `[MIN_ORDER]` log - `base_order_usdt` đủ chưa?
2. `[REGIME]` log - đang TREND ON không?
3. `[WAIT]` log - đang chờ calm không?
4. `max_position` - đã đạt cap chưa?

### "Config changes not applied"

**Nguyên nhân:** Bot cần restart để load config mới

**Fix:** Stop và Start lại bot qua Tray App

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────┐
│              CONFIG QUICK REFERENCE                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ESSENTIAL CHECKS:                                   │
│  ✓ use_testnet matches API keys                     │
│  ✓ base_order_usdt >= 110                           │
│  ✓ max_position >= minQty                           │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  PRO DYNAMIC FORMULAS:                               │
│                                                      │
│  step = max(min, base + k × atr%)                   │
│  size = max(min, base / (1 + k × atr%))             │
│  skew = clamp((inv - target) × k, ±max)             │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  SAFE DEFAULTS (PRO):                                │
│  - base_step_bps = 10                               │
│  - atr_step_k = 1400                                │
│  - vol_size_k = 30                                  │
│  - inv_skew_k = 40                                  │
│  - max_skew_bps = 8                                 │
│  - trend_on = 2.0, trend_off = 1.3                  │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  KEY LOGS TO WATCH:                                  │
│  [MIN_ORDER] - Check minQty/minNotional             │
│  [SKEW] - Inventory and step adjustments            │
│  [MM] - maker_ratio, toxicity, cancels              │
│  [REGIME] - TREND ON/OFF status                     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

**Last Updated:** 2026-01-28 (PRO v2.9)

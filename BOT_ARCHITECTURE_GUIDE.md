## Tổng quan bot grid (hedge + oneway)

Bot này là grid bot chạy trên Binance Futures (UM), hỗ trợ:

- **Oneway**: `btcgridbot2modes.py` (tài khoản One-Way, chỉ 1 vị thế / side).
- **Hedge**: `btcgridbot_hedgemode.py` (tài khoản Hedge Mode, có LONG & SHORT độc lập).

Tray app (`bnbot-tray`) chỉ là vỏ GUI để:

- Quản lý **instances** (mỗi instance = 1 cặp, 1 config TOML, 1 log).
- Start/Stop process bot thực sự (`bnbot-hedge.exe` / `bnbot-oneway.exe`).
- Hiển thị logs, Activities, metrics (PnL, balances, open orders...).

Tất cả logic giao dịch chính nằm trong file bot (`btcgridbot_hedgemode.py` / `btcgridbot2modes.py`) + config TOML.

---

## Chuỗi xử lý chính của bot (hedge)

### 1. Khởi động

1. Đọc **env**:
   - `BINANCE_API_KEY`, `BINANCE_API_SECRET` từ `.env.local` cạnh file `.exe` hoặc source.
   - `USE_TESTNET` (từ env hoặc TOML, tuỳ mode / bot).
2. Tạo client:
   - `UMFutures(key, secret, base_url)`.
3. Đọc cấu hình từ TOML (instance):
   - `[app]` và `[profiles.SAFE]` / `[profiles.HIGH]`.
4. Tải filter symbol:
   - `minQty`, `stepSize`, `tickSize`, `baseAsset`.
5. Gọi `GridBot.run()`.

### 2. `GridBot.run()` (loop chính)

Ở mức cao, flow là:

1. Log version, mode, symbol, leverage.
2. **Set leverage + margin**:
   - `change_margin_type` → ISOLATED / CROSS theo config.
   - `change_leverage` theo config (có xử lý -4046, -4161).
3. Lấy equity ban đầu (= `start_equity`).
4. Tính **regime** ban đầu:
   - `px`, `atr`, `ema_fast`, `ema_slow`, `trend_score`, `atr_pct`, `market_bias`.
5. `anchor = px`, cancel tất cả lệnh cũ (`reason=init`).
6. Nếu `starter_entry = true` và `atr_pct <= atr_pct_threshold`:
   - Chọn book LONG/SHORT (theo `grid_book` / `AUTO` + EMA bias).
   - Vào **starter MARKET** theo `base_order_usdt` / `starter_entry_usdt`.
   - Đặt TP limit đầu tiên đối diện (1 bước `step_bps`).
7. Gọi `place_fresh_grid(...)` để dựng full grid quanh giá.

Sau đó vào vòng:

```text
while True:
  - nếu có STOP_FILE -> cancel_all + exit
  - compute_regime (px, atr, ema_f, ema_s, trend_score, atr_pct, bias)
  - equity_kill_switch (dựa trên equity_dd_stop)
  - price_kill_switch (dựa trên hard_kill_dev & anchor)
  - ATR regime (atr_pct_threshold) -> pause/resume grid
  - TREND regime (trend_on / trend_off) -> trend_protect / resume grid
  - Nếu đang range (không trend, không ATR pause):
      - maybe_reanchor (reanchor_dev)
      - nếu dev > reanchor_dev -> cancel_all("reanchor") + place_fresh_grid
      - handle_fills_and_regrid (regrid local quanh các fill)
  - Mỗi ~30s:
      - dump_open_orders (ORDERS[LIVE] + LOCAL)
      - log PNL (Realized / Unrealized / Equity)
  - sleep(POLL_SECONDS)
```

---

## Các block logic quan trọng

### A. Regime & trend

```python
atr_pct = atr / price                # ATR% 1m
trend_score = abs(ema_f - ema_s) / atr
```

- `atr_pct` dùng cho **ATR regime filter**:
  - Nếu `atr_pct > atr_pct_threshold` → coi là volatility quá cao → **pause grid**.
  - Chỉ khi `atr_pct` rơi lại dưới threshold thì bot mới **resume grid** (rebuild).
- `trend_score` dùng cho **TREND regime** (dựa vào độ chênh EMA so với ATR):
  - Nếu đang **range** (`self.trending == False`) và `trend_score > trend_on` → TREND ON:
    - Log `[REGIME] TREND ON ...`.
    - Log `[PAUSE] Grid cancelled due to TREND ON ...`.
    - Gọi `trend_protect()`:
      - `cancel_all("trend_protect")`.
      - Đóng bớt vị thế của book hiện tại: `qty = current_pos * reduce_pct_on_trend`.
  - Nếu đang **TREND ON** (`self.trending == True`) và `trend_score < trend_off` → TREND OFF:
    - Log `[REGIME] TREND OFF ...`.
    - Reset anchor, `cancel_all("trend_off")`.
    - Nếu không bị ATR pause → `[RESUME] Rebuilding grid after TREND OFF` + `place_fresh_grid(...)`.

Hysteresis (`trend_on > trend_off`) tránh flip ON/OFF liên tục khi score dao động quanh ngưỡng.

### B. Grid building & vị thế

- `build_grid_prices(mid, atr)`:
  - Dùng `step_bps` (basis points) + `band_atr_mult * atr` để tính **dải giá** grid.
  - Tạo list levels BUY dưới `mid` và SELL trên `mid`, tối đa `levels_each_side` mỗi phía.
- `order_qty_for_level(px, level_i)`:
  - Notional cơ bản: `base_notional = base_order_usdt`.
  - Có optional `size_ramp`: `notional = base_notional * (1 + size_ramp * (level_i - 1))`.
  - Chuyển sang qty: `qty = notional / px`.
- Điều kiện Binance:
  - `qty >= minQty` (theo filters).
  - `qty * price >= 100 USDT` (minNotional) – có logic **tự tăng qty** nếu được.
- `max_position`:
  - `enforce_inventory_cap` đảm bảo tổng vị thế book (LONG hoặc SHORT) không vượt `max_position`.

---

## Tham số cấu hình trong TOML

### 1. Block `[app]`

```toml
[app]
bot_kind = "hedge"        # "hedge" hoặc "oneway" (trong instances)
symbol = "BTCUSDT"        # cặp futures
use_testnet = true        # true = testnet, false = production
mode = "SAFE"             # "SAFE" hoặc "HIGH" (chọn profile)
poll_seconds = 2.0        # chu kỳ loop chính (giây)
dry_run = false           # true = không gửi lệnh thật
initial_capital_usdt = 5000.0  # dùng để log & scale risk (tuỳ người dùng)
starter_entry = true      # có vào lệnh MARKET ban đầu không
starter_entry_usdt = 0    # 0 = dùng base_order_usdt làm size starter
grid_book = "AUTO"        # "AUTO", "LONG", "SHORT" (hedge mode)
```

**Gợi ý:**

- `use_testnet = true` khi test, `false` khi chạy tiền thật.
- `starter_entry = true` giúp có vị thế ban đầu rồi mới dựng grid xung quanh.
- `grid_book = AUTO` an toàn cho BTC, có thể fix LONG/SHORT nếu bạn chủ ý chơi 1 phía.

### 2. Block `[profiles.SAFE]` và `[profiles.HIGH]`

Mỗi profile là một **bộ thông số risk riêng**. Tray dùng `mode` từ `[app]` để chọn SAFE/HIGH.

#### 2.1. Tham số chung

```toml
leverage          # đòn bẩy: 2 (SAFE), 10 (HIGH)
margin_type       # "ISOLATED" hoặc "CROSSED"
kline_interval    # "1m"
atr_len           # số nến dùng để tính ATR
ema_fast          # EMA nhanh (dùng trong trend_score)
ema_slow          # EMA chậm
levels_each_side  # số levels BUY/SELL mỗi phía
step_bps          # khoảng cách grid, tính bằng basis points (bps). 10 = 0.10%
band_atr_mult     # bội số ATR cho "band" tối đa quanh mid; tránh grid quá xa giá
base_order_usdt   # size lệnh cơ bản (USDT)
size_ramp         # mỗi level xa hơn có notional = base * (1 + size_ramp * (level_i - 1))
max_position      # cap tổng size book (BTC, ETH, SOL...)
max_open_orders   # số lệnh tối đa track cùng lúc
```

**Gợi ý scale theo vốn (initial_capital_usdt):**

- SAFE: `base_order_usdt ≈ 1–1.5% vốn`.
- HIGH: `base_order_usdt ≈ 2–3% vốn`.
- `max_position` nên là **20–40% vốn** tính theo notional, tuỳ khẩu vị.

#### 2.2. Tham số bảo vệ (risk & regime)

```toml
trend_on              # ngưỡng trend_score để bật TREND ON
trend_off             # trend_score dưới mức này thì TREND OFF (resume)
reduce_pct_on_trend   # % vị thế bị đóng trong trend_protect (0.2 = 20%)
equity_dd_stop        # equity drawdown tối đa (0.12 = 12%) trước khi kill
reanchor_dev          # % lệch anchor để re-anchor + rebuild grid
hard_kill_dev         # % lệch anchor để kill hẳn bot (SystemExit)
atr_pct_threshold     # ATR% (atr / price) ngưỡng để pause do volatility
```

**Diễn giải:**

- `trend_on` / `trend_off`:
  - Dùng `trend_score = |EMA_fast - EMA_slow| / ATR`.
  - Ví dụ SAFE:
    - `trend_on = 2.0` → chỉ khi EMA lệch nhau > 2 ATR mới coi là trend mạnh.
    - `trend_off = 1.3` → đợi đến khi lệch < 1.3 ATR mới resume.
  - HIGH có thể dùng `2.6 / 1.8` → ít bảo vệ hơn, chịu trend sâu hơn trước khi bỏ grid.
- `reduce_pct_on_trend`:
  - Khi TREND ON, bot **cancel toàn bộ grid** và đóng bớt `reduce_pct_on_trend` * position (market).
  - Giá trị 0.2–0.35 là hợp lý (20–35%).
- `equity_dd_stop`:
  - Dùng equity account: nếu `(start_equity - current_equity) / start_equity > equity_dd_stop` → kill.
  - SAFE: 0.12 (12%), HIGH: 0.18–0.2.
- `reanchor_dev`:
  - Nếu giá lệch anchor > reanchor_dev (vd 2%): cancel grid, set anchor mới và build lại quanh px mới.
  - Trị số nhỏ → bot rebuild grid thường xuyên hơn; trị số lớn → “lì” hơn, ít cancel.
- `hard_kill_dev`:
  - Nếu dev > hard_kill_dev (vd 8–10%), bot log `[KILL]` + cancel_all + exit.
- `atr_pct_threshold`:
  - Nếu 1m ATR% > threshold (vd 2–3%) → coi là volatility quá cao → pause grid cho đến khi hạ nhiệt.

---

## Cách tweak an toàn cho người mới

Giả sử vốn `initial_capital_usdt = 5000`:

### SAFE (khuyến nghị)

- `leverage = 2`
- `base_order_usdt = 60` (~1.2% vốn).
- `max_position`:
  - BTC: 0.015 BTC (~1.3–1.5k notional).
  - ETH/SOL scale theo giá (0.5 ETH, 5 SOL, v.v.).
- `trend_on = 2.0`, `trend_off = 1.3`.
- `equity_dd_stop = 0.12`.
- `atr_pct_threshold = 0.020`.
- `reanchor_dev = 0.02`, `hard_kill_dev = 0.08`.

### HIGH (rủi ro cao nhưng vẫn có phanh)

- `leverage = 10`
- `base_order_usdt = 120` (~2.4% vốn).
- `max_position`:
  - BTC: 0.03 BTC (~2.7k notional).
- `trend_on = 2.6`, `trend_off = 1.8`.
- `equity_dd_stop = 0.20`.
- `atr_pct_threshold = 0.030`.
- `reanchor_dev = 0.015`, `hard_kill_dev = 0.055`.

Nếu muốn **ít bảo vệ hơn** (ít cancel hơn, chấp nhận DD cao hơn):

- Tăng `trend_on`, `trend_off` (vd 3.0 / 2.0).
- Tăng `equity_dd_stop` (SAFE 0.15, HIGH 0.25).
- Tăng `atr_pct_threshold` (SAFE 0.03, HIGH 0.04).

Nếu muốn **an toàn hơn**:

- Giảm `base_order_usdt`, `max_position`.
- Giảm `trend_on`, tăng `reduce_pct_on_trend`.
- Giảm `equity_dd_stop`.

---

## Checklist nhanh khi share cho người khác

1. **Không share `.env.local`** – chỉ gửi `.env.example` / `env.example` + file hướng dẫn này.
2. Bảo họ:
   - Copy `.env.example` → `.env.local` và điền API key (testnet trước).
   - Chỉnh `[app]` (`symbol`, `mode`, `grid_book`) phù hợp.
   - Chỉnh `[profiles.SAFE]`/`[profiles.HIGH]` theo vốn (base_order_usdt, max_position).
3. Nhắc rõ:
   - Luôn test trên **testnet** ít nhất 1–2 ngày.
   - Không tăng `leverage` / `max_position` / `base_order_usdt` bừa bãi.
   - Hiểu rõ `equity_dd_stop`, `trend_on/off`, `atr_pct_threshold` trước khi giảm các giá trị bảo vệ.


# Hướng Dẫn Sử Dụng Inventory Flush (PRO v2.9)

## Mục Lục

1. [Flush Là Gì?](#flush-là-gì)
2. [Cách Flush Hoạt Động](#cách-flush-hoạt-động)
3. [Cài Đặt Khởi Đầu](#cài-đặt-khởi-đầu)
4. [Logs Cần Theo Dõi](#logs-cần-theo-dõi)
5. [Tweak & Tối Ưu](#tweak--tối-ưu)
6. [Troubleshooting](#troubleshooting)

---

## Flush Là Gì?

### Định Nghĩa

**Inventory Flush = "Xả hàng" để bảo vệ lợi nhuận và giảm rủi ro**

| Flush LÀ | Flush KHÔNG PHẢI |
|----------|------------------|
| Risk-off (giảm rủi ro) | Take-profit (chốt lời tự động) |
| Đóng vị thế khi có tín hiệu nguy hiểm | Đóng vị thế khi đạt target |
| Bảo vệ equity đã đạt được | Maximize profit |
| Trigger bởi điều kiện thị trường | Trigger bởi PnL cố định |

### Khi Nào Flush Trigger?

Flush chỉ trigger khi **TẤT CẢ** điều kiện sau đúng:

1. `enabled = true` (bật tính năng)
2. `total_fills >= 50` (đủ dữ liệu)
3. `inv_pressure >= 0.15` (có inventory đáng kể)
4. Một trong các trigger active (xem bên dưới)
5. Không trong warmup/resume cooldown
6. Không trong flush cooldown

### 4 Loại Trigger (Theo Thứ Tự Ưu Tiên)

| # | Trigger | Mục Đích | Mặc Định |
|---|---------|----------|----------|
| 1 | **VOLATILITY** | Circuit breaker cho tail risk | TẮT |
| 2 | **TREND_PROLONGED** | Trend kéo dài quá lâu | TẮT |
| 3 | **WAIT_TIMEOUT** | Inventory bị kẹt trong WAIT mode | BẬT |
| 4 | **PROFIT_CUSHION** | Bảo vệ lợi nhuận đã đạt | BẬT |

---

## Cách Flush Hoạt Động

### State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                         FLUSH STATE MACHINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [OFF] ──trigger──> [ARMED] ──confirm──> [FLUSHING_MAKER]      │
│     ↑                   │                        │               │
│     │                   │ (disarm)               │ (45s timeout) │
│     │                   ↓                        ↓               │
│     │                 [OFF]              [FLUSHING_MARKET]       │
│     │                                            │               │
│     │                                            │ (complete)    │
│     │                                            ↓               │
│     └───────(30 min cooldown)──────────── [COOLDOWN]            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Chi Tiết Từng State

| State | Mô Tả | Thời Gian |
|-------|-------|-----------|
| **OFF** | Không có trigger active | - |
| **ARMED** | Trigger đã active, chờ confirm | 2 cycles (~4s) |
| **FLUSHING_MAKER** | Đang đặt lệnh LIMIT GTX để đóng | Tối đa 45s |
| **FLUSHING_MARKET** | Maker không fill, dùng MARKET | Ngay lập tức |
| **COOLDOWN** | Nghỉ sau khi flush xong | 30 phút |

### Execution Flow

```
1. Detect trigger (VD: equity +0.5% với trend ON)
      ↓
2. ARMED - chờ 2 lần confirm liên tiếp
      ↓
3. Cancel grid orders
      ↓
4. Lấy mark price làm reference
      ↓
5. Đặt LIMIT GTX với offset (mark_price + 4bps cho SELL)
      ↓
6. Chờ fill, re-quote mỗi 5s nếu cần
      ↓
7. Sau 45s không fill → chuyển MARKET
      ↓
8. Log metrics: pnl, fee, slippage
      ↓
9. Vào COOLDOWN 30 phút
```

---

## Cài Đặt Khởi Đầu

### Giai Đoạn 1: Quan Sát (Tuần 1-2)

**Mục tiêu:** Thu thập data, hiểu bot hoạt động

```toml
[flush]
enabled = false   # TẮT - chỉ quan sát
```

**Trong giai đoạn này, theo dõi:**
- maker_ratio (nên > 80%)
- toxicity (nên < 8 bps)
- cancel_count (không quá cao)
- Số fills/ngày

### Giai Đoạn 2: Test Conservative (Tuần 3-4)

**Mục tiêu:** Test flush với settings an toàn

```toml
[flush]
enabled = true
mode = "AUTO"

# Safety
cooldown_after_flush_sec = 1800
cooldown_after_failed_flush_sec = 600
min_fills_before_enable = 50
flush_confirm_hits = 2

# Execution
maker_first = true
maker_first_timeout_sec = 45
maker_requote_sec = 5
max_flush_time_sec = 90
close_offset_bps = 4.0

# Triggers - CHỈ BẬT 2 CÁI NÀY
profit_cushion_on = true
profit_cushion_pct = 0.50
profit_cushion_hysteresis_pct = 0.25
profit_requires_risk_signal = true
profit_cushion_once_per_day = true

wait_timeout_on = true
wait_timeout_sec = 1800
wait_stuck_sec = 600
wait_stuck_delta = 0.05
wait_requires_loss = false
wait_requires_risk_signal = true

# Triggers - TẮT
trend_prolonged_on = false
vol_flush_on = false

# Gating
min_inventory_pressure = 0.15
```

### Giai Đoạn 3: Production (Sau 100+ fills)

```toml
[flush]
enabled = true
mode = "AUTO"

# Có thể giảm ngưỡng nếu cần
profit_cushion_pct = 0.50
min_fills_before_enable = 50

# Cân nhắc bật thêm nếu cần
trend_prolonged_on = false   # Bật nếu thường xuyên stuck trong trend
vol_flush_on = false         # Bật nếu cần circuit breaker cho news
```

---

## Logs Cần Theo Dõi

### 1. Log Khởi Động

```
[FLUSH] enabled=true mode=AUTO cooldown=1800s
[FLUSH] profit_cushion=0.50% wait_timeout=1800s
```

✅ Kiểm tra config đã load đúng

### 2. Log Ngày Mới (UTC)

```
[FLUSH] New day: equity_start=5000.00
```

✅ Reset HWM tracking cho ngày mới

### 3. Log Trạng Thái (Mỗi 30s)

```
[FLUSH] state=OFF hwm%=0.32 inv_p=0.18 flushes=0
```

| Field | Ý Nghĩa | Cần Chú Ý Khi |
|-------|---------|---------------|
| `state` | Trạng thái hiện tại | Khác OFF |
| `hwm%` | % lợi nhuận so với đầu ngày | > profit_cushion_pct |
| `inv_p` | Inventory pressure | > 0.15 (có thể flush) |
| `flushes` | Số lần đã flush | > 3 (quá nhiều?) |

### 4. Log Khi Flush

**ARMED:**
```
[FLUSH] ARMED: PROFIT_CUSHION (0.52%) inv_pressure=0.25
```
→ Trigger active, chờ confirm

**FLUSHING:**
```
[FLUSH] FLUSHING_MAKER: PROFIT_CUSHION (0.52%) inv=0.012 BTC ref_price=87500.00
```
→ Đang thực hiện flush

**Maker Fill:**
```
[FLUSH] Maker order filled at 87504.20
```
→ Lệnh maker đã khớp (tốt!)

**Market Fallback:**
```
[FLUSH] Switching to MARKET (maker timeout)
[FLUSH] MARKET SELL LONG qty=0.012 id=123456
[FLUSH] Market slippage: 8.50bps
```
→ Phải dùng market (có slippage)

**Complete:**
```
[FLUSH] COOLDOWN: flush #1 complete | pnl=2.35 USDT | fee_est=0.17 USDT | avg_slippage=0.48bps
[FLUSH] Cumulative: total_pnl=2.35 USDT | total_fees=0.17 USDT
```
→ Xong! Xem metrics

### 5. Log Cảnh Báo

```
[FLUSH] Disarmed: trigger no longer active
```
→ OK - market đã calm trước khi flush

```
[FLUSH] FAILED: could not complete flush, cooldown=600s
```
→ Có vấn đề! Check lại

---

## Metrics Quan Trọng

### Bảng Đánh Giá

| Metric | Tốt | Chấp Nhận | Cần Xem Lại |
|--------|-----|-----------|-------------|
| `avg_slippage` | < 2 bps | 2-10 bps | > 10 bps |
| `fee_est / pnl` | < 10% | 10-30% | > 30% |
| `flushes/day` | 0-2 | 3-5 | > 5 |
| `maker vs market` | 100% maker | 80% maker | < 50% maker |

### Cách Tính

**Slippage:**
```
slippage_bps = |fill_price - expected_price| / expected_price × 10000
```

**Fee/PnL Ratio:**
```
ratio = fee_est / flush_pnl × 100%
```

---

## Tweak & Tối Ưu

### Scenario 1: Flush Không Bao Giờ Trigger

**Triệu chứng:** Chạy nhiều ngày, `flushes=0`

**Nguyên nhân có thể:**
- Chưa đủ fills
- Ngưỡng profit quá cao
- Luôn có risk_signal = false

**Giải pháp:**
```toml
# Giảm ngưỡng
profit_cushion_pct = 0.30              # Từ 0.50 → 0.30
min_inventory_pressure = 0.10          # Từ 0.15 → 0.10
min_fills_before_enable = 30           # Từ 50 → 30

# Hoặc bỏ yêu cầu risk signal
profit_requires_risk_signal = false
```

### Scenario 2: Flush Quá Nhiều

**Triệu chứng:** `flushes > 5` mỗi ngày

**Nguyên nhân có thể:**
- Ngưỡng quá thấp
- Cooldown quá ngắn
- Market volatile

**Giải pháp:**
```toml
# Tăng ngưỡng
profit_cushion_pct = 0.75              # Từ 0.50 → 0.75
cooldown_after_flush_sec = 3600        # Từ 1800 → 3600 (1 giờ)
min_fills_before_enable = 100          # Từ 50 → 100
flush_confirm_hits = 3                 # Từ 2 → 3
```

### Scenario 3: Slippage Cao

**Triệu chứng:** `avg_slippage > 10 bps`, nhiều MARKET fills

**Giải pháp:**
```toml
# Tăng thời gian maker
maker_first_timeout_sec = 60           # Từ 45 → 60
maker_requote_sec = 3                  # Từ 5 → 3 (re-quote nhanh hơn)
close_offset_bps = 6.0                 # Từ 4 → 6 (price hấp dẫn hơn)
```

### Scenario 4: Fee Ăn Hết Profit

**Triệu chứng:** `fee_est > 30% pnl`

**Nguyên nhân:** Flush inventory nhỏ

**Giải pháp:**
```toml
# Chỉ flush inventory lớn
min_inventory_pressure = 0.25          # Từ 0.15 → 0.25
profit_cushion_pct = 0.75              # Cần profit lớn hơn trước khi flush
```

### Scenario 5: Cần Circuit Breaker Cho News

**Triệu chứng:** Bot bị hit mạnh khi có news/spike

**Giải pháp:**
```toml
# Bật volatility flush
vol_flush_on = true
vol_atr_pct_on = 0.020                 # Flush khi ATR% > 2%
vol_atr_pct_off = 0.015                # Tắt khi ATR% < 1.5%
```

---

## Troubleshooting

### "Flush triggered nhưng không đóng được"

**Check:**
1. `min_qty` có đủ không?
2. Position có tồn tại trên exchange không?
3. API rate limit?

**Log tìm:**
```
[FLUSH] Market order failed: ...
[FLUSH] FAILED: could not complete flush
```

### "Flush đóng sai side"

**Không nên xảy ra!** Nếu có:
1. Check `active_book` trong log
2. Check position trên Binance
3. Report bug

### "HWM% không tăng dù có profit"

**Nguyên nhân:** HWM dùng `totalMarginBalance` (bao gồm unrealized)

**Giải thích:** HWM chỉ tăng khi equity thực sự đạt đỉnh mới, không phải chỉ realized PnL.

### "Flush trigger liên tục disarm"

**Nguyên nhân:** Trigger có hysteresis, market đang oscillate quanh ngưỡng

**Giải pháp:** Tăng `flush_confirm_hits` lên 3-4

---

## Checklist Hàng Ngày

### Sáng (Bắt đầu ngày)
- [ ] Kiểm tra `[FLUSH] New day:` đã log chưa
- [ ] Xem `equity_start` có đúng không

### Trong Ngày
- [ ] Monitor `state=` (nên là OFF phần lớn thời gian)
- [ ] Xem `hwm%` tăng dần nếu bot đang profitable
- [ ] Check `inv_p` (nếu > 0.15 có thể flush)

### Tối (Kết thúc ngày)
- [ ] Xem `flushes=` (0-2 là tốt)
- [ ] Nếu có flush: check `pnl`, `slippage`, `fee_est`
- [ ] Review bất kỳ `[FLUSH] FAILED` nào

### Hàng Tuần
- [ ] Tính tổng `flush_total_pnl` và `flush_total_fees`
- [ ] So sánh với tổng realized PnL
- [ ] Điều chỉnh config nếu cần

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────┐
│              FLUSH QUICK REFERENCE                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  STATE FLOW:                                         │
│  OFF → ARMED → FLUSHING_MAKER → COOLDOWN            │
│                      ↓                               │
│               FLUSHING_MARKET (if timeout)           │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  TRIGGER PRIORITY (cao → thấp):                      │
│  1. VOLATILITY     (circuit breaker)                │
│  2. TREND_PROLONGED (trend kéo dài)                 │
│  3. WAIT_TIMEOUT    (inventory kẹt)                 │
│  4. PROFIT_CUSHION  (bảo vệ profit)                 │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  KEY METRICS:                                        │
│  - avg_slippage < 5 bps (tốt)                       │
│  - fee/pnl < 20% (tốt)                              │
│  - flushes/day: 0-2 (tốt)                           │
│                                                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  DEFAULT TIMINGS:                                    │
│  - Confirm: 2 cycles (~4s)                          │
│  - Maker timeout: 45s                               │
│  - Re-quote: 5s                                     │
│  - Max flush: 90s                                   │
│  - Cooldown: 30 min                                 │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

**Last Updated:** 2026-01-28 (PRO v2.9)

# Hướng Dẫn Setup Bot Grid Trading

## 📋 Mục Lục
1. [Chọn Bot Phù Hợp](#1-chọn-bot-phù-hợp)
2. [Cấu Hình Cơ Bản (.env)](#2-cấu-hình-cơ-bản-env)
3. [Chọn MODE: SAFE vs HIGH](#3-chọn-mode-safe-vs-high)
4. [Điều Chỉnh Thông Số Quan Trọng](#4-điều-chỉnh-thông-số-quan-trọng)
5. [Chạy Bot](#5-chạy-bot)

---

## 1. Chọn Bot Phù Hợp

### Bot One-Way Mode (`btcgridbot2modes.py`)
- ✅ **Dùng khi**: Tài khoản Binance của bạn ở chế độ **One-Way Mode**
- ✅ **Đơn giản hơn**: Chỉ có 1 vị thế (long hoặc short)
- ✅ **Phù hợp**: Người mới bắt đầu

### Bot Hedge Mode (`btcgridbot_hedgemode.py`)
- ✅ **Dùng khi**: Tài khoản Binance của bạn ở chế độ **Hedge Mode**
- ✅ **Linh hoạt hơn**: Có thể mở LONG và SHORT đồng thời
- ✅ **Phù hợp**: Trader có kinh nghiệm, muốn kiểm soát rủi ro tốt hơn

**⚠️ QUAN TRỌNG**: Kiểm tra mode của tài khoản trên Binance trước khi chọn bot!

---

## 2. Cấu Hình Cơ Bản (.env)

### File `.env.local` hoặc `.env` (copy từ `.env.example`)

```env
# API Keys (BẮT BUỘC)
BINANCE_API_KEY=your_api_key_here
BINANCE_API_SECRET=your_api_secret_here

# Môi trường
USE_TESTNET=true          # true = testnet, false = production (tiền thật!)

# Chọn Bot và Mode
MODE=SAFE                 # SAFE hoặc HIGH

# Chỉ cho Hedge Mode bot
GRID_BOOK=AUTO            # AUTO | LONG | SHORT (chỉ dùng với btcgridbot_hedgemode.py)

# Cấu hình khác
SYMBOL=BTCUSDT            # Cặp giao dịch
POLL_SECONDS=2.0          # Tần suất kiểm tra (giây)
DRY_RUN=false             # true = chỉ test không đặt lệnh thật
```

---

## 3. Chọn MODE: SAFE vs HIGH

### 🔵 SAFE Mode (An Toàn)

**Đặc điểm:**
- Leverage: **2x**
- Step: **12 bps** (0.12%)
- Max position: **0.03 BTC**
- Base order: **25 USDT**
- Equity stop: **12% drawdown**

**Phù hợp cho:**
- ✅ Vốn nhỏ-trung bình: **500 - 2,000 USDT**
- ✅ Người mới bắt đầu
- ✅ Muốn ổn định, ít rủi ro
- ✅ Chấp nhận lợi nhuận thấp hơn nhưng an toàn

**Cách chọn:**
```env
MODE=SAFE
```

---

### 🔴 HIGH Mode (Rủi Ro Cao)

**Đặc điểm:**
- Leverage: **10x**
- Step: **9 bps** (0.09%)
- Max position: **0.06 BTC**
- Base order: **35 USDT**
- Equity stop: **18% drawdown**

**Phù hợp cho:**
- ✅ Vốn lớn hơn: **2,000+ USDT**
- ✅ Trader có kinh nghiệm
- ✅ Chấp nhận rủi ro cao để có lợi nhuận cao hơn
- ✅ Muốn tần suất trade nhiều hơn

**Cách chọn:**
```env
MODE=HIGH
```

---

## 4. Điều Chỉnh Thông Số Quan Trọng

### ⚙️ Các Thông Số Có Thể Điều Chỉnh Trong Code

Nếu bạn muốn tùy chỉnh sâu hơn, mở file bot và sửa trong phần `SAFE` hoặc `HIGH` config:

#### A. Điều Chỉnh Theo Vốn Của Bạn

**Nếu vốn nhỏ (< 500 USDT):**
```python
# Trong SAFE config
max_position_btc=0.01,      # Giảm từ 0.03 xuống 0.01
base_order_usdt=10,         # Giảm từ 25 xuống 10
```

**Nếu vốn lớn (> 5,000 USDT):**
```python
# Trong SAFE config
max_position_btc=0.05,      # Tăng từ 0.03 lên 0.05
base_order_usdt=50,         # Tăng từ 25 lên 50
```

#### B. Điều Chỉnh Step (Khoảng Cách Grid)

**Step nhỏ hơn = Nhiều lệnh hơn = Rủi ro cao hơn**

```python
# SAFE mode
step_bps=12,  # 0.12% - AN TOÀN
step_bps=10,  # 0.10% - Cân bằng
step_bps=8,   # 0.08% - Rủi ro cao (không khuyến nghị)

# HIGH mode
step_bps=9,   # 0.09% - Mặc định
step_bps=7,   # 0.07% - Rất rủi ro (không khuyến nghị)
```

**⚠️ Lưu ý**: Step quá nhỏ sẽ bị fee "ăn" lợi nhuận!

#### C. Điều Chỉnh Risk Management

```python
# Equity drawdown stop (dừng bot khi lỗ quá nhiều)
equity_dd_stop=0.12,  # 12% - SAFE
equity_dd_stop=0.15,  # 15% - Cân bằng
equity_dd_stop=0.20,  # 20% - Rủi ro cao

# Trend protection (giảm vị thế khi có trend)
reduce_pct_on_trend=0.35,  # Giảm 35% khi trend
reduce_pct_on_trend=0.50,  # Giảm 50% khi trend (an toàn hơn)
```

---

## 5. Chạy Bot

### Bước 1: Kiểm Tra Cấu Hình

Đảm bảo file `.env.local` hoặc `.env` đã được điền đầy đủ:
```bash
# Windows PowerShell
cat .env.local

# Linux/Mac
cat .env.local
```

### Bước 2: Chọn Bot Phù Hợp

**Nếu dùng One-Way Mode:**
```bash
python btcgridbot2modes.py
```

**Nếu dùng Hedge Mode:**
```bash
python btcgridbot_hedgemode.py
```

### Bước 3: Kiểm Tra Logs

Bot sẽ hiển thị:
```
[MODE] SAFE  symbol=BTCUSDT  testnet=True  leverage=2
[EQUITY] start=1000.00
[GRID] Active book: LONG (config: AUTO, bias: LONG)
[ORDER] LIMIT BUY LONG qty=0.001 price=95000.00 id=12345
```

---

## 📊 Bảng So Sánh Nhanh

| Thông Số | SAFE Mode | HIGH Mode |
|----------|-----------|-----------|
| **Leverage** | 2x | 10x |
| **Step** | 12 bps (0.12%) | 9 bps (0.09%) |
| **Max Position** | 0.03 BTC | 0.06 BTC |
| **Base Order** | 25 USDT | 35 USDT |
| **Vốn Khuyến Nghị** | 500-2,000 USDT | 2,000+ USDT |
| **Rủi Ro** | Thấp | Cao |
| **Lợi Nhuận** | Ổn định | Cao hơn |

---

## 🎯 Khuyến Nghị Cho Người Mới

1. **Bắt đầu với SAFE Mode** trên **TESTNET**
2. **Chạy ít nhất 1-2 ngày** để hiểu cách bot hoạt động
3. **Theo dõi logs** để xem PnL và fills
4. **Điều chỉnh `max_position_btc`** theo vốn thực tế của bạn
5. **Chỉ chuyển sang HIGH Mode** khi đã quen và có vốn lớn

---

## ⚠️ Lưu Ý Quan Trọng

1. **Luôn test trên TESTNET trước** (`USE_TESTNET=true`)
2. **Không đặt step quá nhỏ** (< 7 bps) - sẽ bị fee ăn
3. **Theo dõi equity drawdown** - bot sẽ tự dừng nếu lỗ quá nhiều
4. **Điều chỉnh `max_position_btc`** theo vốn thực tế
5. **Không chạy bot khi không theo dõi** - luôn monitor logs

---

## 🆘 Troubleshooting

### Bot không đặt lệnh?
- Kiểm tra API keys đúng chưa
- Kiểm tra `USE_TESTNET` đúng với tài khoản
- Kiểm tra vốn có đủ không

### Bot bị lỗi "positionSide required"?
- Bạn đang dùng `btcgridbot2modes.py` nhưng tài khoản ở Hedge Mode
- Chuyển sang dùng `btcgridbot_hedgemode.py`

### Bot dừng đột ngột?
- Kiểm tra `equity_dd_stop` - có thể đã chạm ngưỡng
- Kiểm tra `hard_kill_dev` - giá đã lệch quá xa anchor

---

## 📝 Checklist Trước Khi Chạy Production

- [ ] Đã test trên TESTNET ít nhất 1-2 ngày
- [ ] Hiểu rõ cách bot hoạt động
- [ ] Đã điều chỉnh `max_position_btc` theo vốn
- [ ] Đã chọn MODE phù hợp (SAFE/HIGH)
- [ ] Đã đặt `USE_TESTNET=false`
- [ ] Đã backup API keys
- [ ] Đã đọc và hiểu các rủi ro

**Chúc bạn trade thành công! 🚀**

# BNBOT Quick Start Guide

## Bắt Đầu Trong 5 Phút

### Bước 1: Chuẩn Bị API Keys

1. Đăng nhập Binance Futures Testnet: https://testnet.binancefuture.com
2. Tạo API Key
3. Lưu vào file `.env.local`:

```env
BINANCE_API_KEY=your_testnet_api_key
BINANCE_API_SECRET=your_testnet_secret
```

### Bước 2: Chọn Bot Mode

| Nếu bạn... | Chọn |
|------------|------|
| Tài khoản Hedge Mode | `bnbot-hedge-pro.exe` |
| Tài khoản One-Way Mode | `bnbot-oneway-pro.exe` |
| Không biết | Kiểm tra trên Binance Futures → Settings |

### Bước 3: Config Cơ Bản

Mở `instances/BTCUSDT/bot_config.toml`:

```toml
[app]
symbol = "BTCUSDT"
use_testnet = true        # ← QUAN TRỌNG: true cho testnet
mode = "SAFE"
starter_entry = true

[profiles.SAFE]
leverage = 2
base_order_usdt = 160     # ← >= 110 USDT
max_position = 0.02       # ← Theo vốn của bạn
```

### Bước 4: Chạy Bot

**Qua Tray App:**
1. Mở `bnbot-tray.exe` (hoặc `bnbot-tray-pro.exe`)
2. Chọn pair BTCUSDT
3. Click Start

**Qua Command Line:**
```cmd
cd instances/BTCUSDT
..\..\bnbot-hedge-pro.exe
```

### Bước 5: Xác Nhận Bot Chạy Đúng

Tìm các log này:

```
[VERSION] v0.2.9-pro-v2              ✓ Version
[MODE] SAFE  symbol=BTCUSDT          ✓ Mode & Symbol
[MIN_ORDER] minQty=0.001 BTC         ✓ Exchange info
[STARTER] MARKET BUY LONG qty=0.002  ✓ Entry placed
[ORDER][GTX] LIMIT SELL LONG ...     ✓ Grid orders
```

---

## Checklist Trước Khi Chạy

### Testnet Checklist

- [ ] API keys là testnet keys
- [ ] `use_testnet = true` trong config
- [ ] Có USDT trong testnet wallet

### Mainnet Checklist (SAU KHI TEST)

- [ ] API keys là real keys
- [ ] `use_testnet = false` trong config
- [ ] Đã test trên testnet >= 24h
- [ ] Hiểu các logs và metrics
- [ ] Vốn sẵn sàng

---

## Config Khuyến Nghị Theo Vốn

### Vốn $500-1000

```toml
[profiles.SAFE]
leverage = 2
base_order_usdt = 120
max_position = 0.01
levels_each_side = 5
```

### Vốn $1000-5000

```toml
[profiles.SAFE]
leverage = 2
base_order_usdt = 160
max_position = 0.02
levels_each_side = 10
```

### Vốn $5000-20000

```toml
[profiles.SAFE]
leverage = 3
base_order_usdt = 200
max_position = 0.05
levels_each_side = 15
```

---

## Logs Quan Trọng Cần Hiểu

### Logs Bình Thường

```
[EQUITY] start=5000.00                      # Vốn ban đầu
[ORDER][GTX] LIMIT BUY LONG qty=0.002       # Đặt lệnh mua
[ORDER][GTX] LIMIT SELL LONG qty=0.002      # Đặt lệnh bán
[PNL] Realized: +2.50 USDT | Total: +3.20   # PnL tracking
```

### Logs Cần Chú Ý

```
[REGIME] TREND ON                           # Grid paused - OK
[WAIT] Waiting for trend to subside         # Đang chờ market calm - OK
[WARN] base_order_usdt < min_notional       # Config sai! Fix ngay
```

### Logs Nguy Hiểm

```
[KILL] equity drawdown exceeded             # Bot stop vì lỗ
[KILL] price deviation exceeded             # Bot stop vì giá move mạnh
[FATAL] ...                                 # Lỗi nghiêm trọng
```

---

## FAQ Nhanh

### "Bot không đặt orders"
→ Kiểm tra: `base_order_usdt >= 110`? `max_position >= minQty`?

### "TREND ON liên tục"
→ Market đang trend. Đợi hoặc tăng `trend_on` threshold.

### "maker_ratio thấp"
→ Tăng `base_step_bps` hoặc bật `use_post_only = true`

### "Muốn chuyển sang real"
→ Đổi API keys + `use_testnet = false` + Restart

---

## Tài Liệu Chi Tiết

| File | Nội Dung |
|------|----------|
| `CONFIG_USAGE_GUIDE.md` | Giải thích tất cả config |
| `FLUSH_USAGE_GUIDE.md` | Hướng dẫn Inventory Flush |
| `BNBOT_COMPLETE_GUIDE.md` | Tài liệu đầy đủ |
| `BOT_PRO_GUIDE.md` | PRO features chi tiết |

---

## Support Flow

```
1. Đọc logs → tìm [WARN] hoặc [ERROR]
2. Check FAQ trong guide
3. Xem Troubleshooting trong CONFIG_USAGE_GUIDE.md
4. So sánh config với examples
```

---

**Chúc bạn trade thành công!**

Last Updated: 2026-01-28 (PRO v2.9)

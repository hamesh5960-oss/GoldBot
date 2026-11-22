# مشخصات اجرایی GoldBot چندپلتفرمی
## نسخهٔ ۱.۱ — MT5 (XAUUSD) + Toobit USDT-M Futures (XAU-SWAP-USDT)

| مشخصه | مقدار |
|---|---|
| نسخهٔ سند | **v1.1** (جایگزین کامل v1.0) |
| نسخهٔ مبنا | `GoldBot_Final_Release_v8.2.5.zip` — ۲۹۳ فایل، `git_commit 894560d`، build `2026-11-21T01:18:50Z` |
| هدف | حفظ کامل MT5 موجود + افزودن Runtime مستقل Toobit Futures روی XAU-SWAP-USDT |
| نوع قرارداد | USDT-M Perpetual، شناسهٔ API: `XAU-SWAP-USDT` |
| وضعیت | **قابل استفاده برای Kickoff** · **غیرمجاز برای معاملهٔ واقعی** |
| تاریخ | 2026-11-21 |
| دامنهٔ اعتبار | بخش ۷ با مستندات رسمی Toobit در تاریخ ۲۰۲۶-۱۱-۲۱ تطبیق داده شده؛ پیش از هر انتشار دوباره اعتبارسنجی شود |

### نحوهٔ خواندن این سند
هر گزاره یکی از سه برچسب را دارد:
- `[SRC]` — با اجرا یا بازرسی مستقیم سورس تحویلی اثبات شده است.
- `[DOC]` — با مستندات رسمی Toobit (api-docs.toobit.com / اعلان‌های رسمی) تطبیق داده شده است.
- `[OPEN]` — تأیید نشده است؛ **پیاده‌سازی نباید بر آن بنا شود** تا زمانی که شاهد بسته شود.

> این سند مشخصات توسعه است، نه گواهی سودآوری و نه مجوز معاملهٔ با پول واقعی. نسخهٔ مبنا خودش وضعیت `READY FOR DEMO VALIDATION` دارد و بک‌تست واقعی گزارش‌شده منفی است.

---

## ۱. تصمیم معماری (قفل‌شده)

**یک کدبیس، یک Shared Strategy Core، دو Runtime و دو Deployment کاملاً مستقل.**

MT5 و Toobit در هیچ‌کدام از این‌ها شریک نیستند: نشست اجرایی، دیتابیس state ریسک، ژورنال، چرخهٔ سفارش، حساب، feed، چرخهٔ استقرار، Kill Switch.
تنها چیز مشترک: کد استراتژی و سیاست تصمیم/ریسک در سطح «درصد» — بدون هیچ دانشی از قرارداد، مارجین یا venue.

| تصمیم | الزام قطعی | تأیید |
|---|---|---|
| Runtime | دو فرآیند مستقل؛ MT5 فعلی دست‌نخورده | `[SRC]` معماری فعلی تزریق‌پذیر است |
| Strategy | همان پکیج‌های `zones` و `numeric`؛ **بدون fork** | `[SRC]` `src/strategies/*` |
| Market data | کندل/تیک **Toobit** مرجع تحلیل و اجرا در Runtime Toobit | §۸ |
| Risk Policy | مشترک در سطح درصد؛ sizing/margin/liquidation مخصوص Futures | §۹ |
| State | SQLite ژورنال و risk-state مستقل per runtime/account | §۱۲ |
| Fail-safe | در هر ابهام وضعیت سفارش/پوزیشن/حفاظت: ورود ممنوع، اول reconciliation | §۱۰ |
| Release gate | هیچ مسیر پول واقعی تا عبور کامل §۱۶ | §۱۶ |
| ساختار کد | **افزودنی (additive)**؛ جابه‌جایی درخت ممنوع | §۵ |

**قاعدهٔ تغییرناپذیری:** در فازهای ۰ تا ۵، این شش فایل مجاز به تغییر نیستند:
`src/goldbot/domain/model.py` · `src/goldbot/domain/execution.py` · `src/goldbot/ports/broker.py` · `src/goldbot/adapters/mt5_broker.py` · `src/goldbot/app/engine.py` · `src/goldbot/runtime/live.py`
تضمین: `tests/test_mt5_contract_frozen.py` (هش SHA-256 هر شش فایل).

---

## ۲. وضعیت مبنا (شواهد، نه ادعا)

| کنترل | نتیجه | برچسب |
|---|---|---|
| اجرای کامل test suite | `1085 collected` · **`1084 passed, 1 xfailed`** · ۱۳۵.۹ ثانیه · Python 3.14.7 / Windows | `[SRC]` |
| تنها استثنا | `xfail` مستند: تکرار delivery برای close پوزیشنِ از‌پیش‌بسته → `uncertain` به‌جای ack کش‌شده. **هیچ سفارش دومی ارسال نمی‌شود** | `[SRC]` |
| ماژول‌های تست | ۷۱ فایل `tests/test_*.py` (نه ۷۷/۷۹/۸۰ که اسناد بسته ادعا می‌کنند) | `[SRC]` |
| کد Toobit/Futures در بسته | **هیچ.** جست‌وجوی `toobit` در کل ۲۹۳ فایل: صفر نتیجه. تنها آداپتور venue: `mt5_broker.py` + `fake_broker.py` | `[SRC]` |
| `reports/audit` و `reports/review` | **در ZIP نیستند** (README به آن‌ها ارجاع می‌دهد) | `[SRC]` |
| `SHA256SUMS` | با نرمال‌سازی CRLF معتبر (۲۹۲ ورودی)؛ روی POSIX بدون نرمال‌سازی هر ۲۹۲ ورودی FAIL | `[SRC]` |
| متغیرهای محیطی در بسته | ۱۴ فایل `var/` شامل ۴ اسکریپت `probe_*.py` و ۶ فایل `*.heartbeat.json` | `[SRC]` |

**اقدام لازم قبل از Kickoff (Gate فاز ۰):** اصلاح اعداد manifest از منبع واقعی، تولید `SHA256SUMS` با LF، حذف `var/*` از بسته. این‌ها پیش‌شرط عبور Gate فاز ۰ هستند و اکنون قرمزند.

---

## ۳. وضعیت عملیاتی و ریسک استراتژی مبنا

- بک‌تست ۱۲۰ روزه روی دادهٔ واقعی M15 بروکر: ۱۴۷ معامله، خالص **−۱۵۸.۳۴ USD**، PF **۰.۹۵**، MaxDD **۸.۶۶٪** (IS −۶۷۱.۴۴ / OOS +۱۸۰.۴۵). **منفی.** `[SRC]`
- tape از OHLC با مسیر intrabar قطعی ساخته شده؛ ترتیب واقعی حرکت درون کندل معلوم نیست و slippage/requote تاریخی صفر فرض شده است. حتی همین بک‌تست هم مدل‌محور است. `[SRC]`
- نشست دمو در زمان **بسته بودن بازار** ثبت شده: startup، recovery، reconcile، journal و heartbeat بررسی شده‌اند؛ **ارسال/پر شدن سفارش، بستن، trailing و kill-flatten روی venue آزمایش نشده‌اند.** `[SRC]`

**نتیجه:** نسخهٔ ۸.۲.۵ فقط برای ادامهٔ Demo Validation مناسب است. اتصال Toobit **نباید** به‌عنوان ارتقای سودآوری یا آماده‌سازی سرمایهٔ واقعی معرفی شود. اعتبار استراتژی و اعتبار اجرا دو مسئلهٔ جدا هستند؛ این سند فقط دومی را می‌سازد.

---

## ۴. معماری فعلی و نقاط اتصال

| ماژول | نقش فعلی | حکم برای Toobit |
|---|---|---|
| `domain/model.py` · `domain/execution.py` | `Tick`, `Bar`, `SymbolInfo`, `AccountSnapshot`, `PositionSnapshot`, `ApprovedOrder` | **reuse (بدون تغییر)**؛ مدل‌های Futures جدا افزوده شوند |
| `ports/broker.py` | `IBrokerGateway`: connect/account/positions/orders/deals/submit/modify/close/find_by_client_ref | **reuse برای MT5**؛ پورت Futures جدا |
| `ports/marketdata.py` | `IMarketDataFeed` + `MarketSnapshot` ساده | **adapt**؛ `FuturesMarketSnapshot` جدید |
| `app/engine.py` | `TradingEngine` یک symbol/feed/broker | **reuse الگو**؛ `app/futures/engine.py` جدا |
| `app/risk/capital.py` (۲۷۸ خط) | گام‌های سایزینگ با `lot`/`contract_size`/Gate D درصدی | **replace** برای Futures — مستقیم قابل فراخوانی نیست |
| `app/risk/profiles.py` · `app/risk/checks.py` | پروفایل‌ها و چک‌های ریسک درصدی | **reuse** |
| `app/order_coordinator.py` (۳۸۵ خط) | write-ahead، idempotency، حالت UNKNOWN | **reuse الگو**؛ توسعه برای conditional و partial |
| `app/retry_guard.py` | طبقه‌بندی retry با re-seal؛ حلقهٔ واقعی retry در سورس **وجود ندارد** | **adapt**؛ الزام §۱۰ |
| `app/reconciler.py` (۱۵۸ خط) | تطبیق پوزیشن/سفارش + `adopt_orphans` | **adapt** به سه‌طرفه (position/regular/algo) |
| `adapters/mt5_broker.py` (۱۴۸۸ خط) | تنها venue فعلی | **بدون تغییر** |
| `runtime/bootstrap.py` · `cli.py` | composition root تزریق‌پذیر | **reuse الگو**؛ `runtime/toobit/` جدا |
| `web/app.py` (۵۶۴ خط) | پنل تک‌نمونه با Bearer/Basic | **additive route**؛ UI هرگز API صرافی را مستقیم صدا نزند |

**محدودیت‌های واقعی مدل فعلی که مستقیماً به Futures منتقل نمی‌شوند:**

- `SymbolInfo` شامل `stops_level_points`, `freeze_level_points`, `filling_mode`, `swap_long/short` — همه مفاهیم MT5.
- `Tick` فقط `bid`/`ask` دارد؛ `mark`, `index`, `funding`, `depth` ندارد.
- `ApprovedOrder` فقط market/limit با SL/TP ساده دارد؛ `positionSide` و conditional lifecycle ندارد.
- `AccountSnapshot` فیلدهای `available_balance` (USDT)، `position_margin`، `maintenance_margin`، `liquidation_price` و `funding` ندارد.
- state DB **تک‌کلید** است (خودِ `docs/SPEC-MERGED-v1.1.md` §۱۶.۲ این را «باز» اعلام کرده).

استفادهٔ مستقیم از این مدل‌ها برای Futures = خطای واحد و ریسک مالی. این سند آن را ممنوع می‌کند.

---

## ۵. معماری هدف (افزودنی، بدون جابه‌جایی)

```
src/goldbot/
├── domain/
│   ├── model.py                  [MT5 — تغییر ممنوع]
│   ├── execution.py              [MT5 — تغییر ممنوع]
│   ├── futures.py                + FuturesAccountSnapshot, FuturesPositionSnapshot,
│   │                               FuturesOrderCommand, ConditionalOrderSnapshot,
│   │                               FuturesMarketSnapshot, FuturesFill
│   └── instruments.py            + InstrumentSpec, LeverageBracket, MarginTier, VenueCapabilities
├── ports/
│   ├── broker.py                 [MT5 — تغییر ممنوع]
│   ├── futures_broker.py         + IFuturesBroker (capability-based)
│   └── marketdata.py             [MT5]
├── app/
│   ├── engine.py                 [MT5 — تغییر ممنوع]
│   ├── decision.py, confirmations.py, gates.py, order_budget.py, arbiter.py   [مشترک — reuse]
│   ├── risk/profiles.py, checks.py                                            [مشترک — reuse]
│   └── futures/
│       ├── engine.py             + FuturesEngine
│       ├── capital.py            + sizing / margin / liquidation
│       ├── protection.py         + state machine حفاظت
│       ├── reconcile.py          + تطبیق سه‌طرفه
│       └── basis.py              + analytics (advisory only)
├── adapters/
│   ├── mt5_broker.py, fake_broker.py    [تغییر ممنوع]
│   └── toobit/
│       ├── signer.py             + امضای byte-exact
│       ├── rest.py               + rate-limit aware
│       ├── ws.py                 + فقط اگر REQ-TB-036 اثبات شود
│       ├── futures_broker.py     + پیاده‌سازی IFuturesBroker
│       └── marketdata.py         + feed + کیفیت‌سنجی
└── runtime/
    ├── live.py, bootstrap.py     [MT5 — تغییر ممنوع]
    └── toobit/
        ├── live.py               + composition root مستقل
        └── cli.py                + CLI مستقل
```

### ۵.۱ مرز مسئولیت (قابل استناد در Code Review)

| لایه | مسئول | ممنوع |
|---|---|---|
| Strategy Core (`strategies/*`, `decision.py`, `confirmations.py`, `gates.py`) | تولید سیگنال از کندل‌های همان venue | هر دانشی از `positionSide`, `leverage`, `margin_mode`, `contract_multiplier`, `funding` |
| Risk Policy (مشترک) | درصد ریسک، سقف روزانه، DD، concurrency — **currency-neutral** | محاسبهٔ حجم یا مارجین |
| Futures Capital | `risk_budget` → `contracts`، مارجین، لیکوییدیشن، هزینه‌ها | تصمیم Tier یا تأییدیه |
| Futures Protection | FSM حفاظت، تشخیص بی‌حفاظتی، close اضطراری | ارسال سفارش ورود |
| Toobit Adapter | ترجمه به API، امضا، rate limit — **تنها جای دانش `XAU-SWAP-USDT`** | هر تصمیم ریسک یا سیاست |
| Runtime Toobit | سیم‌کشی، feature flag، قفل تک‌نمونه، Kill Switch | منطق تجاری |

هر process فقط **یک** venue + یک حساب + یک instrument فعال دارد. Multi-account orchestration در این فاز **خارج از دامنه** است.

---

## ۶. قراردادهای دامنه‌ای جدید

### ۶.۱ `InstrumentSpec` — منبع همهٔ ثابت‌های قرارداد
| فیلد | نوع | قید |
|---|---|---|
| `venue`, `market_type` | str | `toobit` / `USDT_M_PERPETUAL` |
| `canonical_symbol` | str | `XAUUSDT` (نمایشی/پنل) |
| `venue_symbol` | str | `XAU-SWAP-USDT` (تنها ارسالی به API) |
| `base`, `quote`, `settlement_asset` | str | `XAU` / `USD` / `USDT` |
| `contract_multiplier` | Decimal | **اجباری، بدون default**؛ نبود ⇒ startup fail `[OPEN: COND-1]` |
| `tick_size`, `qty_step`, `min_qty`, `max_qty`, `min_notional` | Decimal | از `exchangeInfo` زنده با `fetched_at` و `expires_at` |
| `price_precision`, `qty_precision` | int | از metadata |
| `status` | str | فقط `TRADING` مجاز |
| `leverage_brackets` | list[LeverageBracket] | `max_leverage`, `min_notional`, `max_notional`, `maintenance_margin_rate` |
| `fetched_at`, `expires_at`, `source` | datetime/str | TTL پیش‌فرض ۶۰ دقیقه |

### ۶.۲ `FuturesAccountSnapshot`
`account_id`, `equity_usdt`, `wallet_balance`, `available_balance`, `used_margin`, `position_margin`, `order_margin`, `unrealized_pnl`, `timestamp_exchange`, `timestamp_received`
**قید:** تمام مقادیر پولی `Decimal`؛ `float` برای پول **ممنوع**.

### ۶.۳ `FuturesPositionSnapshot`
`position_id`, `venue_symbol`, `position_side` (`LONG|SHORT`), `contracts` (**واحد رسمی venue**), `underlying_qty` (مشتق محلی = `contracts × contract_multiplier`), `avg_entry`, `mark_price`, `liquidation_price|None`, `liquidation_source` (`venue|estimated|unavailable`), `initial_margin`, `maintenance_margin`, `unrealized_pnl`, `leverage`, `margin_mode`
**قید:** `underlying_qty` **هرگز** به venue ارسال نمی‌شود.

### ۶.۴ `FuturesOrderCommand`
`client_order_id`, `venue_symbol`, `side` (`BUY|SELL`), `position_side` (`LONG|SHORT`), `type` (`MARKET|LIMIT`), `contracts`, `price|None`, `time_in_force` (`GTC|IOC|FOK|POST_ONLY`), `reduce_intent` (bool محلی)، `protection` (`attached` | `separate` | `none`), `trigger_config|None`
**قید سخت:** فیلد `value_quantity` وجود ندارد. اگر کسی سعی کند واحد USDT بفرستد، validator با `OrderUnitError` رد می‌کند (fail-closed).

### ۶.۵ `ConditionalOrderSnapshot`
`venue_order_id`, `client_order_id`, `stop_category` (`STOP|STOP_PROFIT_LOSS`), `stop_order_type` (`TAKE_PROFIT|STOP_LOSS|None`), `trigger_price`, `trigger_by` (`CONTRACT_PRICE|MARK_PRICE`), `execution_type` (`MARKET|LIMIT`), `contracts`, `status`, `linked_position_side`

### ۶.۶ `FuturesMarketSnapshot`
`bid`, `ask`, `bid_qty`, `ask_qty`, `last`, `mark`, `index`, `funding_rate`, `next_funding_time`, `funding_interval_h`, `timestamp_exchange`, `timestamp_received`, `source`, `quality` (`ok|stale|invalid|crossed|partial`)

### ۶.۷ `VenueCapabilities`
`supports_market`, `supports_limit`, `supports_ioc`, `supports_attached_tp_sl`, `supports_conditional`, `supports_algo_orders_query`, `supports_positionside_hedge_only`, `supports_leverage_tiers`
**قاعده:** قابلیت ناشناخته = `unsupported`. هیچ قابلیت بر اساس فرض فعال نمی‌شود.

---

## ۷. Toobit API — قرارداد پیاده‌سازی

### ۷.۱ هویت و احراز
| مورد | مقدار | برچسب |
|---|---|---|
| Base URL | `https://api.toobit.com` (production؛ sandbox فرض نشود) | `[DOC]` |
| هدر کلید | `X-BB-APIKEY` | `[DOC]` |
| `Content-Type` | `application/json` (مسیرهای v2) | `[DOC]` |
| امضا | HMAC-SHA256، lowercase hex، key = secret | `[DOC]` |
| رشتهٔ امضا (v2 + JSON) | `queryString + rawJsonBody` — **بدون `&` بین آن‌ها** | `[DOC]` |
| الزام بدنه | JSON **compact تک‌خطی**، byte-for-byte یکسان با آنچه امضا شده؛ بدون `\n`/`\r\n` | `[DOC]` |
| پارامترهای query | `timestamp`, `recvWindow` (اختیاری، محدود)، `category=USDT` | `[DOC]` |
| testnet | **وجود ندارد.** تست واقعی باید read-only سپس controlled validation باشد | `[DOC]` |

> پیاده‌سازی: `json.dumps(body, separators=(",", ":"), ensure_ascii=False)` ← همان بایت‌ها هم امضا و هم ارسال شوند. تست goldens با بدنه‌های نمونهٔ رسمی اجباری است.

### ۷.۲ جدول endpointها (مرجع واحد)

| کاربرد | Endpoint | Method / نسخه | پارامترهای الزامی کلیدی | برچسب |
|---|---|---|---|---|
| Metadata قرارداد | `/api/v1/exchangeInfo` | GET v1 | آرایهٔ `contracts`؛ `status` | `[DOC]` |
| ثبت سفارش عادی | `/api/v2/futures/order` | POST v2 | `symbol, side, positionSide, type, newClientOrderId` + `quantity` **یا** `valueQuantity` | `[DOC]` |
| سفارش شرطی (plan) | `/api/v2/futures/algo-order` | POST v2 | `type=STOP\|STOP_MARKET`, `stopPrice`, `newClientOrderId` | `[DOC]` |
| **TP/SL موجود** | `/api/v2/futures/open-algo-orders` | GET v2 | **`stopCategory=STOP_PROFIT_LOSS`** | `[DOC]` |
| plan orderهای موجود | `/api/v2/futures/open-algo-orders` | GET v2 | **`stopCategory=STOP`** | `[DOC]` |
| تغییر اهرم | `/api/v2/futures/leverage` | POST v2 | `symbol`, `leverage` | `[DOC]` |
| **خواندن اهرم + margin mode** | `/api/v1/futures/accountLeverage` | GET v1 (signed) | `symbol` → `leverage`, `marginType` | `[DOC]` |
| **تغییر margin mode** | `/api/v1/futures/marginType` | POST v1 | `symbol`, `marginType=CROSS\|ISOLATED` | `[DOC]` |
| پوزیشن‌ها | `/api/v1/futures/positions` | GET v1 (signed) | `position` = contracts، `available` | `[DOC]` |
| Funding | `/api/v1/futures/fundingRate` | GET v1 | نرخ **و** interval | `[DOC]` |
| Order / trades / history | طبق مستندات v2 | GET | برای reconciliation | `[DOC]` |

> **هشدار P0 برای پیاده‌سازی:** پیش‌فرض `open-algo-orders` فقط `STOP` است و **TP/SL را برنمی‌گرداند**. خواندن بدون `stopCategory` = لیست خالی = تشخیص غلط «حفاظت وجود ندارد». استعلام TP/SL **فقط** با `STOP_PROFIT_LOSS`.

### ۷.۳ قواعد اجباری سفارش
- `newClientOrderId` **اجباری** است و باید یکتا در سطح runtime/account باشد.
- `side` و `positionSide` دو فیلد مستقل‌اند. نگاشت V1 (`BUY_OPEN`/`SELL_CLOSE`) **ممنوع** است.
- `reduceOnly` در پارامترهای مستند V2 وجود ندارد. **اختراع یا ارسال نکنید.** بستن با `side` + `positionSide` + مقدار دقیق.
- `quantity` بر حسب **contracts** است. `valueQuantity` (USDT) در فاز ۱ **ممنوع** (validator رد می‌کند).
- `timeInForce` مجاز: `GTC`, `FOK`, `IOC`, `POST_ONLY`.
- TP/SL می‌توانند به سفارش ورود **attached** شوند (`takeProfit`, `stopLoss`, `tpOrderType`, `slOrderType`, `tpLimitPrice`) — این مسیر **ترجیح داده می‌شود** (`[DOC]`) چون پنجرهٔ بی‌حفاظت را صفر می‌کند.

### ۷.۴ خطاها و Rate Limit
- هدرها: `X-Api-Limit-Status`, `X-Api-Limit`, `X-Api-Limit-Reset-Timestamp` — پایش و ثبت در journal. `[DOC]`
- `429` و `5xx`: backoff با jitter، سقف تلاش، **بدون** نقض rate limit.
- خطای auth / permission / symbol نامعتبر / `API_TRADE_FORBIDDEN` / `OPEN_FORBIDDEN`: **retry ممنوع**؛ فوراً `QUARANTINE` و ورود مسدود.
- `status != TRADING` یا نبود قرارداد در `exchangeInfo`: ورود ممنوع.

### ۷.۵ ثابت‌های متغیر — ممنوعیت hardcode
این مقادیر **باید** در runtime از API/اعلان زنده خوانده شوند و در ژورنال با `fetched_at` ثبت شوند:
`tick_size`, `qty_step`, `min_qty`, `max_qty`, `min_notional`, `contract_multiplier` (فقط از config تأییدشده)، `leverage_brackets`, `maintenance_margin_tiers`, `funding_rate`, `funding_interval_h`, `fee_rate_maker/taker`.
هیچ سقف اهرمی (از جمله ۵۰۰x) در کد hardcode نمی‌شود. جدول‌های اهرم/مارجین و بازهٔ funding تغییر می‌کنند (`[DOC]` — بازهٔ XAU از ۴ به ۸ ساعت و سقف از ±۰.۳۷۵٪ به ±۰.۰۵٪ در ۲۰۲۶-۰۱-۳۰ اصلاح شد).

---

## ۸. قیمت، Basis و Market Data

### ۸.۱ قاعدهٔ پایه
قیمت MT5 (XAUUSD) و Toobit (XAU-SWAP-USDT) **جایگزین یکدیگر نیستند**. Strategy در Runtime Toobit باید S/R، Zone، Entry، SL و TP را روی کندل/تیک **Toobit** محاسبه کند. استفاده از کندل MT5 در تصمیم Runtime Toobit **ممنوع** است (تأخیر، اختلاف قیمت، ساعات معاملاتی متفاوت).

### ۸.۲ تبدیل ارز — تعریف دقیق
`f` = **تعداد USDT در برابر ۱ USD** (مثلاً 1.0003). جهت تبدیل صریح است؛ مقدار `1.0000` ثابت **مجاز نیست**.

```
reference_xau_usdt = xauusd_mid × f
crypto_mid         = (toobit_bid + toobit_ask) / 2
basis_abs          = crypto_mid − reference_xau_usdt
basis_pct          = 100 × basis_abs / reference_xau_usdt
```

### ۸.۳ کیفیت و رفتار
| شرط | رفتار الزامی |
|---|---|
| `f` معتبر و هم‌زمان موجود نیست | `quality=unadjusted`؛ Basis فقط نمایش خام؛ **گیت غیرفعال** |
| timestamp دو منبع ناهم‌تراز (بیش از `basis_max_skew_ms`) | `quality=unadjusted`؛ گیت غیرفعال |
| Feed stale / timestamp عقب‌تر از `stale_tick_seconds` پروفایل | ورود جدید block؛ `DATA_STALE` در journal؛ ادامهٔ مدیریت پوزیشن فقط با دادهٔ معتبر یا حقیقت venue |
| `bid ≥ ask` یا قیمت ≤ ۰ | `feed invalid`؛ block |
| `mark`/`index` ناموجود و سیاست نیازمند آن | block؛ **مقدار جعلی جایگزین نشود** |
| Spread/عمق نامناسب | block یا کاهش سایز طبق سیاست مصوب |

### ۸.۴ کاربرد قیمت‌ها
- مقایسهٔ تحلیلی: mid-to-mid با timestamp هم‌تراز.
- خرید اجرایی: Ask (или عمق order book)؛ فروش اجرایی: Bid.
- `mark_price`: mark-to-market و ارزیابی لیکوییدیشن.
- `index_price`: فقط برای کاربردهایی که صرافی تعریف کرده.
- `last_price` **به‌تنهایی** مبنای اجرا نیست.

### ۸.۵ Basis = advisory
Z-score و آمارهٔ rolling تنها ابزار monitoring/anomaly هستند. **هیچ basis مثبت/منفی به‌تنهایی سیگنال آربیتراژ نیست** و هیچ threshold پیش‌فرض قطعی وجود ندارد؛ تا calibration + walk-forward، وضعیت `advisory` است.

---

## ۹. ریسک، Sizing، اهرم و مارجین

### ۹.۱ ترتیب اجباری (تنها منبع حقیقت)
```
1) Risk Policy:  risk_pct از Tier/پروفایل (درصد)            ← مشترک با MT5، currency-neutral
2) risk_budget_usdt = min(equity_usdt × risk_pct, remaining_portfolio_risk_usdt)
3) contracts        = معادلهٔ ۹.۲  (per-contract، بدون circularity)
4) verify_risk      = contracts × loss_per_contract  ≤  risk_budget   (وگرنه کاهش contracts)
5) min_qty / min_notional / max_notional check
6) Margin: available_balance ≥ initial_margin × margin_safety_mult
7) Liquidation: فاصلهٔ entry→liq  ≥  stop_distance × liq_safety_mult  (اگر venue قیمت بدهد)
8) Data quality / spread / news / session gates
9) ارسال، حفاظت، reconciliation
```

### ۹.۲ معادلهٔ Sizing (اصلاح‌شده — جایگزین فرمول v1.0)
واحدها: `price` بر حسب USD (قیمت قرارداد)، `f` = USDT per USD، `m` = `contract_multiplier`، `q` = contracts.

```
notional_per_contract = price × m × f                       [USDT]
fee_per_contract      = fee_rate_taker × notional_per_contract × 2      (ورود + خروج)
slip_per_contract     = slippage_ticks × tick_size × m × f
n_settlements         = ceil(max_holding_hours / funding_interval_h)    (از API زنده)
fund_per_contract     = |funding_rate| × notional_per_contract × n_settlements

loss_per_contract = |price − stop| × m × f
                  + fee_per_contract + slip_per_contract + fund_per_contract

raw_q     = risk_budget_usdt / loss_per_contract
contracts = floor_to_step(raw_q, qty_step)
```

**چرا این نسخه درست است:**
- **هیچ وابستگی دوری ندارد.** در فرمول v1.0، `estimated_fee` و `funding_buffer` به notional وابسته بودند و notional به contracts — یعنی معادلهٔ حل‌نشده. اینجا همه‌چیز per-contract است.
- **`f` صریح است.** v1.0 در §۸ تبدیل USDT/USD را الزام کرده بود ولی در §۹ حذفش کرده بود → ریسک واقعی بزرگ‌تر از محاسبه.

### ۹.۳ قاعدهٔ حاکم
**اهرم اندازهٔ ریسک مجاز را تعیین نمی‌کند؛ فقط مارجین موردنیاز را تغییر می‌دهد.**
هیچ مسیری مجاز نیست `risk_pct` را از `leverage` استخراج کند. ابتدا حجم از بودجهٔ ریسک و فاصلهٔ استاپ؛ سپس مارجین، tier و available balance.

### ۹.۴ بازبینی‌های پس از محاسبه (fail-closed)
```
verify_risk = contracts × loss_per_contract
assert verify_risk ≤ risk_budget + ε                       # ε = 1e-9
assert contracts ≥ min_qty
assert contracts × notional_per_contract ≥ min_notional
assert contracts ≤ max_qty and ≤ max_notional_for_leverage_bracket
initial_margin = contracts × notional_per_contract / leverage
assert available_balance ≥ initial_margin × margin_safety_mult    # پیشنهاد 1.25
if liquidation_price from venue:
    assert |entry − liq| ≥ stop_distance × liq_safety_mult         # پیشنهاد 2.0
else:
    liquidation_source = "estimated" یا "unavailable" → فقط هشدار + ممنوعیت ادعای ضد‌لیکویید
```
اگر هیچ اصلاحی ممکن نبود، **رد معامله** — نه کاهش سکوت‌آمیز پارامتر ریسک.

### ۹.۵ قید اقتصادی (یافتهٔ جدید این ممیزی)
اگر `risk_budget_usdt / min_qty < fee_per_contract` باشد، **در هیچ فاصلهٔ استاپی** معامله از نظر اقتصادی ممکن نیست: هزینهٔ رفت‌وبرگشت به‌تنهایی از بودجهٔ ریسک بیشتر است. این باید در startup به‌عنوان **EconomicViabilityError** هشدار داده شود، نه اینکه در زمان اجرا با `contracts=0` بی‌صدا رد شود.

**مثال عددی (m به‌عنوان متغیر؛ `[OPEN: COND-1]`):**
`equity = 1000 USDT` · `risk_pct = 0.5%` → budget = `5.00 USDT` · `price = 2650.00` · `stop = 2648.00` → stop_dist = `2.00` · `f = 1.0003` · `fee_rate_taker = 0.06%` `[OPEN]` · `slippage = 2 ticks` · `funding = 0.01%/8h × 6` (۴۸ ساعت)

| | `m = 1` | `m = 0.1` |
|---|---:|---:|
| notional / contract | 2650.80 | 265.08 |
| fee (رفت‌وبرگشت) | 3.181 | 0.318 |
| slippage | 0.020 | 0.002 |
| funding (۶ تسویه) | 1.590 | 0.159 |
| price loss | 2.001 | 0.200 |
| **loss_per_contract** | **6.792** | **0.679** |
| raw contracts | 0.736 | 7.361 |
| contracts (step=1) | **0** → SKIP | **7** |
| verify risk | — | 4.754 ≤ 5.00 ✅ |
| notional کل | — | 1855.56 |
| initial margin @20x | — | 92.78 ✅ |

**درس:** با `m=1` این حساب **نمی‌تواند** با ۰.۵٪ ریسک و استاپ ۲ دلاری معامله کند (هزینهٔ رفت‌وبرگشت ۳.۱۸ USDT از بودجهٔ ۵ USDT را می‌بلعد). این نشان می‌دهد `contract_multiplier` فقط یک عدد نیست؛ **کل امکان‌پذیری استراتژی** به آن وابسته است. به همین دلیل COND-1 یک Gate سخت است، نه یک یادداشت.

### ۹.۶ اهرم و Margin Mode
- پیش‌فرض ایمن: `ISOLATED` **فقط اگر** read-back تأیید کند؛ `leverage` از config محافظه‌کارانه (`initial_leverage`)، نه سقف تبلیغاتی صرافی.
- هر تغییر اهرم/margin mode با پوزیشن باز **ممنوع** (کد چک می‌کند `position.contracts == 0`).
- پس از هر تغییر: read-back اجباری از `GET /api/v1/futures/accountLeverage`؛ عدم تطابق ⇒ `QUARANTINE`.
- جدول tier از API زنده؛ `max_notional_for_leverage_bracket` باید در §۹.۴ چک شود.

### ۹.۷ Funding
نرخ **و** interval **و** زمان تسویهٔ بعدی از API زنده. `funding_interval_h` در `n_settlements` استفاده می‌شود. مقدار در journal ثبت می‌شود. هیچ عدد ثابت (۴ یا ۸ ساعت) در کد نیست.

### ۹.۸ لیکوییدیشن
قیمت لیکوییدیشن **از venue**. مدل تخمینی فقط برای هشدار است و **جایگزین قیمت venue یا تضمین ضد‌لیکویید نیست**. حد ضرر معاملاتی با قیمت لیکوییدیشن یکی نیست و نباید به‌جای هم استفاده شوند.

---

## ۱۰. چرخهٔ سفارش و حفاظت

### ۱۰.۱ ماشین حالت
| حالت | معیار عبور | اقدام در timeout |
|---|---|---|
| `CREATED` | intent معتبر + `client_order_id` یکتا | — |
| `SUBMITTING` | write-ahead در journal: `idempotency_key`, `payload_hash`, `signed_body_hash` | → `UNKNOWN` |
| `ACKNOWLEDGED` | venue `orderId` و وضعیت معتبر؛ **به معنی fill نیست** | → `UNKNOWN` |
| `PARTIALLY_FILLED` | `contracts_filled` و `avg_price` از venue | استعلام مجدد |
| `FILLED` | مجموع fills == `quantity` و تأیید با position truth | — |
| `PROTECTION_PENDING` | SL/TP در حال ثبت/تأیید؛ deadline = `T_protect` | → close اضطراری |
| `PROTECTED` | **تأیید از venue**: ردیف SL/TP با `contracts` و `trigger` منطبق | — |
| `UNKNOWN` / `QUARANTINE` | نتیجه یا حفاظت نامعلوم | ورود جدید block، reconciliation اجباری |
| `CLOSED` | `position.contracts == 0` **و** همهٔ algo orders مرتبط terminal/cancelled | — |

### ۱۰.۲ سه قاعدهٔ P0

**P0-1 — بدون تأیید venue، `PROTECTED` وجود ندارد.**
منبع تأیید: `GET /api/v2/futures/open-algo-orders?stopCategory=STOP_PROFIT_LOSS` برای TP/SL و `stopCategory=STOP` برای plan orders. وضعیت داخلی بات هرگز مبنای حفاظت نیست.

**P0-2 — پنجرهٔ بی‌حفاظت باید صفر باشد، نه کوتاه.**
مسیر ترجیحی: ارسال SL/TP به‌صورت **attached** با سفارش ورود (`stopLoss`, `takeProfit`, `slOrderType`, `tpOrderType`) — اتمیک در venue. اگر Gate E اثبات کرد attach پشتیبانی نمی‌شود: `T_protect ≤ 2s`؛ انقضا ⇒ **close اضطراری خودکار**، نه ادامهٔ حیات بدون حفاظت.

**P0-3 — resend کورکورانه ممنوع.**
در هر timeout، اولین اقدام **read** است: order با `client_order_id` → trades → position. ارسال مجدد فقط پس از اثبات absence و با `client_order_id` **جدید** و re-seal ریسک.

### ۱۰.۳ سیاست ورود
- پیش‌فرض: **marketable LIMIT + IOC** با price cap = `min(bid/ask ∓ max_slippage_ticks × tick_size)`.
- اگر fill نشد: `NO_FILL` ثبت شود؛ ارسال مجدد فقط با تصمیم صریح strategy/retry policy و شناسهٔ جدید.
- سفارش Market ساده پیش‌فرض **نیست**.

### ۱۰.۴ Partial Fill
`contracts_filled` هرگز از `quantity` بیشتر نمی‌شود. مقدار حفاظت = مجموع fills، **نه** quantity سفارش. اگر `contracts_filled < min_qty` و پوزیشن شکل نگرفت ⇒ fill باقی‌مانده لغو و ژورنال `PARTIAL_ABANDONED`.

### ۱۰.۵ Restart و بازیابی
ترتیب اجباری در startup:
```
account → instrument metadata → positions → regular open orders
        → algo orders (stopCategory=STOP) → algo TP/SL (stopCategory=STOP_PROFIT_LOSS)
        → recent trades → reconciliation با local state
```
- divergence بین local و venue ⇒ **QUARANTINE**؛ فقط read/close/protect، ورود ممنوع.
- پوزیشن venue بدون رکورد محلی = **exposure واقعی** ⇒ adopt با policy صریح (شبیه `adopt_orphans` موجود `[SRC]`) یا هشدار؛ **هرگز بستن یا نادیده‌گرفتن خودکار بدون policy**.

---

## ۱۱. Position Side، Margin Mode و Close

- **`position_mode_expected = HEDGE_BOOK`** — تنها مقدار مجاز. هر مقدار دیگر ⇒ `ConfigError` در startup. `[DOC]` (API V2 فقط `LONG|SHORT` می‌پذیرد و `BOTH` ندارد)
- **باز کردن LONG:** `side=BUY, positionSide=LONG` · **بستن LONG:** `side=SELL, positionSide=LONG`
- **باز کردن SHORT:** `side=SELL, positionSide=SHORT` · **بستن SHORT:** `side=BUY, positionSide=SHORT`
- **One-Way Mode:** در فاز ۱ **پشتیبانی نمی‌شود.** شبیه‌سازی خاموش آن ممنوع است؛ چون بدون `reduceOnly` خطر باز شدن پوزیشن معکوس دارد.
- **Margin mode:** پیش‌فرض `ISOLATED` مشروط به read-back؛ تغییر با پوزیشن باز ممنوع؛ هر تغییر + read-back در journal.
- **پوزیشن دستی/ناشناخته:** بدون policy صریح نه adopt می‌شود نه بسته.

### ۱۱.۱ الگوریتم Close (جبران نبودِ `reduceOnly`)
```
1) live = GET positions (venue truth)
2) if live.contracts == 0: → CLOSED (بدون ارسال)
3) qty = min(intent_contracts, live.contracts)
4) if qty ≤ 0: abort + journal CLOSE_NOOP
5) submit close (side/positionSide معکوس، qty دقیق)
6) re-read positions + open orders
7) if live.contracts > 0 and delta > tolerance: → QUARANTINE (بدون retry خودکار)
8) if live.contracts == 0: cancel همهٔ algo orders مرتبط → CLOSED
```
**دلیل وجود مرحلهٔ ۳ و ۷:** بین خواندن و ارسال، پوزیشن می‌تواند توسط کاربر/TP/SL/لیکوییدیشن تغییر کند. بدون `reduceOnly`، مقدار اضافی به **باز کردن پوزیشن معکوس** تبدیل می‌شود — بدترین حالت ممکن.

---

## ۱۲. ژورنال، دیتابیس و پنل

### ۱۲.۱ مسیرها
```
var/mt5/                        ← بدون تغییر (سازگاری با نسخهٔ ۸.۲.۵)
var/toobit/<account_alias>/journal.db
var/toobit/<account_alias>/risk_state.db
var/toobit/<account_alias>/heartbeat.json
```
هیچ جدول یا رکورد risk-state بین MT5 و Toobit مشترک نیست. migration **افزودنی و نسخه‌دار**؛ هیچ ستون/جدول موجود MT5 حذف یا تغییر نوع نمی‌شود.

### ۱۲.۲ فیلدهای الزامی ژورنال (به فیلدهای موجود افزوده می‌شود)
| گروه | فیلدها |
|---|---|
| هویت | `platform`, `venue`, `account_alias`, `market_type` |
| نماد | `canonical_symbol`, `venue_symbol` |
| بازتولید تصمیم | `strategy_id`, `strategy_version`, `config_hash`, `signal_id` |
| قیمت | `reference_price`, `execution_bid`, `execution_ask`, `mark`, `index` |
| Basis | `basis_abs`, `basis_pct`, `basis_quality`, `usd_usdt_rate` |
| واحد | `contract_multiplier`, `contracts`, `underlying_qty`, `notional_usdt` |
| ریسک | `entry`, `stop`, `take_profit`, `trigger_by`, `liquidation_price`, `liquidation_source` |
| Futures | `leverage`, `margin_mode`, `position_side`, `initial_margin`, `maintenance_margin` |
| هزینه | `fees`, `funding`, `slippage`, `realized_pnl`, `unrealized_pnl` |
| تطبیق | `client_order_id`, `venue_order_id`, `algo_order_id`, `fill_ids` |
| کیفیت | `exchange_ts`, `received_ts`, `latency_ms`, `data_quality` |
| حفاظت | `protection_state`, `protection_verified_at`, `protection_source` |

`config_hash` باید شامل `venue` و `market_type` باشد و **برای هر Runtime جدا** محاسبه شود (وگرنه مقایسهٔ hash بین دو Runtime بی‌معنا یا گمراه‌کننده می‌شود).

### ۱۲.۳ پنل
حداقل: Exchange Accounts (Add/Edit/Disable/Test) · Account Dashboard · Futures Configuration · وضعیت اتصال/Market Data · پوزیشن‌ها و سفارش‌ها · PnL · وضعیت Stop Loss/Take Profit و **منبع تأیید آن** · وضعیت Reconciliation · خطاها · Kill Switch مستقل · هشدارهای بحرانی.

**قواعد امنیتی UI:**
- Secret فقط **یک بار** هنگام ورود؛ سپس فقط `masked fingerprint`.
- نه API Secret، نه passphrase، نه token در HTML، local storage، log، export یا error trace.
- UI هرگز مستقیم API صرافی را صدا نمی‌زند؛ فقط از runtime می‌خواند.
- تغییرات حساس (leverage/margin mode) از UI فقط با تأیید صریح و read-back نمایش داده شود.
- حذف/غیرفعال‌سازی حساب: اول بررسی پوزیشن و سفارش باز.

---

## ۱۳. امنیت، پایداری، کنترل عملیاتی

| مورد | الزام | معیار آزمون |
|---|---|---|
| ذخیرهٔ secret | OS secure store (Windows DPAPI) یا env فقط برای deployment کنترل‌شده؛ کلید رمزنگاری **کنار ciphertext** ذخیره نشود | اسکن بسته: صفر مقدار حساس |
| مجوز API | فقط **Read + Futures Trade**؛ Withdrawal درخواست نشود؛ IP whitelist اگر پشتیبانی شود | چک‌لیست امضاشده + اسکرین‌شات مجوز |
| امضا | deterministic و unit-tested با goldens؛ timestamp هم‌زمان با server time؛ `recvWindow` محدود | `test_toobit_signer` |
| WebSocket | backoff + jitter، heartbeat، gap detection، resubscription — **فقط اگر channel اثبات شود** `[OPEN]` | REQ-TB-036 |
| Rate limit | پایش هدرها؛ 429/5xx با backoff؛ auth/permission بدون retry | `test_rate_limit` |
| Reconciliation | در startup و دوره‌ای (پیشنهاد ۵ دقیقه) و قبل از هر ورود؛ کهنگی snapshot ⇒ رد ورود | `test_three_way_reconcile` |
| اختلاف local/venue | QUARANTINE؛ فقط read/close/protect | `test_quarantine` |
| Kill Switch | مستقل از MT5؛ `stop_new_entries` از `emergency_flatten` جدا؛ مستقل از پنل (CLI)؛ نتیجه در venue verify شود | `test_kill_switch_toobit` |
| ساعت | `exchange_ts` و `received_ts` جدا؛ میلی‌ثانیهٔ Toobit به UTC نرمال | `test_time_normalization` |
| Correlation | هر order/protection/close با `correlation_id` + `payload_hash` + `response_code` در journal | `test_journal_correlation` |
| Graceful shutdown | ابتدا ورود جدید متوقف، وضعیت و حفاظت از venue خوانده، سپس exit؛ **پوزیشن باز رها نشود** | `test_graceful_shutdown` |
| Feature flag | `toobit_enabled=false` و `trading_enabled=false` پیش‌فرض | `test_feature_flags` |

---

## ۱۴. رجیستری تعارض (وضعیت حل)

| ID | تعارض | حکم نهایی | وضعیت |
|---|---|---|---|
| C-01 | «One-Way/Hedge mode انتخابی» ↔ API فقط `LONG\|SHORT` | HEDGE_BOOK اجباری؛ One-Way پشتیبانی نمی‌شود | **حل شد** (§۱۱) |
| C-02 | «تأیید حفاظت از algo-order» ↔ پیش‌فرض API فقط `STOP` | `stopCategory=STOP_PROFIT_LOSS` الزامی | **حل شد** (§۷.۲، §۱۰) |
| C-03 | `contract_multiplier` تأییدنشده ↔ کل ریاضیات ریسک | config اجباری + Gate سخت COND-1 | **باز تا COND-1** |
| C-04 | فرمول sizing: بدون `f`، با circularity | معادلهٔ per-contract §۹.۲ | **حل شد** |
| H-01 | endpoint تغییر margin mode بدون نام | `/api/v1/futures/marginType` + read-back `/api/v1/futures/accountLeverage` | **حل شد** (§۷.۲) |
| H-02 | هدر auth و شکل body ذکر نشده | `X-BB-APIKEY` + compact single-line + goldens | **حل شد** (§۷.۱) |
| H-03 | درخت پیشنهادی ↔ قاعدهٔ additive | ساختار افزودنی حفظ شد | **حل شد** (§۵) |
| H-04 | واحد عملکردی USD در برابر USDT | ارز حساب هر venue واحد عملکردی همان Runtime؛ تبدیل فقط در analytics | **حل شد** (§۸.۲، §۹.۲) |
| H-05 | close بدون سیاست race و بدون `reduceOnly` | الگوریتم ۸ مرحله‌ای §۱۱.۱ | **حل شد** |
| H-06 | `contracts/value_quantity` هر دو مجاز | فقط `contracts`؛ `value_quantity` رد می‌شود | **حل شد** (§۶.۴، §۷.۳) |
| H-07 | Gate E بدون سقف عددی | جدول `validation_limits` + امضای مالک | **حل شد** (§۱۶.۳) |
| M-01..M-07 | Basis/interface/migration/liquidation/manifest/ws | به‌ترتیب در §۸.۲، §۵.۱، §۱۲.۱، §۹.۸، §۱۶.۲، §۱۵ | **حل شد** (M-07 باز: REQ-TB-036) |
| CR-01..CR-10 (v1.0) | مطابق سند اصلی | همان احکام با اصلاحات H-03، H-06 | **حفظ و اصلاح شد** |

---

## ۱۵. تصمیم‌های باز (با مالک و شرط بستن)

| # | تصمیم | قاعدهٔ پیشنهادی این سند | مالک | شرط بستن | فاز |
|---|---|---|---|---|---|
| OD-1 | `contract_multiplier` | از config تأییدشده؛ نبود ⇒ fail | مالک + تیم | شاهد خام `exchangeInfo` + یک PnL دستی تأییدشده | **۰ (مسدودکننده)** |
| OD-2 | Position mode | `HEDGE_BOOK` تنها مقدار | تیم | read-back روی حساب واقعی | ۰ (مسدودکننده) |
| OD-3 | Margin mode | `ISOLATED` مشروط به read-back | مالک | تأیید کاربر + read-back | ۵ |
| OD-4 | Leverage اولیه | `initial_leverage` محافظه‌کارانه در config | مالک | tier زنده + margin stress test | ۴ |
| OD-5 | نوع ورود | marketable LIMIT + IOC + price cap | تیم | slippage/fill test در ساعات نقدشونده | ۵ |
| OD-6 | `trigger_by` برای SL | `MARK_PRICE` (پیشنهاد) | مالک | تست رفتار trigger و سازگاری با risk model | ۵ |
| OD-7 | نوع اجرای TP | `MARKET` بعد از trigger (پیشنهاد) | مالک | تست partial/no-fill و gap | ۵ |
| OD-8 | آستانهٔ Basis | هیچ؛ advisory | مالک | دادهٔ کافی + calibration + walk-forward | بعد از ۷ |
| OD-9 | منبع `f` (USDT/USD) | feed مستقل با timestamp؛ نبود ⇒ unadjusted | تیم | تأیید منبع + stale policy | ۳ |
| OD-10 | `funding_buffer` | `n_settlements` از interval زنده | تیم | funding history + stress | ۴ |
| OD-11 | ذخیرهٔ credential | DPAPI؛ env فقط استقرار کنترل‌شده | تیم | threat model + تست عدم نشت | ۲ |
| OD-12 | `fee_rate_taker` | از جدول کارمزد حساب (نه فرض) | مالک | اسکرین‌شات/API کارمزد + ثبت در config | ۰ |
| OD-13 | WebSocket user-data | اگر اثبات نشد ⇒ REST polling | تیم | REQ-TB-036 | ۲ |

---

## ۱۶. آزمون و معیارهای پذیرش

### ۱۶.۱ فازهای آزمون
| مرحله | آزمون | معیار قبولی |
|---|---|---|
| **A — MT5 Regression** | کل suite فعلی روی Windows/Python پشتیبانی‌شده + golden fixtures + parity tapes | `failed=0`, `errors=0`, `xfail ⊆ allowlist`؛ hash شش فایل ممنوعه ثابت |
| **B — Unit Futures** | symbol mapping، multiplier، qty step، PnL، fees، funding، margin tiers، liquidation | تمام مرزها/گردکردن/مقادیر نامعتبر fail-closed |
| **C — API Read-only** | server time، exchangeInfo، ticker/depth، mark/index، funding، account، positions/orders | احراز واقعی؛ secret redacted؛ rate-limit compliant |
| **D — Simulated Execution** | دادهٔ واقعی + fill شبیه‌سازی‌شده؛ disconnect/partial/timeout/stale | صفر سفارش واقعی؛ برچسب `SIMULATION` |
| **E — Controlled Venue Validation** | فقط با تأیید مالک، حجم بسیار کوچک: entry, fill, SL/TP, cancel, close | هر fill دارای TP/SL تأییدشده؛ reconcile دقیق |
| **F — Restart/Soak** | restart با پوزیشن و algo orders باز؛ قطع اینترنت/API؛ چند ساعته | صفر duplicate؛ recovery به venue truth؛ صفر پوزیشن بی‌حفاظت |
| **G — Release** | security، runbook، rollback، migration، چک‌لیست اپراتور | صفر P0/P1 در مسیر معامله |

### ۱۶.۲ معیارهای پذیرش (قابل اندازه‌گیری)
| # | معیار | سنجه | شاهد |
|---|---|---|---|
| AC-01 | MT5 بدون تغییر رفتار | تست‌ها سبز + `git diff` خالی برای ۶ فایل ممنوعه | log + hash |
| AC-02 | یکتایی سفارش | `client_order_id` تکراری ⇒ دقیقاً ۱ سفارش در venue | log خام + شمارش |
| AC-03 | بدون resend کورکورانه | ۱۰۰٪ timeoutها: اولین اقدام read | failure-injection trace |
| AC-04 | حفاظت اثبات‌شده | ۱۰۰٪ fillها دارای ردیف TP/SL با `STOP_PROFIT_LOSS` و contracts منطبق | JSON خام |
| AC-05 | پنجرهٔ بی‌حفاظت | ۰ در مسیر attached؛ ≤ `T_protect` در fallback | timestamp journal |
| AC-06 | Partial fill | `contracts_filled ≤ quantity`؛ protection = مجموع fills | fixture |
| AC-07 | Restart | ۰ duplicate، ۰ پوزیشن بی‌حفاظت، reconcile clean | journal + snapshot |
| AC-08 | Metadata زنده | ۱۰۰٪ مقادیر از API با `fetched_at`/`expires_at` | cache dump |
| AC-09 | Fail-closed داده | نبود mark/funding در مسیر نیازمند ⇒ ۰ ورود | test |
| AC-10 | عدم نشت secret | ۰ مورد در log/journal/UI/export (اسکن الگو) | اسکن |
| AC-11 | PnL فقط از fills واقعی | ۰ قاطی‌شدن simulation و real | گزارش |
| AC-12 | صفر P0/P1 باز | defect register | sign-off مالک |

### ۱۶.۳ محدودیت‌های Gate E (عدد سخت، نه توصیه)
| پارامتر | مقدار پیشنهادی |
|---|---|
| `max_attempts` | ۵ در هر نشست |
| `max_notional_usdt` هر تلاش | ۱۰۰ |
| `max_loss_absolute_usdt` کل نشست | ۱۰ |
| `max_position_contracts` | ۱ |
| `abort_conditions` | هر divergence، هر `UNKNOWN`، هر عدم تطابق حفاظت، هر 429 مکرر |
| مجوز | امضای کتبی مالک + تاریخ + حساب مشخص |

---

## ۱۷. Roadmap (ترتیب اجرا)

| فاز | خروجی | Gate خروج |
|---|---|---|
| **۰** | Freeze baseline، اجرای کامل suite (اکنون ۱۰۸۴ pass + ۱ xfail `[SRC]`)، اصلاح manifest، hash شش فایل، **بستن COND-1/2/3** | Gate ۰ سبز — **اکنون قرمز** |
| ۱ | Domain & Ports (`futures.py`, `instruments.py`, `ports/futures_broker.py`) | contract/architecture tests |
| ۲ | Adapter read-only: signer, exchangeInfo, account, positions, funding, leverage/margin read-back | هیچ write endpoint فعال نیست |
| ۳ | Market profile + feed quality + basis (advisory) | stale/invalid fail-closed |
| ۴ | Futures risk: sizing §۹.۲، fees، funding، margin، liquidation | property/stress سبز |
| ۵ | Execution + protection + reconciliation | AC-02..AC-07 |
| ۶ | Runtime جدا + پنل + Kill Switch + DB جدا | AC-10، AC-12 |
| ۷ | Simulation، سپس Gate E کنترل‌شده | AC-04، AC-05، §۱۶.۳ |
| ۸ | Handover: runbook، rollback، ADR، manifest، ZIP | صفر P0/P1 + sign-off |

**قاعده:** ابتدا read-only، بعد write. کلید API با `Trade` permission از روز اول فعال **نمی‌شود**.

---

## ۱۸. استقرار و Rollback

- Toobit در پوشه و process مستقل، با config و DB مستقل از MT5.
- نسخهٔ MT5 8.2.5 به‌عنوان baseline تغییرناپذیر نگهداری می‌شود؛ تغییرات مشترک فقط در شاخهٔ توسعه و با regression gate.
- `toobit_enabled=false` و `trading_enabled=false` پیش‌فرض.
- راه‌اندازی: read-only → simulation → write فقط در فاز کنترل‌شده.
- **Rollback:** ابتدا ورود جدید متوقف → وضعیت سفارش/پوزیشن/حفاظت از Toobit خوانده → سپس **فقط** Runtime جدید خاموش. **پوزیشن باز را با خاموش‌کردن process رها نکنید.**
- Migration باید additive و versioned باشد؛ بازگشت کد نباید DB را destructive کند.
- Emergency stop مستقل از پنل (CLI/ops) و نتیجهٔ آن در venue verify شود.
- SHA256SUMS با LF تولید و self-hash شود؛ release manifest اعداد را از واقعیت بخواند.

---

## ۱۹. تحویل نهایی مورد انتظار

1. سورس کامل با `adapters/toobit/*`, `app/futures/*`, `runtime/toobit/*`.
2. گزارش اجرای کامل suite (MT5 + Toobit) با اعداد واقعی، نه ادعا.
3. نمونهٔ پاسخ خام **redacted** از `exchangeInfo`, `account`, `positions`, `open-algo-orders` (هر دو `stopCategory`).
4. Golden fixtures برای `XAU-SWAP-USDT` و بدنه‌های امضاشده.
5. Migration scripts + schema documentation.
6. چک‌لیست مجوز/امنیت API + شاهد عدم نشت secret.
7. نمونهٔ ژورنال: signal → decision → order → fills → protection (verified) → close → fees/funding.
8. Runbook برای: connect / disconnect / restart / unknown order / protection failure / kill switch / rollback.
9. Conflict Register، Defect Register، ADRها و release manifest اصلاح‌شده.
10. ZIP بازتولیدپذیر با `SHA256SUMS` (LF) و **تمام** مسیرهای اشاره‌شده در README موجود.

---

## ۲۰. مراجع

| منبع | کاربرد در این سند |
|---|---|
| `https://api-docs.toobit.com/api/usdt-m-api-v2` | سفارش عادی، plan order، open-algo-orders، leverage، enumها |
| `https://api-docs.toobit.com/api/basic-information` | امضای HMAC، `totalParams`، قاعدهٔ JSON body |
| `https://api-docs.toobit.com/api/code-examples.html` | نمونهٔ امضای `queryString + rawJsonBody` بدون `&` |
| `https://api-docs.toobit.com/api/usdt-m-market-data` · `/api/v1/exchangeInfo` | ساختار `contracts`، `filters` |
| `https://api-docs.toobit.com/agent/faq.html` | فرمت `X-SWAP-USDT`؛ عدم وجود testnet |
| `https://api-docs.toobit.com/api/usdt-m-account-and-trading` | `marginType` (v1)، `accountLeverage`، `positions` |
| `https://www.toobit.com/support/api-faq` | بند ۱۰ (testnet)، سفارش بر حسب contracts یا USDT |
| اعلان ۲۰۲۶-۰۱-۳۰ (funding XAU/XAG) | ۴h→۸h و سقف ±۰.۳۷۵٪→±۰.۰۵٪ |
| اعلان ۲۰۲۶-۰۳-۲۴ (position limits XAU) | جدول tier تا ۵۰۰x — نباید hardcode شود |
| سورس `GoldBot_Final_Release_v8.2.5.zip` | تمام گزاره‌های `[SRC]` |

**اعتبار:** تطبیق‌ها در ۲۰۲۶-۱۱-۲۱ انجام شده‌اند. API و اعلان‌ها تغییر می‌کنند؛ پیش از هر انتشار، بخش ۷ دوباره اعتبارسنجی شود.

---

## ۲۱. حکم نهایی

GoldBot 8.2.5 از نظر ساختار، وجود پورت تزریق‌پذیر، state recovery، ژورنال، adoption پس از restart و ۱٬۰۸۴ تست سبز، پایهٔ قابل قبولی برای توسعه است — **به‌شرط آن‌که لایهٔ Futures جدا افزوده شود و به مدل‌های MT5 دست نزند.**

**تصمیم:** Toobit به‌صورت Runtime مستقل روی همان کدبیس و Shared Strategy Core ساخته شود؛ دامنه/ریسک/اجرای Futures جدید افزوده شود؛ MT5 بدون تغییر رفتاری باقی بماند؛ نماد `XAU-SWAP-USDT` با `exchangeInfo` زنده تأیید شود؛ API V2 مبنای USDT-M باشد و امضا و conditional orders دقیقاً مطابق مستندات روز پیاده شوند.

**وضعیت: قابل Kickoff — غیرقابل معاملهٔ واقعی.**

پیش‌شرط‌های تبدیل به «آمادهٔ فاز ۱»:
- **COND-1:** شاهد خام `contract_multiplier` + اثبات PnL دستی
- **COND-2:** read-back موفق `accountLeverage` روی حساب واقعی
- **COND-3:** snapshot از `open-algo-orders` با هر دو `stopCategory` برای یک TP/SL واقعی
- **COND-4:** اصلاح manifest بسته (اعداد تست، `SHA256SUMS` با LF، حذف `var/*`)

> تصریح: این سند یک **مشخصات** است. هیچ کد Toobit نوشته، اجرا یا تست نشده است. اصلاح سند، پیاده‌سازی کد و اثبات عملکرد واقعی سه فعالیت مستقل‌اند؛ تنها اولی انجام شده است.

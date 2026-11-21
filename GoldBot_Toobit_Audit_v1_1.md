# ممیزی مستقل و مشخصات اجرایی اصلاح‌شده
## GoldBot v8.2.5 (MT5) + Toobit USDT-M Futures روی XAU-SWAP-USDT

**سند مبنا:** `GoldBot_v8_2_5_Source_Audit_and_Toobit_Futures_Spec_v1_0.docx` (v1.0، 2026-11-21)
**سورس مبنا:** `GoldBot_Final_Release_v8.2.5.zip` (293 فایل، `git_commit 894560d`, build `2026-11-21T01:18:50Z`)
**نسخهٔ این گزارش:** v1.1 — هیئت ممیزی مستقل
**تاریخ ممیزی:** 2026-11-21
**روش:** بازرسی مستقیم سورس + اجرای واقعی test suite + تطبیق تک‌تک ادعاهای API با مستندات رسمی Toobit (api-docs.toobit.com و اعلان‌های رسمی)

> **محدودهٔ شواهد.** هر گزاره در این سند با یکی از سه برچسب همراه است:
> `[SRC]` = با اجرا/بازرسی مستقیم سورس اثبات شد · `[DOC]` = با مستندات رسمی Toobit تطبیق داده شد · `[OPEN]` = تأیید نشده و نیازمند تصمیم/شاهد است.
> هیچ ادعایی برچسب‌نشده باقی نمانده است.

---

## بخش ۰ — حکم نهایی

| معیار | نتیجه |
|---|---|
| **وضعیت پذیرش** | **APPROVED WITH CONDITIONS** — برای جلسهٔ Kickoff قابل استفاده است؛ **پیش از شروع فاز ۱ (Domain & Ports) باید سه شرط C-01، C-02، C-03 بسته شوند.** |
| مجوز معاملهٔ واقعی | **صفر.** سند خودش این را پذیرفته و ممیزی تأیید می‌کند. |
| دقت ادعاهای API سند | بالا — ۱۰ مورد از ۱۲ ادعای کلیدی مستقیماً با مستندات رسمی تأیید شد. |
| دقت ادعاهای سند دربارهٔ سورس | بالا — ادعاهای CR-08/CR-09 و «هیچ کد Toobit وجود ندارد» تأیید شد. |
| مهم‌ترین نقص | فرمول sizing فاقد تبدیل USDT/USD و دارای وابستگی دوری هزینه‌ها (C-04)؛ و فرض وجود «One-Way Mode» که در API V2 وجود ندارد (C-01). |

نقاط قوت واقعی سند (نه تعارف): تفکیک درست «یک کدبیس / دو Runtime»، الزام بروکر به‌عنوان منبع حقیقت، ممنوعیت resend کورکورانه، الزام تأیید حفاظت از سمت صرافه، و صداقت در اعلام «بک‌تست منفی / آماده‌نبودن برای پول واقعی». این‌ها همان نقاطی هستند که اکثر اسناد این‌چنینی در آن‌ها شکست می‌خورند.

---

## خروجی ۱ — گزارش ممیزی مستقل

### ۱.۱ نتیجهٔ اجرای واقعی test suite (بستن ابهام سند)

سند در §2 نوشته است: «Full suite در این محیط تا پایان اجرا نشد… ادعای 1084 PASS مستقلاً تأیید نشده است.» این ابهام در این ممیزی **بسته شد**:

```
$ python -m pytest -q        # Python 3.14.7, Windows
1085 tests collected
1084 passed, 1 xfailed, 4 warnings in 135.99s (0:02:15)
```

`[SRC]` نتیجه: ادعای README و `docs/TEST-COVERAGE-STATUS.md` **صحیح است**. تنها استثنا: یک `xfail` مستند (تکرار delivery برای close پوزیشنِ از‌پیش‌بسته → `uncertain` به‌جای ack کش‌شده؛ **سفارش دومی ارسال نمی‌شود**).

**اصلاح لازم در معیار پذیرش:** عبارت «همه تست‌ها pass» در §16-A قابل اندازه‌گیری نیست. باید باشد: `failed == 0 and errors == 0 and xfail ⊆ allowlist مستند`.

### ۱.۲ صحت‌سنجی ادعاهای سورس

| ادعای سند | نتیجهٔ ممیزی | برچسب |
|---|---|---|
| در بسته هیچ کد/تنظیمات Toobit/XAUUSDT/Futures/ExchangeAdapter نیست | تأیید. `grep -ri toobit` روی کل ۲۹۳ فایل: صفر نتیجه. تنها آداپتور venue: `mt5_broker.py` + `fake_broker.py` | `[SRC]` |
| `reports/audit` و `reports/review` در ZIP نیستند (CR-08) | تأیید. `find reports -type d` فقط `reports` را برمی‌گرداند؛ ۱۰ فایل `.md/.json/.csv/.xlsx` بدون هیچ زیرپوشه‌ای | `[SRC]` |
| تعداد فایل تست در سه سند ۷۷/۷۹/۸۰ است اما ZIP ۷۳ دارد (CR-09) | تأیید و **دقیق‌سازی**: `tests/` شامل ۷۳ فایل `.py` است که ۷۱ مورد آن `test_*.py` است؛ `SHA256SUMS` هم دقیقاً ۷۱ ورودی `tests/test_*.py` دارد. عدد مرجع درست: **۷۱ ماژول تست / ۱۰۸۵ تست جمع‌آوری‌شده** | `[SRC]` |
| `SHA256SUMS` معتبر است (سند: PASS، ۲۹۲ ورودی) | **نیمه‌درست.** پس از نرمال‌سازی CRLF، `sha256sum -c` روی ۲۹۲ ورودی موفق است. اما فایل با **پایان‌خط CRLF** ذخیره شده و روی POSIX بدون نرمال‌سازی، هر ۲۹۲ ورودی FAIL می‌شود (`No such file or directory` به‌خاطر `\r` چسبیده به نام فایل). ادعای «PASS» وابسته به محیط است | `[SRC]` |
| Python compileall PASS | تأیید غیرمستقیم: import و اجرای کامل suite موفق | `[SRC]` |
| `IBrokerGateway` حاضر است ولی برای Futures کافی نیست | تأیید. `ports/broker.py` (۶۴ خط): `submit/modify/close` با `ApprovedOrder`؛ `deals`, `find_by_client_ref`; هیچ `conditional order`, `leverage`, `margin mode`, `position mode`, `funding` ندارد | `[SRC]` |
| `Tick` فقط bid/ask دارد؛ `SymbolInfo` فقط مفاهیم MT5 | تأیید. `SymbolInfo` شامل `stops_level_points`, `freeze_level_points`, `filling_mode`, `swap_*` — همه MT5-specific | `[SRC]` |
| Engine تک‌نماد/تک‌حساب است | تأیید. `engine.py` (۱۹۹۲ خط) برای یک `symbol/feed/broker`؛ state DB تک‌کلید | `[SRC]` |

### ۱.۳ صحت‌سنجی ادعاهای API Toobit

| ادعای سند | نتیجهٔ تطبیق با مستندات رسمی | برچسب |
|---|---|---|
| شناسهٔ قرارداد `XAU-SWAP-USDT` با خط تیره؛ `XAUUSDT` فقط فرم اسپات/نمایشی | **تأیید.** FAQ رسمی: Futures از فرمت `BTC-SWAP-USDT` استفاده می‌کند. اعلان رسمی ۲۰۲۶-۰۳-۲۴ هم عیناً `XAU-SWAP-USDT` را به‌کار می‌برد | `[DOC]` |
| `GET /api/v1/exchangeInfo`، قراردادها از آرایهٔ `contracts` با فیلترهای قیمت/مقدار | **تأیید.** پاسخ نمونهٔ رسمی آرایهٔ `contracts` با `filters[]` شامل `PRICE_FILTER` (`minPrice/maxPrice/tickSize`) | `[DOC]` |
| `POST /api/v2/futures/order`؛ `side=BUY/SELL` + `positionSide=LONG/SHORT`؛ `type=MARKET/LIMIT`؛ `newClientOrderId` **اجباری** | **تأیید کامل**، از جمله اجباری‌بودن `newClientOrderId` | `[DOC]` |
| نگاشت V1 (`BUY_OPEN/SELL_CLOSE`) نباید به V2 ارسال شود | **تأیید.** enum رسمی V2 فقط `BUY`,`SELL` + `LONG`,`SHORT` است | `[DOC]` |
| `POST /api/v2/futures/algo-order`؛ `STOP`/`STOP_MARKET`؛ `triggerBy=CONTRACT_PRICE|MARK_PRICE` | **تأیید** (triggerBy با مقادیر CONTRACT_PRICE/MARK_PRICE در همان صفحه) | `[DOC]` |
| امضای V2: `queryString + rawJsonBody` **بدون `&`**؛ body باید byte-for-byte یکسان و compact تک‌خطی باشد | **تأیید کامل** از صفحهٔ Code Examples و Basic Information | `[DOC]` |
| هدرهای rate limit: `X-Api-Limit-Status`, `X-Api-Limit`, `X-Api-Limit-Reset-Timestamp` | **تأیید کامل** (چنج‌لاگ رسمی) | `[DOC]` |
| `POST /api/v2/futures/leverage` | **تأیید** | `[DOC]` |
| Toobit testnet خارجی اختصاصی ارائه نمی‌کند | **تأیید.** API F.A.Q بند ۱۰: «We currently do not offer a dedicated external testnet environment.» | `[DOC]` |
| Funding قراردادهای XAU/XAG از ۴ ساعت به ۸ ساعت و سقف از ±۰.۳۷۵٪ به ±۰.۰۵٪ (۲۰۲۶-۰۱-۳۰) | **تأیید کامل**، با همان تاریخ و همان اعداد | `[DOC]` |
| `reduceOnly` نباید اختراع/ارسال شود | **تأیید رویکرد.** در پارامترهای مستندشدهٔ V2 order دیده نمی‌شود؛ سند درست عمل کرده که آن را از دامنه حذف کرده | `[DOC]` |
| «Bid/Ask/Mark/Index/Last + depth» از market data و «WebSocket» با sequence/gap detection | **تأیید نشده.** وجود channelهای عمومی محتمل است، اما **هیچ شاهدی برای user-data stream سفارش/پوزیشن دیده نشد** و سند هم آن را اثبات نکرده | `[OPEN]` |
| `contractMultiplier` قرارداد طلا | **تأیید نشده.** در منابع رسمی بررسی‌شده، مقدار ضریب قرارداد XAU-SWAP-USDT صریحاً مستند نشده. سند خودش در §9 به این ابهام اعتراف کرده — درست، اما آن را به یک Gate سخت ارتقا نداده | `[OPEN]` |

**جمع‌بندی ۱.۳:** سند در بخش API، به‌طور غیرمعمول دقیق است. ایرادها بیشتر «شکاف» هستند تا «خطا»: endpointهایی که نام برده نشده‌اند اما برای اجرای الزامات خودش حیاتی‌اند (بخش ۱.۴).

### ۱.۴ شکاف‌های کشف‌شده در ممیزی که سند اصلاً ثبت نکرده

این‌ها در Conflict Register سند (§14) غایب‌اند و توسط هیئت ممیزی افزوده شدند:

1. **`stopCategory` در `GET /api/v2/futures/open-algo-orders`.** مستند رسمی: مقدار پیش‌فرض `STOP` است و **TP/SL را شامل نمی‌شود**؛ برای TP/SL باید `stopCategory=STOP_PROFIT_LOSS` ارسال شود. اگر تیم طبق §7/§10 سند «open algo orders» را بدون این پارامتر بخواند، **همیشه لیست خالی می‌گیرد** و طبق قانون P0 خود سند، وارد حلقهٔ «protection تأیید نشد → close اضطراری» می‌شود — یا بدتر، اگر تیم فقدان ردیف را «حفاظت وجود ندارد» تفسیر نکند، پوزیشن بدون حفاظت می‌ماند. `[DOC]`
2. **Endpoint تغییر Margin Mode در v1 است، نه v2:** `POST /api/v1/futures/marginType` با `marginType=CROSS|ISOLATED`. سند §7 جدول را با عنوان «API V2» بسته و این مسیر را نام نبرده. `[DOC]`
3. **Endpoint read-back رسمی برای leverage + marginType:** `GET /api/v1/futures/accountLeverage` که هم `leverage` و هم `marginType` هر قرارداد را برمی‌گرداند. سند «read-back الزامی» را ۴ بار تکرار کرده اما هرگز نگفته با کدام endpoint — این برای برنامه‌نویس مبهم است. `[DOC]`
4. **نام هدر احراز هویت:** `X-BB-APIKEY`. سند §13 فقط از «signing» حرف زده و نام هدر را نگفته. `[DOC]`
5. **`positionSide` فقط `LONG|SHORT` است و `BOTH` وجود ندارد** → بخش C-01.
6. **چهارشنبه‌سوآپ MT5 در برابر ۸-ساعتهٔ Toobit:** مدل هزینهٔ فعلی (`triple_swap_weekday`, broker-day) از نظر مفهومی به Futures منتقل نمی‌شود؛ CR-06 سند این را گفته اما تعداد تسویه‌ها در افق نگهداری (۳۶/۴۸/۷۲ ساعت → ۴/۶/۹ بازهٔ ۸ ساعته) را کمّی نکرده. `[DOC]`

### ۱.۵ ارزیابی کلی

سند v1.0 یک **مشخصات توسعهٔ خوب** است: تصمیم معماری درست، مرزهای Runtime روشن، و صداقت دربارهٔ وضعیت پروژه. برای Kickoff کافی است. اما در سه جای بحرانی، «الزام» نوشته شده بدون آن‌که امکان‌پذیری‌اش در API اثبات شده باشد (One-Way Mode)، یا بدون آن‌که مسیر اجرایی‌اش named باشد (protection verification، read-back)، یا با ریاضیات نادرست (sizing). این سه دسته باید قبل از فاز ۱ بسته شوند، وگرنه تیم در میانهٔ پیاده‌سازی دامنه را دوباره طراحی می‌کند — گران‌ترین نوع بازکاری.

---

## خروجی ۲ — جدول کامل ایرادات

شدت: **C** = زیان کنترل‌نشده/سفارش تکراری/نقص امنیتی · **H** = نقص جدی معماری/ریسک/بازیابی · **M** = نقص عملکردی/تست/مانیتورینگ · **L** = نگهداری‌پذیری

| ID | شدت | محل در سند | ایراد | پیامد فنی | راهکار اصلاح |
|---|---|---|---|---|---|
| **C-01** | **Critical** | §11 کل بند؛ §6 `VenueCapabilities` | «Position Mode / One-Way / Hedge» به‌عنوان یک انتخاب قابل‌خواندن و قابل‌تغییر فرض شده، در حالی که API V2 `positionSide` را **اجباری** و فقط با `LONG|SHORT` می‌پذیرد؛ `BOTH` و پارامتر position-mode در معامله وجود ندارد | اگر تیم مدل «One-Way» بسازد، هر سفارش در زمان اجرا رد می‌شود یا معنای close اشتباه نگاشته می‌شود (باز کردن پوزیشن معکوس به‌جای بستن) → ریسک مستقیم مالی | دفتر position همیشه hedge-book بـا `positionSide` باشد. Config `position_mode_expected = HEDGE_BOOK`؛ هر مقدار دیگر fail-closed. قابلیت One-Way در فاز ۱ **پشتیبانی نشود** (نه شبیه‌سازی خاموش). read-back از `GET /api/v1/futures/accountLeverage`. |
| **C-02** | **Critical** | §7 (TP/SL)، §10 (`PROTECTED`)، §16 (AC) | تأیید حفاظت از «algo-order» خواسته شده بدون تعیین `stopCategory`؛ پیش‌فرض API فقط `STOP` را برمی‌گرداند و TP/SL را حذف می‌کند | تأیید حفاظت همیشه «یافت نشد» → یا حلقهٔ close اضطراری و بستن پوزیشن‌های سالم، یا (اگر کد فقدان ردیف را نادیده بگیرد) پوزیشن بدون SL با ادعای PROTECTED | استعلام TP/SL **فقط** با `stopCategory=STOP_PROFIT_LOSS`؛ استعلام plan order جدا با `stopCategory=STOP`. Reconciliation هر دو را با هم بخواند. تست اجباری: fixture با هر دو نوع. |
| **C-03** | **Critical** | §9 (پانویس اعتراف)، §6 `contract_multiplier` | `contractMultiplier` قرارداد طلا `[OPEN]` است، اما تمام ریاضیات sizing/margin/liquidation به آن وابسته است؛ سند آن را «تیم باید تأیید کند» گذاشته، نه Gate | یک ضریب اشتباه = ریسک واقعی چند برابر ریسک محاسبه‌شده، بدون هیچ خطای اجرایی | Gate-0 اجباری: شاهد خام `exchangeInfo` برای `XAU-SWAP-USDT` + یک سفارش کوچک واقعی با محاسبهٔ دستی PnL به‌عنوان اثبات ضریب. تا آن زمان `contract_multiplier` از config اجباری و بدون default؛ نبودش = startup fail. |
| **C-04** | **Critical** | §9 فرمول sizing | فرمول `loss_per_contract` نه تبدیل **USDT/USD** دارد و نه circularity هزینه‌ها را حل می‌کند؛ `estimated_entry_fee` و `funding_buffer` به notional وابسته‌اند و notional به contracts | ریسک واقعی به‌اندازهٔ انحراف نرخ USDT/USD و هزینه‌ها بیشتر از بودجهٔ ریسک؛ در USDT depeg شدید، خطای چند درصدی در ریسک | فرمول per-contract بازنویسی شود (بخش ۴ این گزارش، معادلهٔ ۴.۱) تا هیچ وابستگی دوری نماند و `f = USDT per USD` صریح باشد. |
| **H-01** | High | §7 جدول endpointها | تغییر/خواندن Margin Mode با endpoint مشخص نشده؛ مسیر واقعی v1 است (`/api/v1/futures/marginType`, `/api/v1/futures/accountLeverage`) | برنامه‌نویس endpoint حدس می‌زند؛ یا 404 می‌گیرد یا از مسیر اشتباه مارجین تغییر می‌دهد | جدول §7 با ستون «endpoint + method + نسخه» بازنویسی شود؛ برای هر تغییر، read-back اجباری. |
| **H-02** | High | §13 | نام هدر auth (`X-BB-APIKEY`) و الزام `Content-Type: application/json` و body compact تک‌خطی هرگز ذکر نشده | امضای همیشه‌ناامیدکننده؛ خطای 401 مبهم در اولین پیاده‌سازی | افزودن بخش «Canonical signing contract» با نمونهٔ تست‌پذیر byte-exact + unit test goldens. |
| **H-03** | High | §5، §13 | ساختار پوشه‌ای `shared/ runtime_mt5/ runtime_toobit/` پیشنهاد شده، اما §5 خودش می‌گوید تغییر MT5 باید «additive» باشد. جابه‌جایی درخت، import path و packaging را می‌شکند | یا regression بی‌سروصدای MT5 (نقض شرط پذیرش) یا دوباره‌کاری | **گزینهٔ انتخابی:** ساختار فعلی `src/goldbot` حفظ شود؛ پکیج‌های جدید additive: `goldbot/domain/futures.py`, `goldbot/ports/futures.py`, `goldbot/adapters/toobit/`, `goldbot/runtime/toobit_*`. بازآرایی فیزیکی به فاز بعد و فقط با Change Request + parity gate. |
| **H-04** | High | §8، §9 | واحد پول عملکردی تصمیم‌گیری نشده: ریسک MT5 بر حسب USD، حساب Toobit بر حسب USDT. سند در §8 تبدیل را الزامی کرده اما در §9 sizing آن را حذف کرده (همان C-04) و در §6 `equity` را USDT گرفته | دو تعریف متناقض از equity → سقف‌های ریسک ناسازگار بین دو Runtime | اعلام صریح: **واحد عملکردی هر Runtime ارز حساب همان venue است.** تبدیل فقط در لایهٔ Cross-Market Analytics با منبع/تازگی/کیفیت مشخص. |
| **H-05** | High | §10، §11 | قانون بستن «مقدار نباید از پوزیشن واقعی بیشتر/کمتر شود» بدون سیاست race: خواندن position، سپس ارسال close، در فاصلهٔ بین این دو ممکن است پوزیشن توسط کاربر یا SL تغییر کند | close با مقدار اضافی → باز شدن پوزیشن معکوس ناخواسته (چون `reduceOnly` وجود ندارد) | الگوریتم close اجباری: `read truth → compute qty = min(intent, live_remaining) → submit → read truth → اگر drift > tolerance، abort و QUARANTINE`؛ بدون retry خودکار until reconcile. تست failure-injection با کاهش خارجی پوزیشن بین دو خواندن. |
| **H-06** | High | §6 `FuturesOrderCommand` | فیلد `contracts/value_quantity` هر دو واحد را مجاز کرده، در حالی که خود سند در §9 هشدار «مخلوط نکنید» داده | ابهام واحد در دامنه = خطای ×ضریب در حجم | در فاز ۱ فقط `contracts` مجاز؛ `value_quantity` با validator رد شود (fail-closed)، تا وقتی تست واحد اثبات شود. |
| **H-07** | High | §16 (Test Strategy) | برای «No external testnet» راهکار داده شده، اما Gate E («Controlled Venue Validation») بدون معیار حجم/سرمایه/حد ضرر و بدون چک‌لیست تأیید مالک است | ورود به معاملهٔ واقعی با حجم کنترل‌نشده تحت عنوان «validation» | Gate E با اعداد سخت: `equity_snapshot`, `max_notional_usdt`, `max_loss_per_attempt`, `max_attempts`, `abort_conditions`، امضای مالک، و اسکرین‌شات/JSON خام هر fill. |
| **M-01** | Medium | §8 فرمول Basis | نام متغیر `usdt_per_usdcrypto_mid` منبع‌نشده و مبهم؛ جهت تبدیل (`USDT per USD` در برابر `USD per USDT`) بدون تعریف عددی؛ علامت `basis_abs` بدون convention | Basis با علامت/جهت معکوس؛ تفسیر اشتباه انحراف | تعریف: `f = USDT per 1 USD` (مثلاً 1.0003)؛ `reference = xauusd_mid × f`؛ `basis = crypto_mid − reference`؛ `basis_pct = 100 × basis / reference`؛ هر سه با `quality=adjusted|unadjusted`. |
| **M-02** | Medium | §9 جملهٔ اول | «Risk Policy فعلی قابل حفظ است» به‌صورت نثر نوشته شده، نه قرارداد interface؛ ریسک فعلی به `contract_size/lot/point` گره خورده | یا کپی‌برداری از `capital.py` با واحد غلط، یا fork کردن سیاست | تعریف port `IRiskPolicy` با ورودی currency-neutral (`risk_pct`, `stop_distance`) و خروجی `risk_budget` + ترتیب گام‌ها؛ `capital.py` موجود دست‌نخورده بماند و `futures_capital.py` جدا. |
| **M-03** | Medium | §12 (ژورنال) | افزودن ~۱۵ فیلد جدید به ژورنال بدون طرح migration؛ و بدون توجه به این‌که `config_hash` فعلی «resolved risk profile» را پوشش می‌دهد و افزودن venue آن را عوض می‌کند | شکستن سازگاری state DB موجود MT5 یا بی‌اعتبار شدن مقایسهٔ hash بین دو Runtime | migration افزودنی و نسخه‌دار؛ `config_hash` شامل `venue` و `market_type` شود و برای هر Runtime جدا محاسبه شود؛ تست مهاجرت از DB نسخهٔ ۸.۲.۵. |
| **M-04** | Medium | §16-B در برابر §9 | AC می‌گوید «liquidation checks … fail-closed»، اما §9 می‌گوید liquidation_price تخمینی فقط «هشدار» است | تناقض در معیار پذیرش؛ تست غیرقابل‌نوشتن | تفکیک دو چیز: (الف) اگر venue قیمت liquidation بدهد → گیت سخت با ضریب ایمنی؛ (ب) اگر ندهد → هشدار + الزام تخمین محافظه‌کارانه با ثبت `liquidation_source=estimated` و ممنوعیت ادعای «ضد‌لیکویید». |
| **M-05** | Medium | §2 | ادعای «SHA256SUMS PASS» بدون ذکر وابستگی به CRLF؛ و ۲۹۲ در برابر ۲۹۳ فایل | بازتولیدپذیری release در CI لینوکس می‌شکند | `SHA256SUMS` با LF تولید شود؛ فایل خودش هم hash شود (`SHA256SUMS.asc`/`.sha256`); شمارش در manifest از همان فایل تولید شود نه دستی. |
| **M-06** | Medium | §2 | سه سند عدد متفاوت فایل تست دارند و هیچ‌کدام درست نیست (۷۷/۷۹/۸۰ در برابر ۷۱) | بی‌اعتمادی به manifest؛ تست `test_release_metadata` نمی‌تواند این را بگیرد چون خودش از همان اعداد می‌خواند | `scripts/build_release.py` اعداد را از `pytest --co -q` و `find tests -name 'test_*.py' | wc -l` تزریق کند؛ تستی که عدد سند را با واقعیت مقایسه کند. |
| **M-07** | Medium | §5، §13 | WebSocket user-data (سفارش/پوزیشن) بدون اثبات وجود، اما طراحی reconnect/sequence-gap بر آن بنا شده `[OPEN]` | اگر channel وجود نداشته باشد، طراحی settlement دوباره‌کاری کامل | Gate فاز ۲: اثبات channel با یک اتصال read-only مستند؛ در غیر این‌صورت مسیر settlement = REST polling با interval و reconciliation دوره‌ای (الگوی MT5-03 موجود در سورس). |
| **L-01** | Low | بسته | ۱۴ فایل `var/` در release هست، شامل `probe_hours.py`, `probe_sessions.py`, `probe_tape.py`, `probe_tape2.py` و ۶ فایل `*.heartbeat.json` | نویز، و ریسک انتشار artifact عملیاتی/دادهٔ محیطی در بسته | `var/` فقط `.gitkeep`؛ build release باید allowlist داشته باشد. |
| **L-02** | Low | §17 Roadmap | Gate فاز ۰ («شمارش تست‌ها سازگار») در بستهٔ فعلی از قبل قرمز است | Kickoff با یک gate شکسته شروع می‌شود | اصلاح اعداد manifest پیش از Kickoff (کوچک و سریع). |
| **L-03** | Low | §3 | تکرار ادعای بک‌تست منفی بدون ارجاع به فایل شاهد (`BACKTEST-REPORT.md`) در متن سند | غیرقابل ردیابی در جلسه | ارجاع صریح + ثبت `git_commit` و `config_hash` اجرای بک‌تست. |
| **L-04** | Low | §15 | جدول Open Decisions بدون ستون «مالک» و «تاریخ بستن» | تصمیم‌ها معلق می‌مانند | افزودن owner/date/blocking-phase به هر ردیف. |

---

## خروجی ۳ — گزارش تناقض‌ها (Conflict Register یکپارچه)

شامل تناقض‌های داخلی سند + تناقض سند با سورس واقعی. ستون «هم‌زمان اصلاح شود» الزامی است.

| ID | نوع | محل | متن/مفهوم متناقض | حکم نهایی |
|---|---|---|---|---|
| X-01 | داخلی | §11 ↔ §7 | «position mode خوانده و بررسی می‌شود» ↔ `positionSide` اجباری `LONG|SHORT` بدون `BOTH` | §7 واقعیت API حاکم است؛ §11 بازنویسی شود (C-01). هم‌زمان: §6 `VenueCapabilities.supports_position_mode_read` باید به `supports_positionside_hedge_only` تغییر نام دهد. |
| X-02 | داخلی | §10 ↔ §7 | «حفاظت از algo-order استعلام شود» ↔ پاسخ پیش‌فرض API فقط `STOP` را می‌دهد | `stopCategory` الزامی شود (C-02). هم‌زمان: §16-AC و §19 (Journal sample) هم اصلاح شوند. |
| X-03 | داخلی | §9 ↔ §8 | §8 تبدیل USDT/USD را الزام می‌کند ↔ §9 sizing هیچ تبدیلی ندارد | §9 فرمول per-contract با `f` (C-04). هم‌زمان: §6 `FuturesPositionSnapshot` باید ارز PnL را صریح کند. |
| X-04 | داخلی | §5 (additive) ↔ §5 (درخت پوشه‌ای جدید) | «افزودن Toobit باید additive باشد» ↔ پیشنهاد جابه‌جایی کل درخت به `shared/ runtime_mt5/ runtime_toobit/` | جملهٔ additive حاکم است؛ درخت پیشنهادی حذف/به فاز بعد موکول (H-03). هم‌زمان: §17 فاز ۱ و §19 (ZIP ساختار) اصلاح شوند. |
| X-05 | داخلی | §2 ↔ §21 | §2 «۱۰۸۴ PASS تأیید نشده» ↔ §21 حکم نهایی بدون ذکر این ابهام | ممیزی این را بست: ۱۰۸۴ pass + ۱ xfail. §2 با شاهد به‌روز شود. |
| X-06 | داخلی | §16-A ↔ واقعیت | «همه تست‌ها pass» ↔ وجود یک `xfail` مستند | معیار به `failed=0, errors=0, xfail ⊆ allowlist` تغییر کند. |
| X-07 | سند↔سورس | §4 | ادعا: `src/goldbot/app/risk/*` «قابل استفاده» برای Futures | نیمه‌درست: `risk/profiles.py` و `checks.py` قابل استفاده‌اند؛ `capital.py` (۲۷۸ خط) به lot/contract و Gate D درصدی گره خورده. §4 باید ستون «قابل استفاده به‌صورت» اضافه کند: reuse / adapt / replace. |
| X-08 | سند↔سورس | §4 | ادعا: `order_coordinator.py` «منطق قابل استفاده» | درست است (۳۸۵ خط، write-ahead + idempotency + UNKNOWN)، اما `retry_guard.py` طبق سند خودِ سورس فقط «آماده» است و حلقهٔ retry واقعی وجود ندارد. §4 باید این را بگوید. |
| X-09 | سند↔سورس | §12 | «هیچ جدول risk state مشترک بین MT5 و Toobit» | سازگار با نیاز، اما سورس فعلاً **تک‌کلید** است (`SPEC-MERGED §16.2` خودش این را «باز» اعلام کرده). افزودن `account_alias` نیازمند مهاجرت است که §12 ذکر نکرده (M-03). |
| X-10 | سند↔سورس | §7 | «sandbox/testnet فرض نشود» | درست و تأیید شد. اما سند در §16-D «Simulated Execution» را بدون تعریف مرز «simulation در برابر demo صرافی» رها کرده؛ عبارت‌بندی باید یکسان شود. |
| X-11 | داخلی | §6 ↔ §9 | `FuturesAccountSnapshot` ارز `USDT` ↔ Risk policy درصدی از `equity` بدون واحد | H-04. |
| X-12 | داخلی | §15 (Open Decisions) ↔ §11 | «Position mode: بدون تغییر خودکار» ↔ «اگر ناسازگار است ورود مسدود شود» — این دو با هم سازگارند اما مکانیزم read-back نامشخص است | با X-01 حل می‌شود؛ read-back از `accountLeverage`. |
| X-13 | داخلی | §9 ↔ §16-B | «لیکوییدیشن: هشدار» ↔ «fail-closed» | M-04. |

**بدون تناقض Critical/High حل‌نشده؟** پس از اعمال اصلاحات بخش ۴، بله. جدول بالا تمام موارد C/H را با حکم صریح می‌بندد.

---

## خروجی ۴ — معماری نهایی پیشنهادی

### ۴.۱ ساختار (حفظ ساختار موجود، افزودن additive)

```
src/goldbot/                          # دست‌نخورده برای MT5
├── domain/                           # خالص، بدون I/O
│   ├── model.py                      # [MT5] بدون تغییر
│   ├── execution.py                  # [MT5] ApprovedOrder بدون تغییر
│   ├── futures.py                    # + جدید: Futures*Snapshot, FuturesOrderCommand
│   └── instruments.py                # + جدید: InstrumentSpec, VenueCapabilities
├── ports/
│   ├── broker.py                     # [MT5] بدون تغییر  ← شرط پذیرش
│   ├── futures_broker.py             # + جدید: IFuturesBroker (capability-based)
│   └── marketdata.py                 # [MT5] بدون تغییر
├── app/
│   ├── engine.py                     # [MT5] بدون تغییر
│   ├── futures/
│   │   ├── engine.py                 # + جدید: FuturesEngine (همان TradingEngine الگو)
│   │   ├── capital.py                # + جدید: sizing/margin/liquidation (per-contract)
│   │   ├── protection.py             # + جدید: state machine حفاظت
│   │   └── reconcile.py              # + جدید: سه‌طرفه (regular/algo/position)
│   └── ...
├── adapters/
│   ├── mt5_broker.py                 # [MT5] بدون تغییر
│   ├── fake_broker.py                # [MT5] بدون تغییر
│   └── toobit/
│       ├── signer.py                 # + تست byte-exact
│       ├── rest.py                   # + rate-limit aware
│       ├── ws.py                     # + فقط اگر M-07 اثبات شود
│       ├── futures_broker.py
│       └── marketdata.py
└── runtime/
    ├── live.py / bootstrap.py        # [MT5] بدون تغییر
    └── toobit/live.py                # + composition root مستقل
```

**قاعدهٔ سخت:** هیچ فایلی از `domain/model.py`, `domain/execution.py`, `ports/broker.py`, `adapters/mt5_broker.py`, `app/engine.py`, `runtime/live.py` مجاز به تغییر در فاز ۱–۵ نیست. `tests/test_architecture.py` و یک تست جدید `test_mt5_contract_frozen.py` (hash فایل‌ها) این را تضمین کنند.

### ۴.۲ مرز مسئولیت (قابل استناد در بازبینی کد)

| لایه | مسئول | ممنوع |
|---|---|---|
| Strategy Core (`strategies/*`, `app/decision.py`, `app/confirmations.py`) | تولید سیگنال از **کندل‌های همان venue** | هرگونه دانش از `positionSide`, leverage, margin mode, contract multiplier |
| Risk Policy (مشترک) | درصد ریسک، سقف روزانه، DD، concurrency — **currency-neutral** | محاسبهٔ حجم/مارجین |
| Futures Capital (Toobit) | تبدیل `risk_budget` → `contracts`، مارجین، لیکوییدیشن | تصمیم دربارهٔ Tier/تأییدیه |
| Protection (`app/futures/protection.py`) | state machine حفاظت و تشخیص عدم‌حفاظت | ارسال سفارش ورود |
| Adapter (Toobit) | ترجمه به API، امضا، rate limit، **تنها جای دانش `XAU-SWAP-USDT`** | تصمیم ریسک یا سیاست |
| Runtime (composition) | سیم‌کشی، feature flag، قفل تک‌نمونه | منطق تجاری |

### ۴.۳ جریان داده

```
Toobit REST/WS (candles, mark, funding)
   → ToobitMarketDataFeed → کیفیت‌سنجی (stale/invalid/unavailable)
   → Strategy Core (بدون تغییر) → Signal
   → Decision/Confirmations (مشترک، بدون تغییر)
   → Futures Capital: risk_budget → contracts (معادلهٔ ۴.۱)
   → Margin/Leverage/Liquidation gates
   → FuturesEngine: entry (marketable LIMIT + IOC)
   → Protection Service: attach/verify SL/TP (stopCategory=STOP_PROFIT_LOSS)
   → Reconcile سه‌طرفه (position ↔ regular orders ↔ algo orders)
   → Journal (per-runtime DB) + Web panel read-only
```

### ۴.۴ چرخهٔ سفارش و حفاظت (نسخهٔ اصلاح‌شدهٔ §10)

```
CREATED
  → SUBMITTING            (write-ahead: client_order_id, payload_hash, signed_body_hash)
  → ACKNOWLEDGED          (venue orderId؛ fill نیست)
  → PARTIALLY_FILLED      (contracts_filled += delta؛ هرگز جای FILLED)
  → FILLED                (تجمیع fills == quantity)
  → PROTECTION_PENDING    (deadline سخت T_protect)
  → PROTECTED             (تأیید از venue: SL/TP با quantity و trigger درست)
  → CLOSED                (position=0 و همهٔ algo orders مرتبط terminal)

هر حالت → UNKNOWN/QUARANTINE   (timeout، divergence، payload mismatch)
UNKNOWN → فقط read → reconcile → تصمیم (هرگز resend کورکورانه)
```

**سه قاعدهٔ P0 قابل تست:**

1. **بدون تأیید venue، PROTECTED وجود ندارد.** منبع تأیید = `GET /api/v2/futures/open-algo-orders?stopCategory=STOP_PROFIT_LOSS` (برای TP/SL) و `stopCategory=STOP` (برای plan orders).
2. **پنجرهٔ بی‌حفاظت باید صفر باشد، نه کوتاه.** سفارش ورود باید SL/TP را **به‌صورت attached** در همان درخواست regular order ببرد (`stopLoss`, `takeProfit`, `slOrderType`, `tpOrderType` — همه در مستند V2 موجودند `[DOC]`). اگر attach در اولین Gate E اثبات نشد، مسیر جایگزین: `T_protect ≤ 2s` و در صورت انقضا **close اضطراری خودکار** — نه ادامهٔ حیات بدون حفاظت.
3. **`T_protect` و `close` هر دو در journal با correlation id ثبت شوند.**

**معادلهٔ ۴.۱ — Sizing اصلاح‌شده (جایگزین فرمول §9 سند):**

واحدها: `price` بر حسب USD/oz (قیمت قرارداد)، `f` = USDT per 1 USD، `m` = contract_multiplier، `q` = contracts (عدد صحیح مضرب `qty_step`).

```
notional_per_contract_usdt = price × m × f
fee_per_contract_usdt     = fee_rate_taker × notional_per_contract_usdt × 2        # ورود+خروج
slip_per_contract_usdt    = slippage_ticks × tick_value_usdt
fund_per_contract_usdt    = |funding_rate| × notional_per_contract_usdt × n_settlements
                            n_settlements = ceil(max_holding_hours / funding_interval_h)   # از API زنده

loss_per_contract = |price − stop| × m × f
                  + fee_per_contract_usdt + slip_per_contract_usdt + fund_per_contract_usdt

risk_budget_usdt = min(equity_usdt × risk_pct, remaining_portfolio_risk_usdt)
raw_q = risk_budget_usdt / loss_per_contract
q     = floor_to_step(raw_q, qty_step)
```

بدون وابستگی دوری، چون همهٔ هزینه‌ها per-contract محاسبه شده‌اند. سپس **بازبینی اجباری**:

```
verify_risk = q × loss_per_contract
assert verify_risk ≤ risk_budget_usdt + ε        # وگرنه کاهش q
assert q ≥ min_qty and q × notional_per_contract_usdt ≥ min_notional
initial_margin = q × notional_per_contract_usdt / leverage
assert available_balance_usdt ≥ initial_margin × margin_safety_mult   # پیشنهاد 1.25
if venue_liq_price:  assert |price − liq| ≥ stop_distance × liq_safety_mult  # پیشنهاد 2.0
```

**قاعدهٔ حاکم (تکرار می‌شود چون در سند فقط یک بار آمده):** اهرم فقط مارجین را تغییر می‌دهد، نه ریسک. هیچ مسیری مجاز نیست `risk_pct` را از leverage استخراج کند.

---

## خروجی ۵ — الحاقات الزامی سند اجرایی (متن آمادهٔ درج)

این بخش، متن جایگزین برای بندهای معیوب است. به‌جای بازنویسی کل ۲۱ بخش، فقط بندهای متأثر.

### ۵.۱ جایگزین §7 (سطرهای endpoint)

| موضوع | Endpoint | Method / Version | الزام |
|---|---|---|---|
| Auth header | — | — | `X-BB-APIKEY` + `Content-Type: application/json` `[DOC]` |
| Market metadata | `/api/v1/exchangeInfo` | GET v1 | آرایهٔ `contracts`؛ `status=TRADING` الزامی |
| Regular order | `/api/v2/futures/order` | POST v2 | `newClientOrderId` اجباری؛ فقط `contracts` |
| Plan order | `/api/v2/futures/algo-order` | POST v2 | `STOP`/`STOP_MARKET` |
| Open plan orders | `/api/v2/futures/open-algo-orders` | GET v2 | **`stopCategory=STOP`** |
| **Open TP/SL** | `/api/v2/futures/open-algo-orders` | GET v2 | **`stopCategory=STOP_PROFIT_LOSS`** |
| Leverage set | `/api/v2/futures/leverage` | POST v2 | read-back اجباری |
| **Leverage + margin read-back** | `/api/v1/futures/accountLeverage` | GET v1 (signed) | `leverage` و `marginType` |
| **Margin mode set** | `/api/v1/futures/marginType` | POST v1 | `CROSS|ISOLATED`؛ read-back اجباری |
| Positions | `/api/v1/futures/positions` | GET v1 | `position` = contracts |
| Funding | `/api/v1/futures/fundingRate` | GET v1 | نرخ **و** interval زنده |

سطر «positions» در مستندات با `position` («Number of open positions (pieces)») و `available` توصیف شده؛ پس `contracts` واحد رسمی است و `underlying_qty` فقط مشتق محلی با `m` است و **نباید** به venue ارسال شود.

### ۵.۲ جایگزین §11 (Position/Margin/Close)

- **Position side:** همیشه `HEDGE_BOOK`. هر سفارش `positionSide` اجباری دارد.
- **باز کردن Long:** `side=BUY, positionSide=LONG`. **بستن Long:** `side=SELL, positionSide=LONG`. متقارن برای Short. (این همان قاعدهٔ درست سند است و حفظ می‌شود.)
- **One-Way Mode:** در فاز ۱ پشتیبانی نمی‌شود. `position_mode_expected` تنها مقدار مجاز `HEDGE_BOOK`؛ هر مقدار دیگر `ConfigError` در startup.
- **Margin mode:** پیش‌فرض `ISOLATED` **فقط اگر** read-back تأیید کند؛ تغییر با پوزیشن باز ممنوع (کد باید چک کند `position.contracts == 0`).
- **Close بدون `reduceOnly`:** الگوریتم سه‌مرحله‌ای H-05 اجباری است.

### ۵.۳ جایگزین §14 (Conflict Register)
پس از افزودن C-01..C-04، H-01..H-07، M-01..M-07، L-01..L-04 و X-01..X-13، جدول §14 سند باید با همین ۳۱ ردیف جایگزین شود (نه فقط ۱۰ ردیف فعلی). ردیف‌های CR-01..CR-10 فعلی معتبرند و حفظ می‌شوند.

### ۵.۴ جایگزین §16 (Acceptance Criteria — قابل اندازه‌گیری)

| # | معیار | سنجهٔ عددی | شاهد |
|---|---|---|---|
| AC-01 | MT5 بدون تغییر رفتار | `failed=0, errors=0`, `xfail ⊆ allowlist`, و hash ۶ فایل ممنوعهٔ تغییر (`domain/model.py`, `domain/execution.py`, `ports/broker.py`, `adapters/mt5_broker.py`, `app/engine.py`, `runtime/live.py`) ثابت | خروجی pytest + `git diff --stat` خالی برای آن ۶ فایل |
| AC-02 | یکتایی سفارش | با `client_order_id` تکراری، دقیقاً ۱ سفارش در venue | log خام + شمارش order |
| AC-03 | بدون resend کورکورانه | در ۱۰۰٪ سناریوهای timeout، اولین اقدام read است | failure-injection trace |
| AC-04 | حفاظت اثبات‌شده | ۱۰۰٪ fillها دارای ردیف TP/SL با `stopCategory=STOP_PROFIT_LOSS` و quantity منطبق | snapshot خام JSON |
| AC-05 | پنجرهٔ بی‌حفاظت | ۰ در مسیر attached؛ ≤ T_protect در مسیر fallback | timestamp journal |
| AC-06 | Partial fill | `contracts_filled` هرگز > `quantity`؛ protection quantity = сумма fills | fixture |
| AC-07 | Restart | پس از restart با پوزیشن + algo orders باز: ۰ duplicate، ۰ پوزیشن بدون حفاظت | journal + venue snapshot |
| AC-08 | Metadata زنده | ۱۰۰٪ مقادیر `tick_size/qty_step/min_qty/min_notional/multiplier` از exchangeInfo با timestamp و expiry | cache dump |
| AC-09 | Fail-closed داده | نبود Mark/Funding در مسیرهای نیازمند → ۰ ورود | test |
| AC-10 | عدم نشت secret | ۰ مورد در log/journal/UI/export (اسکن خودکار الگو) | اسکن |
| AC-11 | PnL فقط از fills واقعی | simulation با برچسب جدا؛ ۰ قاطی‌شدن | گزارش |
| AC-12 | صفر P0/P1 باز در مسیر معامله | جدول defect register | sign-off |

---

## خروجی ۶ — ماتریس ردیابی الزامات (RTM)

`PH` = فاز Roadmap · تست‌ها همه در `tests/` جدیدِ additive.

| REQ | الزام | ماژول مسئول | محل | تست | AC |
|---|---|---|---|---|---|
| REQ-TB-001 | `InstrumentSpec` از exchangeInfo زنده با timestamp/expiry | `adapters/toobit/rest.py` → `domain/instruments.py` | فاز ۲ | `test_toobit_metadata.py` | AC-08 |
| REQ-TB-002 | رد `status != TRADING`, `API_TRADE_FORBIDDEN`, `OPEN_FORBIDDEN` | `app/futures/engine.py` | فاز ۲ | `test_toobit_symbol_status.py` | AC-08 |
| REQ-TB-003 | `contract_multiplier` اجباری، بدون default، fail-closed | `domain/instruments.py` | فاز ۱ | `test_instrument_required_fields.py` | AC-08, C-03 |
| REQ-TB-004 | امضای V2: `queryString + rawJsonBody`, compact, byte-exact | `adapters/toobit/signer.py` | فاز ۲ | `test_toobit_signer.py` (goldens) | AC-01 |
| REQ-TB-005 | هدر `X-BB-APIKEY`؛ secret از OS store؛ ۰ نشت | `signer.py`, `runtime/toobit/live.py` | فاز ۲/۶ | `test_secret_redaction.py` | AC-10 |
| REQ-TB-006 | Rate-limit aware، 429 با backoff، auth/permission بدون retry | `rest.py` | فاز ۲ | `test_rate_limit.py` | AC-03 |
| REQ-TB-007 | ورود = marketable LIMIT + IOC + price cap | `app/futures/engine.py` | فاز ۵ | `test_entry_semantics.py` | AC-02 |
| REQ-TB-008 | `newClientOrderId` یکتا per runtime/account | `app/futures/engine.py` | فاز ۵ | `test_client_order_id.py` | AC-02 |
| REQ-TB-009 | فقط `contracts`؛ `valueQuantity` رد | `domain/futures.py` validator | فاز ۱ | `test_order_unit_guard.py` | H-06 |
| REQ-TB-010 | SL/TP attached در سفارش ورود | `app/futures/engine.py` | فاز ۵ | `test_attached_protection.py` | AC-05 |
| REQ-TB-011 | تأیید حفاظت با `stopCategory=STOP_PROFIT_LOSS` | `app/futures/protection.py` | فاز ۵ | `test_protection_verification.py` | AC-04 |
| REQ-TB-012 | `T_protect` + close اضطراری در انقضا | `protection.py` | فاز ۵ | `test_protection_deadline.py` | AC-05 |
| REQ-TB-013 | Sizing per-contract با `f` = USDT/USD (معادلهٔ ۴.۱) | `app/futures/capital.py` | فاز ۴ | `test_futures_sizing.py` (property) | C-04 |
| REQ-TB-014 | بازبینی ریسک پس از گردکردن؛ کاهش یا رد | `capital.py` | فاز ۴ | `test_risk_verify_after_round.py` | C-04 |
| REQ-TB-015 | Margin/available با `margin_safety_mult` | `capital.py` | فاز ۴ | `test_margin_guard.py` | AC-09 |
| REQ-TB-016 | Liquidation: venue-first، estimated فقط هشدار | `capital.py` | فاز ۴ | `test_liquidation_source.py` | M-04 |
| REQ-TB-017 | Funding از API زنده (نرخ + interval) در cost model | `capital.py`, journal | فاز ۴ | `test_funding_live.py` | C-04 |
| REQ-TB-018 | Position side همیشه HEDGE_BOOK؛ config دیگر fail | `domain/futures.py`, `runtime/toobit` | فاز ۱ | `test_position_mode_guard.py` | C-01 |
| REQ-TB-019 | Margin mode read-back + ممنوعیت تغییر با پوزیشن باز | `app/futures/engine.py` | فاز ۵ | `test_margin_mode_readback.py` | H-01 |
| REQ-TB-020 | Close سه‌مرحله‌ای با min(intent, live) و abort روی drift | `app/futures/engine.py` | فاز ۵ | `test_close_race.py` (failure injection) | H-05 |
| REQ-TB-021 | Reconciliation سه‌طرفه: position ↔ regular ↔ algo(STOP + STOP_PROFIT_LOSS) | `app/futures/reconcile.py` | فاز ۵ | `test_three_way_reconcile.py` | AC-07 |
| REQ-TB-022 | Restart: adopt orphan، QUARANTINE روی divergence | `reconcile.py` | فاز ۵ | `test_restart_toobit.py` | AC-07 |
| REQ-TB-023 | Kill switch مستقل Toobit؛ stop-new ≠ flatten | `runtime/toobit/live.py`, CLI | فاز ۶ | `test_kill_switch_toobit.py` | AC-12 |
| REQ-TB-024 | DB/state مستقل per runtime/account | `var/toobit/<alias>/` | فاز ۶ | `test_state_isolation.py` | M-03 |
| REQ-TB-025 | ژورنال: platform/venue/alias/canonical+venue symbol/quality/latency | `adapters/sqlite_store.py` (additive migration) | فاز ۶ | `test_journal_futures_fields.py` | M-03 |
| REQ-TB-026 | Basis با `f` صریح؛ `quality=adjusted|unadjusted` | `app/futures/basis.py` | فاز ۳ | `test_basis_formula.py` | M-01 |
| REQ-TB-027 | Basis فقط advisory؛ ۰ گیت قطعی | `basis.py` | فاز ۳ | `test_basis_advisory_only.py` | M-01 |
| REQ-TB-028 | Data quality: stale/invalid/crossed → block entry | `adapters/toobit/marketdata.py` | فاز ۳ | `test_feed_quality.py` | AC-09 |
| REQ-TB-029 | Feature flags `toobit_enabled`/`trading_enabled` پیش‌فرض false | `runtime/toobit/live.py` | فاز ۶ | `test_feature_flags.py` | AC-12 |
| REQ-TB-030 | پنل: حساب/وضعیت/اضطراری؛ UI مستقیم API صرافی را صدا نزند | `web/app.py` (additive route) | فاز ۶ | `test_panel_toobit.py` | AC-10 |
| REQ-TB-031 | MT5 contract frozen (۶ فایل) | `tests/test_mt5_contract_frozen.py` | فاز ۰ | خودش | AC-01 |
| REQ-TB-032 | Manifest اعداد از واقعیت تولید شوند | `scripts/build_release.py` | فاز ۰ | `test_release_metadata.py` | M-06 |
| REQ-TB-033 | `SHA256SUMS` با LF + self-hash | `scripts/build_release.py` | فاز ۰ | `test_manifest_lf.py` | M-05 |
| REQ-TB-034 | Gate E با سقف‌های عددی و امضای مالک | runbook | فاز ۷ | دستی + شاهد | H-07 |

---

## خروجی ۷ — برنامهٔ آزمون و تحویل

| فاز | محتوا | Gate خروج | شاهد قابل ارائه |
|---|---|---|---|
| 0 | Freeze baseline؛ اجرای کامل suite (اکنون: ۱۰۸۴ pass + ۱ xfail `[SRC]`)؛ اصلاح اعداد manifest؛ hash ۶ فایل ممنوعه | AC-01, AC-11, M-05, M-06 | `pytest` log + `SHA256SUMS` بازتولیدشده |
| 1 | Domain/Ports (futures.py, instruments.py, ports/futures_broker.py) | contract/architecture tests سبز؛ REQ-003,009,018 | diff + تست‌ها |
| 2 | Read-only adapter: signer + exchangeInfo + account + positions + funding + leverage/margin read-back | هیچ write endpoint در کد فعال نیست | dump خام redacted |
| 3 | Market profile + feed quality + basis (advisory) | stale/invalid fail-closed | fixture + گزارش |
| 4 | Futures risk: sizing (۴.۱), fees, funding, margin, liquidation | property/stress: هیچ ورودی‌ای ریسک واقعی > budget تولید نکند | property test report |
| 5 | Execution + protection + reconciliation | AC-02..AC-07 | failure-injection + reconcile log |
| 6 | Runtime جدا + UI + kill switch | AC-10, AC-12 | اسکرین‌شات/JSON |
| 7 | Simulation، سپس Gate E کنترل‌شده | AC-04, AC-05, H-07 | fill خام + snapshot |
| 8 | Handover: runbook، rollback، ADR، manifest | صفر P0/P1 | sign-off |

**قاعدهٔ عبور:** هر Gate فاقد شاهد مکتوب = Gate شکسته. «تست سبز است» بدون artifact پذیرفته نیست.

---

## خروجی ۸ — Change Log (نسبت به سند v1.0)

| # | بند | تغییر | دلیل | اثر فنی |
|---|---|---|---|---|
| 1 | §2 | ادعای «۱۰۸۴ تأیید نشده» → «۱۰۸۴ pass + ۱ xfail تأیید شد» | اجرای واقعی در این ممیزی | ابهام رفع؛ AC واقع‌بینانه شد |
| 2 | §2 | «SHA256SUMS PASS» → PASS مشروط به نرمال‌سازی CRLF | آزمون `sha256sum -c` | CI لینوکس قابل اتکا شد |
| 3 | §2 | عدد فایل تست ۷۷/۷۹/۸۰ → ۷۱ ماژول / ۱۰۸۵ تست | شمارش مستقیم + `SHA256SUMS` | manifest قابل بازتولید |
| 4 | §5 | درخت `shared/runtime_mt5/runtime_toobit` → افزودنی داخل ساختار موجود | تناقض با قاعدهٔ additive خودش | MT5 با ریسک صفر regression |
| 5 | §7 | افزودن `stopCategory` (هر دو مقدار)، `X-BB-APIKEY`، endpointهای v1 margin/leverage، funding | شکاف مستندشده | حفاظت و read-back قابل پیاده‌سازی |
| 6 | §9 | جایگزینی فرمول sizing با معادلهٔ per-contract + `f` + verify | C-04: وابستگی دوری و حذف USDT/USD | ریسک محاسبه‌شده = ریسک واقعی |
| 7 | §10 | پنجرهٔ بی‌حفاظت: attach الزامی، `T_protect` و close اضطراری | C-02 + اصل «PROTECTED فقط با تأیید venue» | حذف ناحیهٔ کور حفاظت |
| 8 | §11 | One-Way حذف؛ HEDGE_BOOK اجباری؛ close سه‌مرحله‌ای | C-01: API فقط LONG/SHORT | جلوگیری از باز شدن پوزیشن معکوس |
| 9 | §14 | Conflict Register از ۱۰ به ۴۱ ردیف (CR-01..10 + C/H/M/L + X) | پوشش کامل یافته‌ها | قابل پیگیری |
| 10 | §16 | AC از «همه تست‌ها pass» به ۱۲ معیار عددی با artifact | غیرقابل‌اندازه‌گیری بودن | پذیرش قابل داوری |
| 11 | §17 | Gate فاز ۰ پیش از Kickoff قرمز است → افزودن اصلاح manifest به فاز ۰ | L-02 | Kickoff با baseline سبز |
| 12 | §15 | افزودن owner/date/blocking-phase به Open Decisions | L-04 | تصمیم‌ها بسته می‌شوند |
| 13 | §19 | افزودن الزام «شاهد خام `contractMultiplier`» به تحویل | C-03 | تنها ریسک باقی‌ماندهٔ غیرقابل‌کدشدن |
| 14 | عمومی | افزودن برچسب شواهد `[SRC]/[DOC]/[OPEN]` به هر ادعا | §18 دستور مبنا: حدس ممنوع | خواننده می‌داند چه چیزی اثبات شده |

**حذف بی‌دلیل:** هیچ. تمام قابلیت‌های سند v1.0 حفظ شده‌اند؛ فقط یکی (One-Way Mode) به‌دلیل عدم‌وجود در API به «پشتیبانی‌نشده در فاز ۱» تنزل یافته — که تنزل واقعیت‌محور است، نه حذف قابلیت.

**افزودهٔ بدون تحلیل اثر:** هیچ. هر افزوده در RTM دارای REQ-id، ماژول، تست و AC است.

---

## بخش پایانی — شرایط باقی‌ماندهٔ پذیرش

برای تبدیل **APPROVED WITH CONDITIONS** به **APPROVED**، سه شرط زیر باید بسته شوند (هر سه شاهد خارجی لازم دارند، نه کد):

| شرط | مالک | شاهد لازم |
|---|---|---|
| **COND-1 (C-03)** | مالک پروژه + تیم | پاسخ خام `exchangeInfo` برای `XAU-SWAP-USDT` + یک سفارش کوچک با محاسبهٔ دستی PnL تأییدشده |
| **COND-2 (C-01)** | تیم | تأیید `accountLeverage` روی حساب واقعی که `marginType` و `leverage` برگرداند و `positionSide` عملاً `LONG|SHORT` باشد |
| **COND-3 (C-02)** | تیم | snapshot از `open-algo-orders` با هر دو `stopCategory` روی یک TP/SL واقعی که در Gate E ثبت شده |

**تصریح مطابق §18 دستور مبنا:** این ممیزی **سند** را ارزیابی کرده است. هیچ کدی برای Toobit نوشته، اجرا یا تست نشده است؛ test suite موجود اجرا شد و نتیجه‌اش گزارش شد. اصلاح سند، پیاده‌سازی کد، و اثبات عملکرد واقعی سه فعالیت مستقل‌اند و تنها اولی در این تحویل انجام شده است.

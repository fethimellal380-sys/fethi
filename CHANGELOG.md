# Gold Sniper - سجل التغييرات

## V202 - Anti-Repaint Build (نسخة مُصلحة)

### 🔴 إصلاحات حرجة

1. **منع Repaint في بيانات MTF**
   - استبدال `request.security(..., close, lookahead_off)` بـ
     `request.security(..., [close[1], close[2]], lookahead_on)`
   - هذا الـ pattern هو القاعدة الذهبية لمنع repaint: نأخذ قيمة الشمعة المُغلقة فعلياً.
   - النتيجة: تنبيهات MTF لن تظهر/تختفي بعد إغلاق الشمعة.

2. **إصلاح منطق Dynamic Reclaim**
   - الكود القديم كان يحفظ آخر منطقة ملموسة فقط في `hit_zone_idx`.
   - في شمعة طويلة تلمس عدة مناطق، كان يفقد منطق Reclaim بعض الحالات.
   - النسخة الجديدة تستخدم مصفوفة `hit_zones` تتتبع كل المناطق الملموسة.

3. **تجميع تنبيهات MTF**
   - الكود القديم كان يطلق تنبيه MTF منفصل لكل منطقة (حتى 10 تنبيهات لشمعة واحدة قوية).
   - النسخة الجديدة تجمع كل الكسور لنفس الفريم/الاتجاه في **تنبيه واحد** بنطاق `min ⟶ max`.

### 🟡 تحسينات متوسطة

4. **لون الـ Box يعتمد على الـ Role**
   - السابق: اللون يعتمد على موقع المنطقة فوق/تحت السعر فقط.
   - الجديد: SUPPORT = أخضر، RESISTANCE = أحمر، NEUTRAL = حسب الموقع (سلوك سابق).

5. **عناوين Z1...Z10 في الإعدادات**
   - الـ checkboxes كانت بدون نص. الآن مرقّمة لسهولة التمييز.

6. **استخدام `timeframe.change()` بدل `ta.change(time(...))`**
   - أوضح وأكثر idiomatic.

7. **Cooldown قابل للتخصيص بين الإشارات**
   - input جديد `cooldown_bars` للتحكم بكم شمعة بين كل إشارة (افتراضي 0 = نفس السلوك السابق).

### 🟢 تنظيف

8. حذف `int(right_p + label_off)` غير الضرورية.
9. حماية إضافية بـ `not na(...)` قبل مقارنات MTF لتفادي تنبيهات وهمية على أول الشارت.

---

## V201 - النسخة الأصلية
- البناء الأولي لمؤشر Gold Sniper مع نظام المناطق والذاكرة و MTF.


---

## V9.9 Hybrid Stable Edition (Surgical Hybrid على V9.9 Final)

> الهدف: تثبيت سلوك V9.9 Final الأساسي (سريع، خفيف، مناسب للذهب realtime/الموبايل) مع
> إصلاح جراحي للنقاط الأربع المُعطلة فقط — دون UDT، دون lifecycle ثقيل، دون institutional engines،
> ودون لمس object pools الأساسية.

### 🔴 إصلاحات حرجة (Surgical)

1. **Break Logic — `final_buy_break` / `final_sell_break`**
   - حُذف `m15_buy_break_ok` و `m15_sell_break_ok` بصيغتهم القديمة (body-only).
   - الصيغة الجديدة:
     ```
     final_buy_break  = m15_h > top and m15_c > bot
     final_sell_break = m15_l < bot and m15_c < top
     ```
   - يستخدم `m15_h` / `m15_l` / `m15_c` مباشرة من security feed بـ `lookahead_off`
     (anti-repaint مضمون، realtime على الـ M15 close).
   - يصلح: BUY mode لا يتفعل، ضعف retracement، flip غير منطقي، break logic ثقيل.

2. **Touch Logic — `buy_touch` / `sell_touch` / `rebuy_touch` / `resell_touch`**
   - لا تعتمد على `break_state` ولا على `mode` المعقد داخل التعريف الهندسي.
   - شروط wick-aware نظيفة:
     ```
     buy_touch    = close[1] > top and low  <= top + offset and close >= top - offset
     sell_touch   = close[1] < bot and high >= bot - offset and close <= bot + offset
     rebuy_touch  = close[1] > bot and low  <= bot + offset and close >= bot - offset
     resell_touch = close[1] < top and high >= top - offset and close <= top + offset
     ```
   - REBUY / RESELL يفعّلان على نفس الشمعة (sequential ifs، no else).
   - يصلح: BUY/REBUY لا يعملان، عدم التناظر، ضعف القنص، same-candle interaction، wick sniper behavior.

3. **Multi-Break Consolidation — تنبيه واحد فقط لكل دورة M15**
   - `alert()` للكسور انتُقل من داخل `f_zone()` إلى طبقة aggregation عالمية.
   - عند `new_buy_breaks > 1` → رسالة واحدة `🟢⏫ M15 MULTI BREAK [lowest_bot ⟶ highest_top]`.
   - عند `new_sell_breaks > 1` → رسالة واحدة `🔴⏬ M15 MULTI BREAK [highest_top ⟶ lowest_bot]`.
   - عند كسر واحد بالضبط → `🟢⬆️ / 🔴⬇️ M15 BREAK [bot ⟶ top]` / `[top ⟶ bot]`.
   - لا individual spam لكسور متعددة في نفس الدورة.

4. **Realtime Overlay Redraw Stabilization**
   - الـ overlay objects تُعاد محاذاتها كل bar طالما `ov_act && show_trade` مستمران.
   - الـ hide branch يفعَّل **حصرياً** عند الانتقال (active→inactive أو show_trade=off).
   - النتيجة: stable overlays، no flickering، no random disappear، no mid-bar shifting.
   - الـ object pool نفسه (4 TP boxes + SL + entry + glow + 4 lines + 5 labels) لم يُمَس.

### 🟡 تنسيق التنبيهات (Institutional Pills)

5. **Alert Format Cleanup**
   - حُذف: 🚀 / 💥 / `CONFIRMATION` / `TOUCH` / `@` / `Z1` / `Z2`.
   - المعتمد فقط:
     - `🟢 BUY 3000` / `🟩 REBUY 2998`
     - `🔴 SELL 3200` / `🟥 RESELL 3203`
     - `🟢⬆️ M15 BREAK [2990 ⟶ 3000]` / `🔴⬇️ M15 BREAK [3200 ⟶ 3190]`
     - `🟢⏫ M15 MULTI BREAK [2890 ⟶ 3200]` / `🔴⏬ M15 MULTI BREAK [3200 ⟶ 2890]`
     - `🟢⬆️ H1 BREAK` / `🔴⬇️ H1 BREAK` (H1/H4 informational فقط، لا يغيران mode/rotation)
   - كل حدث = تنبيه واحد، latches تمنع التكرار داخل نفس الـ chart bar.

### 🎨 Palette (Institutional Gold)

6. **ألوان مُحدَّثة لطابع ذهب institutional خفيف**
   ```
   buy_c       = color.new(#22C55E, 82)
   sell_c      = color.new(#EF4444, 82)
   tp_col      = color.new(#3B82F6, 84)
   tp_border   = color.new(#2563EB, 25)
   sl_col      = color.new(#EF4444, 88)
   sl_border   = color.new(#DC2626, 20)
   entry_col   = color.new(#FACC15, 10)   // line only
   tp_line_col = color.new(#2563EB, 15)
   sl_line_col = color.new(#DC2626, 15)
   ```
   - جميع ملء البوكسات (TP/SL/entry box) ≥ 70 transparency للحفاظ على خفة الذهب.
   - `entry_col` بـ transparency 10 يستخدم فقط على الـ line الرفيعة، أما الريبون فيستخدم `box_entry` بـ 70.
   - `tp_border` / `sl_border` يحدّان الحواف الخارجية لـ `tp2_out_bx` و `sl_bx` لإعطاء عمق بصري بدون ثقل.

### 🟢 ما لم يتغير (محفوظ كما هو)

- البنية العامة لمحرك `f_zone()` و varip state machine.
- `f_unlock_buy` / `f_unlock_sell` (body-confirmed unlock).
- `f_tp2_buy` / `f_tp2_sell` (opposite-zone targeting + min-gap).
- Object pool: 4 TP boxes + SL + entry + glow + 4 lines + 5 labels per zone.
- Global lock `_lock_arr` / `_zone_arr` و failsafe deadlock protection.
- Mode rotation محصور في M15 فقط، H1/H4 confirmation layers فقط.
- Min overlay lifetime guard (`min_live_bars = 2`).

### 📦 الملف
- `Gold_Sniper_V9.9_Hybrid_Stable.pine`
- `max_boxes_count = max_labels_count = max_lines_count = 100` (headroom محفوظ).


### V9.9 Hybrid Stable — Refinement Pass

تعديلان دقيقان فوق نسخة Hybrid Stable الأولية، لا تمسّ الـ architecture:

1. **REBUY / RESELL gates على الحافة البعيدة (FAR edge)**
   - تصحيح `close[1]` ليُختبَر ضد الحافة البعيدة من الزون لا القريبة:
     ```
     rebuy_touch  = close[1] > top and low  <= bot + offset and close >= bot - offset
     resell_touch = close[1] < bot and high >= top - offset and close <= top + offset
     ```
   - السابق كان `close[1] > bot` / `close[1] < top` — كان يسمح بـ retrace عشوائي عند تذبذب داخل الزون.
   - النتيجة: ✅ الاتجاه الأصلي محفوظ، ✅ التعزيز الحقيقي، ✅ same-candle reinforcement، ✅ wick interaction نظيف، بدون retrace عشوائي.

2. **Hide branch transition-only**
   - حُذف الشرط الثالث `(ov_act and not show_trade)` من الـ hide branch — كان يطلق `box.set_*` كل tick أثناء حالة "hidden مستمرة".
   - الآن ينفَّذ حصراً عند:
     ```
     (ov_act[1] and not ov_act) or (show_trade[1] and not show_trade)
     ```
   - أي على لحظة الـ transition فقط (active→inactive أو show_trade-on→off).
   - النتيجة: ✅ no redundant set_* calls، ✅ no flickering، ✅ realtime stable overlays، خفّة إضافية على المحرك.

#### ما لم يتغير (ثبات معتمد)
- Break logic: `final_buy_break` / `final_sell_break` كما هي.
- BUY/SELL touch: كما هي.
- Object pool: `var box`/`var line`/`var label` فقط، **لا** `box.delete()`/`line.delete()`/`label.delete()` في أي مكان.
- لا `alert()` للكسور داخل `f_zone` — الـ aggregation العالمي وحده هو من يبعث.
- Multi-break consolidation: تنبيه واحد لكل دورة M15.
- Palette: `tp_col`/`sl_col`/`entry_col`/`tp_line_col`/`sl_line_col`/`tp_border`/`sl_border` بقيم institutional gold، transparency ≥ 70 على البوكسات.
- Alert format: `🟢 BUY` / `🟩 REBUY` / `🔴 SELL` / `🟥 RESELL` / `🟢⬆️ M15 BREAK` / `🔴⬇️ M15 BREAK` / `🟢⏫ MULTI` / `🔴⏬ MULTI`. بدون 🚀/💥/CONFIRMATION/TOUCH/@/Z1.


### V9.9 Hybrid Stable — Aggregator Naming Pass

تعديل cosmetic فقط على طبقة multi-break aggregator، لا يمسّ السلوك أو الـ architecture:

- إعادة تسمية متغيرات الـ break aggregator لتطابق spec الاستراتيجية:
  - `buy_break_lo`  → `multi_buy_from`  (lowest bot among breaking buy zones)
  - `buy_break_hi`  → `multi_buy_to`    (highest top among breaking buy zones)
  - `sell_break_hi` → `multi_sell_from` (highest top among breaking sell zones)
  - `sell_break_lo` → `multi_sell_to`   (lowest bot among breaking sell zones)
- أضيفت aliases للحالة الفردية: `first_buy_from` / `first_buy_to` / `first_sell_from` / `first_sell_to` (تعكس الزون الفردي عندما `count == 1`).
- الـ dispatcher أعيد ترتيبه: single break أولاً، multi break ثانياً، باستخدام `if` (لا `else if`) — `count == 1` و `count > 1` متعاكستان منطقياً فقط واحد منهما يطلق.
- النتيجة: الكود يتطابق نصياً مع spec المُعتمد، لا تغيير في الـ runtime behavior، لا تغيير في رسائل التنبيهات.

#### تأكيدات نهائية (لم تتغير، للتوثيق)
- ✅ Object pool: كل `box.new` / `line.new` / `label.new` داخل `var` declarations فقط (init مرة واحدة على أول bar). صفر `delete()`. صفر `*.new` داخل realtime loop.
- ✅ Hide branch: ينفَّذ حصراً عند الـ transition (`ov_act[1] and not ov_act` أو `show_trade[1] and not show_trade`). لا `set_bgcolor(na)` ولا `set_color(na)` كل tick.
- ✅ Break alerts: لا `alert()` للكسر داخل `f_zone` — `buy_break_evt` / `sell_break_evt` ترجع للـ aggregator العالمي فقط.
- ✅ Palette: `#22C55E,82` / `#EF4444,82` / `#3B82F6,84` / `#EF4444,88` / `#FACC15,10` (line) / `#2563EB,15` / `#DC2626,15` / `#2563EB,25` / `#DC2626,20`.
- ✅ Touch logic: BUY/SELL على الحافة القريبة، REBUY/RESELL على الحافة البعيدة (`close[1] > top` للـ rebuy، `close[1] < bot` للـ resell).
- ✅ Break logic: `m15_h > top and m15_c > bot` / `m15_l < bot and m15_c < top` فقط، لا body-only، لا break_state arrays.
- ❌ بدون UDT, lifecycle engine, institutional rotations, heavy anti-spam, architecture rewrite.


### V9.9 Hybrid Stable — Six Surgical Hardening Patches

ست تعديلات مستهدفة على نسخة Hybrid Stable. لا تغيير في الـ strategy logic أو الـ break logic أو الـ TP/SL math أو الـ unlock logic أو الـ rendering أو الـ object pools.

1. **إزالة Reverse Direction Flip blocks**
   - حُذف `if ov_act and ov_dir == "SELL"` من BUY activation و `if ov_act and ov_dir == "BUY"` من SELL activation.
   - `ov_act := false` محفوظ في unlock logic (السطور 346, 353) و overlay removal (السطر 437) — لم يُمَس.
   - النتيجة: لا cleanup عشوائي للـ overlay المعاكس عند activation جديد، الـ unlock الطبيعي وحده هو من يدير الـ cycle.

2. **M15 / H1 / H4 BREAK alerts → `alert.freq_once_per_bar_close`**
   - جميع الـ 8 break alerts (4 M15 single+multi، 2 H1، 2 H4) محوّلة من `freq_all` → `freq_once_per_bar_close`.
   - يمنع التكرار realtime داخل نفس الـ bar.

3. **Entry alerts → `alert.freq_once_per_bar`**
   - 4 alerts (BUY / REBUY / SELL / RESELL) محوّلة من `freq_all` → `freq_once_per_bar`.
   - يسمح بإطلاق فوري داخل الـ bar مع منع التكرار.

4. **`has_prev` guard على touch logic**
   - أُضيف `bool has_prev = not na(close[1])` قبل touch definitions.
   - الأربعة touches الآن تبدأ بـ `has_prev and ...` لمنع false positives على أول bar أو بعد gaps في الداتا.

5. **TP2 edge-case clamp**
   - بعد `f_tp2_buy(top)`: إذا `math.abs(tp2_calc - top) < 0.01` → `tp2_calc := top + 30`.
   - بعد `f_tp2_sell(bot)`: إذا `math.abs(tp2_calc - bot) < 0.01` → `tp2_calc := bot - 30`.
   - يحمي من حالات tp2 = entry (zero-RR) عند زون أمامي بنفس مستوى الزون الحالي.

6. **Deadlock recovery — restructured**
   - تقسيم منطق `lock_idle_bars` إلى مرحلتين:
     - مرحلة الزيادة/الإعادة: `lock_idle_bars += 1` إذا (locked & no overlay) و إلا `:= 0`.
     - مرحلة الإفراج: `if lock_idle_bars >= 2` → release + reset.
   - النتيجة: الـ release يفحص دائماً (حتى بعد إعادة الـ counter لـ 0 مباشرة) — أكثر متانة ضد race conditions.

#### تأكيد المنطق المُجمَّد (لم يتغير)
- Break logic: `final_buy_break = m15_h > top and m15_c > bot` + `final_sell_break = m15_l < bot and m15_c < top` — كما هي.
- Touch logic: BUY/SELL على الحافة القريبة، REBUY/RESELL على الحافة البعيدة — كما هي + `has_prev` guard فقط.
- Unlock logic، Overlay removal، Rendering system، Object pools، Architecture، Naming — صفر تغيير.


### V9.9 Hybrid Stable — H1/H4 Latch Reset Fix

تعديل واحد جراحي على آلية إعادة تعيين latches الكسر للـ H1/H4. باقي توجيهات الـ Patch Order كانت **no-ops** لأنها مطبَّقة بالفعل من الجولات السابقة.

#### تحليل توجيهات Patch Order

| التوجيه | الحالة |
|---|---|
| 1. Fix H1/H4 alert spam (latch reset gating) | ⚠️ **طُبِّق الآن** |
| 2. `has_prev` close[1] protection | ✅ مطبَّق سابقاً (السطر 374) |
| 3. TP2 failsafe clamps | ✅ مطبَّق سابقاً (السطور 389, 415) |
| 4. Replace `alert.freq_all` → `alert.freq_once_per_bar` | ✅ no-op (صفر `freq_all` في الملف) |

#### تفاصيل التعديل (Patch 1)

**قبل:**
```pine
if barstate.isnew
    buy_alert_sent    := false
    sell_alert_sent   := false
    rebuy_alert_sent  := false
    resell_alert_sent := false
    buy_break_sent    := false
    sell_break_sent   := false
    h1_break_sent     := false   ← يُعاد تعيينها كل chart bar
    h4_break_sent     := false   ← يُعاد تعيينها كل chart bar
```

**بعد:**
```pine
if barstate.isnew
    buy_alert_sent    := false
    sell_alert_sent   := false
    rebuy_alert_sent  := false
    resell_alert_sent := false
    buy_break_sent    := false
    sell_break_sent   := false

if h1_just_closed
    h1_break_sent := false       ← يُعاد تعيينها فقط عند H1 close
if h4_just_closed
    h4_break_sent := false       ← يُعاد تعيينها فقط عند H4 close
```

#### لماذا هذا الإصلاح ضروري

الـ H1/H4 BREAK alerts تستخدم `alert.freq_once_per_bar_close` (من patch سابق) الذي يتطلّب أن يُستدعى `alert()` على tick الإغلاق للـ chart bar. مع النمط القديم:

- Tick 1 من chart bar: `barstate.isnew` true → reset latch → الشرط يطلق → `latch := true` → alert() يُستدعى ✓
- Tick 2-N من نفس chart bar: latch=true → الشرط يُتجاوز → alert() **لا** يُستدعى
- Closing tick: latch=true → الشرط يُتجاوز → alert() **لا** يُستدعى ✗
- النتيجة: `freq_once_per_bar_close` لا يصدر التنبيه (لأنه يتطلّب call على closing tick)

مع النمط الجديد:
- كل tick من chart bar حيث `h1_just_closed=true`: latch يُعاد تعيينه → الشرط يطلق → alert() يُستدعى
- closing tick أيضاً يستدعي alert() → `freq_once_per_bar_close` يصدر التنبيه مرة واحدة ✓

#### المنطق المُجمَّد (لم يتغير)
- ✅ `final_buy_break` / `final_sell_break` — صفر تغيير
- ✅ `buy_touch` / `sell_touch` / `rebuy_touch` / `resell_touch` — صفر تغيير
- ✅ Unlock logic, mode engine, TP engine, rendering, object pools — صفر تغيير
- ✅ M15 break latches (`buy_break_sent`, `sell_break_sent`) — صفر تغيير (التوجيه خصّ H1/H4 فقط)


### V9.9 Hybrid Stable — Confirmed-Bar Hardening Pass

ثمانية إصلاحات جراحية تستهدف ثبات الـ realtime: تأخير اتخاذ القرارات (break / unlock) إلى لحظة `barstate.isconfirmed` (إغلاق الـ chart bar فعلياً)، وإضافة تنبيهات مرئية للكسر، وتثبيت الـ overlay ضد flickers الـ ticks. **لا تغيير في الاستراتيجية أو شروط الدخول أو منطق TP/SL.**

#### ملاحظة قبل التطبيق
تم rollback لتعديل غير مُلتَزَم من الجولة السابقة (`buy_flip`/`sell_flip` restructure مع direct alerts) لأن هذا الـ Patch Order يفترض البنية الأصلية (وجود `buy_break_evt := true` وجملة `if final_buy_break and ... not buy_break_sent and cooldown_ok`).

#### تفاصيل الإصلاحات

**FIX 1 — M15 BREAK confirmation**
- `if m15_just_closed` → `if m15_just_closed and barstate.isconfirmed`
- اكتشاف الكسر يحدث فقط على tick إغلاق الـ chart bar، لا على ticks intrabar.

**FIX 2 — SELL BREAK confirmation**
- لا تغيير فعلي (الجزء الـ `else if` الداخلي يرث الحارس من FIX 1's parent guard). بقي شرطه كما هو في الـ spec.

**FIX 3 — H1 spam fix**
- `if use_h1_break and h1_just_closed and not h1_break_sent` → `... and barstate.isconfirmed and not h1_break_sent`
- يمنع التحقّق المتكرر داخل الـ chart bar حيث H1 just closed.

**FIX 4 — H4 spam fix**
- نفس FIX 3 لكن لـ H4.

**FIX 5 — BUY unlock realtime fix**
- `if can_unlock ...` → `if barstate.isconfirmed and can_unlock ...`
- يمنع تنفيذ الـ unlock أثناء ticks مفتوحة، فلا تختفي الـ overlays فجأة في realtime ثم تعود.

**FIX 6 — SELL unlock realtime fix**
- نفس FIX 5 لـ sell unlock.

**FIX 7 — Stable overlay draw**
- `if ov_act and show_trade` → `if show_trade and (ov_act or b1_d or b2_d or s1_d or s2_d)`
- الـ overlay يبقى مرئياً طالما **أيٌّ** من state flags مفعّل، فلا يختفي بسبب flips مؤقتة لـ `ov_act` في realtime.
- ⚠️ أثر جانبي: الـ overlay قد يظل مرئياً بعد ضربة TP2/SL حتى يحدث unlock فعلي (لأن `b1_d`/`s1_d` لا تُمسح إلا عند unlock، بينما `ov_act` يُمسح عند TP2/SL).

**FIX 8 — Break alert visibility (per-zone)**
- إضافة `alert("🟢 BREAK BUY " + f_num(top), alert.freq_once_per_bar_close)` بعد `buy_break_evt := true`.
- إضافة `alert("🔴 BREAK SELL " + f_num(bot), alert.freq_once_per_bar_close)` بعد `sell_break_evt := true`.
- ⚠️ الـ global aggregator alerts (`🟢⬆️ M15 BREAK [bot ⟶ top]`، `🟢⏫ M15 MULTI BREAK [...]`) **تبقى تعمل** لأن `*_break_evt := true` ما زال يُعيَّن. النتيجة: للكسر الواحد قد يصدر تنبيهان (per-zone + global). إذا أردت إلغاء أحدهما، أخبرني.

#### المنطق المُجمَّد (لم يتغير)
- `final_buy_break` / `final_sell_break` — صفر تغيير (السطور 297-298).
- `buy_touch` / `sell_touch` / `rebuy_touch` / `resell_touch` — صفر تغيير (السطور 382-385).
- شروط BUY/SELL/REBUY/RESELL activation — صفر تغيير.
- TP/SL math، التعزيز، الرسم المؤسسي، object pools، architecture، palette — صفر تغيير.


### V9.9 Hybrid Stable — Final Critical Completion Patch

تعديل جراحي على frequency لـ REBUY/RESELL alerts فقط، مطابقاً للـ spec المُرسَل:

#### تحليل الـ patch

| البند | الحالة قبل | المطلوب | الإجراء |
|---|---|---|---|
| BUY state setting (ov_act/ov_dir/e_bar/tp1/tp2/sl/lock) بعد `b1_d := true` | ✅ مطبَّق سابقاً | مطلوب | لا تغيير |
| BUY alert ("complete as is") | `freq_once_per_bar` | unchanged | لا تغيير |
| SELL state setting بعد `s1_d := true` | ✅ مطبَّق سابقاً | مطلوب | لا تغيير |
| SELL alert ("complete as is") | `freq_once_per_bar` | unchanged | لا تغيير |
| **REBUY alert frequency** | `freq_once_per_bar` | `freq_once_per_bar_close` | ⚠️ **تم تعديله** |
| **RESELL alert frequency** | `freq_once_per_bar` | `freq_once_per_bar_close` | ⚠️ **تم تعديله** |
| REBUY/RESELL: لا reset | ✅ لا يوجد | لا reset | لا تغيير |

#### الـ asymmetry المقصودة

السلوك بعد التعديل:
- **BUY / SELL** (entries): `alert.freq_once_per_bar` — تنبيه فوري على أول tick داخل الـ bar.
- **REBUY / RESELL** (reinforcements): `alert.freq_once_per_bar_close` — تنبيه على إغلاق الـ bar للتأكيد.

التصميم: الدخول الأولي يُطلَق بسرعة، أما التعزيز فيُؤكَّد على إغلاق الـ bar.

#### المنطق المُجمَّد (تأكيد نهائي — صفر تغيير)
- ✅ `final_buy_break` / `final_sell_break` (السطور 297-298)
- ✅ touch geometry (السطور 382-385)
- ✅ كل state setting (b1_d/b2_d/s1_d/s2_d, ov_act, ov_dir, e_bar, tp1_v, tp2_v, sl_v, lock acquisition)
- ✅ TP/SL math، التعزيز، الاتجاه، الرسم، object pools، architecture، palette


### V9.9 Hybrid Stable — SAFE FIX PACK (Stability Only)

تعديلان دقيقان (2 fixes فقط) بدون مساس بأي شيء يخص الاستراتيجية أو الـ entry/exit/TP/SL/mode/touch/break/unlock.

#### SAFE FIX #1 — Unify BUY/SELL alerts with close-confirmed architecture

كل تنبيهات الـ entries (BUY/SELL/REBUY/RESELL) أصبحت موحَّدة على `alert.freq_once_per_bar_close`:

```diff
- alert("🟢 BUY "  + f_num(top), alert.freq_once_per_bar)
+ alert("🟢 BUY "  + f_num(top), alert.freq_once_per_bar_close)

- alert("🔴 SELL " + f_num(bot), alert.freq_once_per_bar)
+ alert("🔴 SELL " + f_num(bot), alert.freq_once_per_bar_close)
```

(REBUY/RESELL كانتا بالفعل على `freq_once_per_bar_close` من الـ Final Critical Completion Patch السابق.)

النتيجة: الـ asymmetry السابقة بين BUY/SELL و REBUY/RESELL أُلغيت. كل الأربعة يلتزمن بنفس آلية تأكيد إغلاق الـ bar.

#### SAFE FIX #2 — Stable overlay visibility guard

أُضيف متغيّر `overlay_visible` جديد يوحّد منطق ظهور الـ overlay مع شرط ملكية الـ zone للـ global lock:

```pine
bool overlay_visible =
     ov_act or
     ((b1_d or b2_d or s1_d or s2_d) and array.get(_zone_arr, 0) == name)

if show_trade and overlay_visible
    ...
```

السطر 471 السابق كان `if show_trade and (ov_act or b1_d or b2_d or s1_d or s2_d)`. الفرق المفصلي: الفرع الثاني الآن مشروط بـ `array.get(_zone_arr, 0) == name`.

**ما يصلحه هذا الـ guard:**

1. **Realtime stability** ✅ — إذا flicker لـ `ov_act` (false مؤقتاً ثم true)، فإن `b1_d`/`s1_d` المُحقَّقة + ملكية الـ zone للـ lock تبقي الـ overlay مرسوماً، فلا flicker.

2. **اختفاء نظيف بعد TP2/SL** ✅ — عند ضربة TP2/SL، يُمسح `_zone_arr` (lock release). على الـ tick التالي:
   - `ov_act = false`
   - `(b1_d=true) and (_zone_arr == name)` → `false` (لأن `_zone_arr=""`)
   - `overlay_visible = false` → الـ overlay يختفي ✅
   
   هذا يُصلح الـ side effect الذي أشرتُ إليه في FIX 7 من الـ Confirmed-Bar Hardening Pass السابق (overlay كان يبقى بعد TP2/SL).

3. **عزل بين الـ zones** ✅ — كل zone ترسم فقط عندما تكون هي مالكة الـ lock. لا تتداخل overlays من zones مختلفة.

#### المنطق المُجمَّد (تأكيد نهائي — صفر تغيير)
- ✅ `final_buy_break` / `final_sell_break` (السطور 297-298)
- ✅ touch geometry — buy/sell/rebuy/resell touches (السطور 382-385)
- ✅ Mode flow، unlock engine، TP/SL math، التعزيز، الاتجاه
- ✅ Object pools، rendering pipeline، architecture، palette
- ✅ Break conditions، entry/exit gates


### V9.9 Hybrid Stable — Visual Cleanup + Reinforcement Label

أربعة تعديلات بصرية على الـ overlay rendering. لا تغيير في الاستراتيجية، break logic، touch logic، entry/exit، TP/SL، أو unlock.

#### 1) إزالة اللون الأصفر كلياً
- `entry_col`: `(#FACC15, 10)` → `(#FACC15, 100)` (شفاف تماماً → الـ entry line مخفي)
- `box_entry`: `(#FACC15, 70)` → `(#FACC15, 100)` (الـ entry ribbon مخفي)
- `box_eglow`: `(#FACC15, 85)` → `(#FACC15, 100)` (الـ entry glow مخفي)

#### 2) إزالة السهم الأصفر ▲/▼ كلياً
- `LBL_MARK_C`: `(#FACC15, 15)` → `(#FACC15, 100)` (background شفاف)
- `label.set_text(lbl_mark, ...)` على السهم → `""` فارغ دائماً
- حُذف `label.set_style(lbl_mark, ...)` (لا حاجة له مع نص فارغ)

#### 3) تخفيف الأزرق + إلغاء التدرّج (gradient flat)
- `tp_col`: `(#3B82F6, 84)` → `(#3B82F6, 90)` — أزرق أفتح
- أُضيف ثابت `box_hidden = color.new(color.gray, 100)` للـ inner layers
- `tp2_mid_bx` bgcolor → `box_hidden` (الطبقة الوسطى مخفية)
- `tp2_in_bx` bgcolor → `box_hidden` (الطبقة الداخلية مخفية)
- النتيجة: TP2 box واحد فقط (`tp2_out_bx`) بلون موحَّد، بدون تدرّج layered

#### 4) إضافة رقم تعزيز (Reinforcement Label) على الرسم
- متغيّر جديد: `var label lbl_r` في الـ object pool لكل zone
- ثابت لون جديد: `LBL_R_C = color.new(#0F1116, 8)` — خلفية داكنة pill
- في redraw block:
  - عند `b2_d=true` (REBUY مفعَّل): يظهر `🟩 REBUY <bot_price>` على مستوى bot
  - عند `s2_d=true` (RESELL مفعَّل): يظهر `🟥 RESELL <top_price>` على مستوى top
  - وإلا: نص فارغ (مخفي)
- في hide branch: `lbl_r` يُخفى مع باقي الـ labels عند التحوُّل
- النتيجة: المستخدم يرى رقم/سعر التعزيز على الشارت لحظة تفعيله

#### المنطق المُجمَّد (تأكيد نهائي — صفر تغيير)
- ✅ `final_buy_break` / `final_sell_break` (السطور 297-298)
- ✅ touch geometry (buy/sell/rebuy/resell) — السطور 382-385
- ✅ شروط BUY/SELL/REBUY/RESELL activation
- ✅ TP/SL math، التعزيز، الاتجاه، break logic، unlock engine
- ✅ Object pools structure، architecture، naming

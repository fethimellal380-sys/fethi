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

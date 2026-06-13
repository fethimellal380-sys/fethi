# Gold Sniper - سجل التغييرات

## V11 - Core-Fixed Institutional Edition

ملف `Gold_Sniper_V11.pine` = V10 (Cloud-Fixed) + 8 إصلاحات نواة + 5 توصيات من المراجعة المعمارية المؤسسية. كل إصلاح موسوم بـ `// V11-FIX#N` أو `// V11.1-GUARD` في الكود لسهولة التتبّع.

### 🛡️ توصيات المراجعة المعمارية (V11.1 hardening — مدمجة في نفس الملف)

1. **Timeframe Guard (runtime.error)**
   - حارس على `barstate.isfirst` يمنع التشغيل على H1 أو أعلى.
   - السبب: `request.security("15", ..., lookahead_off)` على H1 يعيد فقط آخر شمعة M15 من كل 4 → فقدان 75% من بيانات الكسر.
   - الرسالة: `⛔ Gold Sniper V11 مصمَّم للتشغيل على M15 أو أقل`.

2. **سقف ناعم على Arrows Loop (Soft Cap)**
   - `math.min(consecutive_count, 30)` في حلقة توليد الـ ⏫/⏬ فقط (لا يؤثر على منطق العدّاد نفسه).
   - زيادة عن 30 تظهر كـ `⏫⏫⏫...⏫+15` (مثلاً عند العدّ 45).
   - يحمي من تجاوز حد TradingView لطول رسالة التنبيه (~4096 حرف) عند re-breaks متكررة لنفس المنطقة.

3. **توثيق `last_*` بـ math.max/min**
   - block توثيق مؤسسي في رأس الملف يوضح أن `last_buy_top` = "ذروة الترند" (math.max عبر كامل الترند) لا "آخر زمنياً".

4. **توثيق Alert Hybrid Model**
   - block يوضح أن BREAK alerts = bar-close-confirmed (non-repaint)، أما TOUCH/REBUY/RESELL/SL/TP/TIMESTOP = intra-bar live.
   - تنبيه صريح للمستخدمين الذين يبنون backtesting خارجي.

5. **توثيق Trade vs Zone State Independence**
   - block يؤكد أن صفقة مفتوحة لا تُغلَق على تغيّر حالة المنطقة، فقط على TP/SL/TIME-STOP.
   - منع سوء الفهم بين Strategy DNA والسلوك المُتوقَّع.

### 🔴 إصلاحات حرجة (8 أصلية من تحليل V10)

### 🔴 إصلاحات حرجة

1. **FIX#1 — إزالة لاج 15 دقيقة في بيانات M15**
   - V10: `request.security("15", [open[1], close[1], high[1], low[1]], lookahead=barmerge.lookahead_off)` → دمج `[1]` مع `lookahead_off` يعطي شمعة قبل الأخيرة المُغلقة (لاج إضافي 15د).
   - V11: `request.security("15", [close, high, low], lookahead=barmerge.lookahead_off)` → آخر شمعة M15 مُغلقة فعلاً، بدون repaint وبدون لاج.

2. **FIX#3 — تصفير العدّاد المتراكم لم يعد عدوانياً**
   - V10: عودة أيّ منطقة واحدة لداخلها كانت تمحو `consecutive_buy_count` و `consecutive_sell_count` لكل المناطق.
   - V11: التصفير يحدث **فقط** عند كسر معاكس فعلي في محرك الكسور المتتالية، لا عند العودة لـ "NONE".
   - بونص: عند الانعكاس، تُمحى ذاكرة الترند المعاكس بالكامل (`first_*`/`last_*` تعود `na`) لمنع تنبيهات بنطاقات قديمة.

3. **FIX#4 — Time-Stop لم يعد صامتاً + منفصل عن الامتداد البصري**
   - V10: `extend_right` (الافتراضي 35) كان يُستخدم للأمرين معاً: امتداد البوكس **والـ** time-stop. النتيجة: صفقة سوينج تُغلق بعد ~8.7 ساعة بدون تنبيه.
   - V11: input جديد `time_stop_bars` (افتراضي 96 = 24 ساعة على M15) منفصل عن `extend_right`، **مع تنبيه صريح** `⏰ TIME-STOP BUY/SELL Z#` عند الإغلاق.

4. **FIX#6 — ترتيب فحص TP/SL تحفّظي + تنبيهات إغلاق**
   - V10: `if high >= tp2 or low <= sl` — في شمعة متذبذبة تضرب الاثنين، تُحسب ربحاً.
   - V11: SL يُفحص أولاً (افتراض ضرب SL قبل TP)، ثم TP. تنبيهات صريحة:
     - `🛑 STOP LOSS BUY/SELL Z# @ price`
     - `🎯 TAKE PROFIT BUY/SELL Z# @ price`
   - النتيجة: backtesting أكثر صدقاً + تتبّع نتائج أوضح.

### 🟡 تحسينات متوسطة

5. **FIX#2 — حذف dead code وشروط زائدة**
   - `m15_open_r` كان مجلوب من `request.security` لكن لا يُستخدم في أي شرط. حُذف.
   - الشرط `m15_high_r > top_v` متضمَّن رياضياً في `m15_close_r > top_v` (لأن high ≥ close). حُذف.
   - الشرط `m15_low_r < bot_v` متضمَّن في `m15_close_r < bot_v`. حُذف.
   - شرط `and not valid_sell_break` على `final_buy_break` كان مستحيلاً منطقياً (close لا يكون فوق top وتحت bot معاً). حُذف.

6. **FIX#5 — حماية `bar_index` السالب**
   - `box.set_left(bx, bar_index - 500)` → `box.set_left(bx, math.max(0, bar_index - 500))`.
   - يمنع سلوك بصري غير محدّد عندما `bar_index < 500` في بداية الشارت.

7. **FIX#7 — `last_buy_top`/`last_sell_bot` بأقصى/أدنى قيمة عبر الترند**
   - V10: تُعاد كتابتها بكل كسر جديد، حتى لو القيمة الجديدة أقل (في الترند الصاعد).
   - V11: `math.max(last_buy_top, top_v)` و `math.min(last_sell_bot, bot_v)` لضمان أن النطاق المُبلَّغ يمثّل الترند بالكامل.
   - `first_buy_bot`/`first_sell_top` تُحفظ فقط عند بداية ترند جديد (`count == 0`).

8. **FIX#8 — السحابة تحترم `show_dist`**
   - السحابة المؤسسية تختفي بصرياً إذا كان السعر بعيداً عن منتصف نطاقها بمقدار > `show_dist` (متّسق مع فلسفة V10 الأصلية لإخفاء المناطق البعيدة).

### ⚠️ ملاحظات

- V11 لم يحلّ محل V10 — كلاهما يعيش جنباً إلى جنب في الريبو (`Gold_Sniper_V10.pine` و `Gold_Sniper_V11.pine`).
- Cloud Engine في V11 = نفس الـ Hybrid Stable Pool في V10، مع إضافة احترام `show_dist`.
- لمستخدمي V10: الترقية إلى V11 لا تتطلب تغييرات في الإعدادات الحالية، فقط نسخ الكود الجديد إلى Pine Editor.

---

## V10 - Original Institutional Edition (Cloud-Fixed)

ملف جديد `Gold_Sniper_V10.pine` بمعمارية مختلفة عن V202: تركيز على **دورة حياة الصفقة** على M15 (TP1/TP2 ديناميكي + SL + REBUY/RESELL) بدلاً من تنبيهات MTF متعددة.

### ☁️ Cloud Engine — نسخة هجينة (Stable Pool + User Inputs + LastBar Guard)

النسخة النهائية تجمع أفضل ما في النمطين:

1. **Object Pool المستقر** (إنشاء مرة واحدة في `barstate.isfirst`)
   - 4 خطوط + 2 linefills طوال عمر السكربت — لا تسريب، لا حذف، لا إعادة إنشاء.
   - مطابق لنمط V10 الأصلي مع `boxes` و `entry_lines`.

2. **inputs المستخدم محفوظة**
   - `show_clouds` toggle.
   - `buy_cloud_col` و `sell_cloud_col` كـ inputs قابلة للتعديل.
   - اللون يُسند في الـ render (عبر `linefill.set_color`)، ليس وقت الإنشاء → **يقبل التغيير المباشر من إعدادات المؤشر**.

3. **`barstate.islast` guard**
   - الرسم على آخر بار فقط، متّسق مع `should_redraw` في V10.
   - يمنع آلاف استدعاءات `set_xy1/2` على البارات التاريخية.

4. **`extend=extend.right` على إنشاء الخطوط**
   - الخط يمتد للأمام تلقائياً، لا حاجة لتحديث `xy2` على البار المستقبلي.

5. **إخفاء كنسي عبر `linefill.set_color(... color(na))`**
   - بدلاً من `set_xy1(na, na)` (نمط ضمني غير موثّق).

6. **حماية ترتيب linefill** (`math.max/min` على top/bot).

### 🔧 تعديلات نواة V10 لدعم السحابة

- إضافة `var int buy_trend_start_bar` و `sell_trend_start_bar`.
- التقاط `bar_index` عند عبور `consecutive_count` من 0 → ≥1.
- مسح بداية الترند المعاكس عند الانعكاس.
- مسح بداية الترند عند تصفير العدّاد في حالة "NONE".
- تصريح `max_linefills_count=10` في `indicator(...)`.

### ⚠️ ملاحظات

- نواة V10 (request.security pattern، الشروط الميتة، تصفير العدّاد العدواني، الـ time-stop الصامت، وغيرها) **لم تُعدَّل** في هذه النسخة. تعديلها مخطّط لـ V11 المنفصل.
- V10 و V202 معماريتهما مختلفة جوهرياً ولا يحلّ أحدهما محلّ الآخر — يمكن استخدامهما معاً (ولكن ليس على نفس الشارت في الغالب).

---

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

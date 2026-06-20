# סוכן חשבוניות ורכש אוטומטי — סיכום שלב 1 (MVP Backend) לאישור ארכיטקט

**פרויקט:** Supabase `Invoices-tracker` (`vzeowbriddhvhpishmhn`)
**עיקרון מנחה:** אדיטיבי בלבד — לא שונו/נמחקו טבלאות או פונקציות קיימות. `products`, `suppliers`, `product_price_history`, `get_price_anomalies()` נשארו ללא נגיעה.
**סטטוס:** Backend לשלב 1 נבנה ונבדק מקצה-לקצה (9/9 בדיקות עברו). שכבת Make/Whapi עדיין לא חוברה.

---

## 1. מטרה
קליטת חשבוניות מקבוצות וואטסאפ → חילוץ עם Gemini Vision → staging → סיכום לאישור → ורק לאחר אישור עם **token** כתיבה ל-`product_price_history`, זיהוי שינויי מחיר, ויצירת מוצרים חדשים (לא פעילים עד סקירה).

## 2. ארכיטקטורה (תשתית קיימת + שכבה חדשה)
```
קבוצת וואטסאפ ──(Whapi webhook)──► Make
   │ הודעה עם תמונה/PDF
   ▼
Make: הורדת מדיה → Storage פרטי → Gemini Vision (JSON) → RPC wa_ingest_invoice
   ▼
סיכום עברית + token חזרה לקבוצה  →  המשתמש שולח "אישור <token>"
   ▼
RPC wa_confirm_invoice → כותב ל-product_price_history (קיים) + מוצרים חדשים (active=false)
   ▼
get_price_anomalies() (קיים) מזהה חריגות מחיר
```

**נכסים קיימים בשימוש חוזר (לא שונו):**
`product_price_history` (מנוע המחירים, 858 שורות), `get_price_anomalies()` (זיהוי עליות/ירידות מול ממוצע 3 חשבוניות), `price_anomalies_acknowledged`, `inventory` + `upsert_stock()`, `order_drafts`, `orders_log`, `products` (id=uuid, sku ב-91%), `suppliers` (id=text).

## 3. מה נבנה (4 מיגרציות)
### טבלאות staging חדשות (RLS פעיל; ללא policies → רק RPCs מסוג SECURITY DEFINER ניגשים)
- **`wa_invoice_intake`** (header): `id, token, source, source_chat_id, source_msg_id, storage_path, image_url, supplier_id→suppliers(SET NULL), supplier_name_raw, invoice_ref, invoice_date, location='פסאו', currency='ILS', subtotal, vat_rate, vat_amount, discount_amount, total_amount, computed_lines_total, totals_match, totals_diff, raw_json(jsonb), confidence, status(pending/confirmed/rejected), created_at, confirmed_at`
  - אינדקסים: `UNIQUE(source_msg_id)`; `UNIQUE(supplier_id,invoice_ref) WHERE status<>'rejected'`; `UNIQUE(token) WHERE status='pending'`; `(source_chat_id,status)`
- **`wa_invoice_lines`**: `id, intake_id→intake(CASCADE), line_no, product_name_raw, sku_raw, qty, unit_raw, pack_size, base_unit, base_qty, unit_price_raw, unit_price_normalized, line_total, matched_product_id→products(SET NULL), match_method, match_score, is_new`
- **`wa_invoice_events`** (אודיט): `id, intake_id, event_type, detail(jsonb), created_at`
- **`wa_product_review`** (תור סקירת מוצרים חדשים): `product_id(PK→products CASCADE), intake_id, supplier_id, name_raw, status, note, created_at, reviewed_at, reviewed_by`
- **Storage bucket `invoices`** — פרטי.

### פונקציות RPC (SECURITY DEFINER, `search_path` נעול, מוגנות בסוד `app_config.WA_INGEST_SECRET`, הרשאה ל-anon/authenticated/service_role)
- `wa_ingest_invoice(p_secret, p_payload jsonb, p_chat_id, p_msg_id, p_image_url)` → staging + matching + סיכום + token
- `wa_confirm_invoice(p_secret, p_chat_id, p_token)` → כתיבה ל-product_price_history + מוצרים חדשים
- `wa_reject_invoice(p_secret, p_chat_id, p_token)`
- עזר: `wa_match_supplier`, `wa_match_product`, `wa_gen_token`

### קונפיג (`app_config`)
`WA_INGEST_SECRET` (סוד), `WA_INVOICE_GROUPS` (allowlist), `WA_DEFAULT_LOCATION='פסאו'`, `WA_PRICE_ALERT_PCT=8`, `WA_FUZZY_THRESHOLD=0.45`, `WA_WRITE_ORDERS_LOG=false`.

## 4. הכרעות תכן
- **התאמת מוצרים:** SKU → שם מדויק → fuzzy (pg_trgm) **בתוך אותו ספק בלבד**. כולל מוצרים `discontinued` (כדי לא ליצור כפילות), עם העדפה ל-active.
- **מוצרים חדשים:** נוצרים ב-`products` עם **`active=false`** (מוסתרים מהאפליקציה) + רשומה ב-`wa_product_review`. אפס שינוי סכמה ל-products.
- **בסיס מחיר:** **נטו לפני מע"מ** ל-`product_price_history` (עקבי עם 858 השורות הקיימות). מע"מ/הנחות/raw נשמרים ב-staging.
- **Idempotency:** דה-דופ ב-ingest (msg_id, supplier+ref); אישור כפול = no-op (status guard) + `NOT EXISTS` לפני INSERT ל-price_history.
- **נורמליזציית יחידות:** `base_qty=qty*pack_size`, `unit_price_normalized=line_total/base_qty` — מונע ששינוי אריזה ייראה כשינוי מחיר.
- **אבטחה:** Make קורא ב-anon key + secret בלבד (ללא service_role). הטבלאות מוגנות RLS.

## 5. תוצאות בדיקות (9/9 ✅)
ingest · התאמת SKU · fuzzy בתוך-ספק · שם מדויק · מוצר חדש (active=false+סקירה) · אישור עם token · **אישור כפול → 4 שורות בלבד (idempotent)** · דחייה (rejected, 0 מחירים) · כתיבה ל-product_price_history · **get_price_anomalies זיהה +100% אחרי אישור** · בדיקת סכומים (אזהרה 87.5₪ מול 100₪) · נורמליזציה (₪10/קרטון-12 → ₪0.833).
כל נתוני הבדיקה נמחקו; הנתונים הקיימים נשארו שלמים.

🐞 **באג שתוקן:** מוצר `discontinued=true` שהספק עדיין מחייב עליו — תוקן כך שמתאים אליו במקום ליצור כפילות.

---

## 6. משימות להמשך

### שלב 1 — השלמה (חיבור Make/Whapi) — חוסם, דורש צד משתמש
1. הפעלת ארגון Make (כרגע paused) + אישור הרשאות ל-Make MCP.
2. מזהי קבוצות וואטסאפ → `WA_INVOICE_GROUPS`.
3. תרחיש Make יחיד עם Router:
   - ענף A: Whapi webhook → סינון (קבוצה מאושרת + תמונה/PDF) → הורדת מדיה → העלאה ל-Storage פרטי (signed upload) → Gemini Vision → `wa_ingest_invoice` → שליחת סיכום.
   - ענף B: טקסט "אישור/ביטול <token>" מקבוצה מאושרת → `wa_confirm/reject_invoice` → תשובה.
4. פרומפט Gemini סופי שמפיק את כל השדות (sku, pack_size, vat_rate, subtotal, discount).
5. מנגנון העלאה מאובטח ל-bucket הפרטי (signed upload URL).

### שלב 2 — אינטליגנציית הזמנה
- `recommend_orders(location, supplier)` מ-`orders_log` (קצב+כמויות) + תזמון מחיר (לקנות כשזול) → `order_drafts` (status='suggested').
- דייג'סט שבועי בוואטסאפ/Telegram; מילוי מראש של אפליקציית ההזמנות.
- מסך/תהליך סקירת מוצרים חדשים: אישור `active=false → true` + קביעת קטגוריה/יחידה כדי שיופיעו באפליקציה.

### שלב 3 — דיוק ולמידה
- חיבור **Alfred POS** (מכירות→צריכה אמיתית) → דיוק כמויות; מילוי `inventory.avg_weekly_usage`.
- לולאת משוב: אישור/עריכת המלצות → כיול ספים.
- ריבוי סניפים (אומינו) דרך מיפוי קבוצה→location במקום ברירת מחדל 'פסאו'.
- נורמליזציה v2: טבלת אריזות/המרות + שימוש ב-`unit_to_base_factor/base_unit`; התראות רק על מחיר מנורמל.

### Backlog — קשיחות ותפעול
- מדיניות שמירה/תפוגה ל-Storage; signed URLs זמניים.
- ניטור מגבלות Make Free (1000 פעולות/חודש, 2 תרחישים) — שדרוג לפי צורך.
- סיכון ToS של Whapi (לא רשמי) — לשקול Cloud API רשמי למספר ייעודי בהמשך.
- סף confidence: חילוץ בביטחון נמוך → סקירה מחמירה.
- טיפול בספק שלא זוהה: סימון בסיכום + שלב מיפוי ידני.
- Observability: תצוגת אדמין מעל `wa_invoice_events` / intakes ממתינים / תור סקירה.
- Backfill אופציונלי של חשבוניות היסטוריות.

## 7. סיכונים פתוחים
- דיוק חילוץ Gemini בעברית — ממותן ע"י שלב האישור + `raw_json` + אפשרות fallback ל-Claude.
- Whapi לא-רשמי (אזור אפור ToS).
- Make Free מוגבל.

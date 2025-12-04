اکنون که جستجوی ما به لطف `websearch_to_tsquery` به خوبی کار می‌کند، زمان آن رسیده که **رتبه‌بندی (ranking)** نتایج را تنظیم کنیم تا نتایجی که به نظر ما به عنوان انسان مرتبط‌تر هستند، در بالای لیست قرار بگیرند.

---

### **۱. اهمیت انتخاب زبان صحیح**

اولین قدم برای تنظیم دقیق نتایج، استفاده از پیکربندی زبان صحیح برای full-text search است. شما می‌توانید با بررسی `default_text_search_config`، زبان پیش‌فرض را مشاهده کنید.

```postgresql
SHOW default_text_search_config; -- pg_catalog.english
```

با دادن آرگومان به تابع `to_tsvector` می توان زبان پیشفرض را override کرد: 

```postgresql
SELECT
	title,
	to_tsvector('simple', title) AS simple,
	to_tsvector('english', title) AS english
FROM
	movies;
```

??? note "خروجی کوئری بالا"
    <div class='centered-div'>
      <img src="../../assets/images/full_text_search/94_1.png">
    </div>

**`simple` در مقابل `english`:**

  *   پیکربندی `simple` هیچ پردازش زبانی خاصی انجام نمی‌دهد و کلمات توقف (stop words) مانند `by`, `of`, `the` را حذف نمی‌کند.
  *   پیکربندی `english` (یا هر زبان دیگری) به صورت هوشمند کلمات توقف را حذف کرده و کلمات را به ریشه زبانی (lexeme) آن‌ها تبدیل می‌کند.

استفاده از پیکربندی زبان صحیح، هم به تطابق بهتر نتایج و هم به رتبه‌بندی دقیق‌تر کمک می‌کند.

---

### **۲. تنظیم تابع `ts_rank` برای در نظر گرفتن طول سند**

تابع `ts_rank` به صورت پیش‌فرض، طول سند (document) را در محاسبات خود در نظر نمی‌گیرد. این ممکن است باعث شود که عناوین طولانی که چندین بار شامل کلمه کلیدی می‌شوند، رتبه بهتری نسبت به آنچه انتظار داریم، کسب کنند.

شما می‌توانید با ارسال یک آرگومان عددی به `ts_rank`، رفتار آن را تغییر دهید.

```postgresql
SELECT 
	title, 
	ts_rank(to_tsvector(title), to_tsquery('fight'), 1) AS rank
FROM 
	movies
WHERE 
	to_tsvector(title) @@ to_tsquery('fight')
ORDER BY 
	rank DESC;
```

??? note "خروجی کوئری بالا"
    <div class='centered-div'>
      <img src="../../assets/images/full_text_search/94_2.png">
    </div>

*   **آرگومان `0`:** این مقدار پیشفرض است. در این حالت تابع `ts_rank`طول سند را در محاسبات خود در نظر نمی‌گیرد. 
*   **آرگومان `1`:** این گزینه باعث می‌شود که `ts_rank` به **اسناد کوتاه‌تر** وزن بیشتری بدهد. این کار معمولاً باعث می‌شود نتایجی که به یک تطابق دقیق (exact match) نزدیک‌تر هستند، به بالای لیست بیایند. این گزینه، رتبه را بر لگاریتم طول سند تقسیم می‌کند.
*   **آرگومان `2`:** این گزینه وزن **بیشتری** به اسناد کوتاه‌تر می‌دهد و رتبه را مستقیماً بر طول سند تقسیم می‌کند.

این یک ابزار عالی برای تنظیم دقیق تعادل بین ارتباط کلمات کلیدی و فشردگی عنوان است.
دیگر آپشن های این تابع را می توانید از [اینجا](https://www.postgresql.org/docs/current/textsearch-controls.html#TEXTSEARCH-RANKING) مطالعه کنید. 


---

### **۳. جستجو در چند ستون و وزن‌دهی با `setweight`**

برای جستجو در چند ستون (مانند `title` و `plot`)، می‌توانیم `tsvector`های آن‌ها را با هم الحاق کنیم.

```postgresql
SELECT 
	title, 
	ts_rank(
		to_tsvector(title) || ' ' || to_tsvector(plot),
		to_tsquery('fight')
	) AS rank
FROM 
	movies
WHERE 
	(to_tsvector(title) || ' ' || to_tsvector(plot))  @@ to_tsquery('fight')
ORDER BY 
	rank DESC;
```

??? note "خروجی کوئری بالا"
    <div class='centered-div'>
      <img src="../../assets/images/full_text_search/94_3.png">
    </div>

**مشکل:** این رویکرد ممکن است باعث شود که نتایجی که کلمه کلیدی را فقط در `plot` دارند، به دلیل تکرار زیاد، رتبه بالاتری از نتایجی کسب کنند که کلمه کلیدی را در `title` دارند.

**راه حل:** ما می‌توانیم با استفاده از تابع `setweight`، به `tsvector`های مختلف **وزن‌های متفاوتی** اختصاص دهیم. این تابع یک `tsvector` و یک حرف از `A` (بیشترین وزن) تا `D` (کمترین وزن) را به عنوان ورودی می‌گیرد.

```postgresql
SELECT 
	title, 
	ts_rank(
		setweight(to_tsvector(title), 'A') || ' ' || to_tsvector(plot),
		to_tsquery('fight')
	) AS rank
FROM 
	movies
WHERE 
	(to_tsvector(title) || ' ' || to_tsvector(plot))  @@ to_tsquery('fight')
ORDER BY 
	rank DESC;
```

??? note "خروجی کوئری بالا"
    <div class='centered-div'>
      <img src="../../assets/images/full_text_search/94_4.png">
    </div>

این کار به تابع `ts_rank` می‌گوید که به تطابق‌ها در ستون `title` اهمیت بیشتری نسبت به تطابق‌ها در ستون `plot` بدهد. تابع `setweight` این وزن را مستقیماً در داخل `tsvector` ذخیره می‌کند که یک طراحی بسیار هوشمندانه است.

---

### **۴. دستکاری دستی رتبه برای افزودن منطق کسب‌وکار**

از آنجایی که خروجی `ts_rank` فقط یک عدد (از نوع `float`) است، شما می‌توانید آن را با مقادیر دیگر ترکیب کرده و رتبه نهایی را دستکاری کنید. این یک روش عالی برای افزودن **منطق کسب‌وکار (business logic)** به فرآیند رتبه‌بندی است.

**مثال:** فرض کنید می‌خواهیم به فیلم‌های ژانر "اکشن" در نتایج جستجو برای کلمه "fight"، یک امتیاز اضافی (bonus) بدهیم.

```postgresql
SELECT 
	title,
	genre, 
	ts_rank(
		setweight(to_tsvector(title), 'A') || ' ' || to_tsvector(plot),
		to_tsquery('fight')
	) + (
		CASE
  		WHEN genre LIKE '%action%' THEN 0.1
  		ELSE 0
		END
	) AS rank
FROM 
	movies
WHERE 
	(to_tsvector(title) || ' ' || to_tsvector(plot))  @@ to_tsquery('fight')
ORDER BY 
	rank DESC;
```

این کار به ما اجازه می‌دهد تا علاوه بر ارتباط متنی، عواملی مانند محبوبیت، امتیاز کاربران یا هر معیار دیگری را در رتبه‌بندی نهایی نتایج دخیل کنیم.

---

#### **۵. جمع‌بندی و چالش عملکرد**

تا به اینجا، ما یک موتور جستجوی بسیار خوب و باکیفیت ساخته‌ایم. ما یاد گرفتیم که چگونه با استفاده از `setweight` و دستکاری دستی رتبه، نتایج را به شکلی کاملاً سفارشی و مرتبط مرتب کنیم.

با این حال، کوئری ما هنوز از نظر عملکردی بهینه نیست. تمام این محاسبات `tsvector` و `setweight` در زمان اجرا (`runtime`) انجام می‌شوند. در جلسه بعد، یاد می‌گیریم که چگونه با استفاده از ایندکس‌گذاری مناسب، این موتور جستجو را به یک راه‌حل بسیار سریع و کارآمد تبدیل کنیم.
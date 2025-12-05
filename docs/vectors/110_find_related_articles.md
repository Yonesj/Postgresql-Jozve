در این جلسه، به سراغ یک مثال واقعی و بسیار جذاب می‌رویم. ما قصد داریم متن مقالات یک وب‌سایت را به مدل هوش مصنوعی OpenAI ارسال کرده، برای آن‌ها امبدینگ (`embedding`) تولید کنیم و سپس از این امبدینگ‌ها برای پیدا کردن مقالات مرتبط و مشابه استفاده کنیم.

---

### **۱. فرآیند تولید امبدینگ از متن**

کدی که در این جلسه نمایش داده می‌شود به زبان PHP و فریمورک Laravel است، اما این فرآیند در هر زبان یا فریمورک دیگری کاملاً مشابه است.

1.  **خواندن محتوای مقاله:** ابتدا محتوای متنی هر مقاله را از فایل مربوطه می‌خوانیم.
2.  **ارسال به API هوش مصنوعی:** سپس، این متن را به API یک مدل تولید امبدینگ (در اینجا، مدل `text-embedding` از OpenAI) ارسال می‌کنیم.
3.  **دریافت بردار امبدینگ:** خروجی که از API دریافت می‌کنیم، یک ساختار داده است که شامل بردار امبدینگ مقاله به صورت یک آرایه طولانی از اعداد ممیز شناور می‌باشد.

در این‌جا کد لاراول (PHP) و معادل پایتون آن را در دو تب می‌بینید.

=== "PHP (Laravel)"
    
    ```php
    // PHP (Laravel) - خلاصه
    try {
        $contents = file_get_contents(resource_path('views/articles/' . $article->markdown_file));
        $response = OpenAI::embeddings()->create([
            'model' => 'text-embedding-3-small',
            'input' => $contents,
        ]);
        $embedding = $response->embeddings[0]->embedding;
        $vector = '[' . implode(',', $embedding) . ']';

        DB::connection('pgsql')->statement(
            "INSERT INTO articles (slug, title, embedding) VALUES (?, ?, ?::vector)
            ON CONFLICT (slug) DO UPDATE SET title = EXCLUDED.title, embedding = EXCLUDED.embedding",
            [$article->slug ?? $article->id, $article->title, $vector]
        );
    } catch (\Throwable $e) {
        info("Embedding error for {$article->id}: " . $e->getMessage());
    }
    ```

=== "Python"
    
    ```python
    # Python - خلاصه
    import json, openai, psycopg2
    from pathlib import Path

    openai.api_key = "..."

    contents = Path("/path/to/resources/views/articles/file.md").read_text(encoding="utf-8")

    resp = openai.Embedding.create(model="text-embedding-3-small", input=contents)

    vector_literal = json.dumps(embedding)

    sql = """
    INSERT INTO articles (slug, title, embedding)
    VALUES (%s, %s, %s::vector)
    ON CONFLICT (slug) DO UPDATE SET title = EXCLUDED.title, embedding = EXCLUDED.embedding;
    """

    conn = psycopg2.connect(DATABASE_URL)
    with conn:
        with conn.cursor() as cur:
            cur.execute(sql, (slug, title, vector_literal))
    conn.close()
    ```

برای مثال، مدل `text-embedding-3-small` از OpenAI یک بردار با **۱۵۳۶ بعد (dimensions)** تولید می‌کند.

---

### **۲. طراحی جدول برای ذخیره امبدینگ‌ها**

اکنون باید جدولی در PostgreSQL طراحی کنیم که بتواند این امبدینگ‌ها را ذخیره کند.

```postgresql
CREATE TABLE articles (
	id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	slug TEXT UNIQUE,
	title TEXT,
	embedding VECTOR(1536)
);
```

*   **`slug`:** یک شناسه متنی منحصر به فرد برای هر مقاله که بعداً برای عملیات `Upsert` از آن استفاده خواهیم کرد.
*   **`embedding`:** ستون اصلی ما از نوع `VECTOR` که تعداد ابعاد آن را دقیقاً برابر با خروجی مدل (۱۵۳۶) تنظیم کرده‌ایم.

---

### **۳. درج امبدینگ‌ها در پایگاه داده**

پس از دریافت امبدینگ از API، باید آن را به فرمت رشته‌ای مورد قبول PostgreSQL تبدیل کرده و در ستون `embedding` جدول `articles` درج کنیم. این فرآیند را برای تمام مقالات تکرار می‌کنیم.

=== "PHP"
    ```php
    $vector = '[' . implode(',', $embedding) . ']';
    ```

=== "Python"
    ```python
    def vector_literal_from_list(vec: List[float]) -> str:
        return "[" + ",".join(map(str, vec)) + "]"
    ```

---

### **۴. کوئری اصلی: پیدا کردن مقالات مرتبط**

اکنون که امبدینگ تمام مقالات را در پایگاه داده داریم، می‌توانیم به راحتی مقالات مرتبط با یک مقاله خاص را پیدا کنیم. این سناریو را در نظر بگیرید که کاربری در حال مطالعه یک مقاله (مثلاً با `id = 18`) است و ما می‌خواهیم لیستی از مقالات مشابه را به او پیشنهاد دهیم.

**راه‌حل:**

1.  **انتخاب بردار مرجع:** ابتدا با یک زیرکوئری (`subquery`)، بردار امبدینگ مقاله فعلی (`id = 18`) را به عنوان بردار مرجع خود انتخاب می‌کنیم.
2.  **محاسبه فاصله و مرتب‌سازی:** سپس، با استفاده از اپراتور فاصله L2 (`<->`)، فاصله تمام مقالات دیگر را از این بردار مرجع محاسبه کرده و نتایج را بر اساس این فاصله به صورت صعودی (`ASC`) مرتب می‌کنیم.
3.  **حذف مقاله فعلی:** با یک شرط `WHERE id <> 18`، خود مقاله فعلی را از لیست پیشنهادات حذف می‌کنیم.

```postgresql
SELECT 
	id,
	title,
	embedding <-> (
		SELECT embedding FROM articles WHERE id = 18
	) AS distance
FROM 
	articles
WHERE 
	id != 18
ORDER BY 
	distance ASC
LIMIT 5;
```

خروجی این کوئری، پنج مقاله‌ای را که از نظر معنایی بیشترین شباهت را با مقاله فعلی دارند، به ما نشان می‌دهد. این تکنیک می‌تواند برای متن، تصویر یا هر نوع داده دیگری که بتوان برای آن امبدینگ تولید کرد، به کار رود و یک روش بسیار ساده و قدرتمند برای پیاده‌سازی سیستم‌های توصیه‌گر است.
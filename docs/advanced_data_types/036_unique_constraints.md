### **جلسه ۳۶: قید یکتایی (UNIQUE)**

وقتی صحبت از قید یکتایی (unique) می‌شود، ذهن اکثر افراد به سمت کلید اصلی (primary key) می‌رود که کاملاً درست است. یک کلید اصلی باید هم `NOT NULL` باشد و هم `UNIQUE`. اما شما می‌توانید محدودیت `UNIQUE` را به ستون‌های دیگر یا حتی ترکیبی از چند ستون نیز اضافه کنید.

---

#### **۱. اعمال محدودیت `UNIQUE` روی یک ستون**

برای اعمال این محدودیت، کافی است کلمه کلیدی `UNIQUE` را پس از تعریف نوع داده یک ستون قرار دهید، مانند :

```postgresql
CREATE TABLE products (
	id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	product_number TEXT UNIQUE,
	price NUMERIC NOT NULL CHECK(price > 0)
);
```

با این کار، اگر تلاش کنید مقداری را وارد کنید که از قبل در آن ستون وجود داشته باشد، PostgreSQL با یک خطای واضح جلوی شما را می‌گیرد و از درج داده تکراری جلوگیری می‌کند.

```postgresql
INSERT INTO products VALUES (DEFAULT, 'ABC New Product', 9.99);
-- INSERT 0 1, 1 rows affected
INSERT INTO products VALUES (DEFAULT, 'ABC New Product', 9.99);
-- ERROR:  duplicate key value violates unique constraint "products_product_number_key"
-- DETAIL:  Key (product_number)=(ABC New Product) already exists.
```

---

#### **۲. رفتار عجیب `UNIQUE` با مقادیر `NULL`**

یک نکته بسیار مهم و کمی گیج‌کننده در مورد محدودیت `UNIQUE`، رفتار آن با مقادیر `NULL` است. به صورت پیش‌فرض، اگر یک ستون `UNIQUE` را به صورت `nullable` (یعنی بدون `NOT NULL`) تعریف کنید، شما می‌توانید **چندین مقدار `NULL`** را در آن ستون وارد کنید. دلیل این اتفاق این است که `NULL` به معنای "نامشخص" است و وقتی دو مقدار `NULL` با هم مقایسه می‌شوند، نتیجه "برابر" نیست، بلکه "نامشخص" است. به همین دلیل، PostgreSQL نمی‌تواند دو `NULL` را با هم برابر در نظر بگیرد و اجازه ورود چندین `NULL` را می‌دهد. این رفتار ممکن است دقیقاً همان چیزی باشد که شما می‌خواهید: تمام مقادیر غیر `NULL` باید منحصر به فرد باشند، اما مقادیر نامشخص می‌توانند متعدد باشند.

<div class='centered-div'>
  <img src="../../assets/images/advanced_data_types/36_1.png">
</div>

---

#### **۳. کنترل رفتار `NULL` در محدودیت `UNIQUE`**

اگر رفتار پیش‌فرض را نمی‌خواهید و قصد دارید جلوی ورود چندین `NULL` را بگیرید، دو راه حل دارید. راه حل اول این است که ستون را به صورت `NOT NULL UNIQUE` تعریف کنید. با این کار، هیچ مقدار `NULL`ی هرگز وارد جدول نخواهد شد. راه حل دوم استفاده از قابلیت `UNIQUE NULLS NOT DISTINCT` است.

```postgresql
CREATE TABLE products (
	id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	product_number TEXT UNIQUE NULLS NOT DISTINCT,
	price NUMERIC NOT NULL CHECK(price > 0)
);
```

با افزودن این عبارت، شما به PostgreSQL می‌گویید که در زمینه این محدودیت `UNIQUE`، مقادیر `NULL` را با هم "برابر" در نظر بگیرد. در نتیجه، شما اجازه دارید **فقط یک مقدار `NULL`** را در ستون وارد کنید و در تلاش برای ورود `NULL` دوم، با خطای تکراری مواجه خواهید شد. این برای سناریوهایی که می‌خواهید "یک" مقدار نامشخص داشته باشید اما بقیه مقادیر منحصر به فرد باشند، بسیار مفید است.

```postgresql
INSERT INTO products VALUES (DEFAULT, NULL, 9.99);
-- INSERT 0 1, 1 rows affected
INSERT INTO products VALUES (DEFAULT, NULL, 9.99);
-- ERROR:  duplicate key value violates unique constraint "products_product_number_key"
-- DETAIL:  Key (product_number)=(null) already exists.
```

---

#### **۴. محدودیت `UNIQUE` روی چند ستون (Composite Unique)**

شما می‌توانید محدودیت منحصر به فرد بودن را روی ترکیبی از چند ستون اعمال کنید. در این حالت، **ترکیب مقادیر** آن ستون‌ها باید منحصر به فرد باشد، نه هر ستون به تنهایی. برای این کار، باید محدودیت را به صورت یک محدودیت جدول (table constraint) در انتهای تعریف جدول خود قرار دهید، مانند:

```postgresql
CREATE TABLE products (
	id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	brand TEXT NOT NULL,
	product_number TEXT NOT NULL,

	UNIQUE(brand, product_number)
);
```

با این تعریف، شما می‌توانید چندین محصول با `product_number` یکسان داشته باشید، به شرطی که `brand` آن‌ها متفاوت باشد.

---

#### **۵. تعریف `UNIQUE` به عنوان محدودیت جدول و نام‌گذاری آن**

حتی برای یک ستون تکی نیز می‌توانید محدودیت `UNIQUE` را به صورت یک محدودیت جدول تعریف کنید. این کار به شما اجازه می‌دهد تا با استفاده از کلمه کلیدی `CONSTRAINT`، یک نام مشخص برای محدودیت خود انتخاب کنید، مانند `CONSTRAINT must_be_unique UNIQUE (product_number)`. نام‌گذاری محدودیت‌ها می‌تواند به خوانایی بیشتر پیام‌های خطا کمک کند، هرچند پیام خطای پیش‌فرض `UNIQUE` نیز به خودی خود بسیار واضح است.

---

#### **۶. جمع‌بندی**

به یاد داشته باشید که `PRIMARY KEY` به صورت خودکار یک ستون را `NOT NULL` و `UNIQUE` می‌کند، اما شما آزاد هستید که محدودیت `UNIQUE` را به هر ستون یا مجموعه‌ای از ستون‌های دیگر در جدول خود اضافه کنید. PostgreSQL گزینه‌های بسیار قدرتمندی برای کنترل دقیق رفتار این محدودیت، به خصوص در مورد مقادیر `NULL` و ستون‌های ترکیبی، در اختیار شما قرار می‌دهد.

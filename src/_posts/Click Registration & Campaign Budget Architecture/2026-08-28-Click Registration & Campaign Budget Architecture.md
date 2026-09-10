---
title: معماری مدیریت بودجه و ثبت کلیک
date: 2026-08-28 08:58:47 +03:30
tags: [AdTech, یکتانت, yektanet]
description: ین پروژه یک Mini AdTech Platform است که چرخه‌ی اصلی یک سیستم تبلیغات کلیکی را در قالب یک MVP شبیه‌سازی می‌کند.
image: "/simorq/simorq.png"
---








# معماری مدیریت بودجه و ثبت Click


### 1. هدف و مسئولیت

در زمان ثبت یک `Click`، هزینه‌ی تبلیغ (`CPC`) باید از بودجه‌ی کمپین کسر شود. سیستم باید هم‌زمان دو محدودیت را کنترل کند:

* **Daily Budget** — سقف هزینه‌ی روزانه‌ی کمپین
* **Total Budget** — سقف کل هزینه‌ی کمپین

اگر با ثبت یک Click هرکدام از این محدودیت‌ها به پایان برسد، وضعیت کمپین باید به `stop` تغییر کند تا تبلیغ دیگر نمایش داده نشود.

بنابراین ثبت Click صرفاً ایجاد یک رکورد در دیتابیس نیست و شامل بخشی از منطق کسب‌وکار سیستم AdTech نیز می‌شود.

---

### 2. محل قرارگیری Business Logic

این منطق در یک **Service Layer** و در فایل `campaigns/services.py` قرار گرفته است.

قرار دادن این منطق در Service Layer به این دلیل انتخاب شده که ثبت Click می‌تواند در آینده علاوه بر ایجاد رکورد و مدیریت بودجه، مسئولیت‌هایی مانند موارد زیر نیز داشته باشد:

* Click Validation
* Campaign Status Update
* Transaction Recording
* Fraud / Invalid Traffic Checks
* سایر عملیات مرتبط با Billing

قرار دادن همه‌ی این مسئولیت‌ها در مدل `Campaign` باعث می‌شود مدل به‌تدریج به محل تجمع Business Logicهای مختلف تبدیل شود.

Service Layer این منطق را از مدل‌ها جدا کرده و مسئولیت عملیات کسب‌وکار را در یک محل مشخص و قابل تست نگه می‌دارد.

---

### 3. طراحی Service

عملیات اصلی سیستم از طریق تابع زیر انجام می‌شود:

`register_click(ad, publisher, ip_address, user_agent, impression=None)`

ورودی‌های `ip_address` و `user_agent` به دلیل اجباری بودن این فیلدها در مدل `Click` مستقیماً از درخواست کاربر دریافت می‌شوند. پارامتر `impression` نیز اختیاری است.

برای مدیریت خطا، به‌جای بازگرداندن یک `True/False` ساده، از **Custom Exceptions** استفاده شده است تا نوع خطا به‌صورت مشخص بیان شود.

Exceptionهای اصلی عبارت‌اند از:

* `AdNotActiveError`
* `CampaignNotActiveError`
* `CampaignOutOfDateRangeError`
* `InsufficientDailyBudgetError`
* `InsufficientTotalBudgetError`

این ساختار در آینده امکان نگاشت خطاهای کسب‌وکار به HTTP Status Codeهای مناسب در لایه API را نیز فراهم می‌کند.

---

### 4. Validation قبل از ثبت Click

پیش از ثبت Click، شرایط زیر باید برقرار باشند:

1. تبلیغ (`Ad`) فعال باشد.
2. وضعیت کمپین `active` باشد.
3. کمپین در بازه‌ی زمانی معتبر خود قرار داشته باشد.
4. بودجه‌ی روزانه برای پرداخت CPC کافی باشد.
5. بودجه‌ی کل کمپین برای پرداخت CPC کافی باشد.

بودجه‌ی روزانه و بودجه‌ی کل به‌صورت مستقل بررسی می‌شوند؛ زیرا ممکن است بودجه‌ی کل کافی باشد اما سقف روزانه اجازه‌ی ثبت Click را ندهد.

---

### 5. محاسبه مصرف بودجه

برای جلوگیری از قرار دادن Queryهای محاسباتی در مدل‌ها، دو تابع مستقل برای محاسبه‌ی مصرف بودجه در Service Layer در نظر گرفته شده است:

* `get_daily_cpc_consumption(campaign)`
* `get_total_cpc_consumption(campaign)`

مصرف روزانه برابر مجموع `CPC` مربوط به Clickهای همان کمپین در تاریخ جاری است و مصرف کل برابر مجموع CPC تمام Clickهای ثبت‌شده برای کمپین است.

این محاسبات با `aggregate()` و `Sum()` انجام می‌شوند و در صورت نبود Click، مقدار `None` به صفر تبدیل می‌شود.

---

### 6. Atomicity و مدیریت همزمانی

ثبت Click و مدیریت بودجه باید در یک **Database Transaction** انجام شوند.

استفاده از:

`transaction.atomic()`

باعث می‌شود عملیات به‌صورت یک واحد انجام شود؛ بنابراین اگر در هر مرحله خطایی رخ دهد، تغییرات انجام‌شده Rollback خواهند شد.

اما Transaction به‌تنهایی برای این مسئله کافی نیست، زیرا امکان **Race Condition** وجود دارد.

برای مثال، اگر بودجه‌ی باقی‌مانده ۱۰ واحد و CPC برابر ۹ باشد، دو درخواست هم‌زمان ممکن است هر دو وضعیت بودجه را کافی تشخیص دهند و هر دو Click را ثبت کنند. در نتیجه مصرف واقعی از سقف بودجه عبور خواهد کرد.

برای جلوگیری از این وضعیت، رکورد `Campaign` در ابتدای Transaction با `select_for_update()` قفل می‌شود:

`Campaign.objects.select_for_update().get(pk=ad.campaign.pk)`

به این ترتیب، درخواست‌های هم‌زمان برای همان Campaign تا پایان Transaction منتظر می‌مانند.

نکته‌ی مهم این است که تمام Validationهای مرتبط با Campaign نیز باید روی همان نمونه‌ی Lock شده انجام شوند تا تصمیم‌گیری بر اساس وضعیت معتبر و فعلی رکورد انجام شود.

---

### 7. جریان اصلی ثبت Click

جریان منطقی `register_click` به این صورت است:

**Lock Campaign → Validate → Calculate Budget → Create Click → Update Campaign Status → Commit**

به بیان دقیق‌تر:

1. آغاز Transaction
2. Lock کردن Campaign
3. بررسی فعال بودن Ad
4. بررسی وضعیت Campaign
5. بررسی بازه‌ی زمانی Campaign
6. بررسی Daily Budget
7. بررسی Total Budget
8. ایجاد Click
9. بررسی اتمام بودجه و تغییر وضعیت Campaign در صورت نیاز
10. Commit Transaction

اگر هر مرحله‌ی اعتبارسنجی شکست بخورد، Exception مربوطه ایجاد شده و Transaction بدون ثبت Click پایان می‌یابد.

---

### 8. مدیریت اتمام بودجه

ثبت Click باید با وضعیت Campaign نیز هماهنگ باشد.

برای این منظور، مسئولیت بررسی اتمام بودجه در تابع مستقلی مانند:

`close_campaign_if_exhausted(campaign)`

قرار گرفته است.

این تابع مصرف روزانه و کل را بررسی می‌کند و اگر یکی از آن‌ها به سقف بودجه برسد، وضعیت Campaign را به `stop` تغییر می‌دهد.

این تابع از نظر کد مستقل است، اما از نظر زمان اجرا بخشی از همان Transaction مربوط به `register_click` است و بلافاصله پس از ایجاد Click اجرا می‌شود.

در نتیجه:

* منطق بستن Campaign قابل تست و استفاده‌ی مجدد است.
* تغییر Click و تغییر Status در یک Transaction انجام می‌شوند.
* بین ثبت Click و تغییر Status وقفه‌ی پردازشی یا فرآیند دوره‌ای ایجاد نمی‌شود.

---

### 9. تصمیم معماری نهایی

معماری نهایی ثبت Click بر چهار اصل اصلی استوار است:

**Service Layer**
برای جداسازی Business Logic از مدل‌ها.

**Transaction Atomicity**
برای جلوگیری از ثبت ناقص عملیات و اطمینان از Rollback شدن تغییرات در صورت خطا.

**Row-Level Locking**
با استفاده از `select_for_update()` برای جلوگیری از Race Condition هنگام مصرف هم‌زمان بودجه.

**Separation of Responsibilities**
با جدا کردن محاسبه‌ی مصرف بودجه، مدیریت Exceptionها و بررسی اتمام بودجه از منطق اصلی ثبت Click.

این طراحی در MVP پیچیدگی غیرضروری ایجاد نمی‌کند، اما ساختار مناسبی برای توسعه‌ی بعدی سیستم در جهت Billing، Reporting، Fraud Detection و API فراهم می‌کند.

---

### 10. سناریوهای پایه‌ی اعتبارسنجی

رفتار مورد انتظار Service در سناریوهای اصلی:

| Scenario                              | Expected Result                |
| ------------------------------------- | ------------------------------ |
| Ad فعال + Campaign معتبر + بودجه کافی | Click ایجاد می‌شود             |
| `ad.is_active=False`                  | `AdNotActiveError`             |
| Campaign با وضعیت `stop`              | `CampaignNotActiveError`       |
| Campaign خارج از بازه‌ی زمانی         | `CampaignOutOfDateRangeError`  |
| Daily Budget ناکافی                   | `InsufficientDailyBudgetError` |
| Total Budget ناکافی                   | `InsufficientTotalBudgetError` |
| مصرف بودجه پس از Click به سقف برسد    | Campaign → `stop`              |

پس از نهایی شدن Service، این سناریوها باید به‌صورت Automated Test نیز پوشش داده شوند؛ به‌خصوص رفتار مربوط به **Concurrency و Race Condition**.

---
title: معماری مدل های Impression و Click 
date: 2026-08-10 08:58:47 +03:30
tags: [AdTech, یکتانت, yektanet]
description: ین پروژه یک Mini AdTech Platform است که چرخه‌ی اصلی یک سیستم تبلیغات کلیکی را در قالب یک MVP شبیه‌سازی می‌کند.
image: "/simorq/simorq.png"
---
# Impression & Click Tracking Architecture

## Overview

در یک سیستم **Pay-Per-Click (PPC)**، رویدادهای `Impression` و `Click` بخش مرکزی چرخه‌ی تبلیغات هستند. این رویدادها علاوه بر ثبت رفتار کاربر، مبنای محاسبه‌ی شاخص‌های تبلیغاتی، Billing و گزارش‌گیری سیستم قرار می‌گیرند.

در این فاز، مدل داده‌ی مربوط به این دو رویداد طراحی شده است تا بتواند علاوه بر ثبت ارتباط تبلیغ با رویداد، منبع نمایش تبلیغ و اطلاعات مورد نیاز برای تحلیل رویدادها را نیز حفظ کند.

---

# Architectural Decision: Event Context

یک `Impression` زمانی ایجاد می‌شود که یک `Ad` در فضای تبلیغاتی یک `Publisher` نمایش داده شود و `Click` نیز نشان‌دهنده‌ی تعامل کاربر با همان تبلیغ است.

بنابراین صرفاً نگهداری رابطه‌ی رویداد با `Ad` برای نیازهای سیستم کافی نیست.

ساختار اصلی رابطه به شکل زیر در نظر گرفته شده است:

```text
Advertiser
    │
 Campaign
    │
    Ad
    │
    ├──────── Impression
    │              │
    │              └── Publisher
    │
    └──────── Click
                   │
                   ├── Publisher
                   └── Impression
```

نگهداری مستقیم `Publisher` روی رویدادها امکان پاسخ‌گویی ساده و مستقل به پرسش‌هایی مانند موارد زیر را فراهم می‌کند:

* یک Publisher چه تعداد Impression دریافت کرده است؟
* کدام Publisher بیشترین Click را ایجاد کرده است؟
* درآمد یک Publisher در یک بازه‌ی زمانی چقدر بوده است؟
* عملکرد تبلیغات روی هر Publisher چگونه است؟

این تصمیم به‌خصوص در فازهای **Reporting، Analytics و Billing** اهمیت خواهد داشت.

---

# Impression Model

`Impression` نمایانگر یک بار نمایش موفق یک تبلیغ است.

مدل شامل اطلاعات زیر است:

| Field        | Purpose                                  |
| ------------ | ---------------------------------------- |
| `ad`         | تبلیغی که نمایش داده شده است             |
| `publisher`  | ناشری که تبلیغ در فضای او نمایش داده شده |
| `ip_address` | IP بازدیدکننده                           |
| `user_agent` | مشخصات مرورگر یا دستگاه                  |
| `created_at` | زمان ثبت Impression                      |

رابطه‌ی اصلی:

```text
Ad ───< Impression >─── Publisher
```

برای `ip_address` از `GenericIPAddressField` استفاده شده است تا امکان نگهداری IPv4 و IPv6 وجود داشته باشد.

`user_agent` نیز به‌عنوان داده‌ی متنی ذخیره می‌شود تا اطلاعات مرورگر و دستگاه در زمان ثبت رویداد قابل تحلیل باشد.

---

# Click Model

`Click` نمایانگر تعامل کاربر با یک تبلیغ نمایش‌داده‌شده است.

مدل شامل اطلاعات زیر است:

| Field        | Purpose                              |
| ------------ | ------------------------------------ |
| `ad`         | تبلیغی که روی آن کلیک شده است        |
| `publisher`  | ناشری که کلیک در فضای او رخ داده است |
| `impression` | Impression مرتبط با کلیک             |
| `ip_address` | IP کاربر                             |
| `user_agent` | مشخصات مرورگر یا دستگاه              |
| `created_at` | زمان ثبت کلیک                        |

رابطه‌ی اصلی:

```text
Ad ───< Click >─── Publisher
           │
           └── Impression
```

---

# Click → Impression Relationship

هر `Click` از نظر دامنه‌ی کسب‌وکار می‌تواند به یک `Impression` مشخص مرتبط باشد؛ زیرا کلیک معمولاً نتیجه‌ی نمایش قبلی یک تبلیغ است.

به همین دلیل، رابطه‌ی زیر در مدل `Click` در نظر گرفته شده است:

```text
Click → Impression
```

با این حال، این رابطه در MVP اجباری نشده و `impression` می‌تواند `NULL` باشد.

دلیل این تصمیم، پذیرش این واقعیت است که در تمام شرایط عملیاتی ممکن است امکان انتساب یک Click به Impression مشخص وجود نداشته باشد.

بنابراین مدل بین **روابط ایده‌آل دامنه** و **محدودیت‌های ثبت رویداد در دنیای واقعی** انعطاف ایجاد می‌کند.

---

# Data Redundancy in Click

در مدل `Click`، علاوه بر رابطه با `Impression`، دو رابطه‌ی `ad` و `publisher` نیز به‌صورت مستقیم نگهداری می‌شوند:

```text
Click
 ├── Ad
 ├── Publisher
 └── Impression (optional)
```

در نگاه اول این طراحی می‌تواند redundant به نظر برسد، زیرا `Impression` نیز خود به `Ad` و `Publisher` متصل است.

اما این redundancy آگاهانه است.

از آنجا که `impression` می‌تواند `NULL` باشد، وابسته کردن اطلاعات اصلی Click به مسیر:

```text
Click → Impression → Ad / Publisher
```

باعث می‌شود در برخی رویدادها Context اصلی Click در دسترس نباشد.

بنابراین `ad` و `publisher` به‌صورت مستقیم روی `Click` نگهداری می‌شوند تا Context اصلی رویداد مستقل از وجود Impression باقی بماند.

این تصمیم یک نوع **Controlled Denormalization** است که در ازای افزایش جزئی redundancy، دسترسی مستقیم به اطلاعات کلیدی رویداد و استقلال بیشتر داده را فراهم می‌کند.

---

# Event Metadata

دو فیلد `ip_address` و `user_agent` در هر دو مدل `Impression` و `Click` ذخیره می‌شوند.

```text
Impression
 ├── IP Address
 └── User Agent

Click
 ├── IP Address
 └── User Agent
```

دلیل نگهداری این اطلاعات، تنها گزارش‌گیری نیست؛ بلکه این داده‌ها می‌توانند در آینده برای تحلیل کیفیت ترافیک و شناسایی رفتارهای مشکوک مورد استفاده قرار گیرند.

برای مثال، الگوهایی مانند:

* تعداد غیرطبیعی Click از یک IP
* تکرار Click در فاصله‌ی زمانی کوتاه
* User-Agentهای غیرمعمول
* الگوهای تکرارشونده‌ی Impression و Click

می‌توانند به عنوان ورودی یک سیستم **Fraud Detection** یا **Invalid Traffic Detection** مورد استفاده قرار گیرند.

در MVP، پیاده‌سازی Fraud Detection خارج از Scope است و این فیلدها صرفاً داده‌ی خام مورد نیاز برای توسعه‌ی احتمالی این قابلیت را ثبت می‌کنند.

---

# Tracking Module

برای مدیریت رویدادهای تبلیغاتی، یک Django App مستقل با نام `tracking` ایجاد شده است.

ساختار ماژولار پروژه به شکل زیر توسعه پیدا می‌کند:

```text
accounts
    │
    └── User & Role Management

campaigns
    │
    └── Campaign & Ad Management

tracking
    │
    └── Impression & Click Events
```

این جداسازی با هدف **Separation of Concerns** انجام شده است.

هر App مسئول یک بخش مشخص از Domain است و این ساختار امکان توسعه‌ی مستقل‌تر قابلیت‌های Tracking را در مراحل بعد فراهم می‌کند.

---

# Data Integrity Considerations

وجود چند رابطه‌ی مستقیم روی `Click` به این معناست که سازگاری بین داده‌ها باید در منطق ثبت رویداد کنترل شود.

برای مثال، در صورت وجود یک `impression`، باید سازگاری زیر برقرار باشد:

```text
Click.ad       == Click.impression.ad

Click.publisher == Click.impression.publisher
```

این موضوع اهمیت بیشتری پیدا می‌کند زیرا `ad` و `publisher` به‌صورت مستقیم روی `Click` ذخیره شده‌اند.

بنابراین در مراحل بعدی، هنگام طراحی Event Registration، اعتبارسنجی این روابط باید بخشی از منطق ثبت Click باشد تا denormalization باعث ایجاد داده‌ی ناسازگار نشود.

---

# Future Reporting & Analytics

ساختار فعلی داده، پایه‌ی لازم برای توسعه‌ی قابلیت‌های تحلیلی آینده را فراهم می‌کند.

برای مثال:

```text
Publisher
   │
   ├── Impressions
   ├── Clicks
   └── Revenue
```

و:

```text
Ad
   │
   ├── Impressions
   ├── Clicks
   ├── CTR
   └── CPC
```

با استفاده از این رویدادها می‌توان در فازهای بعد شاخص‌هایی مانند موارد زیر را محاسبه کرد:

* Impression Count
* Click Count
* CTR
* CPC
* Publisher Revenue
* Campaign Performance

بنابراین مدل‌های Tracking تنها برای ثبت رویداد طراحی نشده‌اند؛ بلکه به‌عنوان لایه‌ی داده‌ی مورد نیاز برای **Analytics و Billing** نیز عمل می‌کنند.

---

# Architectural Summary

تصمیم‌های اصلی این فاز:

| موضوع              | تصمیم                                    |
| ------------------ | ---------------------------------------- |
| Tracking Module    | App مستقل با نام `tracking`              |
| Impression         | ارتباط مستقیم با `Ad` و `Publisher`      |
| Click              | ارتباط مستقیم با `Ad` و `Publisher`      |
| Click → Impression | رابطه‌ی اختیاری                          |
| Event Metadata     | ذخیره‌ی IP و User-Agent                  |
| IP Type            | `GenericIPAddressField`                  |
| Click Redundancy   | نگهداری مستقیم `Ad` و `Publisher`        |
| هدف Redundancy     | استقلال Context اصلی Click از Impression |
| Future Use         | Analytics، Billing و Fraud Detection     |

در این طراحی، `Impression` و `Click` به‌عنوان **Domain Events** اصلی سیستم تبلیغاتی در نظر گرفته شده‌اند. ساختار آن‌ها به‌گونه‌ای طراحی شده است که ضمن حفظ سادگی MVP، اطلاعات لازم برای توسعه‌ی قابلیت‌های مهمی مانند **Reporting، Billing و Fraud Detection** در مراحل بعدی در دسترس باشد.

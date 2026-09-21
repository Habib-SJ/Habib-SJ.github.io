---
title: معماری پایگاه داده و مهاجرت به پستگرس
date: 2026-09-13 08:58:47 +03:30
tags: [AdTech, یکتانت, yektanet]
description: ین پروژه یک Mini AdTech Platform است که چرخه‌ی اصلی یک سیستم تبلیغات کلیکی را در قالب یک MVP شبیه‌سازی می‌کند.
image: "/simorq/simorq.png"
---

# معماری پایگاه داده و مهاجرت به PostgreSQL


### 1. متن

نسخه‌ی اولیه‌ی پروژه با SQLite توسعه داده شده بود. با توجه به نیازهای معماری پروژه، به‌خصوص استفاده از Transaction و Row-Level Locking برای مدیریت همزمانی در فرآیند ثبت Click، تصمیم گرفته شد پایگاه داده به PostgreSQL منتقل شود.

این تغییر صرفاً یک جایگزینی Database Engine نیست؛ بلکه بخشی از آماده‌سازی زیرساخت پروژه برای رفتارهای همزمان، تست‌های واقعی‌تر و محیط‌های نزدیک‌تر به Production است.

---

## 2. تصمیم معماری

### تصمیم گرفته شده

پایگاه داده‌ی اصلی پروژه از **SQLite به PostgreSQL** منتقل شد.

در محیط توسعه‌ی فعلی، به دلیل محدودیت سیستم‌عامل و معماری سخت‌افزاری، PostgreSQL به‌صورت Cloud Database روی **Neon** اجرا می‌شود.

اتصال Django به Database از طریق یک Configuration مستقل از کد و با استفاده از `DATABASE_URL` انجام می‌شود.

### منطق تصمیم

دلایل اصلی این تصمیم:

* نیاز پروژه به رفتار مناسب‌تر Database در سناریوهای Concurrency
* استفاده از `select_for_update()` برای Lock کردن رکورد Campaign
* امکان اجرای تست‌های مرتبط با همزمانی روی یک Database واقعی‌تر
* نزدیک‌تر شدن محیط توسعه به الگوی رایج Production
* جداسازی Configuration دیتابیس از Source Code
* امکان جابه‌جایی بین PostgreSQL محلی و Cloud بدون تغییر Business Logic

---

## 3. الزامات همزمانی پایگاه داده 

یکی از دلایل اصلی مهاجرت، نیاز پروژه به کنترل همزمانی در فرآیند ثبت Click است.

در این فرآیند، چند درخواست ممکن است تقریباً هم‌زمان برای یک Campaign وارد شوند و وضعیت بودجه را بررسی کنند. بنابراین عملیات بررسی و مصرف بودجه باید در برابر Race Condition محافظت شود.

معماری ثبت Click از Transaction و Row-Level Locking استفاده می‌کند:

```text
Click Request
     │
     ▼
Transaction
     │
     ▼
Lock Campaign Row
     │
     ▼
Validate Budget
     │
     ▼
Create Click
     │
     ▼
Update Campaign State
     │
     ▼
Commit
```

استفاده از `select_for_update()` به Row-Level Locking نیاز دارد؛ بنابراین نوع Database انتخاب‌شده باید رفتار مناسب برای این الگو داشته باشد.

SQLite در این معماری انتخاب مناسبی برای شبیه‌سازی رفتار مورد انتظار در سناریوهای واقعی Concurrency نیست، زیرا مدل Locking آن با Row-Level Locking مورد استفاده در PostgreSQL متفاوت است.

---

## 4. پستگرس به عنوان پایگاه داده اصلی

PostgreSQL به‌عنوان Database Engine اصلی پروژه انتخاب شده است.

این انتخاب علاوه بر نیازهای فعلی، پروژه را به زیرساختی نزدیک می‌کند که برای محیط‌های واقعی Backend مناسب‌تر است.

در نتیجه، Database بخشی از معماری Production-oriented پروژه در نظر گرفته می‌شود و SQLite صرفاً به‌عنوان Database ساده‌ی اولیه کنار گذاشته شده است.

---

## 5. پستگرس محلی در مقابل ابری

در حالت ایده‌آل، PostgreSQL می‌تواند به‌صورت Local روی سیستم توسعه اجرا شود.

با این حال، به دلیل محدودیت محیط فعلی توسعه، استفاده از PostgreSQL محلی امکان‌پذیر نبود. برای جلوگیری از توقف توسعه‌ی پروژه، PostgreSQL روی سرویس Cloud **Neon** قرار گرفت.

در این معماری:

```text
Django Application
       │
       │ PostgreSQL connection
       ▼
     Neon
       │
       ▼
 PostgreSQL
```

Application نیازی به دانستن محل اجرای Database ندارد و تنها از طریق Configuration به آن متصل می‌شود.

این موضوع باعث می‌شود در آینده بتوان PostgreSQL را از Neon به یک PostgreSQL محلی یا سرویس دیگری منتقل کرد، بدون اینکه Business Logic پروژه تغییر کند.

---

## 6. مدیریت پیکربندی اطلاعات

اطلاعات اتصال Database نباید مستقیماً در `settings.py` یا Source Code قرار گیرند.

اطلاعاتی مانند:

* Database Name
* Username
* Password
* Host
* Port
* Database URL

از طریق Environment Configuration مدیریت می‌شوند.

در محیط توسعه از فایل `.env` استفاده شده و این فایل در `.gitignore` قرار گرفته است تا Credentials وارد Repository نشوند.

در معماری نهایی، Connection String به‌صورت یک متغیر:

```text
DATABASE_URL
```

در اختیار Application قرار می‌گیرد.

این رویکرد باعث می‌شود Configuration از Application Code جدا باشد.

---

## 7. استتفاده از Database_URL

برای اتصال به PostgreSQL، به‌جای Hard-code کردن اجزای Connection در `settings.py`، از یک `DATABASE_URL` استفاده شده است.

ساختار کلی:

```text
DATABASE_URL
      │
      ▼
Configuration Layer
      │
      ▼
Django DATABASES
      │
      ▼
PostgreSQL
```

این abstraction باعث می‌شود تغییر Environment تنها با تغییر Configuration انجام شود.

برای مثال، Application می‌تواند بدون تغییر کد از:

```text
Neon PostgreSQL
```

به:

```text
Local PostgreSQL
```

منتقل شود.

در این حالت Business Logic و Application Code بدون تغییر باقی می‌مانند و فقط مقدار `DATABASE_URL` تغییر می‌کند.

---

## 8. مدیریت وابستگی ها

برای ارتباط Django با PostgreSQL، Driver مربوط به PostgreSQL در Dependencies پروژه قرار گرفته است.

همچنین برای مدیریت Configuration و Database URL از ابزارهای مربوط به Environment Configuration استفاده شده است.

Dependencyهای مورد استفاده در `requirements.txt` ثبت می‌شوند تا Environment پروژه قابل بازسازی باشد.

---

## 9. مقداردهی اولیه پایگاه داده

پس از تغییر Database Engine، Schema پروژه از طریق Django Migrationها روی PostgreSQL ایجاد می‌شود.

Migrationها Source of Truth مربوط به Schema Application هستند و شامل ساخت جداول اصلی پروژه، از جمله:

* User
* Advertiser
* Publisher
* Campaign
* Ad
* Impression
* Click

می‌شوند.

در این مهاجرت، داده‌های تستی موجود در SQLite به PostgreSQL منتقل نشده‌اند و Database جدید از طریق Migrationها ساخته شده است؛ زیرا در این مرحله پروژه هنوز در مرحله‌ی MVP و دارای حجم محدودی از داده‌های آزمایشی است.

---

## 10. نکنه های مربوط به Connection Pooling

در استفاده از Neon، دو نوع Connection مورد توجه قرار گرفت:

* **Pooled Connection**
* **Direct Connection**

Connection Pooling برای Applicationهای دارای تعداد زیادی Connection می‌تواند مفید باشد، اما در برخی عملیات مدیریتی Database مانند ایجاد یا حذف Database تستی، وجود Connectionهای فعال از طریق Pooler می‌تواند مانع اجرای عملیات DDL شود.

به همین دلیل برای عملیات مربوط به ساخت یا حذف Test Database، استفاده از Direct Connection در نظر گرفته شد.

این موضوع یک تفکیک مهم بین:

**Application Runtime Connection**

و

**Database Administration / DDL Connection**

ایجاد می‌کند.

---

## 11. ملاحظات امنیتی

Credentials مربوط به Database بخشی از Source Code نیستند و باید خارج از Repository نگهداری شوند.

اصول اصلی این بخش:

* عدم Hard-code کردن Password در `settings.py`
* استفاده از Environment Variables
* قرار دادن `.env` در `.gitignore`
* عدم انتشار Connection String دارای Password در Repository
* استفاده از SSL برای Connection به Database Cloud

Connection String مورد استفاده در محیط Cloud دارای SSL Configuration است تا ارتباط Application با PostgreSQL به‌صورت رمزنگاری‌شده انجام شود.

---

## 12. مستقل کردن Application از محل اجرای Database

یکی از اهداف معماری این بخش، مستقل کردن Application از محل اجرای Database است.

Application تنها به یک Configuration Contract وابسته است:

```text
DATABASE_URL
```

بنابراین:

```text
                  ┌── Local PostgreSQL
                  │
Django Application ─── Neon PostgreSQL
                  │
                  └── Other PostgreSQL Provider
```

تغییر Infrastructure نباید نیازمند تغییر در Business Logic یا Application Layer باشد.

این اصل در آینده امکان انتقال پروژه به محیط‌هایی مانند Deployment Platformهای مختلف را نیز ساده‌تر می‌کند.

---

## 13. استراتژی Migration

به دلیل کوچک بودن Dataset در مرحله‌ی فعلی، Migration به PostgreSQL به‌صورت Clean Migration انجام شد:

```text
SQLite
  │
  │  Schema migration
  ▼
PostgreSQL
  │
  ▼
Django Migrations
  │
  ▼
Fresh Database Schema
```

داده‌های آزمایشی قبلی بخشی از Migration محسوب نمی‌شوند و در صورت نیاز می‌توانند مجدداً ایجاد شوند.

در پروژه‌ای با داده‌ی Production، این استراتژی کافی نخواهد بود و باید برای Data Migration، Backup، Verification و Rollback Strategy برنامه‌ریزی شود.

---

## 14. Validation After Migration

پس از انتقال Database، رفتار Application با اجرای Test Suite بررسی می‌شود.

هدف این تست‌ها تنها بررسی اتصال Django به PostgreSQL نیست؛ بلکه باید اطمینان حاصل شود Business Logic قبلی، از جمله:

* User & Role
* Campaign
* Ad
* Impression
* Click
* Budget Management
* Transactional Click Registration

روی Database جدید نیز رفتار مورد انتظار را حفظ می‌کنند.

به‌خصوص تست‌های مرتبط با Transaction و Concurrency باید روی Databaseای اجرا شوند که رفتار آن با معماری نهایی سیستم سازگار باشد.

---

## 15. دامنه و محدودیت‌های MVP

در این مرحله، PostgreSQL به‌عنوان Database اصلی انتخاب شده اما برخی موضوعات در سطح Production هنوز خارج از Scope این MVP هستند، از جمله:

* High-availability architecture
* Database replication
* Read replicas
* Advanced connection pooling strategy
* Automated backup and disaster recovery
* Database monitoring
* High-volume data partitioning
* Production-grade database migration strategy

این موارد در صورت افزایش Scale و نزدیک شدن پروژه به Production باید به‌صورت مستقل طراحی شوند.

---

## 16. تکامل زیرساخت‌های آینده

معماری فعلی این امکان را فراهم می‌کند که Database بدون تغییر Application Code جابه‌جا شود.

برای مثال:

```text
Current:
Django → Neon PostgreSQL

Future:
Django → Managed PostgreSQL
```

یا:

```text
Development:
Django → Local PostgreSQL

Production:
Django → Cloud PostgreSQL
```

در هر دو حالت، Application تنها از طریق `DATABASE_URL` به Database متصل می‌شود.

بنابراین تغییر Infrastructure به یک Configuration Change محدود می‌شود و نیاز به بازنویسی Application Logic ندارد.

---

## 17. تصمیم نهایی در مورد معماری

معماری Database پروژه بر اساس این تصمیم‌ها نهایی شده است:

| موضوع                      | تصمیم                           |
| -------------------------- | ------------------------------- |
| Database Engine            | PostgreSQL                      |
| Development Cloud Provider | Neon                            |
| Django Database Driver     | PostgreSQL Driver               |
| Configuration              | Environment Variables           |
| Connection Configuration   | `DATABASE_URL`                  |
| Secrets Storage            | `.env` خارج از Git              |
| Schema Management          | Django Migrations               |
| Concurrency Requirement    | Transaction + Row-Level Locking |
| Application Dependency     | مستقل از محل اجرای Database     |
| SQLite                     | حذف از معماری اصلی پروژه        |

### نتیجه

مهاجرت از SQLite به PostgreSQL بخشی از آماده‌سازی معماری پروژه برای **Concurrency، Transactional Workflows و محیط‌های نزدیک‌تر به Production** است.

در کنار انتخاب PostgreSQL، جداسازی Database Configuration از Application Code نیز باعث شده Infrastructure قابل تعویض باشد؛ به‌طوری که انتقال بین PostgreSQL محلی و Cloud Database نیازمند تغییر در Business Logic پروژه نباشد.

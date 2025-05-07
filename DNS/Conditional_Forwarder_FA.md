# DNS Conditional Forwarder  
## یک راهنمای آموزشی برای مدیران شبکه

---

## مقدمه

در محیط‌های شبکه‌ای پیچیده و توزیع‌شده، کنترل و بهینه‌سازی نام‌گذاری دامنه (DNS) نقش بسیار مهمی در بهبود عملکرد، امنیت و یکپارچگی سیستم‌ها ایفا می‌کند. یکی از قابلیت‌های پیشرفته در DNS ویندوز سرور، **Conditional Forwarding** یا *ارسال شرطی درخواست DNS* است.

---

## مفاهیم پایه

### DNS چیست؟

DNS (Domain Name System) سیستمی برای تبدیل نام‌های دامنه (مثل `example.com`) به آدرس‌های IP (مثل `93.184.216.34`) است.

### Forwarding در DNS

Forwarding یعنی ارسال درخواست‌های نام‌گذاری که سرور قادر به پاسخ‌گویی آن‌ها نیست، به یک سرور دیگر برای پاسخ.

---

## Conditional Forwarder چیست؟

Conditional Forwarder به سرور DNS این امکان را می‌دهد که فقط درخواست‌های مربوط به یک دامنه خاص (مثلاً `corpB.local`) را به یک DNS Server مشخص ارسال کند.

### ساختار عملکرد

اگر کاربر در شبکه A درخواست `server1.corpB.local` را داشته باشد، و یک Conditional Forwarder برای دامنه `corpB.local` تعریف شده باشد، DNS سرور شبکه A درخواست را مستقیماً به DNS سرور شبکه B هدایت می‌کند.

---

## مزایای Conditional Forwarder

- **کاهش بار شبکه**: فقط درخواست‌های دامنه خاص هدایت می‌شوند.
- **بهبود امنیت**: داده‌های DNS محدود به دامنه مورد نیاز باقی می‌مانند.
- **افزایش سرعت پاسخگویی**: مسیر مستقیم برای دامنه‌های هدف.
- **مناسب برای محیط‌های چنددامنه‌ای و سازمانی**.

---

## سناریوی کاربردی

دو سازمان با دامنه‌های زیر:

- `corpA.local` (DNS Server: 10.0.0.1)
- `corpB.local` (DNS Server: 192.168.1.1)

سازمان A برای دسترسی به منابع `corpB.local`، یک Conditional Forwarder به IP سرور DNS سازمان B تنظیم می‌کند.

---

## آموزش گام به گام در Windows Server

1. اجرای **DNS Manager** از Server Manager یا ابزار `dnsmgmt.msc`.
2. راست‌کلیک روی نام سرور و انتخاب **New Conditional Forwarder**.
3. وارد کردن نام دامنه (مثلاً `corpB.local`).
4. وارد کردن IP آدرس سرور مقصد (مثلاً `192.168.1.1`).
5. تأیید و ذخیره تنظیمات.

---

## نکات مهم

- اگر سرور مقصد قابل دسترس نباشد، درخواست با شکست مواجه می‌شود.
- DNS Forwarding با Conditional Forwarding هم‌زمان قابل استفاده است.
- بهتر است بین دامنه‌هایی که ارتباط مستقیم دارند از Conditional Forwarder استفاده شود.

---

## منابع بیشتر

- [Microsoft Docs - Conditional Forwarders](https://docs.microsoft.com/en-us/windows-server/networking/dns/deploy/conditional-forwarders)
- [RFC 1035 – Domain Names Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)

---

## نتیجه‌گیری

Conditional Forwarder ابزاری هوشمند برای بهینه‌سازی ترافیک DNS، افزایش امنیت، و مدیریت پیشرفته شبکه‌های پیچیده است. در دنیای شبکه‌های سازمانی، آشنایی و استفاده صحیح از این ویژگی می‌تواند مزیت رقابتی قابل توجهی ایجاد کند.

---

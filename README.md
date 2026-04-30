# 🌐 Kian CDN Netlify — رله اینترنت آزاد

راهنمای کامل راه‌اندازی **VLESS+XHTTP** از طریق **Netlify CDN** روی VPS خام — بدون دانش فنی، فقط کپی‌پیست

> **📖 صفحه تعاملی:** [kian-irani.github.io/Kian-CDN-Netlify](https://kian-irani.github.io/Kian-CDN-Netlify)

---

## درباره این روش

ترافیک از طریق **CDN جهانی Netlify** رد می‌شه. ISP فقط یه اتصال HTTPS عادی به یه سایت می‌بینه — نه VPN، نه پروکسی، نه چیز مشکوکی.

---

## معماری

```
📱 کاربر ایران
  ↓  VLESS+XHTTP over TLS (SNI: kubernetes.io)
🌐 Netlify CDN (Edge Function)
  ↓  HTTP relay به VPS
🖥️ VPS هلند/آلمان (3x-ui + Xray)
  ↓
🌍 اینترنت آزاد
```

---

## چرا این روش؟

| ویژگی | وضعیت |
|---|---|
| نیاز به دانش فنی | ❌ ندارد |
| نیاز به root | ❌ ندارد |
| هزینه اضافه | ❌ رایگان (Netlify Free Plan) |
| شناسایی توسط DPI | ❌ نمی‌شه |
| کار روی موبایل | ✅ کاملاً |
| تلگرام | ✅ کار می‌کنه |
| مرورگر | ✅ کار می‌کنه |
| YouTube | ✅ کار می‌کنه |
| WebSocket (Discord، ChatGPT streaming) | ⚠️ محدود |

---

## مراحل راه‌اندازی

> برای راهنمای کامل تعاملی با کپی خودکار دستورات → **[صفحه راهنما](https://kian-irani.github.io/Kian-CDN-Netlify)**

### ۱. نیازمندی‌ها

- یک VPS با Ubuntu 22.04 یا 24.04
- اکانت رایگان Netlify
- اپ SSH (Termius) روی موبایل

### ۲. نصب 3x-ui

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

در هنگام نصب:
- **SSL setup** → گزینه `2` (Let's Encrypt for IP)
- **Port for ACME** → `80` (نه ۸۰۸۰!)

### ۳. ساخت inbound خودکار

اسکریپت موجود در صفحه تعاملی، یک inbound VLESS+XHTTP روی پورت `8443` می‌سازه و مقادیر `UUID` و `PATH` رو نمایش می‌ده.

### ۴. دانلود ZIP و آپلود در Netlify

صفحه تعاملی یک فایل `netlify-relay.zip` می‌سازه که IP سرور از قبل داخلش تنظیم شده. فقط در [Netlify Drop](https://app.netlify.com/drop) آپلودش کن.

### ۵. کانفیگ نهایی VLESS

صفحه تعاملی کانفیگ کامل رو می‌سازه. در **v2rayNG** → دکمه `+` → **Import from Clipboard**.

---

## جزئیات فنی

### ساختار فایل Netlify

```
netlify-relay/
├── netlify.toml                    # Edge Function config
├── package.json
├── public/index.html               # Placeholder page
└── netlify/edge-functions/
    └── relay.js                    # Relay logic (IP hardcoded)
```

### کانفیگ VLESS

```
vless://{UUID}@kubernetes.io:443?
  mode=auto
  &path=%2F{PATH}
  &security=tls
  &alpn=h2%2Chttp%2F1.1
  &encryption=none
  &host={NETLIFY_DOMAIN}
  &fp=chrome
  &type=xhttp
  &sni=kubernetes.io
  #Netlify-Relay
```

---

## مشکلات رایج

### SSL در نصب 3x-ui fail شد
نگران نباش — TLS از طریق Netlify برقرار می‌شه، نه پنل. می‌تونی ادامه بدی.

### پورت ۸۰۸۰ در تداخل با سرویس دیگه‌ایه
در سؤال ACME port باید `80` وارد کنی، نه `8080`.

### Build failed در Netlify
مطمئن شو که فایل ZIP رو از صفحه تعاملی دانلود کردی و دستکاری نشده.

### اتصال برقرار نمی‌شه
با `ss -tlnp | grep 8443` چک کن که پورت ۸۴۴۳ روی سرور لیسن هست.

---

## ارتباط و پشتیبانی

- 📊 کانال فارکس: [@kian_forex](https://t.me/kian_forex)
- ✈️ تلگرام شخصی: [@Kian_irani_t](https://t.me/Kian_irani_t)
- 🐙 GitHub: [KIAN-IRANI](https://github.com/KIAN-IRANI)

---

## پروژه‌های مرتبط

| پروژه | لینک |
|---|---|
| 🛡️ mhrv-rs Full Tunnel (راه‌حل اول) | [mhrv-setup-full-tunell](https://github.com/KIAN-IRANI/mhrv-setup-full-tunell) |
| 🖥️ 3x-ui پنل | [MHSanaei/3x-ui](https://github.com/MHSanaei/3x-ui) |
| 📦 netlify-relay اصلی | [amirshaker000/netlify-relay](https://github.com/amirshaker000/netlify-relay) |

---

*ساخته‌شده توسط **Kian Irani** — آخرین به‌روزرسانی: آوریل ۲۰۲۶*

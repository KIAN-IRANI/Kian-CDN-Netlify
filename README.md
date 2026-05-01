# 🌐 Kian CDN Netlify — رله اینترنت آزاد

راهنمای کامل راه‌اندازی **VLESS+XHTTP** از طریق **Netlify CDN** روی VPS خام

> **📖 صفحه تعاملی:** [kian-irani.github.io/Kian-CDN-Netlify](https://kian-irani.github.io/Kian-CDN-Netlify)

---

## چرا این روش؟

ترافیک از طریق **CDN جهانی Netlify** رد می‌شه. ISP فقط یه اتصال HTTPS عادی به یه سایت می‌بینه — نه VPN، نه پروکسی.

```
📱 کاربر ایران
  ↓  VLESS+XHTTP over TLS
🌐 Netlify Edge Function
  ↓  HTTP relay
🖥️ VPS (3x-ui + Xray)
  ↓
🌍 اینترنت آزاد
```

---

## نکات مهم (از تجربه)

### ✅ چیزهایی که کار می‌کنه
- تلگرام، مرورگر، YouTube، اینستاگرام
- بدون root، بدون certificate روی گوشی
- رایگان (Netlify Free Plan)

### ❌ چیزهایی که کار نمی‌کنه
- Discord Voice (WebSocket)
- ChatGPT Streaming
- هر سرویسی که WebSocket نیاز داشته باشه

### ⚠️ نکات فنی مهم

**۱. ساختار ZIP برای Netlify Drop**
فایل‌ها باید **مستقیم در root ZIP** باشن — نه داخل پوشه:
```
✅ درست:
relay.zip/
  netlify.toml
  public/index.html
  netlify/edge-functions/relay.js

❌ اشتباه:
relay.zip/
  my-project/          ← این پوشه اضافه مشکل می‌سازه
    netlify.toml
    ...
```

**۲. روش Netlify Drop در مقابل GitHub Import**
- **Netlify Drop** → نمی‌شه Environment Variable تنظیم کرد → IP باید hardcode باشه
- **GitHub Import** → می‌شه `TARGET_DOMAIN` رو به عنوان env var تنظیم کرد → بهتر

**۳. پروتکل داخلی: HTTP نه HTTPS**
Netlify Edge Function باید با **HTTP** به VPS وصل بشه:
```
TARGET_DOMAIN = http://YOUR_IP:8080    ✅
TARGET_DOMAIN = https://YOUR_IP:8443   ❌ (SSL cert لازم داره)
```

**۴. پورت 8080 ممکنه توسط Docker گرفته شده باشه**
قبل از شروع چک کن:
```bash
ss -tlnp | grep 8080
docker ps 2>/dev/null
```
اگه Docker روی 8080 بود، از پورت دیگه‌ای استفاده کن.

**۵. مشکل نصب 3x-ui**
وقتی با `tar` نصب می‌کنی، ساختار اینطوریه:
```
/usr/local/x-ui/x-ui/x-ui   ← فایل اجرایی واقعی اینجاست!
```
باید محتوای پوشه داخلی رو به بالا بیاری:
```bash
cp -r /usr/local/x-ui/x-ui/* /usr/local/x-ui/
rm -rf /usr/local/x-ui/x-ui
chmod +x /usr/local/x-ui/x-ui
```

**۶. DB x-ui بلافاصله بعد از نصب ساخته نمی‌شه**
باید چند ثانیه صبر کنی:
```bash
systemctl start x-ui
sleep 5
# حالا /etc/x-ui/x-ui.db وجود داره
```

**۷. فایروال**
برای راحتی کار، فایروال رو غیرفعال کن (نه پاک):
```bash
ufw disable
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```

**۸. Fork کردن ریپو**
مستقیم Fork نکن — اسم ریپو رو بعد از Fork تغییر بده تا اکانت Netlify ساسپند نشه.

---

## راه‌اندازی سریع

### پیش‌نیازها
- یک VPS با Ubuntu 22.04 یا 24.04
- اکانت GitHub (رایگان)
- اکانت Netlify (رایگان)

### مرحله ۱ — نصب خودکار روی VPS

```bash
python3 << 'EOF'
import subprocess, os, time, json, random, string

def run(cmd, capture=False):
    return subprocess.run(cmd, shell=True, capture_output=capture, text=True)

# غیرفعال کردن فایروال
run("ufw disable 2>/dev/null || true")
run("iptables -P INPUT ACCEPT 2>/dev/null || true")
run("iptables -P FORWARD ACCEPT 2>/dev/null || true")
run("iptables -P OUTPUT ACCEPT 2>/dev/null || true")

# نصب ابزارها
run("apt-get update -qq && apt-get install -y -qq wget sqlite3 ca-certificates python3")

# نصب x-ui
run("rm -rf /tmp/xui && mkdir /tmp/xui")
run("wget -qO /tmp/xui.tar.gz https://github.com/MHSanaei/3x-ui/releases/download/v2.9.3/x-ui-linux-amd64.tar.gz")
run("tar -xzf /tmp/xui.tar.gz -C /tmp/xui/")
real = run("find /tmp/xui -name 'x-ui' -type f", capture=True).stdout.strip().split("\n")[0]
real_dir = os.path.dirname(real)
run("rm -rf /usr/local/x-ui && mkdir -p /usr/local/x-ui")
run(f"cp -r {real_dir}/* /usr/local/x-ui/")
run("chmod +x /usr/local/x-ui/x-ui")

svc = """[Unit]
Description=x-ui Service
After=network.target
[Service]
Type=simple
User=root
WorkingDirectory=/usr/local/x-ui/
ExecStart=/usr/local/x-ui/x-ui
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target"""
with open("/etc/systemd/system/x-ui.service","w") as f: f.write(svc)
run("systemctl daemon-reload && systemctl enable x-ui && systemctl start x-ui")
time.sleep(5)

DB = "/etc/x-ui/x-ui.db"
for _ in range(15):
    if os.path.exists(DB) and os.path.getsize(DB) > 1000: break
    time.sleep(2)

run("/usr/local/x-ui/x-ui setting -port 27925 -username kianadmin -password 'NetKian@2025!'")
run("systemctl restart x-ui && sleep 3")

NEW_UUID = run("cat /proc/sys/kernel/random/uuid", capture=True).stdout.strip()
NEW_PATH = "relay-" + "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
PORT = 8080

settings = json.dumps({"clients":[{"id":NEW_UUID,"flow":"","email":"user","limitIp":0,"totalGB":0,"expiryTime":0,"enable":True,"tgId":"","subId":"","reset":0}],"decryption":"none","fallbacks":[]})
stream = json.dumps({"network":"xhttp","security":"none","externalProxy":[],"xhttpSettings":{"path":f"/{NEW_PATH}","host":"","headers":{},"scMaxBufferedPosts":30,"scMaxEachPostBytes":"1000000","noSSEHeader":False,"xPaddingBytes":"100-1000","mode":"auto"}})
sniffing = json.dumps({"enabled":True,"destOverride":["http","tls","quic","fakedns"],"metadataOnly":False,"routeOnly":False})

def sq(s): return s.replace("'","''")
sql = f"INSERT INTO inbounds (user_id,up,down,total,all_time,remark,enable,expiry_time,traffic_reset,last_traffic_reset_time,listen,port,protocol,settings,stream_settings,tag,sniffing) VALUES (1,0,0,0,0,'Netlify-Relay',1,0,'never',0,'',{PORT},'vless','{sq(settings)}','{sq(stream)}','inbound-{PORT}','{sq(sniffing)}');"
with open("/tmp/ib.sql","w") as f: f.write(sql)
run("systemctl stop x-ui && sleep 1")
run(f"sqlite3 '{DB}' < /tmp/ib.sql")
run("systemctl start x-ui && sleep 2")

print(f"UUID={NEW_UUID}")
print(f"PATH={NEW_PATH}")
print(f"PORT={PORT}")
EOF
```

### مرحله ۲ — Fork ریپو در GitHub

۱. به [github.com/amirshaker000/netlify-relay](https://github.com/amirshaker000/netlify-relay) برو
۲. Fork کن
۳. Settings → اسم ریپو رو عوض کن

### مرحله ۳ — Deploy در Netlify

۱. [app.netlify.com/start](https://app.netlify.com/start) → Import from GitHub
۲. ریپوی fork شده رو انتخاب کن
۳. Build command: `npm run build` | Publish: `public`
۴. Environment variables → Add:
   - Key: `TARGET_DOMAIN`
   - Value: `http://YOUR_VPS_IP:8080`
۵. Deploy site

### مرحله ۴ — کانفیگ VLESS

```
vless://YOUR_UUID@kubernetes.io:443?mode=auto&path=%2FYOUR_PATH&security=tls&alpn=h2%2Chttp%2F1.1&encryption=none&insecure=0&host=YOUR_NETLIFY_DOMAIN.netlify.app&fp=chrome&type=xhttp&allowInsecure=0&sni=kubernetes.io#Netlify-Relay
```

---

## SNI های جایگزین

اگه `kubernetes.io` فیلتر شد، اینا رو امتحان کن:
- `helm.sh`
- `letsencrypt.org`
- `www.google.com`

---

## عیب‌یابی

| مشکل | راه‌حل |
|---|---|
| Bad Gateway | `TARGET_DOMAIN` باید `http://` باشه نه `https://` |
| -1ms در v2rayNG | v2rayNG رو آپدیت کن یا Hiddify امتحان کن |
| پورت 8080 بسته | Docker ممکنه اون رو گرفته باشه — چک کن |
| Edge Function نشون نمی‌ده | ساختار ZIP رو چک کن — فایل‌ها باید در root باشن |
| x-ui اجرا نمی‌شه | فایل اجرایی ممکنه داخل پوشه داخلی باشه |
| اکانت Netlify ساسپند شد | Fork مستقیم کردی — اسم ریپو رو عوض کن |

---

## پروژه‌های مرتبط

| پروژه | لینک |
|---|---|
| 🛡️ mhrv-rs Full Tunnel | [mhrv-setup-full-tunell](https://github.com/KIAN-IRANI/mhrv-setup-full-tunell) |
| 📦 netlify-relay اصلی | [amirshaker000/netlify-relay](https://github.com/amirshaker000/netlify-relay) |
| 🖥️ 3x-ui پنل | [MHSanaei/3x-ui](https://github.com/MHSanaei/3x-ui) |

---

## ارتباط

- 📊 کانال: [@kian_forex](https://t.me/kian_forex)
- ✈️ تلگرام: [@Kian_irani_t](https://t.me/Kian_irani_t)
- 🐙 GitHub: [KIAN-IRANI](https://github.com/KIAN-IRANI)

---

*آخرین به‌روزرسانی: می ۲۰۲۶*

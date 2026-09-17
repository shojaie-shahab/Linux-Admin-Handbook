# سناریوهای عملی ۱۲: Security

> پیش‌نیاز مطالعه: `theory/12-Security.md`
> هر دستور بلافاصله بعد از خودش یک بخش «🔍 تحلیل دستور» دارد که تک‌تک سوییچ‌های آن را جداگانه توضیح می‌دهد.

---

## سناریو ۱ (مبتدی): تنظیم ACL ساده روی یک فایل مشترک (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده SELinux/فایروال، باید مفهوم پایه ACL را با یک مثال ساده لمس کنید.

### قدم‌به‌قدم

**۱. ساخت فایل تست**
```bash
touch /tmp/shared-report.txt
```

**۲. اعطای دسترسی نوشتن به یک کاربر خاص**
```bash
setfacl -m u:alice:rwx /tmp/shared-report.txt
```
🔍 **تحلیل دستور:** `-m`: مخفف `--modify` — اضافه/تغییر یک قانون؛ `u:alice:rwx`: فرمت سه‌بخشی — نوع (`u` برای کاربر)، نام (`alice`)، مجوز (`rwx`).

**۳. اعطای دسترسی فقط‌خواندن به یک گروه**
```bash
setfacl -m g:finance:r-- /tmp/shared-report.txt
```
🔍 **تحلیل دستور:** همان ساختار، این‌بار `g` برای گروه؛ `r--` یعنی فقط بیت خواندن فعال، نوشتن و اجرا غیرفعال.

**۴. مشاهده تمام قوانین فعلی**
```bash
getfacl /tmp/shared-report.txt
```
🔍 **تحلیل دستور:** بدون سوییچ؛ نمایش کامل مجوزهای کلاسیک rwx به‌همراه تمام قوانین ACL اضافه‌شده.

**۵. حذف یک قانون خاص**
```bash
setfacl -x u:alice /tmp/shared-report.txt
```
🔍 **تحلیل دستور:** `-x`: مخفف `--remove` — حذف فقط همان یک قانون مشخص (کاربر alice)، بدون تاثیر روی بقیه قوانین.

### ✅ Best Practice
همیشه بعد از هر `setfacl -m`، با `getfacl` نتیجه را تایید کنید.

---

## سناریو ۲ (مبتدی): باز کردن یک پورت ساده در firewalld (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده nftables/DNAT، باید ساده‌ترین کار روزمره فایروال را مسلط باشید.

### قدم‌به‌قدم

**۱. بررسی zone فعال فعلی**
```bash
firewall-cmd --get-active-zones
```
🔍 **تحلیل دستور:** بدون آرگومان اضافه؛ نمایش این‌که کدام اینترفیس به کدام zone تعلق دارد.

**۲. بررسی قوانین فعلی یک zone**
```bash
firewall-cmd --zone=public --list-all
```
🔍 **تحلیل دستور:** `--zone=public`: مشخص کردن صریح zone هدف؛ `--list-all`: نمایش کامل سرویس‌ها/پورت‌های مجاز آن zone.

**۳. افزودن یک سرویس شناخته‌شده (موقت، فقط runtime)**
```bash
firewall-cmd --zone=public --add-service=<" http ">
```
🔍 **تحلیل دستور:** بدون `--permanent`؛ این تغییر فقط تا reload/reboot بعدی باقی می‌ماند — مناسب تست سریع.

**۴. افزودن دائمی یک پورت خاص**
```bash
firewall-cmd --zone=public --add-port=8080/tcp --permanent
```
🔍 **تحلیل دستور:** `--add-port=8080/tcp`: فرمت `شماره/پروتکل`؛ `--permanent`: ذخیره در تنظیمات دائمی (اما فوراً در runtime اعمال نمی‌شود).

**۵. اعمال نهایی**
```bash
firewall-cmd --reload
```
🔍 **تحلیل دستور:** بدون سوییچ اضافه؛ تمام تغییرات `--permanent` ذخیره‌شده را در runtime فعلی هم بارگذاری می‌کند.

### 📌 نکته آزمون
بدون `--permanent` موقتی، بدون `--reload` بعد از `--permanent` فوراً اعمال نمی‌شود.

---

## سناریو ۳: عیب‌یابی صحیح یک سرویس مسدودشده توسط SELinux (سطح متوسط تا پیشرفته)

### قدم‌به‌قدم

**۱. تایید وضعیت فعلی**
```bash
getenforce
```

**۲. تشخیص موقت با Permissive**
```bash
setenforce 0
curl -I http://localhost/
setenforce 1
```
🔍 **تحلیل دستور:** `setenforce 0`/`1`: بدون سوییچ دیگر، فقط یک عدد آرگومان — `0` یعنی Permissive، `1` یعنی Enforcing؛ این تغییر فقط تا ریبوت بعدی یا فراخوانی مجدد باقی می‌ماند (موقت).

**۳. بررسی دقیق لاگ Audit**
```bash
ausearch -m avc -ts recent | grep nginx
```
🔍 **تحلیل دستور:** `-m avc`: مخفف `--message` — نوع پیام مورد جستجو (Access Vector Cache، رکوردهای تصمیم SELinux)؛ `-ts recent`: مخفف `--time-start` — فقط رویدادهای اخیر (نه کل تاریخچه).

**۴. رفع صحیح با تعریف Context دائمی**
```bash
semanage fcontext -a -t httpd_sys_content_t "/data/webcontent(/.*)?"
```
🔍 **تحلیل دستور:** `-a`: مخفف `--add` — افزودن یک قانون جدید؛ `-t`: مخفف `--type` — Type هدف؛ الگوی مسیر `(/.*)?` یعنی این مسیر و تمام زیرپوشه‌هایش.

**۵. اعمال فوری روی فایل‌های موجود**
```bash
restorecon -Rv /data/webcontent
```
🔍 **تحلیل دستور:** `-R`: مخفف `--recursive` — اعمال روی تمام زیرپوشه‌ها؛ `-v`: مخفف `--verbose` — نمایش هر فایلی که Context آن تغییر کرد.

### ⚠️ نکته امنیتی حیاتی
هرگز `setenforce 0` را به‌عنوان راه‌حل دائمی نگه ندارید — فقط برای تشخیص لحظه‌ای.

---

## سناریو ۴: طراحی فایروال کامل با nftables برای سرور چندنقشه (سطح پیشرفته)

### قدم‌به‌قدم

**۱. ساخت جدول و chain با policy پیش‌فرض امن**
```bash
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
```
🔍 **تحلیل دستور:** `inet`: خانواده آدرس که هم IPv4 و هم IPv6 را هم‌زمان پوشش می‌دهد؛ `hook input`: این chain به نقطه ورودی بسته‌ها متصل است؛ `priority 0`: ترتیب اجرا نسبت به سایر chain های احتمالی روی همان hook؛ `policy drop`: رفتار پیش‌فرض اگر هیچ قانونی مطابقت نداشت.

**۲. قوانین پایه**
```bash
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept
```
🔍 **تحلیل دستور:** `iif lo`: مخفف `--input-interface` — فقط بسته‌هایی که از اینترفیس loopback می‌آیند؛ `ct state established,related`: بررسی Connection Tracking، دو وضعیت با کاما (نه فاصله) جدا شده‌اند.

**۳. محدود کردن SSH به شبکه مدیریتی**
```bash
nft add rule inet filter input ip saddr 10.0.9.0/24 tcp dport 22 accept
```
🔍 **تحلیل دستور:** `ip saddr 10.0.9.0/24`: مخفف `source address` — فقط بسته‌هایی با IP مبدا در این محدوده؛ ترکیب با `tcp dport 22` (مقصد پورت ۲۲) یعنی هر دو شرط باید هم‌زمان برقرار باشند.

**۴. لاگ کردن بسته‌های رد‌شده قبل از drop نهایی**
```bash
nft add rule inet filter input log prefix \"nftables-dropped: \" drop
```
🔍 **تحلیل دستور:** `log prefix "..."`: قبل از drop کردن، یک خط لاگ با این پیشوند مشخص در سیستم لاگ کرنل ثبت می‌شود — پیشوند برای فیلتر راحت‌تر بعدی با `grep`.

**۵. ذخیره دائمی**
```bash
nft list ruleset > /etc/nftables.conf
```
🔍 **تحلیل دستور:** `list ruleset` بدون سوییچ، خروجی متنی کامل تمام قوانین فعلی را چاپ می‌کند که با `>` مستقیماً در فایل تنظیمات ذخیره می‌شود.

### ✅ Best Practice
همیشه از خاص‌ترین به عمومی‌ترین قانون بچینید و یک قانون لاگ‌گیری قبل از policy نهایی drop قرار دهید.

---

## سناریو ۵: پیاده‌سازی Port Forwarding و NAT با nftables (سطح پیشرفته)

### قدم‌به‌قدم

**۱. فعال‌سازی IP Forwarding**
```bash
sysctl -w net.ipv4.ip_forward=1
```

**۲. تعریف DNAT در prerouting**
```bash
nft add table inet nat
nft add chain inet nat prerouting { type nat hook prerouting priority -100 \; }
nft add rule inet nat prerouting tcp dport 8080 dnat to 10.0.1.10:80
```
🔍 **تحلیل دستور:** `hook prerouting`: این chain **قبل از** تصمیم مسیریابی کرنل اجرا می‌شود؛ `dnat to IP:PORT`: تغییر آدرس/پورت مقصد بسته به یک مقصد داخلی جدید.

**۳. ا Masquerade در postrouting برای مسیر برگشت**
```bash
nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule inet nat postrouting oif eth0 masquerade
```
🔍 **تحلیل دستور:** `oif eth0`: مخفف `--output-interface` — فقط بسته‌های خروجی از این اینترفیس؛ `masquerade` بدون آرگومان، آدرس مبدا را خودکار به IP همین اینترفیس تغییر می‌دهد.

**۴. اجازه عبور از FORWARD**
```bash
nft add rule inet filter forward ip daddr 10.0.1.10 tcp dport 80 accept
```
🔍 **تحلیل دستور:** `ip daddr`: مخفف `destination address` — این قانون باید جداگانه (علاوه بر DNAT) اضافه شود چون DNAT فقط آدرس را تغییر می‌دهد، تصمیم عبور/رد را کنترل نمی‌کند.

### ⚠️ نکته حیاتی
بدون Masquerade، پاسخ سرور داخلی مستقیم به کلاینت می‌رود (نه از طریق Gateway) و ارتباط شکست می‌خورد.

---

## سناریو ۶: دفاع در عمق ترکیبی — ACL + SELinux + فایروال (سطح پیشرفته، جمع‌بندی)

### قدم‌به‌قدم

**۱. لایه اول — ACL دقیق**
```bash
setfacl -d -m g:finance-uploaders:rwx /srv/finance-uploads
```
🔍 **تحلیل دستور:** `-d`: مخفف `--default` — این قانون نه فقط روی خود پوشه، بلکه به‌صورت **پیش‌فرض** روی هر فایل/زیرپوشه‌ای که در آینده داخل این پوشه ساخته شود هم اعمال می‌شود.

**۲. لایه دوم — Context اختصاصی SELinux**
```bash
semanage fcontext -a -t httpd_sys_rw_content_t "/srv/finance-uploads(/.*)?"
restorecon -Rv /srv/finance-uploads
```

**۳. لایه سوم — Boolean محدودکننده**
```bash
setsebool -P httpd_can_network_connect off
```
🔍 **تحلیل دستور:** `-P`: مخفف `--persistent` روی `setsebool` — تغییر را دائمی (حتی بعد از ریبوت) می‌کند؛ بدون آن، تغییر فقط موقت است.

**۴. لایه چهارم — محدودیت فایروال**
```bash
nft add rule inet filter input ip saddr 10.0.0.0/16 tcp dport 443 accept
```

**۵. لایه پنجم — Sandboxing سطح systemd**
```bash
systemctl edit nginx.service
```
با محتوای:
```ini
[Service]
ProtectSystem=strict
ReadWritePaths=/srv/finance-uploads
NoNewPrivileges=true
```
🔍 **تحلیل تنظیمات:** `ProtectSystem=strict`: کل فایل‌سیستم پیش‌فرض فقط‌خواندنی می‌شود؛ `ReadWritePaths=`: استثنای صریح فقط برای این یک مسیر؛ `NoNewPrivileges=true`: جلوگیری از کسب امتیاز بیشتر حتی در صورت سوءاستفاده از یک باگ.

### 📌 نکته آزمون نهایی
هیچ لایه امنیتی واحد را به‌عنوان تنها خط دفاعی در نظر نگیرید — نفوذ موفق باید نیازمند دور زدن همه لایه‌ها هم‌زمان باشد.

---

## جمع‌بندی مهارت‌های این بخش

- تنظیم پایه ACL با `setfacl -m`/`-x` و تحلیل فرمت `u:نام:مجوز`
- کار پایه firewalld با تمایز دقیق `--permanent` و `--reload`
- عیب‌یابی صحیح SELinux با `ausearch` و رفع با `semanage fcontext`+`restorecon`
- طراحی فایروال کامل nftables با تحلیل عمیق `hook`/`priority`/`policy`
- پیاده‌سازی DNAT+Masquerade با درک چرایی نیاز هم‌زمان به هر دو
- معماری دفاع در عمق پنج‌لایه با تحلیل کامل هر تنظیم

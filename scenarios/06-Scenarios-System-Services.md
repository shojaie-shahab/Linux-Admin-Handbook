# سناریوهای عملی ۶: System Services

> پیش‌نیاز مطالعه: `theory/06-System-Services.md`
> هر دستور بلافاصله بعد از خودش یک بخش «🔍 تحلیل دستور» دارد که تک‌تک سوییچ‌های آن را جداگانه توضیح می‌دهد.

---

## سناریو ۱ (مبتدی): ساخت یک systemd Service ساده از صفر و مشاهده چرخه کامل آن (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده Socket Activation و Slice، باید یک بار کامل ساخت یک Unit ساده از صفر تا اجرا و توقف را تجربه کنید.

### قدم‌به‌قدم

**۱. ساخت یک اسکریپت ساده**
```bash
vi /usr/local/bin/hello-loop.sh

#!/bin/bash
while true; do
    echo "Hello at $(date)"
    sleep 10
done

```
```bash
chmod +x /usr/local/bin/hello-loop.sh
```
🔍 **تحلیل دستور:** `chmod +x` بیت اجرایی را اضافه می‌کند — بدون این، systemd هرگز نمی‌تواند این فایل را به‌عنوان یک برنامه اجرا کند، حتی اگر مسیر در `ExecStart=` درست باشد.

**۲. ساخت فایل Unit**

```bash
vi /etc/systemd/system/hello-loop.service 

[Unit]
Description=Simple Hello Loop Service

[Service]
ExecStart=/usr/local/bin/hello-loop.sh

[Install]
WantedBy=multi-user.target

```

**۳. بارگذاری مجدد systemd**
```bash
systemctl daemon-reload
```
🔍 **تحلیل دستور:** بدون سوییچ؛ این دستور systemd را وادار می‌کند فایل‌های Unit روی دیسک را دوباره بخواند — بدون آن، فایل جدیدی که ساختیم اصلاً شناخته نمی‌شود.

**۴. استارت و بررسی وضعیت**
```bash
systemctl start hello-loop
systemctl status hello-loop
```
🔍 **تحلیل دستور:** `start` فقط همین الان اجرا می‌کند (بدون فعال‌سازی برای بوت‌های بعدی)؛ `status` (بدون سوییچ) خلاصه‌ای از وضعیت فعلی، چند خط آخر لاگ، و PID اصلی را نشان می‌دهد.

**۵. مشاهده لاگ زنده**
```bash
journalctl -u hello-loop -f
```
🔍 **تحلیل دستور:** `-u hello-loop` فیلتر فقط همین سرویس؛ `-f` مخفف `--follow` — حالت زنده مثل `tail -f`.

**۶. فعال‌سازی برای بوت‌های بعدی**
```bash
systemctl enable hello-loop
```
🔍 **تحلیل دستور:** بدون سوییچ اضافه؛ `enable` یک symlink در پوشه Target مربوطه (`multi-user.target.wants/`) می‌سازد — این جداست از `start`؛ `enable` بدون `start` یعنی «در بوت بعدی اجرا شود اما همین الان نه».

**۷. توقف و غیرفعال‌سازی**
```bash
systemctl stop hello-loop
systemctl disable hello-loop
```

### ✅ Best Practice
همیشه بعد از ساخت/ویرایش هر فایل Unit، `daemon-reload` را فراموش نکنید — یکی از رایج‌ترین اشتباهات مبتدیان.

---

## سناریو ۲ (مبتدی): ساخت یک systemd Timer ساده به‌جای cron (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده Path Unit، باید ساده‌ترین جایگزین cron را با systemd Timer تجربه کنید.

### قدم‌به‌قدم

**۱. اسکریپت ساده**
```bash
cat > /usr/local/bin/log-uptime.sh << 'EOF'
#!/bin/bash
echo "$(date): $(uptime)" >> /var/log/uptime-check.log
EOF
chmod +x /usr/local/bin/log-uptime.sh
```

**۲. سرویس oneshot**
```bash
cat > /etc/systemd/system/log-uptime.service << 'EOF'
[Unit]
Description=Log system uptime

[Service]
Type=oneshot
ExecStart=/usr/local/bin/log-uptime.sh
EOF
```
🔍 **تحلیل دستور:** `Type=oneshot` مشخص می‌کند این سرویس یک کار کوتاه انجام می‌دهد و تمام می‌شود (نه یک برنامه پیوسته در حال اجرا مثل سناریو ۱) — systemd صبر می‌کند این پردازش کامل تمام شود، سپس آن را «موفق» علامت می‌زند.

**۳. تعریف Timer**
```bash
cat > /etc/systemd/system/log-uptime.timer << 'EOF'
[Unit]
Description=Run log-uptime every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF
```
🔍 **تحلیل دستور:** `OnCalendar=*:0/5` یک فرمت تقویمی systemd — `*` یعنی هر ساعت، `0/5` یعنی از دقیقه صفر شروع کن و هر ۵ دقیقه تکرار کن؛ `Persistent=true` یعنی اگر سیستم در زمان اجرا خاموش بود، بلافاصله بعد از روشن شدن جبران شود.

**۴. فعال‌سازی Timer (نه سرویس)**
```bash
systemctl daemon-reload
systemctl enable --now log-uptime.timer
```
🔍 **تحلیل دستور:**
- ا `enable --now`: ترکیب دو عملیات در یک دستور — هم فعال‌سازی دائمی (برای بوت‌های بعدی) و هم استارت فوری همین الان؛ معادل اجرای `enable` و `start` جداگانه.
- دقت کنید فایل فعال‌شده `.timer` است، نه `.service` — Timer خودش مسئول فراخوانی سرویس هم‌نام در زمان مقرر است.

**۵. بررسی زمان‌بندی بعدی**
```bash
systemctl list-timers log-uptime.timer
```
🔍 **تحلیل دستور:** نمایش دقیق «چه زمانی آخرین بار اجرا شد» و «چه زمانی بعدی اجرا خواهد شد».

### 📌 نکته آزمون
ا systemd Timer نسبت به cron مزیت لاگ‌گیری بهتر (از طریق `journalctl -u`) و مدیریت خطای یکپارچه‌تر دارد.

---

## سناریو ۳: تبدیل یک سرویس همیشه-فعال به Socket Activation (سطح پیشرفته)

### قدم‌به‌قدم

**۱. بررسی مصرف پایه فعلی**
```bash
systemd-cgtop -n 1 | grep -E "api-.*\.service"
```
🔍 **تحلیل دستور:** `-n 1`: مخفف `--iterations 1` روی `systemd-cgtop` — فقط یک‌بار نمونه‌برداری کن و خارج شو (به‌جای رفرش زنده بی‌نهایت).

**۲. ساخت Socket Unit**
```bash
cat > /etc/systemd/system/api-inventory.socket << 'EOF'
[Socket]
ListenStream=9001
Accept=no

[Install]
WantedBy=sockets.target
EOF
```
🔍 **تحلیل دستور:** `Accept=no` یعنی یک نمونه واحد از سرویس مسئول مدیریت چندین اتصال هم‌زمان است (روش مدرن و توصیه‌شده).

**۳. وابسته کردن سرویس به سوکت با Drop-in**
```bash
mkdir -p /etc/systemd/system/api-inventory.service.d
cat > /etc/systemd/system/api-inventory.service.d/socket-activation.conf << 'EOF'
[Unit]
Requires=api-inventory.socket
EOF
```
🔍 **تحلیل دستور:** ساخت پوشه‌ای با نام دقیق `<unit-name>.service.d/` — این یک قرارداد اجباری نام‌گذاری systemd است؛ هر فایل `.conf` داخل آن با فایل اصلی Unit ادغام می‌شود.

**۴. غیرفعال کردن راه‌اندازی مستقیم سرویس**
```bash
systemctl disable api-inventory.service
systemctl enable --now api-inventory.socket
```
🔍 **تحلیل دستور:** حالا سوکت (نه سرویس) چیزی است که در بوت فعال می‌شود.

**۵. تست فعال‌سازی خودکار**
```bash
curl -s http://localhost:9001/health
systemctl status api-inventory.service
```
🔍 **تحلیل دستور:** `curl -s`: مخفف `--silent` — بدون نمایش نوار پیشرفت دانلود، فقط خروجی خام پاسخ سرور.

### 📌 نکته آزمون
ا Socket Activation برای سرویس‌های با الگوی استفاده متناوب بیشترین سود را دارد.

---

## سناریو ۴: پردازش موازی صف پیام با Template Unit و Slice (سطح پیشرفته)

### قدم‌به‌قدم

**۱. ساخت Slice**
```bash
cat > /etc/systemd/system/queue-workers.slice << 'EOF'
[Slice]
CPUQuota=150%
MemoryMax=2G
EOF
```
🔍 **تحلیل دستور:** `CPUQuota=150%` سقف مجموع مصرف CPU **کل گروه** (نه هر عضو جداگانه) معادل ۱.۵ هسته.

**۲. ساخت Template Unit**
```bash
cat > /etc/systemd/system/queue-worker@.service << 'EOF'
[Unit]
Description=Queue Worker Instance %i
PartOf=queue-workers.slice

[Service]
Slice=queue-workers.slice
ExecStart=/usr/local/bin/queue-worker.sh %i
Restart=on-failure
EOF
```
🔍 **تحلیل دستور:**
- ا `%i` در نام Description و در آرگومان `ExecStart`: یک نماد ویژه Template Unit که در لحظه فعال‌سازی با مقدار بعد از `@` جایگزین می‌شود.
- ا `PartOf=`: اعلام می‌کند این Unit «بخشی از» Slice است؛ اگر Slice متوقف شود، این هم متوقف می‌شود.

**۳. راه‌اندازی چند نمونه**
```bash
for i in 1 2 3 4; do
    systemctl enable --now queue-worker@${i}.service
done
```
🔍 **تحلیل دستور:** `queue-worker@${i}.service` — متغیر شل `${i}` قبل از اجرا با مقدار عددی جایگزین می‌شود، پس چهار Unit مجزا (`@1`, `@2`, `@3`, `@4`) فعال می‌شوند، همه از یک الگوی مشترک.

**۴. مشاهده ساختار سلسله‌مراتبی**
```bash
systemd-cgls /queue-workers.slice
```
🔍 **تحلیل دستور:** آرگومان مسیر Slice به‌جای بدون آرگومان (که کل درخت سیستم را نشان می‌داد) — محدود کردن نمایش فقط به زیرمجموعه این یک Slice.

### ✅ Best Practice
ترکیب Template Unit + Slice برای معماری‌های worker/queue بسیار قدرتمند است.

---

## سناریو ۵: خودکارسازی نظافت پوشه موقت با systemd-tmpfiles و Path Unit (سطح متوسط)

### قدم‌به‌قدم

**۱. تعریف قانون tmpfiles**
```bash
cat > /etc/tmpfiles.d/app-uploads.conf << 'EOF'
d /var/tmp/app-uploads 0750 appuser appuser 1d
EOF
```
🔍 **تحلیل دستور:** فرمت خط: نوع عملیات (`d` = اطمینان از وجود پوشه)، مسیر، مجوز (`0750`)، مالک، گروه، و مدت نگهداری (`1d` = یک روز، بعد از آن فایل‌های داخل خودکار پاک می‌شوند).

**۲. اجرای دستی فوری برای تست**
```bash
systemd-tmpfiles --clean /etc/tmpfiles.d/app-uploads.conf
```
🔍 **تحلیل دستور:** `--clean`: بدون این سوییچ (یا با `--create`)، فقط پوشه ساخته می‌شود؛ `--clean` مشخصاً بخش پاک‌سازی مبتنی‌بر سن فایل را هم اجرا می‌کند.

**۳. ساخت Path Unit برای نظارت آنی**
```bash
cat > /etc/systemd/system/watch-large-upload.path << 'EOF'
[Path]
DirectoryNotEmpty=/var/tmp/app-uploads

[Install]
WantedBy=multi-user.target
EOF
```
🔍 **تحلیل دستور:** `DirectoryNotEmpty=` یک نوع خاص از شرط Path Unit — به‌جای نظارت روی یک فایل خاص، هر بار که این پوشه از حالت خالی به غیرخالی تغییر کند (یعنی حداقل یک فایل در آن ظاهر شود) trigger می‌شود.

**۴. فعال‌سازی و تست**
```bash
systemctl daemon-reload
systemctl enable --now watch-large-upload.path
dd if=/dev/zero of=/var/tmp/app-uploads/bigtest.bin bs=1M count=150
```

### ✅ Best Practice
ترکیب tmpfiles (نظافت دوره‌ای) با Path Unit (واکنش آنی) — بدون نیاز به اسکریپت cron سفارشی.

---

## سناریو ۶: عیب‌یابی کرش مکرر سرویس با systemd-coredump (سطح پیشرفته)

### قدم‌به‌قدم

**۱. لیست Core Dump های اخیر**
```bash
coredumpctl list myservice
```
🔍 **تحلیل دستور:** آرگومان `myservice` فیلتر خروجی به Core Dump های مربوط به فقط این یک برنامه.

**۲. جزئیات آخرین کرش**
```bash
coredumpctl info myservice
```
🔍 **تحلیل دستور:** بدون سوییچ اضافه؛ نمایش timestamp، سیگنال (مثل SIGSEGV)، و Stack Trace خلاصه.

**۳. تحلیل عمیق با gdb**
```bash
coredumpctl debug myservice
```
🔍 **تحلیل دستور:** `debug` (به‌جای `info`) مستقیماً یک نشست `gdb` را با Core Dump بارگذاری‌شده باز می‌کند.

**۴. محدودسازی منابع با Drop-in**
```bash
systemctl edit myservice.service
```
🔍 **تحلیل دستور:** بدون سوییچ؛ این دستور به‌طور خودکار یک فایل Drop-in برای ویرایش باز می‌کند و بعد از ذخیره، خودش `daemon-reload` را هم انجام می‌دهد — نیازی به ساخت دستی پوشه `.d/` نیست.

در ویرایشگر:
```ini
[Service]
MemoryMax=512M
OOMPolicy=stop
```

**۵. هشدار خودکار در صورت شکست مجدد**
```bash
cat > /etc/systemd/system/myservice.service.d/on-failure-alert.conf << 'EOF'
[Unit]
OnFailure=alert-oncall@%n.service
EOF
```
🔍 **تحلیل دستور:** `%n` نماد ویژه‌ای که نام کامل Unit شکست‌خورده را در لحظه جایگزین می‌کند، ترکیب‌شده با Template Unit (`alert-oncall@`).

### 📌 نکته آزمون
تسلط بر `coredumpctl` یکی از تمایزهای مدیر سیستم سطح ۲ حرفه‌ای است.

---

## جمع‌بندی مهارت‌های این بخش

- ساخت پایه یک systemd Service از صفر با چرخه کامل start/enable/status/stop
- ساخت یک systemd Timer ساده به‌جای cron
- ا Socket Activation برای صرفه‌جویی منابع
- ا Template Unit + Slice برای مدیریت دسته‌جمعی worker ها
- خودکارسازی نظافت با tmpfiles + Path Unit
- عیب‌یابی عمیق کرش با `coredumpctl` و Drop-in محدودیت منابع

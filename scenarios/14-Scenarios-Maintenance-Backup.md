# سناریوهای عملی ۱۴: System Maintenance & Backup

> پیش‌نیاز مطالعه: `theory/14-Maintenance-Backup.md`
> هر دستور بلافاصله بعد از خودش یک بخش «🔍 تحلیل دستور» دارد که تک‌تک سوییچ‌های آن را جداگانه توضیح می‌دهد.

---

## سناریو ۱ (مبتدی): اولین Backup ساده با rsync و درک تفاوت اسلش انتهایی (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده Hard-Link، باید ساده‌ترین اشتباه رایج rsync (اسلش انتهایی) را با چشم خودتان تجربه کنید.

### قدم‌به‌قدم

**۱. ساخت داده تست**
```bash
mkdir -p /tmp/source-test/subdir
echo "test file" > /tmp/source-test/file1.txt
```

**۲. کپی با اسلش انتهایی (کپی محتویات)**
```bash
mkdir -p /tmp/dest-with-slash
rsync -av /tmp/source-test/ /tmp/dest-with-slash/
ls /tmp/dest-with-slash/
```
🔍 **تحلیل دستور:** `/tmp/source-test/` (با اسلش انتهایی) — نتیجه این است که **محتویات** پوشه (فایل و زیرپوشه) مستقیم داخل مقصد کپی می‌شوند؛ باید `file1.txt` و `subdir` را مستقیم در `dest-with-slash/` ببینید.

**۳. کپی بدون اسلش انتهایی (کپی خود پوشه)**
```bash
mkdir -p /tmp/dest-no-slash
rsync -av /tmp/source-test /tmp/dest-no-slash/
ls /tmp/dest-no-slash/
```
🔍 **تحلیل دستور:** `/tmp/source-test` (بدون اسلش انتهایی) — این‌بار خودِ پوشه `source-test` هم کپی می‌شود؛ نتیجه این است که یک زیرپوشه `source-test/` داخل `dest-no-slash/` ساخته می‌شود، نه محتویات مستقیم.

**۴. مقایسه دو نتیجه**
```bash
find /tmp/dest-with-slash /tmp/dest-no-slash
```
🔍 **تحلیل دستور:** `find` بدون سوییچ اضافه، با چند آرگومان مسیر هم‌زمان — نمایش درختی کامل هر دو پوشه برای مقایسه مستقیم.

### ✅ Best Practice
همیشه قبل از اجرای واقعی، با `-n` (dry-run) نتیجه دقیق را پیش‌نمایش کنید تا این اشتباه رایج پیش نیاید.

---

## سناریو ۲ (مبتدی): چرخش دستی یک فایل لاگ با logrotate (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوی پیچیده postrotate، باید ساده‌ترین چرخه چرخش لاگ را ببینید.

### قدم‌به‌قدم

**۱. ساخت یک فایل لاگ تست**
```bash
mkdir -p /var/log/testapp
echo "log entry 1" > /var/log/testapp/app.log
```

**۲. تعریف قانون ساده logrotate**
```bash
cat > /etc/logrotate.d/testapp << 'EOF'
/var/log/testapp/*.log {
    rotate 3
    compress
}
EOF
```
🔍 **تحلیل دستور:** `rotate 3`: نگهداری حداکثر ۳ نسخه قدیمی قبل از حذف کامل؛ `compress`: فشرده‌سازی نسخه‌های چرخانده‌شده با gzip.

**۳. اجرای اجباری فوری (بدون صبر برای زمان‌بندی طبیعی)**
```bash
logrotate -f /etc/logrotate.d/testapp
```
🔍 **تحلیل دستور:** `-f`: مخفف `--force` — چرخش را همین الان اجبار می‌کند، صرف‌نظر از این‌که طبق برنامه واقعاً موعدش رسیده باشد یا نه.

**۴. مشاهده نتیجه**
```bash
ls -la /var/log/testapp/
```
🔍 **تحلیل دستور:** باید یک فایل جدید `app.log` (خالی) و یک فایل `app.log.1` (محتوای قبلی، احتمالاً هنوز فشرده‌نشده به‌خاطر عدم `delaycompress`) ببینید.

**۵. تست dry-run برای دیدن رفتار بدون اجرای واقعی**
```bash
logrotate -d /etc/logrotate.d/testapp
```
🔍 **تحلیل دستور:** `-d`: مخفف `--debug` — نمایش دقیق «چه کاری قرار است انجام شود» بدون واقعاً انجام آن.

### 📌 نکته آزمون
ا `rotate N` تعداد نسخه‌های نگهداری‌شده را کنترل می‌کند؛ بعد از N بار چرخش، قدیمی‌ترین نسخه حذف می‌شود.

---

## سناریو ۳: طراحی Backup روزانه با Hard-Link و چرخش خودکار (سطح متوسط تا پیشرفته)

### قدم‌به‌قدم

**۱. اسکریپت اصلی**
```bash
cat > /usr/local/bin/backup-daily.sh << 'EOF'

#!/bin/bash
SOURCE="/srv/data/"
DEST="/backup/daily"
DATE=$(date +%Y-%m-%d)
LATEST="$DEST/latest"

if [ -d "$LATEST" ]; then
    rsync -avz --delete --link-dest="$LATEST" "$SOURCE" "$DEST/$DATE"
else
    rsync -avz --delete "$SOURCE" "$DEST/$DATE"
fi
rm -f "$LATEST"
ln -s "$DEST/$DATE" "$LATEST"
find "$DEST" -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;

EOF
```
🔍 **تحلیل دستور:**
- `if [ -d "$LATEST" ]`:

 بررسی شرطی — آیا یک symlink/پوشه به این مسیر از قبل وجود دارد؛ اگر بله (یعنی حداقل یک backup قبلی موجود است)، از `--link-dest` استفاده می‌شود.
- `--link-dest="$LATEST"`: 

به rsync می‌گوید فایل‌های بدون تغییر را به‌جای کپی، به‌عنوان Hard Link به نسخه قبلی ذخیره کند.
- `find "$DEST" -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;`:

 ا `-maxdepth 1` فقط زیرپوشه‌های مستقیم (نه عمیق‌تر)؛ `-mtime +30` پوشه‌هایی که قدیمی‌تر از ۳۰ روز هستند؛ `-exec ... {} \;` برای هر نتیجه پیداشده، دستور `rm -rf` را با آن نتیجه (جایگزین `{}`) اجرا می‌کند.

**۲. تست dry-run**
```bash
rsync -avzn --delete /srv/data/ /backup/daily/test/
```
🔍 **تحلیل دستور:** `-n`: مخفف `--dry-run` — نمایش دقیق چه چیزی کپی/حذف می‌شود بدون اجرای واقعی؛ حیاتی قبل از هر `--delete`.

**۳. زمان‌بندی**
```bash
systemctl enable --now backup-daily.timer
```

**۴. اثبات صرفه‌جویی فضا**
```bash
du -sh /backup/daily/2026-08-15
du -sh --apparent-size /backup/daily/2026-08-15
```
🔍 **تحلیل دستور:** `--apparent-size`: به‌جای فضای واقعی دیسک، اندازه «ظاهری» هر فایل را می‌شمارد (یعنی هر Hard Link را دوباره کامل حساب می‌کند)؛ مقایسه با `du -sh` عادی تفاوت واقعی صرفه‌جویی را نشان می‌دهد.

### ✅ Best Practice
همیشه دستور `find ... -exec rm` را ابتدا بدون `-exec` (فقط با `-print`) تست کنید.

---

## سناریو ۴: رفع رشد بی‌رویه لاگ به‌خاطر عدم reopen صحیح (سطح متوسط)

### قدم‌به‌قدم

**۱. تشخیص با lsof**
```bash
lsof +L1 | grep myapp
```
🔍 **تحلیل دستور:** همان الگوی بخش ۳ (Filesystems) — `+L1` فایل‌های با Link Count صفر که هنوز باز نگه داشته شده‌اند.

**۲. اصلاح logrotate با postrotate صحیح**
```bash
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 14
    postrotate
        systemctl kill -s HUP myapp.service 2>/dev/null || true
    endscript
}
EOF
```
🔍 **تحلیل دستور:** `2>/dev/null || true`: ترکیب دو محافظت — تغییر مسیر خطا به `/dev/null`، و `|| true` که تضمین می‌کند حتی اگر این دستور شکست بخورد (مثلاً سرویس در حال حاضر متوقف است)، کل اسکریپت `logrotate` با خطا متوقف نشود.

**۳. تست اجباری**
```bash
logrotate -f /etc/logrotate.d/myapp
```

**۴. تایید آزاد شدن فضا**
```bash
lsof +L1 | grep myapp
```

### 📌 نکته آزمون
این سناریو دقیقاً همان مکانیزم فضای گمشده دیسک (بخش ۳) را از زاویه علت logrotate نشان می‌دهد.

---

## سناریو ۵: Backup رمزنگاری‌شده با Borg برای داده حساس (سطح پیشرفته)

### قدم‌به‌قدم

**۱. راه‌اندازی مخزن**
```bash
borg init --encryption=repokey /backup/finance-repo
```
🔍 **تحلیل دستور:** `--encryption=repokey`: کلید رمزنگاری داخل خود مخزن (رمزنگاری‌شده با passphrase) ذخیره می‌شود — جایگزین‌های دیگر مثل `keyfile` کلید را جداگانه و خارج از مخزن نگه می‌دارند.

**۲. اولین backup**
```bash
borg create --stats --progress /backup/finance-repo::backup-{now} /srv/finance-data
```
🔍 **تحلیل دستور:**
- `--stats`: نمایش آمار خلاصه بعد از اتمام (چقدر داده جدید، چقدر Deduplicated).
- `--progress`: نمایش زنده پیشرفت در حین اجرا.
- `backup-{now}`: نام آرشیو با یک نماد Template — `{now}` خودکار با timestamp لحظه اجرا جایگزین می‌شود.

**۳. ا Pruning خودکار نسخه‌های قدیمی**
```bash
borg prune --keep-daily=14 --keep-weekly=8 --keep-monthly=6 /backup/finance-repo
```
🔍 **تحلیل دستور:** سه سوییچ نگهداری چندسطحی — روزانه ۱۴ نسخه آخر، هفتگی ۸ نسخه آخر، ماهانه ۶ نسخه آخر؛ Borg خودش الگوریتم انتخاب می‌کند کدام آرشیوها را نگه دارد تا این معیارها برآورده شود.

**۴. بازیابی**
```bash
borg extract /backup/finance-repo::backup-2026-08-15
```

### ✅ Best Practice
ترکیب رمزنگاری built-in Borg با نگهداری مخزن جدا از سرور اصلی، محافظت در برابر هم سرقت هم خرابی سخت‌افزار.

---

## سناریو ۶: عیب‌یابی Backup که هنگام Restore واقعی شکست می‌خورد (سطح پیشرفته)

### قدم‌به‌قدم

**۱. شبیه‌سازی مشکل**
```bash
rsync -avz /var/lib/mysql/ /backup/mysql-test/
```
🔍 **تحلیل دستور:** کپی مستقیم فایل‌های دیتابیس **در حال نوشتن** — بدون هیچ مکانیزم سازگاری، ریسک کپی یک وضعیت میانی و ناسازگار.

**۲. راه‌حل صحیح — Snapshot LVM**
```bash
lvcreate -L 5G -s -n mysql-snap /dev/vg_data/lv_mysql
mount -o ro /dev/vg_data/mysql-snap /mnt/mysql-snap
```
🔍 **تحلیل دستور:** `-s`: مخفف Snapshot (که در بخش ۴ دیدیم)؛ `-o ro`: mount فقط‌خواندنی برای اطمینان از عدم تغییر تصادفی حین backup.

**۳. یا راه‌حل جایگزین — dump سطح دیتابیس**
```bash
mysqldump --single-transaction --all-databases > /backup/mysql-dump-$(date +%Y%m%d).sql
```
🔍 **تحلیل دستور:** `--single-transaction`: تضمین می‌کند کل dump از یک نمای سازگار (در سطح تراکنش دیتابیس) گرفته شود، نه فایل‌های خام ناهماهنگ.

**۴. تست واقعی Restore**
```bash
systemctl stop mysql
mv /var/lib/mysql /var/lib/mysql.bak
mysql < /backup/mysql-dump-20260816.sql
systemctl start mysql
```
🔍 **تحلیل دستور:** `mysql < file.sql`: تغییر مسیر ورودی — به‌جای این‌که `mysql` منتظر تایپ تعاملی دستورات SQL بماند، محتوای فایل را به‌عنوان ورودی می‌خواند و اجرا می‌کند.

### ⚠️ نکته امنیتی/عملیاتی حیاتی
هیچ backup فایل‌های خام یک دیتابیس در حال اجرا، بدون Snapshot یا native dump، قابل‌اعتماد نیست.

---

## جمع‌بندی مهارت‌های این بخش

- درک عملی و دیداری تفاوت اسلش انتهایی در rsync
- چرخش پایه یک لاگ با logrotate و تحلیل `-f`/`-d`
- ا Backup کم‌حجم روزانه با Hard-Link و چرخش خودکار نسخه‌های قدیمی
- رفع رشد بی‌رویه لاگ با postrotate صحیح
- ا Backup رمزنگاری‌شده با Deduplication با Borg
- تشخیص و رفع مشکل بنیادین backup ناسازگار از دیتابیس فعال

# بخش ۱۴: System Maintenance & Backup (نگهداری سیستم و پشتیبان‌گیری)

> پیش‌نیاز مفهومی: بخش ۳ (Filesystems، برای درک Hard Link که پایه تکنیک backup افزایشی است) و بخش ۶ (systemd Timer).

---

## ۱۴.۱ چرا Backup یک موضوع فنی «ساده» اما استراتژیک «پیچیده» است

کپی کردن یک فایل کار ساده‌ای است؛ اما یک **استراتژی** Backup باید به چند سوال پاسخ دهد: چند نسخه نگه داریم؟ هر چند وقت یک‌بار؟ Backup کامل باشد یا افزایشی؟ آیا واقعاً قابل بازیابی است (تست شده)؟ آیا در برابر باج‌افزار/حذف تصادفی مقاوم است؟ بدون پاسخ به این‌ها، صرف «داشتن یک کپی» امنیت کاذب ایجاد می‌کند.

## ۱۴.۲ انواع استراتژی Backup

| نوع | توضیح | مزیت | هزینه |
|---|---|---|---|
| **Full** | کپی کامل همه داده | ساده‌ترین بازیابی | بیشترین فضا/زمان |
| **Incremental** | فقط تغییرات از آخرین backup (هر نوع) | کمترین فضا | بازیابی نیازمند زنجیره کامل backup ها |
| **Differential** | فقط تغییرات از آخرین Full | فضای متوسط | بازیابی فقط نیاز به Full + آخرین Differential |

✅ الگوی رایج Production: یک Full هفتگی + Incremental روزانه — تعادل بین فضا و سادگی بازیابی.

## ۱۴.۳ ا rsync — ابزار اصلی Backup در دنیای لینوکس

```bash
rsync -avz /source/ /destination/
```
ا `-a (archive) معادل -rlptgoD است: -r بازگشتی، -l حفظ symlink، -p حفظ مجوز، -t حفظ زمان، -g حفظ گروه، -o حفظ مالک، -D حفظ فایل‌های دستگاه؛ و -v نمایش جزئیات و -z فشرده‌سازی حین انتقال.`
⚠️ نکته حیاتی که در سناریوهای این بخش تکرار می‌شود: اسلش انتهای مسیر مبدا معنا دارد — `/source/` یعنی محتویات کپی شود؛ `/source` (بدون اسلش) یعنی خود پوشه هم کپی شود.

```bash
rsync -avz --delete /source/ /destination/
```
ا `--delete` فایل‌هایی که در مقصد هستند اما در مبدا دیگر نیستند را حذف می‌کند — یعنی مقصد را دقیقاً آینه (mirror) مبدا می‌کند؛ همیشه با `-n` (dry-run) تست شود قبل از اجرای واقعی.

### تکنیک Hard-Link Incremental — «Backup روزانه با هزینه فضای افزایشی»

```bash
rsync -avz --delete --link-dest=/backup/2026-08-16 /source/ /backup/2026-08-17/
```
ا `--link-dest` (که مستقیماً به مفهوم Hard Link از بخش ۳ متکی است): فایل‌هایی که نسبت به backup قبلی تغییر نکرده‌اند، به‌جای کپی فیزیکی، به‌عنوان Hard Link ذخیره می‌شوند — نتیجه: هر پوشه روزانه از بیرون شبیه یک backup **کامل و مستقل** به‌نظر می‌رسد (می‌توانید مستقیماً وارد `2026-08-17/` شوید و کل ساختار را ببینید)، اما فضای واقعی دیسک فقط معادل تغییرات آن روز است.

## ۱۴.۴ معرفی tar — آرشیو کلاسیک

```bash
tar czf backup-full.tar.gz /home /etc
tar tzf backup-full.tar.gz            # لیست محتویات بدون استخراج
tar xzf backup-full.tar.gz -C /restore/path
```

### ا Incremental واقعی با tar (متفاوت از rsync)

```bash
tar --create --file=full.tar --listed-incremental=snapshot.file /data
tar --create --file=inc1.tar --listed-incremental=snapshot.file /data
```
فایل `snapshot.file` وضعیت را بین اجراها ردیابی می‌کند؛ هر اجرای بعدی فقط تغییرات از آخرین اجرا را در آرشیو جدید می‌نویسد.

## ۱۴.۵ ا dd — Image کامل سطح بلاک

```bash
dd if=/dev/sda of=/backup/disk.img bs=4M status=progress
```
⚠️ ا `dd` کل دیسک (حتی فضای خالی) را کپی می‌کند، مگر با ابزار sparse ترکیب شود؛ مناسب برای imaging کامل، نه backup روزانه فایل.

## ۱۴.۶ ابزارهای تخصصی مدرن — Borgbackup

```bash
borg init --encryption=repokey /backup/repo
borg create /backup/repo::backup-{now} /home /etc
borg list /backup/repo
borg extract /backup/repo::backup-2026-08-16
```
ا Borg مزایای فراتر از rsync/tar دارد: **Deduplication** (اگر همان بلاک داده در چند فایل/backup تکرار شود، فقط یک‌بار ذخیره می‌شود) و **رمزنگاری built-in** — بدون نیاز به لایه جداگانه مثل LUKS.

## ۱۴.۷ ا logrotate — چرخش لاگ، پایه نگهداری سیستم

```
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        systemctl kill -s HUP myapp.service
    endscript
}
```
ا `postrotate` حیاتی است: بدون سیگنال reopen به برنامه، آن برنامه همچنان به file descriptor فایل قدیمی (که rename شده) می‌نویسد — این دقیقاً همان مکانیزم «فضای گمشده دیسک» است که در بخش ۳ با `lsof +L1` دیدیم.

```bash
logrotate -d /etc/logrotate.d/myapp     # dry-run
logrotate -f /etc/logrotate.d/myapp     # اجرای اجباری فوری
```

## ۱۴.۸ زمان‌بندی Backup — cron در برابر systemd Timer

هر دو روش (که در بخش ۶ عمیق دیدیم) قابل استفاده‌اند؛ برای backup، systemd Timer با `Persistent=true` ارجح است چون اگر سرور در زمان زمان‌بندی‌شده خاموش بوده، بعد از روشن شدن بلافاصله backup عقب‌افتاده را جبران می‌کند.

## ۱۴.۹ اصل طلایی — Backup بدون تست Restore، Backup نیست

```bash
mkdir -p /tmp/restore-test
rsync -avz /backup/daily/2026-08-15/ /tmp/restore-test/
diff -r /tmp/restore-test /original/data
```
حداقل ماهی یک‌بار باید یک بازیابی کامل آزمایشی انجام شود.

## ۱۴.۱۰ خلاصه نکات کلیدی برای آزمون

- تفاوت Full/Incremental/Differential را از نظر فضا و پیچیدگی بازیابی بدانید
- اسلش انتهای مسیر مبدا در rsync معنای متفاوتی دارد
- ا `--link-dest` مکانیزم Hard Link (از بخش ۳) را برای backup افزایشی کم‌حجم به‌کار می‌گیرد
- ا `postrotate`/`endscript` در logrotate برای reopen صحیح فایل لاگ توسط برنامه ضروری است
- ا Backup هرگز کامل نیست مگر restore آن حداقل یک‌بار تست شده باشد

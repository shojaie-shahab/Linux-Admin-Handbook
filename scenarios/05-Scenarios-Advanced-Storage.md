# سناریوهای عملی ۵: Advanced Storage

> پیش‌نیاز مطالعه: `theory/05-Advanced-Storage.md`
> هر دستور بلافاصله بعد از خودش یک بخش «🔍 تحلیل دستور» دارد که تک‌تک سوییچ‌های آن را جداگانه توضیح می‌دهد.

---

## سناریو ۱ (مبتدی): اشتراک‌گذاری ساده یک پوشه با NFS و mount آن از یک کلاینت (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای امنیتی پیچیده NFS، باید ساده‌ترین حالت ممکن را یک‌بار کامل تجربه کرده باشید.

### قدم‌به‌قدم

**۱. نصب سرور NFS**
```bash
apt install nfs-kernel-server -y
```

**۲. ساخت پوشه‌ای برای اشتراک**
```bash
mkdir -p /srv/nfs/shared
chmod 777 /srv/nfs/shared
```
🔍 **تحلیل دستور:**
- ا `chmod 777`: مجوز کامل خواندن/نوشتن/اجرا برای مالک/گروه/سایرین؛ فقط برای این تمرین ساده استفاده می‌شود (در Production هرگز `777` مناسب نیست).

**۳. تعریف export**
```bash
echo "/srv/nfs/shared 192.168.1.0/24(rw,sync,no_subtree_check)" >> /etc/exports
```
🔍 **تحلیل دستور:** `>>` خط جدید را به انتهای فایل اضافه می‌کند بدون پاک کردن export های قبلی.

**۴. اعمال export**
```bash
exportfs -ra
```
🔍 **تحلیل دستور:**
- ا `-r`: مخفف `--reexport` — تمام تعریف‌های `/etc/exports` را دوباره می‌خواند.
- ا `-a`: مخفف `--all` — روی همه export ها عمل کن.

**۵. ا mount از سمت کلاینت**
```bash
mkdir -p /mnt/nfs-test
mount -t nfs 192.168.1.10:/srv/nfs/shared /mnt/nfs-test
```
🔍 **تحلیل دستور:**
- ا `-t nfs`: مخفف `--types` — نوع فایل‌سیستم را صریح مشخص می‌کند چون این یک منبع شبکه‌ای است.
- آرگومان `IP:/path`: فرمت خاص NFS.

**۶. تست نوشتن**
```bash
touch /mnt/nfs-test/testfile.txt
ls -la /mnt/nfs-test/
```

### ✅ Best Practice
همیشه بعد از هر تغییر `/etc/exports`، `exportfs -ra` را اجرا کنید.

---

## سناریو ۲ (مبتدی): ساخت یک Share ساده Samba و اتصال از یک کلاینت لینوکس (سطح مبتدی)

### قدم‌به‌قدم

**۱. نصب Samba**
```bash
apt install samba -y
```

**۲. تعریف یک share ساده**
```bash
cat >> /etc/samba/smb.conf << 'EOF'

[testshare]
    path = /srv/samba/testshare
    browsable = yes
    writable = yes
    guest ok = yes
EOF
```
🔍 **تحلیل دستور:** `guest ok = yes` یعنی این share بدون نیاز به نام‌کاربری/رمز قابل‌دسترسی است — فقط برای تست ساده.

**۳. ساخت پوشه واقعی**
```bash
mkdir -p /srv/samba/testshare
chmod 777 /srv/samba/testshare
```

**۴. بررسی صحت تنظیمات**
```bash
testparm
```
🔍 **تحلیل دستور:** بدون سوییچ؛ `testparm` فایل را از نظر syntax بررسی می‌کند — باید همیشه قبل از restart اجرا شود.

**۵. راه‌اندازی سرویس**
```bash
systemctl restart smbd
```

**۶. اتصال از کلاینت لینوکس**
```bash
smbclient -L localhost -N
```
🔍 **تحلیل دستور:**
- ا `-L`: مخفف `--list` — لیست share های موجود بدون اتصال کامل تعاملی.
- ا `-N`: مخفف `--no-pass` — تلاش بدون درخواست رمز عبور.

### 📌 نکته آزمون
همیشه `testparm` قبل از هر restart سرویس Samba.

---

## سناریو ۳: راه‌اندازی NFSv4 امن با تنظیمات root_squash و تست عملی محدودیت (سطح متوسط تا پیشرفته)

### قدم‌به‌قدم

**۱. تعریف export امن**
```bash
cat >> /etc/exports << 'EOF'
/srv/nfs/uploads   192.168.10.0/24(rw,sync,root_squash,no_subtree_check)
EOF
```
🔍 **تحلیل دستور:** `root_squash` کاربر root کلاینت را به `nobody` تبدیل می‌کند؛ `no_subtree_check` بهینه‌سازی کارایی.

**۲. اعمال و تایید**
```bash
exportfs -ra
exportfs -v
```
🔍 **تحلیل دستور:** `-v` مخفف `--verbose` — نمایش دقیق گزینه‌های کامل هر export.

**۳. باز کردن پورت**
```bash
firewall-cmd --permanent --add-service=nfs
```
🔍 **تحلیل دستور:**
- ا `--permanent`: ذخیره دائمی؛ بدون `--reload` بعدی فوراً اعمال نمی‌شود.
- ا `--add-service=nfs`: استفاده از تعریف سرویس آماده به‌جای پورت خام.

**۴. تست عملی root_squash**
```bash
mount -t nfs4 192.168.10.5:/srv/nfs/uploads /var/www/uploads
touch /var/www/uploads/test-as-root.txt
ls -la /var/www/uploads/test-as-root.txt
```
🔍 **تحلیل دستور:** با وجود اجرا به‌عنوان root روی کلاینت، مالکیت فایل روی سرور باید `nobody` باشد.

### ✅ Best Practice
ترکیب `sync`+`root_squash`+`hard` بالاترین سطح یکپارچگی داده را تضمین می‌کند.

---

## سناریو ۴: یکپارچه‌سازی Samba با Active Directory (سطح پیشرفته)

### قدم‌به‌قدم

**۱. کشف دامین**
```bash
realm discover corp.example.com
```
🔍 **تحلیل دستور:** بدون سوییچ؛ صرفاً یک بررسی خواندنی از طریق DNS.

**۲. اتصال به دامین**
```bash
realm join corp.example.com -U administrator
```
🔍 **تحلیل دستور:** `-U administrator` مخفف `--user` — حساب مجاز افزودن کامپیوتر.

**۳. تست شناسایی کاربر AD**
```bash
id alice@corp.example.com
```
🔍 **تحلیل دستور:** تایید زنجیره NSS→SSSD→AD.

**۴. تنظیم Samba برای استفاده از AD**
```bash
testparm
systemctl restart smbd winbind
```
🔍 **تحلیل دستور:** `restart` با دو نام سرویس هم‌زمان.

### 📌 نکته آزمون
تفاوت `security = user` و `security = ads`.

---

## سناریو ۵: راه‌اندازی یک دیسک iSCSI برای سرور دیتابیس (سطح پیشرفته)

### قدم‌به‌قدم

**۱. کشف Target ها**
```bash
iscsiadm -m discovery -t sendtargets -p 192.168.20.10
```
🔍 **تحلیل دستور:**
- `-m discovery`: مخفف `--mode` — حالت عملیاتی.
- `-t sendtargets`: مخفف `--type` — روش کشف استاندارد.
- `-p`: مخفف `--portal` — آدرس سرور.

**۲. اتصال (Login) به Target**
```bash
iscsiadm -m node -T iqn.2026-08.com.example:storage.db01 -p 192.168.20.10 --login
```
🔍 **تحلیل دستور:**
- `-m node`: حالت `node`، عملیات روی Target مشخص.
- `-T`: مخفف `--targetname`.
- `--login`: اتصال واقعی.

**۳. تایید دیسک**
```bash
lsblk
```

**۴. فعال‌سازی اتصال خودکار در بوت**
```bash
iscsiadm -m node -T iqn.2026-08.com.example:storage.db01 -p 192.168.20.10 --op update -n node.startup -v automatic
```
🔍 **تحلیل دستور:** `--op update` عملیات تنظیم؛ `-n`/`-v` نام و مقدار پارامتر.

### ⚠️ نکته امنیتی
امنیت مبتنی‌بر IQN به‌تنهایی ضعیف است؛ CHAP باید برای محیط حساس فعال شود.

---

## سناریو ۶: پیکربندی Multipath برای افزونگی سخت‌افزاری (سطح پیشرفته)

### قدم‌به‌قدم

**۱. تایید یکسان بودن دو دستگاه**
```bash
/lib/udev/scsi_id -g -u /dev/sdc
```
🔍 **تحلیل دستور:** `-g` لیست استاندارد دستورات SCSI؛ `-u` فرمت خروجی یکتا.

**۲. فعال‌سازی سرویس multipath**
```bash
mpathconf --enable --with_multipathd y
```
🔍 **تحلیل دستور:** `--enable` فعال‌سازی تنظیمات؛ `--with_multipathd y` اجرای فوری سرویس.

**۳. بررسی وضعیت دستگاه یکپارچه**
```bash
multipath -ll
```
🔍 **تحلیل دستور:** `-ll` نمایش وضعیت تازه هر مسیر.

**۴. تست قطع یک مسیر**
```bash
ip link set eth2 down
```

### ✅ Best Practice
همیشه روی دستگاه یکپارچه `/dev/mapper/mpathX` کار کنید، نه دستگاه خام.

---

## جمع‌بندی مهارت‌های این بخش

- راه‌اندازی پایه NFS و Samba از صفر تا mount موفق
- پیکربندی امن NFSv4 با `root_squash` و تست عملی
- یکپارچه‌سازی Samba با Active Directory
- راه‌اندازی کامل iSCSI با تحلیل دقیق حالت‌های `iscsiadm`
- پیکربندی و تست Multipath

# بخش ۱۲: Security (امنیت)

> پیش‌نیاز مفهومی: این بخش  روی چند لایه دفاعی مکمل تمرکز دارد: کنترل دسترسی فایل (از پایه تا SELinux/AppArmor پیشرفته)، و فایروال شبکه (از iptables کلاسیک تا nftables مدرن).

---

## ۱۲.۱ مدل امنیتی پایه لینوکس — از rwx تا ACL

### مرور بسیار مختصر مدل کلاسیک (پیش‌فرض LPIC-1)

فرض پایه این است که شما با مدل کلاسیک مجوز یونیکس (`rwx` برای Owner/Group/Other، و بیت‌های ویژه SUID/SGID/Sticky) از LPIC-1 آشنا هستید. تمرکز این بخش روی چیزی است که **فراتر** از این مدل کلاسیک می‌رود.

### محدودیت بنیادین مدل کلاسیک — چرا به ACL نیاز داریم

مدل کلاسیک rwx فقط اجازه می‌دهد **یک** مالک و **یک** گروه برای هر فایل تعریف شود. اما در دنیای واقعی، اغلب نیاز دارید مجوزهای متفاوت برای **چند** کاربر یا گروه مختلف روی همان فایل تعریف کنید — مثلاً «alice باید بتواند بنویسد، bob فقط بخواند، و گروه finance اصلاً دسترسی نداشته باشد» — این با فقط یک Owner و یک Group ممکن نیست.

ا **POSIX ACL (Access Control List)** این محدودیت را برطرف می‌کند: به شما اجازه می‌دهد مجوزهای دقیق و مجزا برای **هر تعداد** کاربر یا گروه دلخواه روی یک فایل تعریف کنید.

```bash
# <" tools "> modify <" users,group,other:rwx "> <" file ">

setfacl -m u:alice:rwx /srv/shared/report.doc
setfacl -m u:bob:r-- /srv/shared/report.doc
setfacl -m g:finance:--- /srv/shared/report.doc
```
این `-m` (modify) یک قانون جدید اضافه/تغییر می‌دهد؛ `u:` برای یک کاربر خاص، `g:` برای یک گروه خاص.

```bash
getfacl /srv/shared/report.doc
```
نمایش تمام قوانین ACL فعلی این فایل، شامل مجوزهای کلاسیک rwx و تمام قوانین اضافه‌شده.

### ا Default ACL — وراثت خودکار مجوزها برای فایل‌های آینده

یک محدودیت ACL معمولی این است که فقط روی فایل‌های **موجود** اعمال می‌شود — اگر یک فایل جدید در همان پوشه ساخته شود، آن ACL خودکار به آن اعمال نمی‌شود. **Default ACL** این مشکل را حل می‌کند: با تنظیم یک ACL پیش‌فرض روی یک **پوشه**، هر فایل/زیرپوشه جدیدی که داخل آن ساخته می‌شود، به‌طور خودکار همان مجوزها را به ارث می‌برد.

```bash
setfacl -d -m g:finance:rwx /srv/finance-reports/
```
ا `-d` (default) این قانون را به‌عنوان پیش‌فرض برای فایل‌های آینده تنظیم می‌کند، نه فقط برای خود پوشه.

```bash
touch /srv/finance-reports/newfile.txt
getfacl /srv/finance-reports/newfile.txt
```
باید ببینید فایل جدید، بدون هیچ دستور دستی اضافی، خودکار قانون ACL گروه `finance` را از پوشه والد به ارث برده.

📌 نکته فنی مهم: برای استفاده از ACL، فایل‌سیستم باید با گزینه `acl` mount شده باشد (در ext4/XFS مدرن معمولاً پیش‌فرض فعال است، اما برای اطمینان می‌توان صراحتاً در `fstab` اضافه کرد).

---

## ۱۲.۲ ا SELinux — کنترل دسترسی اجباری (Mandatory Access Control)

### تفاوت بنیادین MAC و DAC — چرا SELinux کاملاً متفاوت از rwx/ACL است

مدل کلاسیک rwx (و حتی ACL) را **DAC (Discretionary Access Control)** می‌نامند — «صلاحدیدی»، چون **مالک فایل** خودش تصمیم می‌گیرد چه کسی به آن دسترسی دارد. اگر یک برنامه با یک آسیب‌پذیری (مثلاً یک وب‌سرور که هک شده) با امتیاز کاربری خاص در حال اجراست، آن برنامه هک‌شده می‌تواند به هر فایلی که آن کاربر مجاز به دسترسی است، دسترسی پیدا کند — DAC هیچ محدودیت اضافی روی «این برنامه خاص» اعمال نمی‌کند.

ا **SELinux (Security-Enhanced Linux)** یک لایه کاملاً مستقل و اضافی به نام **MAC (Mandatory Access Control)** پیاده‌سازی می‌کند: حتی اگر مجوزهای rwx/ACL اجازه دسترسی بدهند، SELinux می‌تواند بر اساس یک **سیاست مرکزی** (که کاربر عادی یا حتی مالک فایل نمی‌تواند آن را تغییر دهد) دسترسی را **اضافه محدود** کند. این «اجباری» است چون تصمیم دیگر به صلاحدید مالک فایل نیست — یک سیاست سیستمی مرکزی، مستقل از خواست کاربر، اعمال می‌شود.

### ا Context — برچسب امنیتی هر فایل و پردازش

در SELinux، هر فایل و هر پردازش یک **Context امنیتی** دارد که از چهار بخش تشکیل شده: `user:role:type:level`. برای اکثر کاربردهای عملی روزمره، بخش **Type** مهم‌ترین است — این همان چیزی است که تعیین می‌کند «این فایل چه نوعی است» و «این پردازش چه نوعی است»، و سیاست SELinux قوانینی تعریف می‌کند که کدام Type از پردازش، به کدام Type از فایل، چه نوع دسترسی‌ای دارد.

```bash
ls -Z /var/www/html/index.html
```
ا `-Z` Context امنیتی SELinux فایل را (علاوه بر مجوزهای معمول) نمایش می‌دهد — نمونه خروجی: `system_u:object_r:httpd_sys_content_t`. بخش `httpd_sys_content_t` نشان می‌دهد این فایل از نوع «محتوای وب‌سرور» است.

```bash
ps -eZ | grep nginx
```
نمایش Context پردازش‌های در حال اجرا؛ یک پردازش nginx معمولاً با Type چیزی شبیه `httpd_t` اجرا می‌شود.

### چرا این Type مهم است — یک مثال عملی

سیاست SELinux می‌گوید: «پردازش‌هایی با Type `httpd_t` فقط مجاز به خواندن فایل‌هایی با Type `httpd_sys_content_t` هستند» — این یعنی حتی اگر مجوز rwx یک فایل حساس (مثلاً `/etc/shadow`) اجازه خواندن بدهد و nginx به هر دلیلی (باگ یا حمله) بخواهد آن را بخواند، SELinux این تلاش را **مستقل از rwx** مسدود می‌کند، چون `/etc/shadow` هرگز Type `httpd_sys_content_t` ندارد.

### سه حالت اصلی SELinux

```bash
getenforce
```

ا **`Enforcing`**: سیاست به‌طور کامل اجرا می‌شود — هر تخلف از سیاست، مسدود و ثبت می‌شود.

ا **`Permissive`**: سیاست بررسی می‌شود اما هیچ‌چیز واقعاً مسدود نمی‌شود — فقط تخلفات احتمالی در لاگ ثبت می‌شوند. این حالت برای **عیب‌یابی** بسیار مفید است: می‌توانید ببینید اگر Enforcing بود، چه چیزهایی مسدود می‌شدند، بدون این‌که واقعاً چیزی خراب شود.

ا **`Disabled`**: SELinux کاملاً غیرفعال است.

```bash
setenforce 0     # تغییر موقت به Permissive (فقط تا ریبوت بعدی، یا تا تغییر مجدد)
setenforce 1     # بازگشت به Enforcing
```
تغییر دائمی در `/etc/selinux/config` با `SELINUX=enforcing`/`permissive`/`disabled`.

### عیب‌یابی مشکلات SELinux

⚠️ رایج‌ترین اشتباه مبتدیان: وقتی یک سرویس با خطای مبهم مواجه می‌شود، بلافاصله SELinux را کاملاً `disabled` می‌کنند — این یک اقدام امنیتی بسیار ضعیف است. روش صحیح، تشخیص **دقیق** مشکل و رفع آن است:

```bash
ausearch -m avc -ts recent
```
ا `ausearch` لاگ‌های Audit مربوط به تصمیمات SELinux (چه چیزی مسدود شده، `AVC` — Access Vector Cache — همان رکورد تصمیم‌های SELinux) را نشان می‌دهد.

```bash
sealert -a /var/log/audit/audit.log
```
ا `sealert` (اگر نصب باشد) تحلیل خودکار و پیشنهاد راه‌حل برای هر تخلف ثبت‌شده ارائه می‌دهد — بسیار مفید برای مبتدیان که هنوز با خواندن مستقیم لاگ Audit راحت نیستند.

### رفع صحیح — تغییر Context، نه غیرفعال کردن SELinux

فرض کنید یک وب‌سرور محتوایش را از یک مسیر غیرمعمول (نه `/var/www/html` استاندارد) اجرا می‌کند و SELinux آن را مسدود کرده — چون آن مسیر Type صحیح `httpd_sys_content_t` را ندارد:

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/mywebsite(/.*)?"
restorecon -Rv /srv/mywebsite
```
ا `semanage fcontext -a` یک **قانون دائمی** اضافه می‌کند که می‌گوید «هر فایلی داخل `/srv/mywebsite` باید Type `httpd_sys_content_t` داشته باشد» (این قانون در فایل تنظیمات ذخیره می‌شود و در `relabel` های بعدی هم دوباره اعمال می‌شود)؛ `restorecon -Rv` این قانون را **فوراً** روی فایل‌های موجود اعمال می‌کند (بدون این دستور، قانون فقط برای آینده ثبت شده اما فایل‌های فعلی هنوز Context قدیمی دارند).

### ا Boolean — سوییچ‌های ساده برای رفتارهای رایج

بسیاری از تصمیمات رایج سیاست SELinux به‌جای نیاز به تغییر پیچیده Context، از طریق **Boolean** های ساده (روشن/خاموش) کنترل می‌شوند:

```bash
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
```
ا `httpd_can_network_connect` مشخص می‌کند آیا nginx/Apache اجازه دارد اتصالات خروجی شبکه برقرار کند (مثلاً برای یک اپلیکیشن PHP که به یک API خارجی وصل می‌شود) — `-P` تغییر را دائمی (Persistent، حتی بعد از ریبوت) می‌کند.

---

## ۱۲.۳ ا AppArmor — رویکرد جایگزین ساده‌تر MAC

### تفاوت فلسفی با SELinux

ا **AppArmor** همان هدف کلی SELinux (MAC — کنترل اجباری فراتر از rwx) را دنبال می‌کند، اما با یک فلسفه ساده‌تر: به‌جای برچسب‌گذاری Type روی هر فایل در سراسر سیستم (که SELinux انجام می‌دهد)، AppArmor **بر اساس مسیر فایل (Path-based)** کار می‌کند — یک **Profile** برای هر برنامه تعریف می‌کند که دقیقاً می‌گوید «این برنامه فقط اجازه دارد این مسیرهای خاص را بخواند/بنویسد».

این رویکرد ساده‌تر برای یادگیری و نگهداری است (Ubuntu و SUSE به‌طور پیش‌فرض از AppArmor استفاده می‌کنند، به‌جای SELinux)، هرچند از نظر برخی معیارهای امنیتی پیشرفته، دقت کمتری نسبت به مدل Type-based SELinux دارد.

```bash
aa-status
```
نمایش تمام Profile های فعال، و این‌که هرکدام در حالت `enforce` (اجرای کامل) یا `complain` (مشابه Permissive در SELinux — فقط لاگ، بدون مسدودسازی) هستند.

```bash
aa-complain /etc/apparmor.d/usr.sbin.nginx
```
تغییر یک Profile خاص به حالت complain (برای عیب‌یابی، مشابه `setenforce 0` اما فقط برای یک برنامه، نه کل سیستم — یک مزیت دقت بالاتر نسبت به SELinux).

```bash
aa-enforce /etc/apparmor.d/usr.sbin.nginx
```
بازگشت به حالت کامل اجرایی.

---

## ۱۲.۴ ا Netfilter و iptables — فایروال کلاسیک لینوکس

### معماری Netfilter — سه جدول اصلی

ا **Netfilter** چارچوب سطح کرنل است که فیلترینگ بسته را انجام می‌دهد؛ **iptables** ابزار خط‌فرمان سنتی برای تعریف قوانین آن است. سه **Table (جدول)** اصلی، هرکدام هدف متفاوتی دارند:

ا **`filter`** (پیش‌فرض): فیلترینگ ساده — اجازه یا رد کردن بسته.

ا **`nat`**: تغییر آدرس مبدا/مقصد بسته (که در ادامه با NAT/Masquerade عمیق می‌بینیم).

ا **`mangle`**: تغییر خود بسته (مثل تغییر فیلدهای TTL یا TOS/QoS).

هر Table شامل چند **Chain (زنجیره)** پیش‌فرض است که بسته‌ها بر اساس مسیرشان از یکی از آن‌ها عبور می‌کنند: `INPUT` (بسته‌های ورودی مقصد خودِ این ماشین)، `OUTPUT` (بسته‌های خروجی تولیدشده توسط این ماشین)، `FORWARD` (بسته‌هایی که فقط از این ماشین عبور می‌کنند، مقصد جای دیگری است — مربوط به حالت روتر).

### دستورات پایه

```bash
iptables -L -v -n
```
نمایش تمام قوانین جدول `filter` (پیش‌فرض)، با شمارنده بسته/بایت (`-v`) و بدون تلاش برای resolve نام (`-n`، سریع‌تر).

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```
ا `-A` (Append، اضافه به انتهای زنجیره)؛ `-p tcp --dport 22` شرط (پروتکل TCP، پورت مقصد ۲۲)؛ `-j ACCEPT` اقدام (قبول کردن بسته).

```bash
iptables -P INPUT DROP
```
ا `-P` (Policy) رفتار پیش‌فرض یک زنجیره را تنظیم می‌کند — اگر هیچ قانون دیگری با یک بسته مطابقت نکرد، این رفتار پیش‌فرض اعمال می‌شود. تنظیم `INPUT` روی `DROP` یعنی «هر بسته‌ای که صراحتاً مجاز نشده، رد شود» — یک اصل امنیتی پایه‌ای به نام **Default Deny**.

⚠️ ترتیب قوانین در iptables **بسیار مهم** است — بسته از بالای زنجیره به پایین بررسی می‌شود و به محض تطبیق با اولین قانون، همان اقدام اجرا می‌شود (بقیه قوانین بررسی نمی‌شوند، مگر قانون‌های خاص با `-j LOG` که فقط ثبت می‌کنند و ادامه می‌دهند). اگر یک قانون `ACCEPT` عمومی را قبل از یک قانون `DROP` خاص‌تر قرار دهید، آن قانون DROP هرگز اجرا نمی‌شود.

### ا NAT و Masquerade — تبدیل سرور لینوکسی به یک روتر

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
ا `ip_forward=1` (که در بخش ۲ با sysctl دیدیم) به کرنل اجازه می‌دهد بسته‌هایی که مقصدشان خودِ این ماشین نیست را عبور دهد (روتینگ)؛ `MASQUERADE` تمام بسته‌های خروجی از `eth0` را طوری تغییر می‌دهد که انگار از IP خودِ این ماشین آمده‌اند (نه IP خصوصی واقعی مبدا داخلی) — این دقیقاً مکانیزمی است که به چند کامپیوتر با IP خصوصی داخلی، اجازه اشتراک یک اتصال اینترنت واحد را می‌دهد (همان چیزی که یک روتر خانگی معمولی انجام می‌دهد).

### ا Connection Tracking — چگونه فایروال «حافظه» دارد

```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```
ا Netfilter می‌تواند وضعیت هر اتصال را ردیابی کند (`conntrack`) — این قانون به بسته‌هایی که بخشی از یک اتصال **از قبل تایید‌شده** (`ESTABLISHED`) یا مرتبط با آن (`RELATED`، مثل یک اتصال FTP داده که از یک اتصال کنترل قبلی می‌آید) هستند، اجازه عبور می‌دهد — بدون این قابلیت، شما مجبور بودید برای هر جهت هر اتصال (رفت و برگشت) جداگانه قانون بنویسید، که هم پیچیده و هم پرریسک است.

---

## ۱۲.۵ ا nftables — جانشین مدرن و یکپارچه

### چرا nftables؟

ا **nftables** جانشین رسمی و مدرن iptables است (توزیع‌های امروزی، مثل Debian 10+ و RHEL 8+، به‌طور پیش‌فرض از nftables استفاده می‌کنند، حتی وقتی دستورات آشنای `iptables` را اجرا می‌کنید — یک لایه سازگاری آن‌ها را به معادل nftables ترجمه می‌کند). مزایای اصلی: سینتکس یکپارچه‌تر (یک ابزار برای IPv4 و IPv6 با هم، به‌جای `iptables` و `ip6tables` جداگانه)، کارایی بهتر (خصوصاً با تعداد بسیار زیاد قوانین)، و ساختار داده داخلی انعطاف‌پذیرتر (مثل Set ها — گروه‌های قابل‌جستجوی سریع از IP یا پورت).

```bash
nft list ruleset
```
نمایش تمام قوانین فعلی (معادل مفهومی `iptables -L` اما فرمت متفاوت).

```bash
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet filter input iif lo accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
```
ا `inet` (به‌جای `ip` یا `ip6`) یعنی این قوانین برای **هر دو** IPv4 و IPv6 هم‌زمان اعمال می‌شوند — یک بهبود مستقیم نسبت به نیاز قدیمی به دو ابزار جداگانه.

### ا Set — گروه‌بندی کارآمد چند مقدار

```bash
nft add rule inet filter input tcp dport { 80, 443, 8080 } accept
```
این syntax فشرده اجازه می‌دهد چند پورت (یا IP) را در یک قانون واحد گروه‌بندی کنید — از نظر داخلی، nftables این را به یک ساختار جستجوی بسیار سریع (نه بررسی خطی هر مقدار) تبدیل می‌کند، که با تعداد زیاد مقادیر، کارایی به‌طور چشمگیری بهتر از نوشتن قوانین جداگانه برای هرکدام است.

### فایل تنظیمات دائمی

```bash
# /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif lo accept
        ct state established,related accept
        tcp dport 22 accept
        tcp dport { 80, 443 } accept
    }
}
```
```bash
nft -f /etc/nftables.conf
systemctl enable --now nftables
```

---

## ۱۲.۶ ا firewalld — لایه مدیریتی سطح بالا (رایج در RHEL)

### مفهوم Zone — چرا یک لایه انتزاعی بالاتر مفید است

نوشتن مستقیم قوانین iptables/nftables برای هر سناریو می‌تواند پرحجم و مستعد خطا باشد. **firewalld** یک لایه مدیریتی بالاتر ارائه می‌دهد که بر اساس مفهوم **Zone** (سطح اعتماد) کار می‌کند — هر اینترفیس شبکه به یک Zone خاص تعلق دارد، و هر Zone مجموعه‌ای از سرویس‌ها/پورت‌های مجاز از پیش‌تعریف‌شده یا قابل‌تنظیم دارد.

```bash

firewall-cmd --state                                      # نمایش وضعیت فایروال (running یا not running)
firewall-cmd --get-active-zones                           # نمایش زون‌های فعال و اینترفیس‌های هر کدام
firewall-cmd --get-default-zone                           # نمایش زون پیش‌فرض سیستم
firewall-cmd --set-default-zone=public                    # تغییر زون پیش‌فرض به public
firewall-cmd --get-zones                                  # لیست تمام زون‌های تعریف‌شده
firewall-cmd --zone=public --list-all                     # نمایش کامل تنظیمات زون public (سرویس‌ها، پورت‌ها، اینترفیس‌ها و...)
firewall-cmd --zone=public --list-services                # فقط لیست سرویس‌های مجاز در زون public
firewall-cmd --zone=public --list-ports                   # فقط لیست پورت‌های باز در زون public

firewall-cmd --zone=public --add-service=http             # افزودن سرویس http به‌صورت موقت (تا ریبوت بعدی)
firewall-cmd --zone=public --add-service=http --permanent # افزودن دائمی سرویس http (نیاز به --reload دارد)
firewall-cmd --zone=public --remove-service=http --permanent  # حذف دائمی سرویس http

firewall-cmd --zone=public --add-port=8080/tcp --permanent    # باز کردن دائمی پورت 8080/tcp
firewall-cmd --zone=public --remove-port=8080/tcp --permanent # بستن دائمی پورت 8080/tcp

firewall-cmd --zone=public --query-service=http           # بررسی اینکه آیا سرویس http در زون public مجاز است یا نه
firewall-cmd --zone=public --query-port=8080/tcp          # بررسی وضعیت باز بودن پورت 8080/tcp

firewall-cmd --zone=public --add-source=192.168.1.0/24 --permanent  # محدود کردن زون public به شبکه 192.168.1.0/24
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="22" protocol="tcp" accept' --permanent  # افزودن قانون پیشرفته (Rich Rule) برای اجازه SSH فقط از IP مشخص

firewall-cmd --zone=public --add-masquerade --permanent   # فعال‌سازی NAT/Masquerade در زون public
firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=192.168.1.10 --permanent  # فوروارد پورت 80 به 8080 روی سرور داخلی

firewall-cmd --reload                                     # بارگذاری مجدد قوانین دائمی بدون قطع اتصال‌های فعلی
firewall-cmd --runtime-to-permanent                       # تبدیل تمام قوانین موقت (runtime) به دائمی

firewall-cmd --panic-on                                   # فعال‌سازی حالت اضطراری (قطع کامل ترافیک)
firewall-cmd --panic-off                                  # غیرفعال‌سازی حالت اضطراری

firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -p tcp --dport 3306 -j ACCEPT  # افزودن قانون مستقیم (Direct Rule) برای باز کردن پورت 3306


```

⚠️ نکته حیاتی که در بخش‌های قبلی هم دیدیم: بدون `--permanent`، تغییرات فقط runtime (موقتی، از بین می‌روند با reload/reboot) هستند؛ اما با `--permanent`، بدون اجرای بعدی `--reload`، تغییرات بلافاصله در runtime فعلی اعمال **نمی‌شوند** — برای اعمال دائمی و فوری هر دو، یا از `--permanent` + `--reload` استفاده کنید، یا هر دستور را دوبار (یک‌بار بدون `--permanent` برای فوری، یک‌بار با آن برای دائمی) اجرا کنید.

### ا Rich Rule — برای نیازهای پیچیده‌تر از سرویس/پورت ساده

```bash
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept' --permanent
```
ا Rich Rule اجازه ترکیب شرایط پیچیده‌تر (منبع خاص + سرویس خاص + اقدام خاص) را در یک خط می‌دهد — برای نیازهایی که مدل ساده Zone/Service به‌تنهایی کافی نیست.

---

## ۱۲.۷ خلاصه نکات کلیدی برای آزمون

- ا ACL (`setfacl`/`getfacl`) محدودیت «فقط یک Owner و یک Group» مدل کلاسیک rwx را برطرف می‌کند؛ Default ACL (`-d`) برای وراثت خودکار روی فایل‌های آینده در یک پوشه
- تفاوت بنیادین DAC (rwx/ACL، صلاحدید مالک فایل) و MAC (SELinux/AppArmor، سیاست مرکزی اجباری، مستقل از خواست مالک) را عمیق درک کنید
- ا SELinux بر اساس Type (بخشی از Context چهاربخشی) کار می‌کند؛ هرگز برای رفع مشکل، SELinux را کاملاً disable نکنید — از `ausearch`/`sealert` برای تشخیص دقیق و `semanage fcontext` + `restorecon` برای رفع صحیح استفاده کنید
- ا AppArmor بر اساس مسیر فایل (نه Type برچسب‌خورده در سراسر سیستم) کار می‌کند؛ ساده‌تر اما با فلسفه متفاوت از SELinux
- سه Table اصلی Netfilter: `filter` (فیلتر ساده)، `nat` (تغییر آدرس)، `mangle` (تغییر بسته)؛ ترتیب قوانین در یک Chain حیاتی است
- ا Connection Tracking (`ESTABLISHED,RELATED`) نیاز به نوشتن قوانین جداگانه برای هر جهت یک اتصال را از بین می‌برد
- ا nftables جانشین رسمی iptables است؛ با `inet` هم‌زمان IPv4 و IPv6 را پوشش می‌دهد
- در firewalld: بدون `--permanent` موقتی، بدون `--reload` بعد از `--permanent` فوراً اعمال نمی‌شود

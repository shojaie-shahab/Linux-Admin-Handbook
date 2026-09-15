# سناریوهای عملی ۱۰: DNS & DHCP

> پیش‌نیاز مطالعه: `theory/10-DNS-DHCP.md`
> هر دستور بلافاصله بعد از خودش یک بخش «🔍 تحلیل دستور» دارد که تک‌تک سوییچ‌های آن را جداگانه توضیح می‌دهد.

---

## سناریو ۱ (مبتدی): جستجوی ساده DNS با dig و تفسیر خروجی (سطح مبتدی)

### زمینه و چرایی
قبل از سناریوهای پیچیده Master/Slave، باید با ساده‌ترین ابزار پرسش DNS راحت باشید.

### قدم‌به‌قدم

**۱. پرسش ساده یک نام**
```bash
dig google.com
```
🔍 **تحلیل دستور:** بدون سوییچ؛ به‌طور پیش‌فرض رکورد نوع A را می‌پرسد و از سرور DNS تعریف‌شده در /etc/resolv.conf استفاده می‌کند.

**۲. پرسش یک نوع رکورد خاص**
```bash
dig example.com MX
```
🔍 **تحلیل دستور:** آرگومان دوم (MX) نوع رکورد درخواستی را مشخص می‌کند؛ بدون آن، پیش‌فرض A است.

**۳. پرسش از یک سرور DNS مشخص**
```bash
dig @8.8.8.8 example.com
```
🔍 **تحلیل دستور:** @8.8.8.8: پیشوند @ مشخص می‌کند این پرسش باید مستقیماً از این IP خاص پرسیده شود، نه از resolver پیش‌فرض سیستم.

**۴. نمایش فقط پاسخ خلاصه**
```bash
dig example.com +short
```
🔍 **تحلیل دستور:** +short یک سوییچ ویژه dig (با پیشوند + نه -) که فقط نتیجه نهایی را چاپ می‌کند، بدون هدرها و آمار اضافه.

**۵. جستجوی معکوس (IP به نام)**
```bash
dig -x 8.8.8.8
```
🔍 **تحلیل دستور:** -x: حالت PTR lookup — dig خودش مسیر معکوس in-addr.arpa را می‌سازد.

### ✅ Best Practice
ا dig +short بهترین گزینه برای استفاده در اسکریپت‌هاست چون خروجی تمیز و قابل پردازش مستقیم دارد.

---

## سناریو ۲ (مبتدی): بررسی وضعیت Lease فعلی DHCP یک کلاینت (سطح مبتدی)

### قدم‌به‌قدم

**۱. بررسی IP فعلی کلاینت**
```bash
ip addr show <" eth0 ">
```

**۲. درخواست دستی و verbose یک IP جدید**
```bash
dhclient -v <" eth0 ">
```
🔍 **تحلیل دستور:** -v: مخفف --verbose — نمایش دقیق هر مرحله از فرآیند DORA روی صفحه.

**۳. آزاد کردن lease فعلی**
```bash
dhclient -r <" eth0 ">
```
🔍 **تحلیل دستور:** -r: مخفف --release — صراحتاً به سرور اعلام می‌کند این IP دیگر لازم نیست.

**۴. بررسی فایل دیتابیس lease های سمت سرور**
```bash
cat /var/lib/dhcp/dhcpd.leases
```
🔍 **تحلیل دستور:** بدون سوییچ؛ شامل رکورد هر IP تخصیص‌یافته، MAC کلاینت، و زمان انقضا.

**۵. مشاهده زنده ترافیک DHCP**
```bash
tcpdump -i eth0 port 67 or port 68
```
🔍 **تحلیل دستور:** port 67 or port 68: این دو پورت استاندارد سمت سرور و کلاینت پروتکل DHCP هستند.

### 📌 نکته آزمون
فرآیند DHCP: Discover → Offer → Request → Acknowledge (DORA).

---

## سناریو ۳: راه‌اندازی جفت Master/Slave DNS با تست کامل چرخه انتشار (سطح متوسط تا پیشرفته)

### قدم‌به‌قدم

**۱. تعریف Zone روی Master**
```bash
cat > /etc/bind/zones/db.corp.example.com << 'EOF'
$TTL    3600
@   IN  SOA ns1.corp.example.com. admin.corp.example.com. (
                2026081501 3600 1800 1209600 3600 )
@   IN  NS  ns1.corp.example.com.
@   IN  NS  ns2.corp.example.com.
app IN  A   192.168.1.50
EOF
```
🔍 **تحلیل دستور:** ساختار SOA — عدد اول (2026081501) Serial است که هر تغییر باید افزایش یابد؛ چهار عدد بعدی به‌ترتیب Refresh/Retry/Expire/Negative-Cache-TTL هستند.

**۲. اجازه Zone Transfer فقط به IP مشخص**
```bash
echo 'allow-transfer { 192.168.1.20; };' >> /etc/bind/named.conf.local
```
🔍 **تحلیل دستور:** محدود کردن صریح — این تنظیم فقط IP سرور Slave شناخته‌شده را مجاز می‌کند.

**۳. بررسی صحت قبل از راه‌اندازی**
```bash
named-checkzone corp.example.com /etc/bind/zones/db.corp.example.com
```
🔍 **تحلیل دستور:** دو آرگومان — نام Zone و مسیر فایل Zone.

**۴. اجبار فوری Slave به بررسی مجدد**
```bash
rndc retransfer corp.example.com
```
🔍 **تحلیل دستور:** retransfer: به‌جای صبر تا Refresh Interval طبیعی، همین الان Slave را مجبور به بررسی/دانلود مجدد می‌کند.

**۵. تست منفی — فراموشی افزایش Serial**
```bash
sed -i 's/192.168.1.51/192.168.1.52/' /etc/bind/zones/db.corp.example.com
rndc reload corp.example.com
```
🔍 **تحلیل دستور:** sed -i تغییر مستقیم فایل بدون افزایش Serial؛ باید ببینید Slave این تغییر را نمی‌گیرد چون Serial همان است.

### ✅ Best Practice
ا Serial را همیشه اولین چیزی که چک می‌کنید باشد، قبل از هر rndc reload.

---

## سناریو ۴: پیکربندی DHCP با Reservation ثابت و مشاهده DORA زنده (سطح متوسط)

### قدم‌به‌قدم

**۱. تعریف subnet با استخر پویا و Reservation**
```bash
cat > /etc/dhcp/dhcpd.conf << 'EOF'
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    host office-printer {
        hardware ethernet 00:1B:44:11:3A:B7;
        fixed-address 192.168.1.50;
    }
}
EOF
```
🔍 **تحلیل دستور:** range استخر پویا؛ بلوک host یک استثنای ثابت با hardware ethernet (MAC) و fixed-address (IP همیشگی) که باید خارج از range باشد.

**۲. بررسی صحت syntax**
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```
🔍 **تحلیل دستور:** -t: مخفف --test — فقط بررسی، بدون راه‌اندازی؛ -cf: مخفف --config-file — مسیر فایل تنظیماتی که باید بررسی شود.

**۳. مشاهده زنده فرآیند DORA**
```bash
tcpdump -i eth0 port 67 or port 68 -v
```
🔍 **تحلیل دستور:** -v این‌بار روی tcpdump، جزئیات بیشتر از نوع پیام DHCP (DISCOVER/OFFER/REQUEST/ACK) نمایش می‌دهد.

**۴. تایید Reservation در فایل lease**
```bash
grep -A5 "192.168.1.50" /var/lib/dhcp/dhcpd.leases
```
🔍 **تحلیل دستور:** -A5: نمایش ۵ خط بعد از خط مطابق، چون جزئیات یک رکورد lease در چند خط پشت‌سرهم می‌آید.

### 📌 نکته آزمون
IP های Reservation باید همیشه خارج از range استخر پویا باشند.

---

## سناریو ۵: عیب‌یابی ناهماهنگی کش DNS ناشی از TTL بالا (سطح پیشرفته)

### قدم‌به‌قدم

**۱. تایید صحت روی سرور Authoritative**
```bash
dig @192.168.1.10 app.corp.example.com +short
```

**۲. بررسی TTL فعلی**
```bash
dig @192.168.1.10 app.corp.example.com
```
🔍 **تحلیل دستور:** بدون +short، برای دیدن عدد TTL کامل کنار نام رکورد.

**۳. کاهش TTL برای یک رکورد خاص**
```bash
sed -i 's/^app.*IN.*A.*/app     300     IN      A       192.168.1.51/' /etc/bind/zones/db.corp.example.com
```
🔍 **تحلیل دستور:** نوشتن صریح عدد 300 بلافاصله بعد از نام رکورد، این رکورد را از TTL سراسری مستقل می‌کند و فقط برای همین یک رکورد TTL کوتاه‌تر تنظیم می‌شود.

**۴. افزایش Serial و اعمال**
```bash
rndc reload corp.example.com
```

**۵. تایید بعد از انتظار کافی**
```bash
dig @8.8.8.8 app.corp.example.com +short
```

### ✅ Best Practice
قبل از هر مهاجرت IP برنامه‌ریزی‌شده، TTL را از قبل کاهش دهید و صبر کنید تا منتشر شود.

---

## سناریو ۶: راه‌اندازی DHCP Relay برای چند VLAN با یک سرور مرکزی (سطح پیشرفته)

### قدم‌به‌قدم

**۱. تعریف subnet های مجزا روی سرور مرکزی**
```bash
cat > /etc/dhcp/dhcpd.conf << 'EOF'
subnet 10.0.0.0 netmask 255.255.248.0 {
    range 10.0.0.10 10.0.7.250;
}
EOF
```

**۲. راه‌اندازی Relay Agent روی مرز هر VLAN**
```bash
cat > /etc/default/isc-dhcp-relay << 'EOF'
SERVERS="10.0.8.5"
INTERFACES="eth0.10 eth0.30"
EOF
```
🔍 **تحلیل دستور:** SERVERS: آدرس سرور DHCP مرکزی که درخواست‌ها باید به آن فوروارد شوند؛ INTERFACES: لیست VLAN هایی که این Relay روی آن‌ها گوش می‌دهد.

**۳. راه‌اندازی**
```bash
systemctl enable --now isc-dhcp-relay
```

**۴. تست از یک کلاینت**
```bash
dhclient -v eth0
```

**۵. مشاهده تبدیل Broadcast به Unicast**
```bash
tcpdump -i eth0.10 port 67 or port 68 -v
```
🔍 **تحلیل دستور:** روی گیت‌وی محلی این ترافیک هنوز Broadcast است؛ روی سرور مرکزی، بسته‌ها اکنون Unicast هستند و شامل فیلد giaddr که آدرس Relay Agent را نشان می‌دهد.

### 📌 نکته آزمون
ا DHCP Relay ضروری است چون بسته‌های Broadcast از روتر عبور نمی‌کنند.

---

## جمع‌بندی مهارت‌های این بخش

- جستجوی پایه و پیشرفته DNS با تحلیل کامل سوییچ‌های dig
- بررسی و مدیریت Lease با dhclient -v/-r
- راه‌اندازی و تست کامل چرخه Master/Slave DNS با تاکید بر Serial
- ا Reservation ثابت DHCP و مشاهده زنده فرآیند DORA
- رفع ناهماهنگی کش با کاهش هدفمند TTL یک رکورد
- ا DHCP Relay برای پل زدن بین چند VLAN و یک سرور مرکزی

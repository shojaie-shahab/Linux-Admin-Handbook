# Linux-Admin-Handbook# کتابچه جامع آموزشی LPIC-2 

## این کتابچه چیست

مجموعه‌ای کامل و عمیق (سطح مبتدی مطلق) برای آمادگی آزمون‌های LPIC-2 (201 و 202)، شامل ۱۶ بخش اصلی + ۱ بخش تکمیلی امنیت، هرکدام با یک فایل **تئوری** (توضیح از صفر، بدون فرض دانش قبلی) و یک فایل **سناریو** ( سناریوی عملی قدم‌به‌قدم با توضیح دستورات و Best Practice).

## ساختار کامل نهایی

```
lpic2-guide/
├── theory/
│   ├── 01-System-Startup-Boot.md
│   ├── 02-Linux-Kernel.md
│   ├── 03-Filesystems.md
│   ├── 04-Storage-Management.md
│   ├── 05-Advanced-Storage.md          (NFS, Samba, iSCSI, Multipath)
│   ├── 06-System-Services.md           (systemd پیشرفته)
│   ├── 07-Logging.md
│   ├── 08-Process-Management.md        (Namespaces, Scheduler پیشرفته)
│   ├── 09-Networking.md
│   ├── 10-DNS-DHCP.md
│   ├── 11-Time-Authentication.md       (NTP/Chrony, PAM, LDAP, Kerberos, sudo)
│   ├── 12-Security.md                  (ACL, SELinux, AppArmor, Firewall)
│   ├── 12b-SSH-VPN-Fail2ban.md         (تکمیلی: SSH Hardening, VPN, Fail2ban)
│   ├── 13-Capacity-Planning.md
│   ├── 14-Maintenance-Backup.md
│   ├── 15-HTTP-Services.md             (Apache, Nginx, Squid)
│   ├── 16-Email-Services.md            (Postfix, Dovecot)
│   ├── 17-PKI-Certificates.md          (CA داخلی، OpenSSL، ACME)
│   ├── 18-RADIUS-Authentication.md     (FreeRADIUS، 802.1x، EAP-TLS)
│   ├── 19-Load-Balancing-HAProxy.md    (Layer 4/7، Health Check، SSL Termination)
│   ├── 20-Packet-Capture-Analysis.md   (BPF، TShark، Wireshark عمیق)
│   ├── 21-Network-Monitoring.md        (SNMP، NetFlow)
│   ├── 22-IPS-Snort-Suricata-Zeek.md   (تشخیص/پیشگیری نفوذ)
│   ├── 23-Honeypot-Services.md         (Cowrie، تشخیص Lateral Movement)
│   └── 24-Security-Standards-Modern-DNS.md  (CIS/OSQuery، DoH/DoT/DNSSEC)
│   └── 25-IP-Routing-Dynamic.md        (RIP/OSPF/BGP با FRRouting)
└── scenarios/
    └── (فایل متناظر هر بخش بالا، پیشوند Scenarios-)
```

## نقشه پوشش آزمون رسمی LPIC-2

| Objective رسمی | بخش(های) مرتبط |
|---|---|
| 200 Capacity Planning | ۱۳ |
| 201 Linux Kernel | ۲ |
| 202 System Startup | ۱ |
| 203 Filesystem and Devices | ۳، ۴ |
| 204 Advanced Storage | ۵ |
| 205 Networking Configuration | ۹ |
| 206 System Maintenance | ۱۴ |
| 207 Domain Name Server | ۱۰، ۲۴ |
| 208 Web Services | ۱۵، ۱۹ |
| 209 File Sharing | ۵ |
| 210 Network Client Management | ۱۰، ۱۱، ۱۸ |
| 211 E-Mail Services | ۱۶ |
| 212 System Security | ۱۲، ۱۲ب، ۱۷، ۲۲، ۲۳، ۲۴ |

با این جدول، کل syllabus رسمی هر دو آزمون (201-450 و 202-450) پوشش داده شده است.



فایل readme ادیت میشود پس از انتشار کامل فایل ها .

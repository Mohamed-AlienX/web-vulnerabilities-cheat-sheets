


# 🔓 Broken Access Control (BAC) – Practical Cheat Sheet

## 1️⃣ What is Broken Access Control?

Broken Access Control happens when the application **fails to properly enforce authorization**, allowing a logged-in user to access **data or actions they should not have access to**.

> ⚠️ Important  
> BAC happens **after authentication**, not by bypassing login.

---

## 2️⃣ Authentication vs Authorization (Very Important)

|Authentication|Authorization|
|---|---|
|Who are you?|What are you allowed to do?|
|Login|Permissions|
|Username / Password / Token|Role, Ownership, Access|

📌 **BAC = Authorization failure**

---

## 3️⃣ Why is BAC Dangerous?

- Data leakage (PII, financial data)
    
- Account takeover
    
- Privilege escalation
    
- Full API compromise
    
- Business impact & legal issues
    

🔥 BAC is **OWASP Top 1 risk**.

---

## 4️⃣ Root Causes of Broken Access Control

- Missing server-side checks
    
- Trusting frontend restrictions
    
- Trusting user input (IDs, role)
    
- Weak API authorization
    
- Business logic mistakes
    
- Over-trusting JWT claims
    

---

## 5️⃣ Main Types of Broken Access Control

---

### 5.1️⃣ IDOR (Insecure Direct Object Reference)

**Description:**  
Accessing objects by changing direct identifiers **without ownership validation**.

**Example:**

```
GET /api/orders/1001
GET /api/orders/1002
```

**Key Question:**

> Does the server check that this object belongs to the logged-in user?

**Impact:**

- Horizontal privilege escalation
    
- Data leakage
    

---

### 5.2️⃣ Horizontal Privilege Escalation

**Description:**  
Accessing data or actions of **another user with the same role**.

**Example:**

```
GET /api/profile?user_id=10 → user_id=11
```

📌

- IDOR = technique
    
- Horizontal escalation = result
    

---

### 5.3️⃣ Vertical Privilege Escalation

**Description:**  
Low-privileged user accesses **admin or higher-role functionality**.

**Examples:**

```
/admin
/api/admin/deleteUser
/api/admin/refund
```

🚫 Hiding buttons ≠ security

---

### 5.4️⃣ Missing Function-Level Access Control

**Description:**  
Sensitive endpoints exist **without role validation**.

**Example:**

```
POST /api/admin/transfer
```

🔥 Very common in APIs.

---

## 6️⃣ Broken Access Control in APIs

### Common API Mistakes:

- No role validation
    
- No ownership validation
    
- Accepting `user_id` from request
    
- Trusting JWT only
    

📌 **Every API endpoint must enforce authorization**

---

## 7️⃣ JWT & Broken Access Control

### Key Truth:

> JWT is for Authentication, not Authorization.

### Common Mistakes:

- Trusting `role` inside JWT
    
- Trusting `user_id` inside JWT
    
- No server-side permission checks
    

**JWT Example:**

```json
{
  "user_id": 42,
  "role": "user"
}
```

### Exploitation Scenarios:

- Change object ID → IDOR
    
- Access admin APIs → Vertical escalation
    
- Token still valid after logout
    

---

## 8️⃣ Business Logic BAC

**Description:**  
Breaking the intended workflow of the application.

**Examples:**

- Skipping approval steps
    
- Approving your own request
    
- Double execution
    
- Jumping from `pending` → `approved`
    

📌 Tools can’t find this — logic analysis only.

---

## 9️⃣ Forced Browsing

**Description:**  
Accessing hidden or unlinked pages/endpoints directly.

**Examples:**

```
/admin
/manage
/config
/reports
```

---

## 🔟 Testing Methodology (Step-by-Step)

### Always Test:

✔ Change IDs  
✔ Change roles  
✔ Call admin APIs  
✔ Reuse tokens  
✔ Skip workflow steps  
✔ Forced browsing  
✔ Test APIs directly

---

## 1️⃣1️⃣ Useful Tools

- Burp Suite
    
- Postman
    
- Browser DevTools
    
- Nuclei (manual verification required)
    

🧠 Tools assist — **thinking finds BAC**

---

## 1️⃣2️⃣ Interview Quick Questions

- Authentication vs Authorization?
    
- Why JWT alone is not enough?
    
- IDOR vs Horizontal escalation?
    
- Why APIs are more vulnerable?
    

---

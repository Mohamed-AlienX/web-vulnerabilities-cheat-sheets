


# 🔓 IDOR Cheat Sheet 

## 🧠 Simple Definition

**IDOR happens when you can access or control another user’s data by changing an ID, while using your own account.**

---

# 📋 Quick Testing Checklist

-  Two accounts
    
-  Change ID only
    
-  Same cookies
    
-  Server returns other user data
    
-  Impact confirmed
    

---

## 🔴 CRITICAL IDOR

### 1️⃣ IDOR – Read / Modify / Delete Other User Data (Direct)

#### 🧠 Description

You can access or change **another user’s data** by changing an identifier.

#### 🎯 Identifiers can be:

`id` / `user_id` / `username` / `email` / `uuid` / `order_id`

---

#### 🪜 Steps to Discover

1. Login as **User A**
    
2. Open any feature (profile, order, document)
    
3. Intercept request in **Burp**
    
4. Find identifier:
    

`user_id=12`

or

`username=mohamed`

5. Change it to another value:
    

`user_id=13`

6. Send request
    

---

#### ✅ Vulnerable If:

- You see **other user data**
    
- OR data is **updated / deleted**
    

#### 💥 Impact

- Account takeover
    
- PII leak
    
- Data manipulation
    

---

### 2️⃣ IDOR via Username / Email (Not Numeric)

#### 🧠 Important

**IDOR does NOT require numeric ID**

---

#### 🪜 Steps

1. Request contains:
    

`email=test1@mail.com`

2. Change to:
    

`email=test2@mail.com`

3. Forward request
    

---

#### ✅ Vulnerable If:

- Response contains test2 data
    
- Or action affects test2 account
    

#### 💥 Impact

Same as numeric IDOR → **CRITICAL**

---

### 3️⃣ JWT-based IDOR (Trusting Request over Token)

#### 🧠 Description

JWT says you are User A  
But request controls User B

---

#### 🪜 Steps

1. JWT payload:
    

`"user_id": 5`

2. Request body:
    

`user_id=7`

3. Send request
    

---

#### ✅ Vulnerable If:

- Backend uses `7` instead of JWT value
    

❌ **NOT IDOR** if you modify JWT itself and resign it  
👉 that’s **JWT tampering**

---

## 🔴 HIGH IDOR

### 4️⃣ GraphQL IDOR (Object ID Abuse)

#### 🧠 Common in modern apps

---

#### 🪜 Steps

1. Capture GraphQL request
    
2. Find:
    

`team_member_id`

3. Decode if Base64:
    

`gid://app/User/43794`

4. Change ID number
    
5. Send request
    

---

#### ✅ Vulnerable If:

- You modify another user object
    
- No permission check
    

#### 💥 Impact

- Role abuse
    
- Privacy breach
    

---

### 5️⃣ Indirect IDOR (ID Chaining)

#### 🧠 Description

You don’t see user ID directly  
But you access it through **another object**

---

#### 🪜 Steps

1. List objects:
    

`GET /orders`

2. Find:
    

`order_id=8821`

3. Open:
    

`GET /order/details?order_id=8821`

4. Change order_id
    

---

#### ✅ Vulnerable If:

- You see other users’ orders
    

#### 💥 Impact

Sensitive business data leak

---

## 🟠 MEDIUM IDOR

### 6️⃣ IDOR on Settings / Visibility / Preferences

#### 🧠 Example

- Profile visibility
    
- Notification settings
    
- Team visibility
    

---

#### 🪜 Steps

1. Change your setting
    
2. Capture request
    
3. Identify:
    

`user_id / team_member_id`

4. Change value
    
5. Send request
    

---

#### 💥 Impact

- Privacy violation
    
- Social engineering risk
    

---

### 7️⃣ File / Document IDOR

#### 🧠 Description

Access files not owned by you

---

#### 🪜 Steps

1. Open file:
    

`/download?file_id=551`

2. Change file_id
    
3. Send request
    

---

#### 💥 Impact

- Document leaks
    
- Contracts / IDs exposure
    

---

## 🟢 LOW IDOR

### 8️⃣ Read-only IDOR (Metadata / Logs)

#### 🧠 Description

Only **non-sensitive** data exposed

---

#### 🪜 Steps

1. Request:
    

`GET /api/user/info?id=9`

2. Change ID
    

---

#### 💥 Impact

- Low risk alone
    
- Can be chained
    

---

## 🧠 Golden Rules (احفظهم)

✔ IDOR ≠ numeric ID only  
✔ Identifier = anything that points to a resource  
✔ Validation ≠ Authorization  
✔ Same cookie + access other data = 🚨  
✔ Always test **Read / Write / Delete**
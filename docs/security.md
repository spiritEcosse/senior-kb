## 10. Security

### SQL Injection

Attack: unescaped user input injected into a SQL query.

```python
# Vulnerable
query = f"SELECT * FROM users WHERE username = '{username}'"
# username = "' OR '1'='1" → returns all users

# Safe — parameterised query
cursor.execute("SELECT * FROM users WHERE username = %s", [username])

# Django ORM — safe by default
User.objects.filter(username=username)

# Raw SQL with params — still safe
User.objects.raw("SELECT * FROM users WHERE username = %s", [username])
```

---

### XSS (Cross-Site Scripting)

Attack: inject malicious JS into a page viewed by other users.

```python
# Vulnerable: render user input as raw HTML
return HttpResponse(f"<h1>Hello, {username}</h1>")

# Safe: Django templates auto-escape by default
return render(request, 'hello.html', {'username': username})
# {{ username }} → escaped: &lt;script&gt;...

# Mark safe only when you trust the content
from django.utils.safestring import mark_safe
safe_html = mark_safe("<b>trusted</b>")   # use with caution
```

Content-Security-Policy header — second line of defence:
```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

---

### Clickjacking

Attack: embed your site in an iframe on an attacker's page.

```python
# Django middleware — add X-Frame-Options header
MIDDLEWARE = ['django.middleware.clickjacking.XFrameOptionsMiddleware']
X_FRAME_OPTIONS = 'DENY'   # or 'SAMEORIGIN'

# Or CSP
Content-Security-Policy: frame-ancestors 'none'
```

---

### Timing attacks

Attack: measure response time to infer secrets (e.g. comparing tokens character by character).

```python
# Vulnerable: short-circuits on first mismatch → timing leak
if token == stored_token:
    ...

# Safe: constant-time comparison
import hmac
if hmac.compare_digest(token, stored_token):
    ...
```

---

### Secret management

```python
# Bad: secrets in code or committed .env
SECRET_KEY = "hardcoded-secret"

# Good: from environment
import os
SECRET_KEY = os.environ['SECRET_KEY']

# Better: use a secrets manager
import boto3
client = boto3.client('secretsmanager', region_name='eu-west-1')
secret = client.get_secret_value(SecretId='myapp/prod/db-password')
```

**Tools:** AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault.
**Never commit:** `.env` files, private keys, API tokens. Add to `.gitignore` + use `git-secrets` or `trufflehog` to scan history.

---

### Password hashing

```python
# Never: plain text, MD5, SHA1
# Always: bcrypt, argon2, PBKDF2

# Django uses PBKDF2 by default; switch to argon2:
# pip install django[argon2]
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',  # preferred
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',  # fallback
]

# Direct usage
from django.contrib.auth.hashers import make_password, check_password
hashed = make_password('my_password')
check_password('my_password', hashed)   # True
```

---

### OWASP Top 10 (2021)

1. **Broken Access Control** — IDOR, privilege escalation, missing authorisation checks
2. **Cryptographic Failures** — weak encryption, HTTP instead of HTTPS, MD5 passwords
3. **Injection** — SQL, NoSQL, OS, LDAP; includes XSS
4. **Insecure Design** — architectural vulnerabilities, missing threat modelling
5. **Security Misconfiguration** — open ports, debug mode in prod, default passwords
6. **Vulnerable and Outdated Components** — outdated deps with CVEs (`pip audit`)
7. **Identification and Authentication Failures** — weak sessions, credential stuffing, missing MFA
8. **Software and Data Integrity Failures** — insecure CI/CD, unverified deserialisation
9. **Security Logging and Monitoring Failures** — no auth logs or anomaly detection
10. **SSRF** — Server-Side Request Forgery

!!! warning
    In 2021, XSS was merged into #3 (Injection). CSRF is no longer a separate entry.

---

### HTTPS/TLS internals

**TLS 1.3 handshake (simplified):**

```
Client → Server: ClientHello (supported ciphers, random)
Server → Client: ServerHello (chosen cipher, certificate, random)
Client:          verify certificate against CA
Client → Server: encrypted pre-master secret (or key exchange)
Both:            derive session keys
Client → Server: Finished (encrypted)
Server → Client: Finished (encrypted)
→ encrypted HTTP traffic begins
```

Key concepts:
- **Certificate** — public key + identity, signed by a CA (Certificate Authority)
- **Chain of trust** — leaf cert → intermediate CA → root CA (pre-installed in OS/browser)
- **HSTS** — tells browsers to always use HTTPS for this domain
- **Certificate pinning** — app only trusts a specific cert/public key (bypasses compromised CAs)

---

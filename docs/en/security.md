## 10. Security

### SQL Injection

Attack: unescaped user input injected into a SQL query.
Defence: parameterised queries, ORM, validation, minimal DB privileges.

### OWASP Top 10 (2021)

1. Broken Access Control — IDOR, privilege escalation
2. Cryptographic Failures — weak encryption, sensitive data leakage (MD5 passwords, HTTP instead of HTTPS)
3. Injection — SQL, NoSQL, OS, LDAP injections
4. Insecure Design — architectural vulnerabilities, missing threat modelling
5. Security Misconfiguration — open ports, debug mode in prod, default passwords
6. Vulnerable and Outdated Components — outdated dependencies with CVEs
7. Identification and Authentication Failures — weak sessions, credential stuffing, missing MFA
8. Software and Data Integrity Failures — insecure CI/CD, unverified deserialisation
9. Security Logging and Monitoring Failures — no auth logs or anomaly detection
10. SSRF — Server-Side Request Forgery

!!! warning
    In 2021, XSS was merged into #3 (Injection). CSRF is no longer a separate entry.

Password hashing: bcrypt, argon2, or PBKDF2. Never MD5/SHA1, never plain text.

---

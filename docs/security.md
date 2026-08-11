## 10. Безопасность

### SQL-инъекции

Атака: неэкранированный ввод пользователя в SQL-запрос.
Защита: параметризованные запросы, ORM, валидация, минимальные привилегии БД.

### OWASP Top 10 (2021)

1. Broken Access Control — нарушение контроля доступа (IDOR, privilege escalation)
2. Cryptographic Failures — слабое шифрование, утечка sensitive data (MD5-пароли, HTTP вместо HTTPS)
3. Injection — SQL, NoSQL, OS, LDAP инъекции
4. Insecure Design — архитектурные уязвимости, отсутствие threat modeling
5. Security Misconfiguration — открытые порты, debug-режим в проде, дефолтные пароли
6. Vulnerable and Outdated Components — устаревшие зависимости с CVE
7. Identification and Authentication Failures — слабые сессии, credential stuffing, отсутствие MFA
8. Software and Data Integrity Failures — небезопасный CI/CD, десериализация без проверки
9. Security Logging and Monitoring Failures — нет логов аутентификации и аномалий
10. SSRF — Server-Side Request Forgery

!!! warning
    В 2021 году XSS вошёл в п.3 (Injection). CSRF больше не отдельная позиция.


Хэширование паролей: bcrypt, argon2 или PBKDF2. Никогда MD5/SHA1, никогда plain text.

---

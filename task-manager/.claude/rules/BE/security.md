---
name: backend-security-ruleset
description: >
  Enforce comprehensive backend security practices including authentication (JWT, OAuth, MFA),
  authorization (RBAC, least privilege), input sanitization, rate limiting, SQL injection prevention,
  XSS/CSRF protection, secure logging, and prohibited insecure patterns. Use this for backend APIs,
  microservices, web services, server-side code, REST/GraphQL endpoints, authentication systems,
  authorization logic, database access layers, API security reviews, security audits, or whenever
  building or reviewing backend code that handles user data, credentials, sensitive information,
  or external requests.
---

# Backend Security Ruleset

This skill enforces production-grade security controls for backend systems. It covers authentication, authorization, data protection, attack prevention, and secure operational practices aligned with OWASP API Security Top 10 2023 and modern backend security standards.

## Core Security Principles

**Defense in depth**: Layer multiple controls so a single failure doesn't compromise the system.

**Least privilege**: Grant only the minimum access required for a task—users, services, and tokens.

**Fail securely**: When errors occur, default to denying access, not granting it.

**Never trust input**: All data from clients, third-party APIs, or external sources is potentially malicious until validated and sanitized.

**Assume breach**: Design systems to limit damage when (not if) an attacker gains partial access.

## Authentication

### Password Handling

- Hash passwords with **bcrypt, scrypt, or Argon2**—never MD5, SHA-1, or plain SHA-256
- Never store plaintext passwords or reversible encryption
- Enforce minimum password strength: 12+ characters, complexity requirements
- Implement account lockout after 5-10 failed login attempts (with exponential backoff or CAPTCHA)
- Use timing-safe comparison for password verification to prevent timing attacks

### Multi-Factor Authentication (MFA)

- Require MFA for admin accounts, privileged roles, and sensitive operations
- Support TOTP (Google Authenticator, Authy) or hardware tokens (WebAuthn, FIDO2)
- Never send MFA codes via SMS for high-security contexts (SIM swapping risk)

### Token-Based Authentication

**JWT (JSON Web Tokens)**:

- Use **RS256 (RSA) or ES256 (ECDSA)** for signing—avoid HS256 unless key distribution is secure
- Set short expiration times: 15 minutes for access tokens, 7 days max for refresh tokens
- Include `exp`, `iat`, `nbf` claims; validate all claims on every request
- Never put sensitive data (passwords, SSNs) in JWT payload—it's base64-encoded, not encrypted
- Validate `aud` (audience) and `iss` (issuer) to prevent token reuse across services
- Implement token rotation for refresh tokens; revoke old tokens on reuse (detect theft)

**OAuth 2.0**:

- Use **Authorization Code flow with PKCE** for web/mobile apps
- Use **Client Credentials flow** for service-to-service communication
- Never use Implicit flow (deprecated, insecure)
- Store client secrets securely (environment variables, secret managers—never in code)
- Validate redirect URIs strictly; use exact match, not pattern matching
- Implement token revocation endpoints and honor revoked tokens

### Session Management

- Generate cryptographically random session IDs (128+ bits entropy)
- Set `HttpOnly`, `Secure`, `SameSite=Strict` (or `Lax`) on session cookies
- Regenerate session IDs on login, privilege escalation, and logout
- Implement absolute and idle timeouts (e.g., 24h absolute, 30min idle)
- Invalidate sessions server-side on logout—don't rely solely on cookie deletion

## Authorization

### Access Control

- Implement **Role-Based Access Control (RBAC)** or **Attribute-Based Access Control (ABAC)**
- Enforce authorization checks at **every API endpoint**—never assume upstream validation
- Validate object-level permissions: users can only access resources they own or are explicitly granted access to (prevents BOLA/IDOR attacks)
- Check function-level authorization: admin endpoints reject non-admin tokens
- Use allowlists for permitted actions, not denylists

### Principle of Least Privilege

- Default to deny; require explicit grants
- Grant minimum necessary permissions for each role or service
- Regularly audit and prune unused permissions
- Use scoped tokens: limit JWT/OAuth scopes to specific resources or actions

## Input Validation and Sanitization

### Validation

- Validate **all input**: query params, headers, body, cookies, file uploads
- Use **allowlists** (permitted characters, formats, values) over denylists
- Enforce strict types: integers as integers, emails as valid email format, UUIDs as valid UUIDs
- Reject requests exceeding size limits (e.g., 10MB for JSON, configurable for file uploads)
- Validate Content-Type headers match actual payload (prevent content-type confusion)

### Sanitization

- Escape output based on context: HTML-encode for HTML, SQL-parameterize for databases, JS-encode for JavaScript
- Use framework-provided sanitization (e.g., Django's template escaping, React's JSX escaping)
- Strip or encode dangerous characters in user-generated content before storage or display

### Parameterized Queries (SQL Injection Prevention)

- **Always use parameterized queries or ORMs**—never string concatenation for SQL
- Example (unsafe): `SELECT * FROM users WHERE id = '${userId}'`
- Example (safe): `SELECT * FROM users WHERE id = ?` with bound parameters
- Validate and sanitize even when using ORMs (some ORM features allow raw queries)
- Apply least privilege to database accounts: read-only for read operations, no DDL rights for app accounts

## Cross-Site Scripting (XSS) Prevention

- **Escape all user-generated content** before rendering in HTML, JavaScript, or JSON responses
- Set `Content-Type: application/json` for JSON APIs; never `text/html`
- Use Content Security Policy (CSP) headers to restrict script sources
- Avoid `eval()`, `innerHTML`, or `dangerouslySetInnerHTML` with user input
- For APIs consumed by browsers, set `X-Content-Type-Options: nosniff`

## Cross-Site Request Forgery (CSRF) Prevention

- Use **CSRF tokens** for state-changing operations (POST, PUT, DELETE) in session-based auth
- Validate `Origin` and `Referer` headers (reject if missing or mismatched)
- For stateless APIs (JWT in `Authorization` header), CSRF risk is lower—but still validate origin for sensitive actions
- Set `SameSite=Strict` or `SameSite=Lax` on cookies
- Require re-authentication for critical actions (password change, fund transfer)

## Rate Limiting and Abuse Prevention

- Implement rate limits per IP, per user, and per endpoint
- Example: 100 requests/minute per IP, 1000/hour per user, 5 login attempts/minute per IP
- Use tiered limits: stricter for unauthenticated users, looser for authenticated
- Apply exponential backoff or CAPTCHA after repeated failures
- Return `429 Too Many Requests` with `Retry-After` header
- Rate-limit expensive operations (password resets, report generation, file uploads) more aggressively
- Monitor for distributed attacks (many IPs, low rate each)—consider behavioral analysis

## Logging and Monitoring

### What to Log

- Authentication events: login success/failure, logout, MFA challenges, password changes
- Authorization failures: access denied, privilege escalation attempts
- Input validation failures: malformed requests, injection attempts
- Rate limit violations, account lockouts
- Critical operations: data exports, admin actions, configuration changes
- Errors and exceptions (with context, not full stack traces in production logs)

### What NOT to Log

- **Never log passwords, tokens, session IDs, API keys, credit card numbers, or PII** in plaintext
- Redact sensitive fields in logs: `email: user@***`, `token: [REDACTED]`
- Avoid logging full request/response bodies in production unless necessary (and sanitized)

### Monitoring and Alerting

- Alert on: multiple failed logins, privilege escalation attempts, unusual access patterns, high error rates
- Use centralized logging (ELK, Splunk, CloudWatch) for correlation and anomaly detection
- Implement real-time monitoring for brute-force attacks, credential stuffing, API abuse
- Retain logs for compliance (30-90 days minimum, often longer for audit trails)

## Data Protection

### Encryption

- **Encrypt data in transit**: Use TLS 1.2+ for all external communication; prefer TLS 1.3
- Enforce HTTPS-only; redirect HTTP to HTTPS; set `Strict-Transport-Security` (HSTS) header
- **Encrypt data at rest**: Use AES-256 for database encryption, file storage, backups
- Use managed encryption services (AWS KMS, Azure Key Vault, GCP KMS) for key management
- Rotate encryption keys regularly; automate key rotation where possible

### Secrets Management

- Store secrets in dedicated secret managers (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
- Never commit secrets to version control (use `.gitignore`, pre-commit hooks, secret scanning)
- Rotate secrets regularly (API keys, database passwords, certificates)
- Use environment variables or runtime injection for secrets—never hardcode
- Limit secret access: only services that need a secret should access it

### Sensitive Data Handling

- Minimize PII collection: only collect what's necessary
- Tokenize or hash sensitive data (e.g., credit cards, SSNs) when possible
- Implement data retention policies: delete data when no longer needed
- Provide user data export and deletion (GDPR, CCPA compliance)

## API Security

- Use HTTPS-only; reject HTTP requests
- Validate `Content-Type` and `Accept` headers
- Implement CORS policies: restrict allowed origins, methods, headers
- Version APIs (`/v1/`, `/v2/`) to allow deprecation without breaking clients
- Document API security requirements (authentication, rate limits, permissions) in OpenAPI/Swagger specs
- Use API gateways for centralized authentication, rate limiting, logging

## Prohibited Insecure Patterns

### Never Do This

- **String concatenation for SQL queries**: Use parameterized queries
- **Storing plaintext passwords**: Hash with bcrypt/Argon2
- **Using MD5 or SHA-1 for passwords**: These are broken; use bcrypt/scrypt/Argon2
- **Trusting client-side validation alone**: Always validate server-side
- **Exposing stack traces or internal errors to clients**: Return generic error messages; log details server-side
- **Using `eval()` or `exec()` with user input**: Code injection risk
- **Disabling SSL/TLS certificate validation**: Man-in-the-middle attacks
- **Hardcoding secrets in code**: Use environment variables or secret managers
- **Allowing unrestricted file uploads**: Validate file type, size, content; store outside web root; scan for malware
- **Using default credentials**: Change all default passwords, API keys, admin accounts
- **Ignoring security headers**: Set CSP, HSTS, X-Frame-Options, X-Content-Type-Options
- **Returning different error messages for "user not found" vs "wrong password"**: Enables user enumeration; return generic "invalid credentials"

### Security Headers

Set these headers on all API responses (especially if consumed by browsers):

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
X-XSS-Protection: 1; mode=block
```

For JSON APIs, omit `X-Frame-Options` and CSP if not rendering HTML.

## Dependency and Configuration Security

- **Keep dependencies updated**: Use automated tools (Dependabot, Snyk, npm audit) to detect vulnerabilities
- Audit dependencies regularly; remove unused libraries
- Pin dependency versions in production; avoid `latest` tags
- Use security-focused linters (Bandit for Python, Brakeman for Ruby, ESLint security plugins for JS)
- Disable debug modes, verbose error messages, and directory listings in production
- Remove or secure admin panels, debug endpoints, and internal APIs from public access
- Use environment-specific configurations: strict in production, permissive in development

## Error Handling

- Return **generic error messages** to clients: `{"error": "Invalid request"}` or `{"error": "Unauthorized"}`
- Log detailed errors server-side with request IDs for correlation
- Never expose: stack traces, database schema, internal paths, library versions, or SQL errors
- Use HTTP status codes correctly: 400 (bad input), 401 (unauthenticated), 403 (forbidden), 404 (not found), 429 (rate limited), 500 (server error)
- Implement global error handlers to catch unhandled exceptions and prevent information leakage

## Security Testing and Audits

- Conduct regular **penetration testing** and security audits
- Use automated scanners (OWASP ZAP, Burp Suite) in CI/CD pipelines
- Perform code reviews with security focus: check for injection, auth bypasses, hardcoded secrets
- Test for OWASP Top 10 vulnerabilities: injection, broken auth, sensitive data exposure, XXE, broken access control, security misconfiguration, XSS, insecure deserialization, using components with known vulnerabilities, insufficient logging
- Simulate attacks: SQL injection, XSS, CSRF, brute force, privilege escalation, IDOR
- Maintain a responsible disclosure policy for security researchers

## Compliance and Standards

- Follow **OWASP API Security Top 10 2023**: Broken Object Level Authorization, Broken Authentication, Broken Object Property Level Authorization, Unrestricted Resource Consumption, Broken Function Level Authorization, Unrestricted Access to Sensitive Business Flows, Server-Side Request Forgery, Security Misconfiguration, Improper Inventory Management, Unsafe Consumption of APIs
- Adhere to industry standards: PCI-DSS (payment data), HIPAA (health data), GDPR/CCPA (privacy)
- Document security controls and threat models
- Implement incident response plans: detection, containment, eradication, recovery, lessons learned

## When to Apply This Skill

Use these rules when:

- Building or reviewing backend APIs, microservices, or web services
- Implementing authentication or authorization logic
- Handling user input, database queries, or external API calls
- Designing systems that process sensitive data (PII, financial, health)
- Conducting security audits or code reviews
- Responding to security incidents or vulnerabilities

These controls are not optional for production systems. Security is foundational, not a feature to add later. Apply defense in depth: multiple layers ensure that a single failure doesn't lead to full compromise.
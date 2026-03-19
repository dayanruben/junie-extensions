---
name: "security-audit"
description: "Security checklist and patterns for Spring Boot applications: OWASP Top 10, input validation, injection prevention, and Spring Security configuration (JWT, OAuth2, method security). Use when reviewing code security, implementing authentication, or user asks about vulnerabilities."
---

# Security Skill

## Scope and Version Notes

- This is the canonical skill for security review, Spring Security configuration, and secure coding checks.
- Examples are security pattern fragments; verify Spring Security DSL/API names against the project version before applying them.
- For Bean Validation details, defer to `validation-patterns` instead of repeating the same rules here.

## When to Use
- Security code review or before production release
- Implementing authentication / authorization
- User asks about "security", "vulnerability", "OWASP", "JWT", "403", "401"
- Checking for injection vulnerabilities

---

## OWASP Top 10 Quick Reference

| # | Risk | Mitigation |
|---|------|------------|
| A01 | Broken Access Control | Role checks at service layer, deny by default |
| A02 | Cryptographic Failures | Strong algorithms, no hardcoded secrets |
| A03 | Injection | Parameterized queries, input validation |
| A04 | Insecure Design | Threat modeling, secure defaults |
| A05 | Security Misconfiguration | Disable debug, secure headers, no default creds |
| A06 | Vulnerable Components | Dependency scanning, keep updated |
| A07 | Authentication Failures | BCrypt/Argon2, MFA, short-lived tokens |
| A08 | Data Integrity Failures | Verify signatures, safe deserialization |
| A09 | Logging Failures | Log security events, never log sensitive data |
| A10 | SSRF | Validate URLs, allowlist domains |

OWASP code patterns → [references/OWASP-PATTERNS.md](references/OWASP-PATTERNS.md)

---

## Spring Security Configuration

For JWT authentication, SecurityFilterChain, OAuth2 resource server, method-level security:

→ [references/SPRING-SECURITY-CONFIG.md](references/SPRING-SECURITY-CONFIG.md)

---

## Key Rules

### Never hardcode secrets
```java
// ❌ private static final String API_KEY = "sk-1234...";
// ✅ @Value("${api.key}") private String apiKey;
```

### Authorize at service layer, not just controller
```java
// ❌ Only checked in @GetMapping — IDOR risk
public Document getDocument(Long id) {
    return documentRepository.findById(id).orElseThrow();
}

// ✅ Check ownership in service
public Document getDocument(Long id, User currentUser) {
    Document doc = documentRepository.findById(id).orElseThrow();
    if (!doc.getOwnerId().equals(currentUser.getId())) {
        throw new AccessDeniedException("Not authorized");
    }
    return doc;
}
```

### Hash passwords with BCrypt
```java
// ✅ Start with strength 10-12 and calibrate for the production latency budget
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
// ❌ MD5/SHA1/SHA256 — never for passwords
```

### Never log sensitive data
```java
// ❌ log.info("Login: user={}, password={}", username, password);
// ✅ log.info("Login attempted", kv("userId", userId), kv("ip", ip));
```

---

## Security Checklist

### Code Review
- [ ] Input validated with allowlist patterns
- [ ] SQL queries use parameters (no string concatenation)
- [ ] Authorization checked at service layer
- [ ] No hardcoded secrets
- [ ] Passwords hashed with BCrypt or Argon2
- [ ] Sensitive data not logged
- [ ] CSRF enabled for browser/session apps; disabled for stateless JWT APIs

### Spring Security (if applicable)
- [ ] JWT secret ≥ 256 bits from environment variable
- [ ] Session policy `STATELESS` for JWT APIs
- [ ] Auth endpoints explicitly `permitAll()`
- [ ] `@EnableMethodSecurity` if using `@PreAuthorize`
- [ ] CORS configured with explicit origins (no `*` with credentials)

### Configuration
- [ ] HTTPS enforced
- [ ] Security headers configured (CSP, HSTS, X-Frame-Options)
- [ ] Debug/dev features disabled in production
- [ ] Error messages don't leak internal details

### Dependencies
- [ ] No known CVEs (run OWASP Dependency Check)
- [ ] Dependencies up to date

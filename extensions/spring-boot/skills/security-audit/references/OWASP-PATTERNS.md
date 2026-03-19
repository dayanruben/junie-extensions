# OWASP Security Patterns Reference

## Input Validation

```java
// ✅ Allowlist — permit only known-good characters
private static final Pattern SAFE_NAME = Pattern.compile("^[a-zA-Z\\s'-]{1,100}$");
if (!SAFE_NAME.matcher(input).matches()) {
    throw new ValidationException("Invalid name format");
}

// ❌ Blocklist — attackers find bypasses
if (input.contains("<script>")) { ... }  // insufficient

// Bean Validation (JSR 380)
public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Alphanumeric and underscore only")
    String username,

    @NotBlank @Email
    String email,

    @NotBlank @Size(min = 8)
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$",
             message = "Must contain uppercase, lowercase, and digit")
    String password
) {}
```

---

## SQL Injection Prevention

```java
// ✅ JPA — parameterized
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// ✅ JDBC — PreparedStatement
String sql = "SELECT * FROM users WHERE email = ? AND active = ?";
try (PreparedStatement stmt = connection.prepareStatement(sql)) {
    stmt.setString(1, email);
    stmt.setBoolean(2, true);
    ResultSet rs = stmt.executeQuery();
}

// ❌ String concatenation — NEVER
String sql = "SELECT * FROM users WHERE email = '" + email + "'";   // VULNERABLE
```

---

## XSS Prevention

```java
// Thymeleaf auto-escapes with th:text (safe)
// <p th:text="${userInput}">...</p>

// Manual encoding when needed
import org.owasp.encoder.Encode;
String safe = Encode.forHtml(userInput);
String safeJs = Encode.forJavaScript(userInput);

// Content Security Policy header
http.headers(h -> h
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'; script-src 'self'"))
    .frameOptions(f -> f.deny())
    .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000))
);
```

Maven dependency:
```xml
<dependency>
    <groupId>org.owasp.encoder</groupId>
    <artifactId>encoder</artifactId>
    <version>1.2.3</version>
</dependency>
```

---

## CSRF

```java
// Stateless JWT REST API — disable
http.csrf(csrf -> csrf.disable());

// Browser app with sessions — enable with cookie token
http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

---

## Password Storage

```java
// ✅ BCrypt (widely supported)
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(rawPassword);
encoder.matches(rawPassword, hash);  // verify

// ✅ Argon2 (recommended for new projects)
PasswordEncoder encoder = Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8();

// ❌ NEVER use MD5, SHA1, SHA256 for passwords
String hash = DigestUtils.md5Hex(password);  // completely insecure
```

---

## Secrets Management

```java
// ❌ Hardcoded
private static final String DB_PASSWORD = "admin123";

// ✅ Environment variables
@Value("${db.password}")
private String dbPassword;
```

```yaml
# ✅ Reference env vars in config
spring.datasource.password: ${DB_PASSWORD}

# ❌ Hardcoded in YAML
spring.datasource.password: admin123
```

`.gitignore` — never commit:
```
.env
*.pem
*.key
*credentials*
application-local.yml
```

---

## Secure Deserialization

```java
// ❌ Java ObjectInputStream — Remote Code Execution risk
ObjectInputStream ois = new ObjectInputStream(untrustedInput);
Object obj = ois.readObject();  // DANGEROUS

// ✅ Jackson with safe configuration
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    mapper.deactivateDefaultTyping();  // prevents gadget attacks
    return mapper;
}
```

---

## Security Headers

| Header | Recommended Value | Purpose |
|---|---|---|
| `Content-Security-Policy` | `default-src 'self'` | Prevent XSS |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `Strict-Transport-Security` | `max-age=31536000` | Force HTTPS |

---

## Dependency Vulnerability Check

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.3</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
    </configuration>
    <executions>
        <execution><goals><goal>check</goal></goals></execution>
    </executions>
</plugin>
```

```bash
mvn dependency-check:check
# Report: target/dependency-check-report.html
```

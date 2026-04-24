# Security — policy & pitfalls

Generic Spring Security 6 knowledge (`SecurityFilterChain`, `HttpSecurity` DSL, `UserDetailsService`, `BCryptPasswordEncoder`) is assumed. This file covers the decisions that actually matter.

## Minimal filter chain (Spring Security 6 / Spring Boot 3.x)

- Exactly one `@Bean SecurityFilterChain`. No `WebSecurityConfigurerAdapter` — it's removed in Spring Security 6.
- Explicit `authorizeHttpRequests { … }` — no `permitAll()` at the end by default. Default deny.
- Explicit choice between stateless (`SessionCreationPolicy.STATELESS`, JWT / resource server) and stateful (default, session cookie, CSRF on).
- `requestMatchers(...)` — never `antMatchers(...)` (removed).

## CSRF

- **Default is ON and should stay on for stateful (cookie-session) apps.** Disabling CSRF on a cookie-auth API is a real vulnerability, not a nuisance.
- Disable **only** for fully stateless APIs with `Authorization: Bearer …` (JWT / opaque token) and `SessionCreationPolicy.STATELESS`. In that case `.csrf(csrf -> csrf.disable())` is correct.
- Never `.csrf(csrf -> csrf.disable())` "to make it work" during development — fix the client (send the token) instead.

## Passwords

- `BCryptPasswordEncoder` or `Argon2PasswordEncoder`. Never `NoOpPasswordEncoder` outside tests.
- `DelegatingPasswordEncoder` (Spring Security default via `PasswordEncoderFactories.createDelegatingPasswordEncoder()`) — lets you upgrade algorithms without migrating all existing hashes at once.
- Never compare passwords with `equals` — always `encoder.matches(raw, encoded)`.

## JWT / OAuth2 resource server

- Prefer `spring-boot-starter-oauth2-resource-server` + `oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()))` over hand-rolled JWT parsing.
- Configure `spring.security.oauth2.resourceserver.jwt.issuer-uri=...` — Spring fetches the JWKS automatically, rotates keys, validates `iss` / `exp` / `nbf`.
- Do not validate tokens manually in a filter — `BearerTokenAuthenticationFilter` does it for you.
- Role / scope mapping: inject a `JwtAuthenticationConverter` with a `JwtGrantedAuthoritiesConverter` (authorities prefix `ROLE_` vs `SCOPE_`) instead of sprinkling `.hasAuthority("SCOPE_read")` ad-hoc.

## Method security

- Enable with `@EnableMethodSecurity` (the `prePostEnabled = true` default is correct in Spring Security 6). `@EnableGlobalMethodSecurity` is deprecated.
- `@PreAuthorize("hasRole('ADMIN')")` on service layer, not controllers — protects the method regardless of how it's called.
- SpEL: `@PreAuthorize("#userId == authentication.principal.id")` for resource-owner checks.
- Method security uses proxies — same rules as `@Transactional`: public non-final, no self-invocation.

## Endpoint hardening

- `/actuator/*` — by default only `/health` and `/info` are exposed over HTTP. Don't blindly `management.endpoints.web.exposure.include=*`. If you need `/prometheus`, expose just that and put actuator behind a separate port / auth.
- `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security` — Spring Security sets sensible defaults. Don't disable them unless you know why.
- `server.error.include-stacktrace=never` and `server.error.include-message=never` in prod — otherwise validation errors leak object paths.

## CORS

- `.cors(Customizer.withDefaults())` picks up a `CorsConfigurationSource` bean — define one, don't sprinkle `@CrossOrigin` annotations.
- `allowedOriginPatterns("*")` is required instead of `allowedOrigins("*")` when `allowCredentials=true` — Spring enforces this (CORS spec forbids `*` + credentials).
- Don't set CORS headers manually in a filter — you'll double-send them when Spring Security's CORS filter also runs. Configure once via the bean.

## SPA + CSRF (cookie-based session)

- Default `CookieCsrfTokenRepository.withHttpOnlyFalse()` lets the JS SPA read the `XSRF-TOKEN` cookie and echo it back as `X-XSRF-TOKEN`. Needed for React/Vue/Angular clients.
- Since Spring Security 6, the default is deferred CSRF resolution — use `CsrfTokenRequestAttributeHandler` + setting its `csrfRequestAttributeName = null` if your SPA calls GETs that need the cookie primed. Otherwise the token isn't materialized until the first unsafe request.

## Common mistakes

| Anti-pattern | Why it bites |
|---|---|
| Disabling CSRF on a cookie-session app | Real CSRF exploit surface |
| Matching routes with `/admin/**` for both GET and POST where GET should be public | `requestMatchers(HttpMethod.POST, "/admin/**")` — always be method-specific |
| Configuring `permitAll()` at the end of the chain "for simplicity" | Any new endpoint is automatically open |
| Setting `SessionCreationPolicy.STATELESS` but still using `HttpSession` somewhere | Bugs that appear only under load |
| Using `@Secured` + `@PreAuthorize` mixed — different SpEL support | Pick `@PreAuthorize` and stick with it |
| Storing JWT secret in `application.yml` committed to git | Rotate by deleting the token? No. Externalize via env / Vault / Config Server. |
| Decoding the JWT yourself with `Jwts.parser()` | Reinvents expiry / kid / jwks — use `oauth2ResourceServer`. |

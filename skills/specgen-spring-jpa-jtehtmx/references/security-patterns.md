# Security Patterns — Keycloak OAuth2 Client with Spring Security

This reference describes the complete security architecture for the spec. Include all of
this content in Section 5 of the generated specification.

---

## Architecture Overview

The application operates as an OAuth2 Client using Authorization Code flow with OIDC.
Keycloak is the external identity provider that handles login UI, user registration,
and role management. The application's responsibilities are:

1. Redirect unauthenticated users to Keycloak login page
2. Handle the OAuth2 callback and establish an HTTP session
3. Extract roles from OIDC ID token claims
4. Enforce role-based access control on endpoints

The application uses server-side sessions (Spring's default). After successful OAuth2
login, the authenticated principal (`OidcUser`) is stored in the session. User identity
and roles are read from the OIDC ID token claims.

The flow:
```
Browser → GET /protected-page → Spring Security (not authenticated)
    → 302 Redirect to Keycloak /auth?response_type=code&client_id=...
    → User logs in at Keycloak
    → Keycloak redirects back with authorization code
    → Spring exchanges code for tokens (ID token + access token)
    → KeycloakGrantedAuthoritiesMapper extracts roles from ID token
    → Session created, OidcUser stored as principal
    → 302 Redirect to original page
```

### Configuration in application.yml

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: {{KEYCLOAK_CLIENT_ID}}
            client-secret: {{KEYCLOAK_CLIENT_SECRET}}
            scope: openid,profile,email
        provider:
          keycloak:
            issuer-uri: {{KEYCLOAK_ISSUER_URI}}

app:
  security:
    keycloak-client-id: {{KEYCLOAK_CLIENT_ID}}
```

---

## SecurityConfig.java

Complete sample code. The application uses OAuth2 Login with Keycloak as the identity
provider. Sessions are managed by Spring (default behavior). No custom login page is
needed — unauthenticated users are automatically redirected to Keycloak.

```java
package {{BASE_PACKAGE}}.config;

import {{BASE_PACKAGE}}.shared.security.CspNonceFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CspNonceFilter cspNonceFilter;
    private final KeycloakGrantedAuthoritiesMapper keycloakGrantedAuthoritiesMapper;
    private final LoginRedirectEntryPoint loginRedirectEntryPoint;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/static/**", "/assets/**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userAuthoritiesMapper(keycloakGrantedAuthoritiesMapper)
                )
                .defaultSuccessUrl("/home", true)
            )
            // Disable Spring's built-in logout to prevent LogoutFilter from intercepting
            // before our custom LogoutController can handle Keycloak end_session
            .logout(logout -> logout.disable())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(loginRedirectEntryPoint)
            )
            .addFilterBefore(cspNonceFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## KeycloakGrantedAuthoritiesMapper.java

This class bridges Keycloak's OIDC ID token claim structure to Spring Security
`GrantedAuthority` objects. It extracts roles from both `realm_access` and
`resource_access` claims in the ID token.

Complete sample code:

```java
package {{BASE_PACKAGE}}.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.authority.mapping.GrantedAuthoritiesMapper;
import org.springframework.security.oauth2.core.oidc.user.OidcUserAuthority;
import org.springframework.stereotype.Component;
import java.util.Collection;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

@Component
public class KeycloakGrantedAuthoritiesMapper implements GrantedAuthoritiesMapper {

    @Value("${app.security.keycloak-client-id}")
    private String clientId;

    @Override
    public Collection<? extends GrantedAuthority> mapAuthorities(
            Collection<? extends GrantedAuthority> authorities) {
        Set<GrantedAuthority> mapped = new HashSet<>();
        for (GrantedAuthority authority : authorities) {
            if (authority instanceof OidcUserAuthority oidc) {
                mapped.addAll(extractRealmRoles(oidc));
                mapped.addAll(extractResourceRoles(oidc));
            }
            mapped.add(authority);
        }
        return mapped;
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractRealmRoles(OidcUserAuthority oidc) {
        Map<String, Object> realmAccess = oidc.getIdToken().getClaim("realm_access");
        if (realmAccess == null) return Collections.emptyList();
        List<String> roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return Collections.emptyList();
        return roles.stream()
            .map(role -> (GrantedAuthority) new SimpleGrantedAuthority("ROLE_" + role))
            .toList();
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractResourceRoles(OidcUserAuthority oidc) {
        Map<String, Object> resourceAccess = oidc.getIdToken().getClaim("resource_access");
        if (resourceAccess == null) return Collections.emptyList();
        Map<String, Object> resource = (Map<String, Object>) resourceAccess.get(clientId);
        if (resource == null) return Collections.emptyList();
        List<String> roles = (List<String>) resource.get("roles");
        if (roles == null) return Collections.emptyList();
        return roles.stream()
            .map(role -> (GrantedAuthority) new SimpleGrantedAuthority("ROLE_" + role))
            .toList();
    }
}
```

---

## Role-Based Access Control

### At the endpoint level (method security)

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public String delete(@PathVariable String id) { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'USER')")
@GetMapping
public String list(Model model) { ... }
```

The `ROLE_` prefix is handled by the mapper. In `@PreAuthorize`, use role names
WITHOUT the prefix (Spring Security strips it automatically in `hasRole()`).

### Role constants

Define roles as a constants class to avoid string duplication:

```java
public final class Roles {
    public static final String ADMIN = "ADMIN";
    public static final String USER = "USER";
    private Roles() {}
}
```

---

## SecurityContextUtil.java

Utility to extract user information from the OIDC principal in the current security
context. Reads from the session-stored `OidcUser` principal.

```java
package {{BASE_PACKAGE}}.shared.util;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Component;
import java.util.Collections;
import java.util.Set;
import java.util.stream.Collectors;

@Component
public class SecurityContextUtil {

    public String getCurrentUserId() {
        OidcUser user = getOidcUser();
        return user != null ? user.getSubject() : null;
    }

    public String getCurrentUsername() {
        OidcUser user = getOidcUser();
        return user != null ? user.getPreferredUsername() : null;
    }

    public String getCurrentUserEmail() {
        OidcUser user = getOidcUser();
        return user != null ? user.getEmail() : null;
    }

    public Set<String> getCurrentRoles() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) return Collections.emptySet();
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .filter(a -> a.startsWith("ROLE_"))
            .map(a -> a.substring(5))
            .collect(Collectors.toSet());
    }

    public OidcUser getOidcUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof OidcUser oidcUser) {
            return oidcUser;
        }
        return null;
    }
}
```

This class is injected via constructor into:
- `AuditAwareImpl` — for `@CreatedBy` / `@LastModifiedBy` fields
- Service classes — when business logic depends on the current user
- The MDC correlation filter — for logging (see Section 11)

---

## AuditAwareImpl.java

Implements `AuditorAware<String>` to populate `@CreatedBy` and `@LastModifiedBy` fields
on JPA entities or MongoDB documents. Uses constructor injection for `SecurityContextUtil`.

```java
package {{BASE_PACKAGE}}.shared.audit;

import {{BASE_PACKAGE}}.shared.util.SecurityContextUtil;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.AuditorAware;
import org.springframework.stereotype.Component;
import java.util.Optional;

@Component
@RequiredArgsConstructor
public class AuditAwareImpl implements AuditorAware<String> {

    private final SecurityContextUtil securityContextUtil;

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(securityContextUtil.getCurrentUsername())
            .or(() -> Optional.of("system"));
    }
}
```

---

## OIDC ID Token Structure Reference

For the spec, include a sample decoded ID token payload showing where Keycloak places
roles, so developers understand the claim structure:

```json
{
  "sub": "f1234-abcd-5678",
  "preferred_username": "john.doe",
  "email": "john.doe@example.com",
  "realm_access": {
    "roles": ["USER", "offline_access"]
  },
  "resource_access": {
    "{{KEYCLOAK_CLIENT_ID}}": {
      "roles": ["ADMIN"]
    }
  },
  "scope": "openid profile email"
}
```

This helps developers understand exactly what `KeycloakGrantedAuthoritiesMapper` is parsing.

---

## CSP Nonce Filter

For the web application, a Content Security Policy nonce is generated per request to allow
inline scripts while maintaining CSP security.

```java
package {{BASE_PACKAGE}}.shared.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;
import java.security.SecureRandom;
import java.util.Base64;

@Component
public class CspNonceFilter extends OncePerRequestFilter {

    private static final SecureRandom RANDOM = new SecureRandom();

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        byte[] nonceBytes = new byte[32];
        RANDOM.nextBytes(nonceBytes);
        String nonce = Base64.getUrlEncoder().withoutPadding().encodeToString(nonceBytes);

        request.setAttribute("cspNonce", nonce);
        // 'unsafe-eval' required for Alpine.js v3 expression evaluation (new Function())
        // 'data:' in font-src required for Vite-bundled font data URIs
        response.setHeader("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' 'nonce-" + nonce + "' 'unsafe-eval'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "font-src 'self' data:; " +
            "img-src 'self' data:; " +
            "connect-src 'self';");

        filterChain.doFilter(request, response);
    }
}
```

---

## AuthAware Mixin

Mixin interface for controllers to extract user information from the OIDC principal.

```java
package {{BASE_PACKAGE}}.shared.security;

import {{BASE_PACKAGE}}.shared.fragment.UserProfile;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;

public interface AuthAware {
    default UserProfile currentUser(HttpServletRequest req) {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof OidcUser oidcUser) {
            return new UserProfile(
                oidcUser.getSubject(),
                oidcUser.getPreferredUsername(),
                oidcUser.getEmail()
            );
        }
        return UserProfile.anonymous();
    }
}
```

---

## LoginRedirectEntryPoint

Custom `AuthenticationEntryPoint` that redirects to the login page when authentication
fails (e.g., session expired, no valid session). For HTMX requests, it returns an
`HX-Redirect` header instead of a 302 redirect so that htmx can handle the full page
navigation.

This prevents the application from showing a raw 401 error page when the user's session
expires while they are idle.

```java
package {{BASE_PACKAGE}}.shared.security;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class LoginRedirectEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException, ServletException {
        if ("true".equals(request.getHeader("HX-Request"))) {
            // HTMX request — send HX-Redirect header so htmx triggers full page navigation
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setHeader("HX-Redirect", "/login");
        } else {
            // Regular browser request — standard redirect
            response.sendRedirect("/login");
        }
    }
}
```

Wire this into SecurityConfig via:
```java
.exceptionHandling(ex -> ex
    .authenticationEntryPoint(loginRedirectEntryPoint)
)
```

---

## LogoutController

Custom logout controller that handles Keycloak end-session. Spring Security's built-in
`LogoutFilter` must be disabled (`.logout(logout -> logout.disable())`) because it
intercepts GET `/logout` before the custom controller when CSRF is disabled.

The controller:
1. Reads the OIDC ID token from the authenticated session
2. Builds the Keycloak `end_session_endpoint` URL with `id_token_hint` and
   `post_logout_redirect_uri`
3. Invalidates the HTTP session
4. Redirects to Keycloak's end_session_endpoint to destroy the SSO session

**Prerequisite:** The Keycloak client must have `post_logout_redirect_uris` configured
(e.g., `http://localhost:8080/*`) via the Keycloak admin console or admin API.

```java
package {{BASE_PACKAGE}}.config;

import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.util.UriComponentsBuilder;

@Controller
@RequiredArgsConstructor
public class LogoutController {

    @Value("${spring.security.oauth2.client.provider.keycloak.issuer-uri}")
    private String issuerUri;

    @GetMapping("/logout")
    public String logout(HttpServletRequest request) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String idToken = null;

        if (auth != null && auth.getPrincipal() instanceof OidcUser oidcUser) {
            idToken = oidcUser.getIdToken().getTokenValue();
        }

        // Invalidate the HTTP session
        request.getSession().invalidate();
        SecurityContextHolder.clearContext();

        // Build Keycloak end_session_endpoint URL
        String endSessionUrl = UriComponentsBuilder
            .fromUriString(issuerUri + "/protocol/openid-connect/logout")
            .queryParam("post_logout_redirect_uri", getBaseUrl(request))
            .queryParamIfPresent("id_token_hint",
                idToken != null ? java.util.Optional.of(idToken) : java.util.Optional.empty())
            .toUriString();

        return "redirect:" + endSessionUrl;
    }

    private String getBaseUrl(HttpServletRequest request) {
        return request.getScheme() + "://" + request.getServerName() + ":" + request.getServerPort();
    }
}
```

---

## ThemeAware Mixin

Mixin interface for controllers to read the theme cookie.

```java
package {{BASE_PACKAGE}}.shared.security;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import java.util.Arrays;

public interface ThemeAware {
    default String theme(HttpServletRequest req) {
        if (req.getCookies() == null) return "light";
        return Arrays.stream(req.getCookies())
            .filter(c -> "theme".equals(c.getName()))
            .map(Cookie::getValue)
            .findFirst()
            .orElse("light");
    }
}
```

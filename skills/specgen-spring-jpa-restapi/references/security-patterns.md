# Security Patterns — Keycloak OAuth2 Resource Server with Spring Security

This reference describes the complete security architecture for the REST API spec. Include
all of this content in Section 6 of the generated specification.

---

## Architecture Overview

The application operates as an OAuth2 Resource Server that validates JWT tokens issued by
Keycloak. Unlike the OAuth2 Client pattern (used in web apps with server-rendered views),
the Resource Server pattern is stateless — no HTTP sessions, no login pages, no cookies.

The client (e.g., a frontend SPA, mobile app, or another service) obtains a JWT access
token from Keycloak directly and includes it in the `Authorization: Bearer` header of
every API request. The REST API validates this token and extracts roles for access control.

The flow:
```
Client → POST /realms/{realm}/protocol/openid-connect/token (to Keycloak)
    → Keycloak returns JWT access token
Client → GET /api/v1/resource (with Authorization: Bearer <token>)
    → Spring Security validates JWT signature via Keycloak's JWKS endpoint
    → JwtAuthenticationConverter extracts roles from JWT claims
    → Request proceeds to @RestController
    → JSON response
```

### Configuration in application.yml

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: {{KEYCLOAK_ISSUER_URI}}
          jwk-set-uri: {{KEYCLOAK_ISSUER_URI}}/protocol/openid-connect/certs

app:
  security:
    keycloak-client-id: {{KEYCLOAK_CLIENT_ID}}
    public-paths:
      - /api-docs/**
      - /swagger-ui/**
      - /swagger-ui.html
      - /actuator/health
      - /error
```

---

## SecurityConfig.java

Complete sample code. The application uses OAuth2 Resource Server with JWT validation.
Stateless — no sessions. Keycloak provides the JWT tokens.

```java
package {{BASE_PACKAGE}}.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfigurationSource;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CorsConfigurationSource corsConfigurationSource;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api-docs/**", "/swagger-ui/**",
                    "/swagger-ui.html", "/actuator/health", "/error").permitAll()
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(new KeycloakJwtGrantedAuthoritiesConverter());
        return converter;
    }
}
```

---

## KeycloakJwtGrantedAuthoritiesConverter.java

This class bridges Keycloak's JWT claim structure to Spring Security `GrantedAuthority`
objects. It extracts roles from both `realm_access` and `resource_access` claims in the
JWT access token.

```java
package {{BASE_PACKAGE}}.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Map;

@Component
public class KeycloakJwtGrantedAuthoritiesConverter
        implements Converter<Jwt, Collection<GrantedAuthority>> {

    @Value("${app.security.keycloak-client-id}")
    private String clientId;

    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        List<GrantedAuthority> authorities = new ArrayList<>();
        authorities.addAll(extractRealmRoles(jwt));
        authorities.addAll(extractResourceRoles(jwt));
        return authorities;
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractRealmRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess == null) return Collections.emptyList();
        List<String> roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return Collections.emptyList();
        return roles.stream()
            .map(role -> (GrantedAuthority) new SimpleGrantedAuthority("ROLE_" + role))
            .toList();
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractResourceRoles(Jwt jwt) {
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
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
public ResponseEntity<Void> delete(@PathVariable String id) { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'OPERATOR')")
@GetMapping
public ResponseEntity<Page<ProductDTO>> list(Pageable pageable) { ... }
```

The `ROLE_` prefix is handled by the converter. In `@PreAuthorize`, use role names
WITHOUT the prefix (Spring Security strips it automatically in `hasRole()`).

### Role Constants

Define roles as a constants class to avoid string duplication:

```java
package {{BASE_PACKAGE}}.shared.security;

public final class Roles {
    public static final String ADMIN = "ADMIN";
    public static final String OPERATOR = "OPERATOR";
    private Roles() {}
}
```

---

## SecurityContextUtil.java

Utility to extract user information from the JWT principal in the current security context.

```java
package {{BASE_PACKAGE}}.shared.security;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;
import java.util.Collections;
import java.util.Set;
import java.util.stream.Collectors;

@Component
public class SecurityContextUtil {

    public String getCurrentUserId() {
        Jwt jwt = getJwt();
        return jwt != null ? jwt.getSubject() : null;
    }

    public String getCurrentUsername() {
        Jwt jwt = getJwt();
        return jwt != null ? jwt.getClaimAsString("preferred_username") : null;
    }

    public String getCurrentUserEmail() {
        Jwt jwt = getJwt();
        return jwt != null ? jwt.getClaimAsString("email") : null;
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

    public Jwt getJwt() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof Jwt jwt) {
            return jwt;
        }
        return null;
    }
}
```

---

## AuditAwareImpl.java

Implements `AuditorAware<String>` to populate `@CreatedBy` and `@LastModifiedBy` fields
on JPA entities or MongoDB documents.

```java
package {{BASE_PACKAGE}}.shared.audit;

import {{BASE_PACKAGE}}.shared.security.SecurityContextUtil;
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

## JWT Access Token Structure Reference

For the spec, include a sample decoded JWT access token payload showing where Keycloak
places roles:

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
  "scope": "openid profile email",
  "iss": "{{KEYCLOAK_ISSUER_URI}}",
  "exp": 1710000000,
  "iat": 1709996400
}
```

---

## Testing with JWT

For integration tests, mock the JWT token:

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;

@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private ProductService productService;

    @Test
    void list_authenticatedUser_returns200() throws Exception {
        when(productService.findAll(any(), any())).thenReturn(Page.empty());

        mockMvc.perform(get("/api/v1/products")
                .with(jwt().authorities(new SimpleGrantedAuthority("ROLE_ADMIN"))))
            .andExpect(status().isOk());
    }

    @Test
    void list_unauthenticated_returns401() throws Exception {
        mockMvc.perform(get("/api/v1/products"))
            .andExpect(status().isUnauthorized());
    }
}
```

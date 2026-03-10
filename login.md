# Authentication & Login Flow Documentation

## User Types
The application architecture segregates user logins into three distinct hierarchical domains, each with a specialized authentication service:

1. **Root user** (`ROOT`): Global platform administrators (handled by [RootAuthService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/RootAuthService.java#19-69)).
2. **Global user / CEO** (`CEO`): Owners/creators of distinct SaaS tenant workspaces (handled by `CeoAuthService`).
3. **Tenant user** (`TENANT`): Regular employees operating exclusively within their company's localized schema (handled by [TenantAuthService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/TenantAuthService.java#22-82)).

---

## 1. Root User Login

### Required Login Inputs
* `identifier`: Username or Email
* [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56): Raw password string
* *Note: No tenant/schema headers are required.*

### Authentication Flow
1. **Request**: POST to `/api/v1/auth/root/login`
2. **Auth Service**: Handled by [RootAuthService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/RootAuthService.java#19-69).
3. **DB Validation**: Looks up user by `username` or `email` in the `public.root_users` table.
4. **Validation Rules**: Verifies user active status (`isActive == true`) and utilizes BCrypt `passwordEncoder.matches()` to validate password credentials.
5. **Token Generation**: Generates Access and Refresh JWTs setting type to `ROOT`.

---

## 2. Global User (CEO) Login

### Required Login Inputs
* `identifier`: Username or Email
* [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56): Raw password string
* *Note:* While a CEO owns a tenant schema, they do *not* require an `X-Tenant-ID` header during login, as their details are persisted globally.

### Authentication Flow
1. **Request**: POST to `/api/v1/auth/ceo/login`
2. **Auth Service**: Handled by `CeoAuthService`.
3. **DB Validation**: Queries the global `public.global_users` table using the provided identifier.
4. **Validation Rules**: Verifies user active status and matching BCrypt password.
5. **Token Generation**: Claims are formulated mapping `userType` to `CEO` and heavily including the `targetSchema` (the tenant schema the CEO owns).

---

## 3. Tenant User Login

### Required Login Inputs
* `identifier`: Username or Email
* [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56): Raw password string
* `X-Tenant-ID`: The HTTP Header mapping to the specific company PostgreSQL schema.

### Authentication Flow
1. **Request**: POST to `/api/v1/auth/login` containing the `X-Tenant-ID` header.
2. **Routing Integration**: The `TenantResolverFilter` traps the header and establishes the context to point Hibernate towards the company's localized schema.
3. **Auth Service**: Handled by [TenantAuthService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/TenantAuthService.java#22-82).
4. **DB Validation**: Searches the `users` table directly *inside* the established company schema.
5. **Validation Rules**: Asserts the user is active, asserts matching BCrypt credentials, updates the `last_login_at` timestamp.
6. **Token Generation**: Generates tokens embedding the matched `tenantSchema`, `roleId`, and `userType` = `TENANT`.

---

## Response Structure
All three login procedures format their response via an `AuthResponse` object.

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR...",
  "tokenType": "Bearer",
  "userType": "TENANT", 
  "expiresIn": 86400000,
  "username": "jane_smith",
  "userId": 45,
  "role": "MANAGER",
  "tenant": "acm_ser_X8B9Q2"
}
```
* **token**: The short-lived JWT used in `Authorization: Bearer <token>` headers.
* **refreshToken**: A longer-lived JWT utilized explicitly at the `/refresh` endpoint.
* **userType**: Informs the client of the account's overarching domain (e.g. `TENANT`).
* **tenant**: The internal system schema designation mapping to their workspace.

---

## JWT Structure
The Application utilizes a sparse JWT approach. The claims map consists primarily of session-centric metadata:

* `sub`: the user identifier (username)
* `userType`: String (`ROOT` | `CEO` | `TENANT`)
* `userId`: Long (Foreign Key mapping)
* `role`: String Base role execution tier (mostly used for basic UI hints rather than granular security limits)
* `roleId`: Long Identifier (for tenant caching associations)
* `tenantSchema`: String (Critical parameter authorizing schema association during subsequent interactions).
* `iat` / `exp`: Issue and Expiry Unix timestamps.

---

## Authorization After Login
Once logged in, the Client application must include the resulting JWT `accessToken` in the `Authorization` header on all subsequent requests. The [JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140) decodes it unconditionally.

During authorization, the embedded `tenantSchema` is dynamically compared against the `X-Tenant-ID` header to prevent cross-tenant spoofing. If validated, the `TenantContext` is locked, preventing unauthorized lateral data access into other tenant tables.

---

## Security Notes
* **Password Encoding**: Implemented globally utilizing standard Spring Boot `BCryptPasswordEncoder`.
* **Stateless Session**: The application exclusively utilizes Spring Security `SessionCreationPolicy.STATELESS`. The JVM memory is purged of all authentication contexts upon the HTTP request terminating (`SecurityContextHolder.clearContext()`).
* **Token Expiry**: Handled implicitly by `jjwt` library (`io.jsonwebtoken.JwtException` catches expiry configurations automatically injected by `JwtProperties`). 

---

## Internal Technical Flow (Spring Security)
1. **Unsecured Bypass**: Login endpoints (`/api/v1/auth/**`) are manually exposed in the `SecurityFilterChain` (`HttpSecurity.authorizeHttpRequests()`).
2. **Context Independence**: Services invoke `UsernamePasswordAuthenticationToken` or raw Repository validation directly. Standard Spring `AuthenticationManagerBuilder` forms are intentionally bypassed as token-oriented domains often require varying metadata properties (like `TenantContext`).
3. **Response**: Tokens are formatted and dispatched locally. Note that since these endpoints are permitted all, `SecurityContextHolder` is not permanently mutated *during* login execution, it is only finalized on subsequent inbound requests validating the spawned token.

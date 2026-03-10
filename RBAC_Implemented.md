# Role-Based Access Control (RBAC) Architecture

## Overview
The access control mechanism implemented in this project is a **Dynamic, Tenant-Aware Hybrid RBAC model**. 

It is "hybrid" because it combines traditional broad Role checks (e.g., `CEO`, `ROOT`) with granular, database-driven dynamic permission checks (`MODULE_ACTION`). It seamlessly handles multitenancy by dynamically switching the database context (`TenantContext`) and regulating users strictly within their localized tenant schemas, while allowing platform-level superusers global capabilities.

## Current Architecture
The security and authorization flow consists of several specialized layers working together:

- **SecurityConfig**: The entry point for Spring Security. It operates statelessly (`SessionCreationPolicy.STATELESS`), disables default CSRF, and enforces security over REST endpoints. It establishes the filter execution order, injecting `TenantResolverFilter` followed by [JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140).
- **JWT Filter ([JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140))**: Intercepts requests, extracts JWT claims, performs tenant-spoofing validation (comparing the token's schema claim to the HTTP header), establishes the `TenantContext`, and initiates the authority materialization process.
- **Authentication Provider**: Authentication is fully token-driven. State is validated not against a session ID but by decrypting and verifying the JWT signature per request.
- **UserDetails / Principal Model ([AuthUserPrincipal](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/AuthUserPrincipal.java#16-64))**: A custom implementation of Spring Security's `UserDetails`. It encapsulates runtime identity semantics including `userId`, `userType`, `tenantSchema`, `roleName`, and dynamically loaded `GrantedAuthority` records. 
- **Authority Loading Mechanism ([AuthorityLoaderService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/AuthorityLoaderService.java#19-77))**: Abstracted service that translates basic JWT claims into exhaustive programmatic authorities. It hydrates the principal's authorities from the `{tenant_schema}.user_permissions` table (leveraging `@Cacheable` for performance).
- **Permission Validation Flow ([RbacPermissionEvaluator](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#18-94))**: A custom implementation of Spring Security's [PermissionEvaluator](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#18-94). It intercepts declarative `@PreAuthorize` checks, dynamically builds the required authority string (`MODULE_ACTION`), and matches it against the principal's loaded authorities. It automatically short-circuits and approves all checks for `ROOT` and `CEO` user types.

## Role Structure
The system distinguishes strictly between Global (Platform) roles and localized Tenant roles.

* **Root User (`ROOT`)**: The highest-level platform administrator. Validated against the `public.root_users` table. Bypasses all granular permission checks.
* **Global User (`CEO`)**: The owner/creator of a tenant SaaS subscription. Validated against the `public.global_users` table. Tied to a specific `targetSchema` but systematically bypasses granular [hasPermission](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#52-80) restrictions to act as a localized superadmin.
* **Tenant User (`TENANT`)**: Scoped strictly to their company’s schema. Governed heavily by specific granular permissions bound to their localized roles and individual `user_permissions` overrides.

## Authority Model
Granular permissions utilize a strict, dynamic concatenation of modules and actions.
*   **Naming Convention**: `[MODULE_NAME]_[ACTION_NAME]` (Always upper case).
*   **Examples**:
    *   `BRANCH_MANAGEMENT_READ`
    *   `PRODUCT_MANAGEMENT_ADD`
    *   `USER_ADD` 

Base roles are also registered as authorities with a `ROLE_` prefix (e.g., `ROLE_MANAGER`).

## Permission Resolution Flow
The end-to-end execution flow of a secured request follows this lifecycle:

1.  **Request** arrives at the container.
2.  **Token Validation**: [JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140) parses the token and extracts `tenantSchema`, `userType`, `roleId`, and `userId`.
3.  **Cross-Tenant Verification**: The filter ensures the incoming `X-Tenant-ID` matches the `tenantSchema` enclosed securely inside the JWT.
4.  **Authority Extraction**: [AuthorityLoaderService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/AuthorityLoaderService.java#19-77) queries the database (or cache) mapping the authenticated user to their specific `MODULE_ACTION` privileges.
5.  **Spring Security Decision**: An [AuthUserPrincipal](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/AuthUserPrincipal.java#16-64) containing the extensive `GrantedAuthority` array is injected into the `SecurityContext`.
6.  **Controller Access**: The router hits the Controller logic, encountering `@PreAuthorize`.
7.  **Evaluation**: [RbacPermissionEvaluator](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#18-94) steps in, validating if the [AuthUserPrincipal](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/AuthUserPrincipal.java#16-64) possesses the explicit authority or if the user is a `CEO`/`ROOT`.

## @PreAuthorize Usage
Permissions are enforced at the Controller method level using Spring Security's Method Security rules. 
Methods can utilize:
* Native Role Verification: `@PreAuthorize("hasRole('CEO')")`
* Literal Authority Checking: `@PreAuthorize("hasAuthority('USER_ADD')")`
* Custom Evaluator Evaluation: `@PreAuthorize("hasPermission('MODULE', 'ACTION')")`

## JWT Permission Storage
To prevent infinite token bloat and optimize the token lifecycle, the system follows a sparse-token approach:

* **What goes inside the token**: Only identity identifiers and grouping claims—`userId`, `userType` (`ROOT`/`CEO`/`TENANT`), `tenantSchema`, and `role`. 
* **How extracted**: `JwtService.extractAllClaims()` parses the payload body inside [JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140).
* **How attached**: Using the extracted sparse claims, [AuthorityLoaderService](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/AuthorityLoaderService.java#19-77) dynamically populates the exhaustive `GrantedAuthority` records locally from the DB/Cache. The full collection is embedded into the [AuthUserPrincipal](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/AuthUserPrincipal.java#16-64) without traveling over the HTTP wire.

## Workflow Diagram

```text
HTTP Request 
   │
   ▼
[ TenantResolverFilter ]  ───── (Extracts & Validates X-Tenant-ID header)
   │
   ▼
[ JwtAuthFilter ]  ──────────── (Extracts JWT, validates signature & tenant spoofing)
   │
   ▼
[ AuthorityLoaderService ] ──── (Loads 'MODULE_ACTION' from DB / Spring Cache)
   │
   ▼
[ SecurityContextHolder ] ───── (Stores AuthUserPrincipal with GrantedAuthorities)
   │
   ▼
[ RbacPermissionEvaluator ] ─── (Intercepts @PreAuthorize, explicitly bypasses CEO/ROOT)
   │
   ▼
[ Controller ] ──────────────── (Executes Business Logic if Authorized)
```

## Advantages of Existing Architecture
* **Strict Tenant Isolation**: [JwtAuthFilter](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/jwt/JwtAuthFilter.java#28-140) cross-verifies `X-Tenant-ID` against the secured JWT `tenantSchema`, effectively eliminating cross-tenant privilege escalation (spoofing).
* **Token Optimization**: By omitting large arrays of authorities from the JWT payload and fetching them dynamically, JWTs remain lightweight and stateless.
* **Performance**: Granular authority derivation requires DB joins. This overhead is mitigated using Spring `@Cacheable` on [loadTenantAuthorityNames](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/auth/service/AuthorityLoaderService.java#48-71).
* **Super-admin Flexibility**: Hardcoding `ROOT` and `CEO` bypass logic in [RbacPermissionEvaluator](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#18-94) prevents overhead operations for administrative accounts.

## Disadvantages / Current Limitations
* **Thread Contamination Risk**: Relying on strict `finally` constructs to clear the `TenantContext` and `SecurityContextHolder`. If a system exception heavily disrupts the filter chain before the `finally` block activates, thread pool contamination could temporarily persist.
* **Cache Invalidation Coupling**: Tenant administrators modifying permissions must programmatically guarantee the execution of `AuthorityLoaderService.evictTenantAuthorities()`. If missed, localized caching can lead to elevated privilege persistence until cache expiry.

## Scalability Notes
The decoupled nature of the DB `search_path` (schema per tenant) combined with JVM-level stateless JWT validation allows immense vertical and horizontal scalability. Standard caching deployments (like Redis integrations) can immediately distribute the `tenant-authorities` cache layer seamlessly to containerized orchestrations.

## Recommended Improvements
* **Active Token Revocation**: Introduce a Redis-based JWT Blacklist mechanism to explicitly kill compromised tokens prior to expiration.
* **Strict Eviction Listeners**: Utilize JPA `@PostUpdate` entity listeners or Spring Application Events when `UserPermission` entities are altered to guarantee cache invalidation automatically.

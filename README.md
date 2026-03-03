I'll generate the complete documentation file for your Hybrid RBAC system. Since I cannot create downloadable files directly, I'll provide the full Markdown content that you can copy into a file named `RBAC_SECURITY.md`.

````markdown
# ERP Security Architecture

## Hybrid RBAC – Static Platform Roles + Dynamic Tenant Permissions

**Version:** 1.0  
**Last Updated:** March 2026  
**Status:** Production Ready

---

## Table of Contents

1. [Overview](#1-overview)
2. [Security Domains](#2-security-domains)
3. [Role Model](#3-role-model)
4. [Database Structure](#4-database-structure)
5. [JWT Strategy](#5-jwt-strategy)
6. [Hybrid Authorization Model](#6-hybrid-authorization-model)
7. [CEO Full Access Strategy](#7-ceo-full-access-strategy)
8. [Permission Versioning & Cache](#8-permission-versioning--cache)
9. [Tenant Provisioning Flow](#9-tenant-provisioning-flow)
10. [Implementation Examples](#10-implementation-examples)
11. [Security Rules](#11-security-rules)
12. [Performance Considerations](#12-performance-considerations)

---

## 1. Overview

This ERP system implements a **Hybrid Role-Based Access Control (RBAC)** model that combines:

| Aspect               | Implementation                             |
| -------------------- | ------------------------------------------ |
| **Platform Access**  | Static roles (`ROLE_SERAVION`, `ROLE_CEO`) |
| **Tenant Access**    | Dynamic permissions (`MODULE:ACTION`)      |
| **Authentication**   | JWT tokens (small payload)                 |
| **Authorization**    | Database + Cache with versioning           |
| **Tenant Isolation** | Schema-per-tenant with JWT-based routing   |

### Key Principles

- **Small JWTs**: Only authentication + tenant context + minimal role claims
- **Dynamic Permissions**: Module/Action permissions stored in database, not JWT
- **Immediate Updates**: Permission versioning for cache invalidation
- **Strict Isolation**: Platform and tenant domains are completely separated

---

## 2. Security Domains

### 2.1 Platform Domain (`/platform/**`)

**Purpose:**

- CEO registration and authentication (pre-tenant)
- Company registration and document submission
- Seravion provider verification and approval
- Subscription management and billing
- Tenant provisioning orchestration

**Base URL:** `/platform/**`

**Static Roles:**

- `ROLE_SERAVION` — Provider/root access for company verification and tenant management
- `ROLE_CEO` — Customer access for onboarding before tenant exists

**Authorization Method:**

```java
@PreAuthorize("hasRole('SERAVION')")
@PostMapping("/platform/company/{id}/approve")
public ResponseEntity<Void> approveCompany(@PathVariable UUID id) { ... }
```
````

**Platform JWT Claims:**

```json
{
  "sub": "uuid",
  "domain": "PLATFORM",
  "type": "PLATFORM",
  "userId": "uuid",
  "email": "admin@seravion.com",
  "roles": ["ROLE_SERAVION"],
  "iat": 1234567890,
  "exp": 1234571490
}
```

---

### 2.2 Tenant Domain (`/tenant/**`)

**Purpose:**

- All ERP operations within an activated tenant
- Branch management, invoicing, ledger, user management
- Dynamic permission-based access control

**Base URL:** `/tenant/**`

**Dynamic Permissions:**

- Format: `MODULE:ACTION` (e.g., `INVOICE:CREATE`, `BRANCH:DELETE`)
- Stored in tenant schema tables
- Evaluated at runtime via `PermissionService`

**Authorization Method:**

```java
@PreAuthorize("@perm.can(authentication, 'INVOICE', 'CREATE')")
@PostMapping("/tenant/invoices")
public ResponseEntity<Invoice> createInvoice(@RequestBody InvoiceRequest request) { ... }
```

**Tenant JWT Claims:**

```json
{
  "sub": "uuid",
  "domain": "TENANT",
  "type": "TENANT",
  "tenantUserId": "uuid",
  "schema": "tenant_abc123",
  "email": "user@company.com",
  "roles": ["TENANT_CEO", "BRANCH_ADMIN"],
  "iat": 1234567890,
  "exp": 1234571490
}
```

> **Important:** No module/action permissions are stored in the JWT.

---

## 3. Role Model

### 3.1 Static Roles (Platform Schema)

**Location:** `public` schema  
**Nature:** Immutable, system-defined  
**Usage:** Platform endpoints only

| Role            | Description               | Permissions                                                      |
| --------------- | ------------------------- | ---------------------------------------------------------------- |
| `ROLE_SERAVION` | Provider superuser        | Company verification, tenant provisioning, system administration |
| `ROLE_CEO`      | Pre-subscription customer | Company registration, document upload, subscription purchase     |

**Database Structure:**

```sql
-- public.seravion_users
CREATE TABLE seravion_users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- public.seravion_user_roles
CREATE TABLE seravion_user_roles (
    user_id UUID REFERENCES seravion_users(id),
    role VARCHAR(50) NOT NULL,
    PRIMARY KEY (user_id, role)
);
```

---

### 3.2 Dynamic Roles (Tenant Schema)

**Location:** Tenant-specific schema  
**Nature:** Mutable, tenant-defined  
**Usage:** ERP module access control

**Predefined System Roles:**
| Role | Description | Typical Permissions |
|------|-------------|---------------------|
| `TENANT_CEO` | Company owner | Full access (bypass checks) |
| `BRANCH_ADMIN` | Branch manager | Branch management, user management within branch |
| `ACCOUNTANT` | Finance user | Invoice, ledger, tax read/write |
| `SALES_EXECUTIVE` | Sales user | Product, invoice, customer management |

**Database Structure:**

```sql
-- tenant_{id}.roles
CREATE TABLE roles (
    id UUID PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    is_system_role BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- tenant_{id}.user_roles
CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id),
    role_id UUID REFERENCES roles(id),
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

---

## 4. Database Structure

### 4.1 Public Schema (Platform)

**Core Tables:**

| Table                    | Purpose                                        |
| ------------------------ | ---------------------------------------------- |
| `meta_admin`             | CEO account before tenant provisioning         |
| `company_details`        | Company registration, verification status, KYC |
| `document_details`       | Uploaded KYC documents                         |
| `subscription_plan`      | Available subscription tiers                   |
| `subscription_purchased` | Active/past subscriptions                      |
| `seravion_users`         | Provider users                                 |
| `seravion_user_roles`    | Provider role assignments                      |
| `tenant_user_map`        | Maps platform user to tenant user              |

**Schema Diagram:**

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  meta_admin     │────▶│ company_details  │◄────│ document_details│
│  (CEO account)  │     │ (Registration)   │     │ (KYC docs)      │
└────────┬────────┘     └────────┬─────────┘     └─────────────────┘
         │                       │
         │              ┌────────┴────────┐
         │              │                 │
         ▼              ▼                 ▼
┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐
│ tenant_user_map │  │ subscription_    │  │ seravion_users  │
│ (Mapping table) │  │ purchased        │  │ (Provider)      │
└─────────────────┘  └──────────────────┘  └─────────────────┘
```

**Platform RBAC Tables (Optional):**

- `public.roles`
- `public.modules`
- `public.actions`
- `public.role_module_action_permission`

---

### 4.2 Tenant Schema (Per Company)

**Core Tables:**

| Table                           | Purpose                                              |
| ------------------------------- | ---------------------------------------------------- |
| `users`                         | Operational users                                    |
| `roles`                         | Dynamic role definitions                             |
| `user_roles`                    | User-to-role mappings                                |
| `modules`                       | ERP modules (INVOICE, BRANCH, etc.)                  |
| `actions`                       | CRUD actions (CREATE, READ, UPDATE, DELETE, APPROVE) |
| `role_module_action_permission` | Role-based permissions                               |
| `user_permission`               | User-specific overrides (optional)                   |
| `tenant_security_state`         | Permission versioning                                |

**Permission Resolution Tables:**

```sql
-- tenant_{id}.modules
CREATE TABLE modules (
    id UUID PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL, -- 'INVOICE', 'BRANCH'
    name VARCHAR(100) NOT NULL,
    description TEXT
);

-- tenant_{id}.actions
CREATE TABLE actions (
    id UUID PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL, -- 'CREATE', 'READ', 'UPDATE', 'DELETE', 'APPROVE'
    name VARCHAR(100) NOT NULL
);

-- tenant_{id}.role_module_action_permission
CREATE TABLE role_module_action_permission (
    id UUID PRIMARY KEY,
    role_id UUID REFERENCES roles(id),
    module_id UUID REFERENCES modules(id),
    action_id UUID REFERENCES actions(id),
    allowed BOOLEAN DEFAULT true,
    UNIQUE(role_id, module_id, action_id)
);

-- tenant_{id}.user_permission (Optional overrides)
CREATE TABLE user_permission (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    module_id UUID REFERENCES modules(id),
    action_id UUID REFERENCES actions(id),
    permission_type VARCHAR(10) CHECK (permission_type IN ('ALLOW', 'DENY')),
    UNIQUE(user_id, module_id, action_id)
);

-- tenant_{id}.tenant_security_state (Permission versioning)
CREATE TABLE tenant_security_state (
    id INTEGER PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    permission_version BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. JWT Strategy

### 5.1 Platform JWT

**Issued When:** CEO or Seravion user logs in via `/platform/auth/login`

**Token Characteristics:**

- Small payload (< 500 bytes)
- No permission claims
- Short expiration (15-30 minutes)
- Refresh token rotation supported

**Claims:**
| Claim | Description | Example |
|-------|-------------|---------|
| `sub` | Subject (user UUID) | `"550e8400-e29b-41d4-a716-446655440000"` |
| `domain` | Security domain | `"PLATFORM"` |
| `type` | Token type | `"PLATFORM"` |
| `userId` | Platform user ID | `"550e8400-e29b-41d4-a716-446655440000"` |
| `email` | User email | `"ceo@company.com"` |
| `roles` | Static roles | `["ROLE_CEO"]` |
| `companyId` | Associated company (CEO only) | `"660e8400-e29b-41d4-a716-446655440001"` |
| `verificationStatus` | Company status | `"PENDING"` |

**Usage:**

- Access `/platform/**` endpoints only
- Cannot access `/tenant/**` endpoints

---

### 5.2 Tenant JWT

**Issued When:**

- CEO exchanges platform token for tenant token after subscription activation
- Tenant user logs in via `/tenant/auth/login`

**Token Characteristics:**

- Small payload (< 600 bytes)
- Contains schema reference for routing
- Role names only, no permission expansion
- Short expiration (15-30 minutes)

**Claims:**
| Claim | Description | Example |
|-------|-------------|---------|
| `sub` | Subject (tenant user UUID) | `"770e8400-e29b-41d4-a716-446655440002"` |
| `domain` | Security domain | `"TENANT"` |
| `type` | Token type | `"TENANT"` |
| `tenantUserId` | Tenant user ID | `"770e8400-e29b-41d4-a716-446655440002"` |
| `schema` | Tenant schema name | `"tenant_abc123"` |
| `email` | User email | `"user@company.com"` |
| `roles` | Dynamic role names | `["TENANT_CEO", "BRANCH_ADMIN"]` |

**Token Exchange Flow:**

```
CEO (Platform Token)
    │
    ▼
POST /platform/auth/tenant-token
    │
    ├──► Validate platform token
    ├──► Check company status = APPROVED
    ├──► Check subscription = ACTIVE
    ├──► Lookup tenant schema
    ├──► Find tenant user mapping
    └──► Issue tenant JWT
```

---

## 6. Hybrid Authorization Model

### 6.1 Architecture Overview

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   API Request   │────▶│  JWT Validation  │────▶│ Tenant Context  │
│                 │     │  (Authentication)│     │  Resolution     │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                              ┌───────────────────────────┼───────────────────┐
                              │                           │                   │
                              ▼                           ▼                   ▼
                    ┌──────────────────┐      ┌──────────────────┐   ┌──────────────────┐
                    │  Platform Route  │      │  Tenant Route    │   │  Invalid Route   │
                    │  (/platform/**)  │      │  (/tenant/**)    │   │  (Reject 403)    │
                    └────────┬─────────┘      └────────┬─────────┘   └──────────────────┘
                             │                         │
                             ▼                         ▼
                    ┌──────────────────┐      ┌──────────────────┐
                    │ Static Role Check│      │ PermissionService│
                    │ @PreAuthorize    │      │ @perm.can()      │
                    │ hasRole('CEO')   │      │                  │
                    └──────────────────┘      └────────┬─────────┘
                                                       │
                              ┌────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ 1. Check Cache   │
                    │    (Caffeine/    │
                    │     Redis)       │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │ Cache Miss?      │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │ Yes              │ No
                    ▼                  ▼
            ┌──────────────────┐  ┌──────────────────┐
            │ 2. Load from DB  │  │ Return Cached    │
            │    - User roles  │  │ Permissions      │
            │    - Role perms  │  │                  │
            │    - User overrides│ └──────────────────┘
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ 3. Evaluate      │
            │    Permission    │
            │    - User DENY?  │
            │    - User ALLOW? │
            │    - Role ALLOW? │
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ 4. Update Cache  │
            │    (TTL: 5 min)  │
            └──────────────────┘
```

### 6.2 PermissionService Implementation

```java
@Service("perm")
public class PermissionService {

    private final TenantRoleRepository roleRepository;
    private final UserPermissionRepository userPermissionRepository;
    private final Cache<String, Set<String>> permissionCache;
    private final TenantSecurityStateRepository securityStateRepository;

    /**
     * Main entry point for permission checks
     */
    public boolean can(Authentication authentication, String module, String action) {
        if (authentication == null || !authentication.isAuthenticated()) {
            return false;
        }

        // Extract tenant context from JWT
        TenantToken token = (TenantToken) authentication.getPrincipal();
        String tenantSchema = token.getSchema();
        String userId = token.getTenantUserId();

        // Build cache key with version
        Long permissionVersion = getCurrentPermissionVersion(tenantSchema);
        String cacheKey = buildCacheKey(tenantSchema, userId, permissionVersion);

        // Get or compute permissions
        Set<String> permissions = permissionCache.get(cacheKey,
            k -> computePermissions(tenantSchema, userId));

        // Check specific permission
        String requiredPermission = module.toUpperCase() + ":" + action.toUpperCase();
        return permissions.contains(requiredPermission) ||
               permissions.contains(module.toUpperCase() + ":*"); // Wildcard support
    }

    /**
     * Compute permissions from database
     */
    private Set<String> computePermissions(String tenantSchema, String userId) {
        Set<String> permissions = new HashSet<>();

        // 1. Check user-specific overrides (highest priority)
        List<UserPermission> userOverrides = userPermissionRepository
            .findByUserIdAndTenantSchema(userId, tenantSchema);

        for (UserPermission override : userOverrides) {
            String perm = override.getModule().getCode() + ":" + override.getAction().getCode();
            if (override.getPermissionType() == PermissionType.DENY) {
                permissions.remove(perm);
                permissions.add("DENY:" + perm); // Track denials
            } else {
                permissions.add(perm);
            }
        }

        // 2. Load role-based permissions (if no explicit denial)
        List<Role> userRoles = roleRepository.findRolesByUserIdAndTenantSchema(userId, tenantSchema);

        for (Role role : userRoles) {
            List<RoleModuleActionPermission> rolePerms =
                roleRepository.findPermissionsByRoleId(role.getId());

            for (RoleModuleActionPermission rmp : rolePerms) {
                String perm = rmp.getModule().getCode() + ":" + rmp.getAction().getCode();
                // Only add if not explicitly denied
                if (!permissions.contains("DENY:" + perm)) {
                    permissions.add(perm);
                }
            }
        }

        // Clean up denial markers
        permissions.removeIf(p -> p.startsWith("DENY:"));

        return permissions;
    }

    private Long getCurrentPermissionVersion(String tenantSchema) {
        return securityStateRepository.findPermissionVersionBySchema(tenantSchema);
    }

    private String buildCacheKey(String tenantSchema, String userId, Long version) {
        return String.format("%s:%s:%d", tenantSchema, userId, version);
    }
}
```

### 6.3 Evaluation Rules (Priority Order)

1. **User-specific DENY** → Immediate rejection
2. **User-specific ALLOW** → Immediate approval
3. **Role-based ALLOW** → Approval if any role has permission
4. **Default** → Deny

```java
public boolean evaluatePermission(String userId, String module, String action,
                                  String tenantSchema) {
    String permissionKey = module + ":" + action;

    // 1. Check user overrides
    Optional<UserPermission> userOverride = userPermissionRepository
        .findByUserIdAndModuleAndAction(userId, module, action);

    if (userOverride.isPresent()) {
        return userOverride.get().getPermissionType() == PermissionType.ALLOW;
    }

    // 2. Check role permissions
    List<String> userRoles = userRolesRepository.findRoleNamesByUserId(userId);

    for (String role : userRoles) {
        boolean hasPermission = roleModuleActionPermissionRepository
            .existsByRoleAndModuleAndAction(role, module, action);
        if (hasPermission) return true;
    }

    // 3. Default deny
    return false;
}
```

---

## 7. CEO Full Access Strategy

### 7.1 Hybrid Check Pattern

Tenant CEO has full system access without explicit permission checks.

**Controller Implementation:**

```java
@RestController
@RequestMapping("/tenant/{tenantId}")
public class InvoiceController {

    @PreAuthorize("hasRole('TENANT_CEO') or @perm.can(authentication, 'INVOICE', 'CREATE')")
    @PostMapping("/invoices")
    public ResponseEntity<Invoice> createInvoice(
            @PathVariable String tenantId,
            @RequestBody @Valid InvoiceRequest request) {

        validateTenantAccess(tenantId);
        return ResponseEntity.ok(invoiceService.create(request));
    }

    @PreAuthorize("hasRole('TENANT_CEO') or @perm.can(authentication, 'INVOICE', 'READ')")
    @GetMapping("/invoices/{id}")
    public ResponseEntity<Invoice> getInvoice(
            @PathVariable String tenantId,
            @PathVariable UUID id) {

        validateTenantAccess(tenantId);
        return ResponseEntity.ok(invoiceService.findById(id));
    }

    @PreAuthorize("hasRole('TENANT_CEO') or @perm.can(authentication, 'INVOICE', 'DELETE')")
    @DeleteMapping("/invoices/{id}")
    public ResponseEntity<Void> deleteInvoice(
            @PathVariable String tenantId,
            @PathVariable UUID id) {

        validateTenantAccess(tenantId);
        invoiceService.delete(id);
        return ResponseEntity.noContent().build();
    }

    private void validateTenantAccess(String requestedTenantId) {
        String currentTenant = TenantContext.getCurrentTenant();
        if (!currentTenant.equals(requestedTenantId)) {
            throw new CrossTenantAccessException("Access denied");
        }
    }
}
```

### 7.2 Database Consistency

Even though CEO bypasses checks, grant all permissions for audit consistency:

```sql
-- On tenant provisioning, grant TENANT_CEO all permissions
INSERT INTO role_module_action_permission (role_id, module_id, action_id, allowed)
SELECT
    (SELECT id FROM roles WHERE name = 'TENANT_CEO'),
    m.id,
    a.id,
    true
FROM modules m
CROSS JOIN actions a;
```

---

## 8. Permission Versioning & Cache

### 8.1 Permission Versioning

**Purpose:** Immediate permission updates without waiting for cache TTL

**Implementation:**

```sql
-- Trigger on permission changes
CREATE OR REPLACE FUNCTION increment_permission_version()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE tenant_security_state
    SET permission_version = permission_version + 1,
        updated_at = CURRENT_TIMESTAMP
    WHERE id = 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all permission-modifying tables
CREATE TRIGGER trigger_user_roles_change
    AFTER INSERT OR UPDATE OR DELETE ON user_roles
    FOR EACH STATEMENT EXECUTE FUNCTION increment_permission_version();

CREATE TRIGGER trigger_role_permissions_change
    AFTER INSERT OR UPDATE OR DELETE ON role_module_action_permission
    FOR EACH STATEMENT EXECUTE FUNCTION increment_permission_version();

CREATE TRIGGER trigger_user_permissions_change
    AFTER INSERT OR UPDATE OR DELETE ON user_permission
    FOR EACH STATEMENT EXECUTE FUNCTION increment_permission_version();
```

### 8.2 Cache Configuration

**Caffeine (Single Instance):**

```java
@Configuration
public class CacheConfig {

    @Bean
    public Cache<String, Set<String>> permissionCache() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build();
    }
}
```

**Redis (Multi-Instance):**

```java
@Configuration
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

**Cache Key Format:**

```
{tenantSchema}:{userId}:{permissionVersion}
```

Example: `tenant_abc123:550e8400-e29b-41d4-a716-446655440000:42`

---

## 9. Tenant Provisioning Flow

### 9.1 Step-by-Step Process

```
┌─────────────────┐
│  1. CEO Registration  │
│  (Platform Domain)    │
│  • Create meta_admin  │
│  • Assign ROLE_CEO    │
│  • Issue Platform JWT │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. Company Registration │
│  • Fill company_details  │
│  • Upload documents      │
│  • Status: SUBMITTED     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. Provider Verification │
│  (ROLE_SERAVION required) │
│  • Review documents       │
│  • Approve/Reject         │
│  • Status: APPROVED       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. Subscription Purchase │
│  • Select plan            │
│  • Payment processing     │
│  • Status: ACTIVE         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. Tenant Provisioning   │
│  • Create schema: tenant_{id}  │
│  • Run Flyway migrations       │
│  • Seed base data:             │
│    - modules, actions          │
│    - roles (TENANT_CEO, etc.)  │
│    - permissions               │
│  • Create tenant CEO user      │
│  • Create tenant_user_map      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  6. Token Exchange        │
│  POST /platform/auth/tenant-token │
│  • Validate platform JWT          │
│  • Check subscription active      │
│  • Issue tenant JWT               │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  7. ERP Operations        │
│  (Tenant Domain)          │
│  • Use tenant JWT         │
│  • Dynamic permissions    │
│  • Full ERP access        │
└─────────────────┘
```

### 9.2 Provisioning Service Implementation

```java
@Service
public class TenantProvisioningService {

    @Autowired
    private Flyway flyway;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private TenantPermissionSeeder permissionSeeder;

    @Transactional("platformTransactionManager")
    public void provisionTenant(UUID companyId, UUID metaAdminId) {
        String tenantSchema = "tenant_" + companyId.toString().replace("-", "");

        try {
            // 1. Create schema
            jdbcTemplate.execute("CREATE SCHEMA IF NOT EXISTS " + tenantSchema);

            // 2. Run Flyway migrations for tenant
            Flyway tenantFlyway = Flyway.configure()
                .dataSource(dataSource)
                .schemas(tenantSchema)
                .locations("db/migration/tenant")
                .load();
            tenantFlyway.migrate();

            // 3. Seed base permissions
            permissionSeeder.seedBasePermissions(tenantSchema);

            // 4. Create tenant CEO user
            UUID tenantUserId = createTenantCEO(tenantSchema, metaAdminId);

            // 5. Create mapping
            createTenantUserMapping(metaAdminId, tenantSchema, tenantUserId);

            // 6. Update subscription with schema name
            subscriptionRepository.updateTenantSchema(companyId, tenantSchema);

        } catch (Exception e) {
            // Rollback: drop schema if exists
            jdbcTemplate.execute("DROP SCHEMA IF EXISTS " + tenantSchema + " CASCADE");
            throw new TenantProvisioningException("Failed to provision tenant", e);
        }
    }

    private void createTenantUserMapping(UUID metaAdminId, String tenantSchema, UUID tenantUserId) {
        TenantUserMap mapping = new TenantUserMap();
        mapping.setMetaAdminId(metaAdminId);
        mapping.setTenantSchema(tenantSchema);
        mapping.setTenantUserId(tenantUserId);
        mapping.setCreatedAt(Instant.now());
        tenantUserMapRepository.save(mapping);
    }
}
```

---

## 10. Implementation Examples

### 10.1 Spring Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Autowired
    private PlatformJwtFilter platformJwtFilter;

    @Autowired
    private TenantJwtFilter tenantJwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Platform routes
            .securityMatcher("/platform/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/platform/auth/**").permitAll()
                .requestMatchers("/platform/admin/**").hasRole("SERAVION")
                .requestMatchers("/platform/ceo/**").hasRole("CEO")
                .anyRequest().authenticated()
            )
            .addFilterBefore(platformJwtFilter, UsernamePasswordAuthenticationFilter.class)

            // Tenant routes (separate chain)
            .securityMatcher("/tenant/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/tenant/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(tenantJwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### 10.2 Tenant Context Filter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TenantContextFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
                                    throws ServletException, IOException {
        try {
            String token = extractToken(request);

            if (token != null && isTenantToken(token)) {
                String tenantSchema = extractTenantSchema(token);
                TenantContext.setCurrentTenant(tenantSchema);
            }

            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

### 10.3 Custom Permission Annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("@permissionService.hasPermission(authentication, #module, #action)")
public @interface RequirePermission {
    String module();
    String action();
}

// Usage
@RestController
public class ProductController {

    @RequirePermission(module = "PRODUCT", action = "CREATE")
    @PostMapping("/tenant/{tenantId}/products")
    public ResponseEntity<Product> createProduct(@RequestBody ProductRequest request) {
        // Implementation
    }
}
```

---

## 11. Security Rules

### 11.1 Must Do

| Rule                             | Implementation                                                  |
| -------------------------------- | --------------------------------------------------------------- |
| **Resolve tenant from JWT only** | Never accept `X-Tenant-ID` header without JWT validation        |
| **Default deny**                 | Return 403 if permission not explicitly granted                 |
| **Keep JWT small**               | Max 1KB, no permission lists in token                           |
| **Separate routes**              | `/platform/**` and `/tenant/**` strictly separated              |
| **Use LAZY fetching**            | `@ManyToMany(fetch = FetchType.LAZY)` for role associations     |
| **Validate tenant match**        | Every tenant endpoint must verify `jwt.schema == path.tenantId` |
| **Permission versioning**        | Increment version on every permission change                    |
| **Audit logging**                | Log all permission checks and failures                          |

### 11.2 Must Not Do

| Rule                                         | Risk                                 |
| -------------------------------------------- | ------------------------------------ |
| **Mix platform roles with tenant endpoints** | CEO could access other tenants' data |
| **Accept schema from request headers**       | Tenant spoofing vulnerability        |
| **Store module/action permissions in JWT**   | Token bloat, stale permissions       |
| **Use EAGER fetching for roles**             | N+1 query performance issues         |
| **Allow platform token on tenant endpoints** | Cross-domain security breach         |
| **Cache without version check**              | Stale permissions after updates      |

---

## 12. Performance Considerations

### 12.1 Database Optimization

```sql
-- Indexes for permission lookups
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_role_permissions_role_id ON role_module_action_permission(role_id);
CREATE INDEX idx_user_permissions_user_id ON user_permission(user_id);
CREATE INDEX idx_role_name ON roles(name);
```

### 12.2 Query Optimization

Use JOIN FETCH to avoid N+1:

```java
@Query("SELECT r FROM Role r " +
       "LEFT JOIN FETCH r.permissions " +
       "WHERE r.id IN (SELECT ur.role.id FROM UserRole ur WHERE ur.user.id = :userId)")
List<Role> findRolesWithPermissionsByUserId(@Param("userId") UUID userId);
```

### 12.3 Cache Metrics

Monitor cache performance:

```java
@Autowired
private Cache<String, Set<String>> permissionCache;

public void printCacheStats() {
    CacheStats stats = ((CaffeineCache) permissionCache).getNativeCache().stats();
    System.out.println("Hit Rate: " + stats.hitRate());
    System.out.println("Miss Rate: " + stats.missRate());
    System.out.println("Eviction Count: " + stats.evictionCount());
}
```

---

## Appendix A: Database Migration Scripts

### A.1 Public Schema Migration (V1\_\_init_platform.sql)

```sql
-- Users
CREATE TABLE meta_admin (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    phone VARCHAR(50),
    enabled BOOLEAN DEFAULT true,
    email_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Company registration
CREATE TABLE company_details (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meta_admin_id UUID REFERENCES meta_admin(id),
    company_name VARCHAR(255) NOT NULL,
    registration_number VARCHAR(100),
    tax_id VARCHAR(100),
    address TEXT,
    status VARCHAR(20) DEFAULT 'DRAFT', -- DRAFT, SUBMITTED, PENDING, APPROVED, REJECTED
    submitted_at TIMESTAMP,
    verified_at TIMESTAMP,
    verified_by UUID,
    rejection_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Subscription
CREATE TABLE subscription_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price_monthly DECIMAL(10,2),
    price_yearly DECIMAL(10,2),
    features JSONB,
    is_active BOOLEAN DEFAULT true
);

CREATE TABLE subscription_purchased (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES company_details(id),
    plan_id UUID REFERENCES subscription_plan(id),
    status VARCHAR(20) DEFAULT 'PENDING', -- PENDING, ACTIVE, EXPIRED, CANCELLED
    started_at TIMESTAMP,
    expires_at TIMESTAMP,
    tenant_schema_name VARCHAR(100),
    payment_reference VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tenant mapping
CREATE TABLE tenant_user_map (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meta_admin_id UUID REFERENCES meta_admin(id),
    tenant_schema_name VARCHAR(100) NOT NULL,
    tenant_user_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(meta_admin_id, tenant_schema_name)
);

-- Seravion provider users
CREATE TABLE seravion_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'SERAVION', -- SERAVION, SUPER_ADMIN
    enabled BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### A.2 Tenant Schema Migration (V1\_\_init_tenant.sql)

```sql
-- Security state
CREATE TABLE tenant_security_state (
    id INTEGER PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    permission_version BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO tenant_security_state (id, permission_version) VALUES (1, 0);

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    phone VARCHAR(50),
    is_active BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- RBAC tables
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    icon VARCHAR(50),
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true
);

CREATE TABLE actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    is_system_role BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE role_module_action_permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    action_id UUID REFERENCES actions(id) ON DELETE CASCADE,
    allowed BOOLEAN DEFAULT true,
    conditions JSONB, -- Optional: time-based, branch-based restrictions
    UNIQUE(role_id, module_id, action_id)
);

CREATE TABLE user_permission (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    action_id UUID REFERENCES actions(id) ON DELETE CASCADE,
    permission_type VARCHAR(10) CHECK (permission_type IN ('ALLOW', 'DENY')),
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    granted_by UUID REFERENCES users(id),
    expires_at TIMESTAMP,
    UNIQUE(user_id, module_id, action_id)
);

-- Indexes
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_role_permissions_role_id ON role_module_action_permission(role_id);
CREATE INDEX idx_user_permissions_user_id ON user_permission(user_id);
```

---

## Appendix B: Glossary

| Term                   | Definition                                             |
| ---------------------- | ------------------------------------------------------ |
| **SERAVION**           | Platform provider role with superuser access           |
| **CEO**                | Customer role for onboarding before tenant exists      |
| **Tenant**             | Isolated database schema per company                   |
| **Permission Version** | Incrementing counter for cache invalidation            |
| **Hybrid RBAC**        | Combination of static roles and dynamic permissions    |
| **TENANT_CEO**         | Full-access role within tenant schema                  |
| **Module**             | ERP functional area (INVOICE, BRANCH, etc.)            |
| **Action**             | CRUD operation (CREATE, READ, UPDATE, DELETE, APPROVE) |

---

**Document Owner:** Security Architecture Team  
**Review Cycle:** Quarterly  
**Next Review:** June 2026

```

```

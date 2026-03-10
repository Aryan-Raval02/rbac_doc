# Signup & Provisioning Workflow Documentation

## Global Signup Flow
The global signup process operates differently from standard user registration. It serves as the primary instantiation anchor for an entire SaaS sub-environment (Tenant).

* **Who can signup:** Any prospective customer attempting to create a new workspace on the platform. This route is fundamentally a public endpoint (`/api/v1/auth/signup`).
* **How company/global user is created:** A company registration inherently spawns two constructs simultaneously: 
  1. A [GlobalUser](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/ceo/service/GlobalUserService.java#7-20) representing the administrative controller (`CEO`) mapped directly to the `public` schema.
  2. A distinct database schema explicitly provisioned and uniquely migrated to accommodate their private corporate structure.

## Tenant Provisioning Flow
Orchestrated fundamentally by the [GlobalUserServiceImpl](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/ceo/service/GlobalUserServiceImpl.java#45-190).

1. **Schema Derivation**: The system invokes cryptographic abbreviation logic to produce a secure, Postgres-safe tenant hash designation. (e.g., `companyName="Acme Corp"` internally translates to `schema="ACM_SER_1A2B3D"`).
2. **Schema Creation & Migration**: Handled centrally by `TenantMigrationService` (frequently utilizing Flyway `locations`). A new PostgreSQL schema is created matching the derived hash, and the framework executes all foundational tenant `.sql` migrations generating `users`, `roles`, `modules`, `actions`, and configuration tables.
3. **Tenant Registration**: The `TenantRegistry` table in the `public` schema is updated.
4. **Initial Configuration**: During the migration pipeline (or subsequent tenant bootstrapping), the newly spawned database receives its baseline defaults, generating default administrative roles, module structures, and preliminary actions mapping.

---

## User Creation Permission Model (Inside the Tenant) 
User generation is strictly hierarchical and isolated inside each workspace.

* **Root User**: Bypasses the application. Root users typically do not create tenant-users as they lack the localized `X-Tenant-ID` context natively, though they technically hold platform authority globally.
* **Global Admin (CEO)**: Possesses absolute localized dominance. When executing tenant-scoped functionality, their underlying native authority (`hasRole('CEO')`) overrides all restrictive evaluation blocks within [RbacPermissionEvaluator](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/security/RbacPermissionEvaluator.java#18-94).
* **Tenant Admin / Tenant Users**: Bound entirely by Explicit Permission parameters. A tenant user can only create another tenant user if their specific identity explicitly maintains the `USER_ADD` authority map.

---

## Required Signup Inputs

**For CEO / Global Registry (`SignUpRequest`)**:
* `companyName`
* `fullName`
* `email`
* `username`
* [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56)
* `phoneNumber` (Optional)

**For Tenant Users (`CreateUserRequest`)**:
* `fullName`
* `email`
* `username`
* [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56)
* `phoneNumber`
* `roleId` (Must exist within the tenant schema)
* `permissions` array (Optional array of `moduleId` and `actionId` mappings constructing unique privilege overrides).

---

## Validation Rules & Duplicate Prevention

**Global Layer Prevention**:
The [GlobalUserServiceImpl](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/ceo/service/GlobalUserServiceImpl.java#45-190) rigorously guards execution preemptively:
1. Validates the generated schema hash does not exist within the `tenant_registry`.
2. Asserts the submitted `email` is absent from `global_users`.
3. Asserts the submitted `username` is absent from `global_users`.

**Tenant Layer Prevention**:
The [UserServiceImpl](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/user/service/UserServiceImpl.java#47-185) strictly defends the local domain:
1. Validates the submitted `email` or `username` do not already exist inside the local tenant's [user](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#79-91) table.
2. Validates that the requested `roleId` explicitly exists.
3. Validates that all supplied override `moduleId` and `actionId` mappings correspond to valid existing application behaviors explicitly located in the tenant database.

---

## Password Handling
Handled symmetrically across both [GlobalUser](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/ceo/service/GlobalUserService.java#7-20) and localized [User](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/user/service/UserServiceImpl.java#64-109) implementations. Passwords received via HTTP DTO endpoints are captured as cleartext and instantly obscured using Spring Security's `BCryptPasswordEncoder.encode(...)`. The persistent data layers (`passwordHash` / [password](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/config/SecurityConfig.java#49-56)) contain purely hashed byte arrays, guaranteeing database exposure cannot compromise plaintext secrets.

---

## Post Signup Internal Flow (Tenant User)

The specific application sequence inside [UserServiceImpl](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/user/service/UserServiceImpl.java#47-185) traces the following life cycle:

1. **Request** hits [UserController](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/user/controller/UserController.java#29-74) attached to HTTP header `X-Tenant-ID`.
2. **Filter & Validation**: Filter populates `TenantContext`. Annotations execute `@PreAuthorize("hasAuthority('USER_ADD') or hasRole('CEO')")`.
3. **Service Logic**: Checks for localized User existence (Email/Username).
4. **Role Validation**: Queries DB for requested `roleId`.
5. **Database Insert**: Triggers `userRepository.save(new User(...))` populating core user metadata.
6. **Role Assignment**: [User](file:///k:/JAVA%20FULL%20STACK%20DEVELOPER/RBAC/rbac/src/main/java/com/security/rbac/modules/user/service/UserServiceImpl.java#64-109) entity accepts the resolved `Role` entity dynamically.
7. **Permission Assignment**: The system loops over the `permissions` DTO array. It executes validations ensuring neither duplicates nor invalid module/action mappings are permitted, invoking `userPermissionRepository.save()` for each granular entity.
8. **Response Return**: Normalizes the finalized creation construct masking sensitive persistence parameters. 

---

## Security Constraints 
Creating users inside an organization leverages explicit endpoint protection.

```java
@PostMapping
@PreAuthorize("hasAuthority('USER_ADD') or hasRole('CEO')")
public ResponseEntity<ResponseStructure<UserResponse>> createUser(...)
```

A malicious user attempting to manufacture new authorized agents is systematically rejected uniformly unless they explicitly maintain either Top-Level Ownership (`CEO`) or precise localized Administrative Authority mappings (`USER_ADD`).

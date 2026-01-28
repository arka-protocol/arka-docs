# ARKA Engine Role-Based Security Model

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Internal / Customer-Facing

---

## Overview

This document defines ARKA Engine's comprehensive role-based access control (RBAC) security model, including role definitions, permission structures, access policies, and implementation guidelines.

---

## Table of Contents

1. [Security Model Architecture](#security-model-architecture)
2. [Role Hierarchy](#role-hierarchy)
3. [Permission System](#permission-system)
4. [Built-in Roles](#built-in-roles)
5. [Custom Roles](#custom-roles)
6. [Tenant & Multi-Tenancy Security](#tenant--multi-tenancy-security)
7. [Resource-Level Permissions](#resource-level-permissions)
8. [Access Control Policies](#access-control-policies)
9. [Delegation & Impersonation](#delegation--impersonation)
10. [Implementation Reference](#implementation-reference)

---

## Security Model Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Security Model                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │     Identity     │    │      Roles       │    │   Permissions    │  │
│  │                  │    │                  │    │                  │  │
│  │  • Users         │───▶│  • System Roles  │───▶│  • Actions       │  │
│  │  • Services      │    │  • Custom Roles  │    │  • Resources     │  │
│  │  • API Clients   │    │  • Tenant Roles  │    │  • Scopes        │  │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘  │
│           │                       │                       │             │
│           │                       ▼                       │             │
│           │              ┌──────────────────┐            │             │
│           │              │    Policies      │            │             │
│           │              │                  │            │             │
│           └─────────────▶│  • RBAC          │◀───────────┘             │
│                          │  • ABAC          │                           │
│                          │  • Conditional   │                           │
│                          └──────────────────┘                           │
│                                   │                                      │
│                                   ▼                                      │
│                          ┌──────────────────┐                           │
│                          │  Access Decision │                           │
│                          │    (Allow/Deny)  │                           │
│                          └──────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Security Layers

| Layer | Purpose | Implementation |
|-------|---------|----------------|
| **Authentication** | Verify identity | SSO, MFA, API Keys |
| **Authorization** | Verify permissions | RBAC + ABAC |
| **Resource Access** | Protect data | Row-level security |
| **Audit** | Track access | Immutable logging |

---

## Role Hierarchy

### Hierarchy Structure

```
                              ┌─────────────────┐
                              │   Super Admin   │
                              │  (Full Access)  │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
            ┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐
            │    System     │  │    Tenant     │  │   Security    │
            │     Admin     │  │     Admin     │  │     Admin     │
            └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
                    │                  │                  │
        ┌───────────┼───────────┐      │                  │
        │           │           │      │                  │
┌───────▼────┐ ┌────▼────┐ ┌────▼────┐ │           ┌──────▼──────┐
│ Compliance │ │   Rule  │ │  Report │ │           │   Auditor   │
│  Manager   │ │  Admin  │ │  Admin  │ │           │             │
└─────┬──────┘ └────┬────┘ └────┬────┘ │           └─────────────┘
      │             │           │      │
      │      ┌──────┴──────┐    │      │
      │      │             │    │      │
┌─────▼──────▼───┐   ┌─────▼────▼──────▼───┐
│   Compliance   │   │      Analyst        │
│    Analyst     │   │     (Read-only)     │
└────────────────┘   └─────────────────────┘
```

### Inheritance Model

```yaml
role_inheritance:
  # Child roles inherit permissions from parent
  mode: "additive"

  # Inheritance chain
  chains:
    - super_admin → system_admin → rule_admin → analyst
    - super_admin → tenant_admin → compliance_manager → compliance_analyst
    - super_admin → security_admin → auditor

  # Override behavior
  overrides:
    # Children can't override parent restrictions
    mode: "parent_restrictions_enforced"
```

---

## Permission System

### Permission Structure

```
permission = resource:action[:scope]

Examples:
  rules:read                    # Read all rules
  rules:write:own               # Write own rules only
  transactions:read:tenant      # Read tenant transactions
  alerts:manage:assigned        # Manage assigned alerts
  users:delete:subordinates     # Delete subordinate users
```

### Resource Categories

| Category | Resources | Description |
|----------|-----------|-------------|
| **Core** | rules, events, decisions | Compliance engine resources |
| **Monitoring** | transactions, alerts, entities | Transaction monitoring |
| **Reporting** | reports, dashboards, analytics | Reporting resources |
| **Administration** | users, roles, settings, tenants | System administration |
| **Audit** | audit_logs, access_logs | Audit resources |
| **Integration** | api_keys, webhooks, connections | Integration resources |

### Action Types

| Action | Description | Typical Use |
|--------|-------------|-------------|
| `read` | View resource | All users |
| `create` | Create new resource | Managers+ |
| `update` | Modify resource | Managers+ |
| `delete` | Remove resource | Admins only |
| `execute` | Run/trigger resource | Varies |
| `approve` | Approve changes | Managers+ |
| `export` | Export data | Controlled |
| `manage` | Full lifecycle | Resource owners |

### Scope Types

| Scope | Description | Example |
|-------|-------------|---------|
| `*` (all) | All resources of type | Admin access |
| `own` | User-owned resources | Personal items |
| `tenant` | Tenant resources | Multi-tenant |
| `team` | Team resources | Team sharing |
| `assigned` | Assigned resources | Work queues |
| `subordinates` | Subordinate users' resources | Manager access |

---

## Built-in Roles

### Super Admin

```yaml
super_admin:
  description: "Full system access - platform owner"
  level: 0  # Highest privilege

  permissions:
    - "*"  # All permissions

  restrictions:
    mfa_required: true
    ip_allowlist_enforced: true
    session_timeout: 1800  # 30 minutes

  capabilities:
    - manage_tenants
    - configure_platform
    - access_all_data
    - impersonate_users
    - view_all_audit_logs
```

### System Admin

```yaml
system_admin:
  description: "System administration - technical operations"
  level: 1
  inherits: []

  permissions:
    # User management
    - users:read
    - users:create
    - users:update
    - users:delete

    # Role management
    - roles:read
    - roles:create
    - roles:update
    - roles:delete

    # System settings
    - settings:read
    - settings:update

    # Integration management
    - api_keys:manage
    - webhooks:manage
    - connections:manage

    # Monitoring
    - system_health:read
    - metrics:read
    - logs:read

  restrictions:
    mfa_required: true
    cannot_access:
      - compliance_decisions
      - regulatory_reports
```

### Tenant Admin

```yaml
tenant_admin:
  description: "Tenant administration - organization owner"
  level: 2
  scope: tenant

  permissions:
    # Tenant users
    - users:read:tenant
    - users:create:tenant
    - users:update:tenant
    - users:delete:tenant

    # Tenant roles (custom)
    - roles:read:tenant
    - roles:create:tenant
    - roles:update:tenant

    # Tenant settings
    - tenant_settings:read
    - tenant_settings:update

    # Billing
    - billing:read
    - billing:update

    # All tenant data
    - rules:manage:tenant
    - transactions:read:tenant
    - alerts:manage:tenant
    - reports:manage:tenant

  restrictions:
    mfa_required: true
    tenant_isolation: strict
```

### Compliance Manager

```yaml
compliance_manager:
  description: "Compliance team lead - rule and alert management"
  level: 3
  inherits: [compliance_analyst]

  permissions:
    # Rule management
    - rules:create
    - rules:update
    - rules:approve
    - rules:publish

    # Alert management
    - alerts:assign
    - alerts:escalate
    - alerts:close
    - alerts:override

    # Report management
    - reports:create
    - reports:approve
    - reports:file  # File with regulators

    # Team management
    - team:read
    - team:assign_work
    - team:view_metrics

    # Entity management
    - entities:update
    - entities:risk_override

  restrictions:
    approval_required_for:
      - rules:publish
      - reports:file
```

### Compliance Analyst

```yaml
compliance_analyst:
  description: "Compliance analyst - day-to-day operations"
  level: 4

  permissions:
    # Rules
    - rules:read
    - rules:propose  # Can propose, not publish

    # Transactions
    - transactions:read
    - transactions:search

    # Alerts
    - alerts:read
    - alerts:claim
    - alerts:investigate
    - alerts:update
    - alerts:request_escalation

    # Entities
    - entities:read
    - entities:add_note

    # Decisions
    - decisions:read:assigned
    - decisions:make:assigned

    # Reports
    - reports:read
    - reports:draft

  restrictions:
    cannot_access:
      - user_management
      - system_settings
      - raw_audit_logs
```

### Auditor

```yaml
auditor:
  description: "Internal/external auditor - read-only audit access"
  level: 5

  permissions:
    # Comprehensive read access
    - rules:read
    - rules:history:read
    - transactions:read
    - alerts:read
    - alerts:history:read
    - decisions:read
    - decisions:history:read
    - entities:read
    - reports:read
    - audit_logs:read
    - access_logs:read
    - change_logs:read
    - user_activity:read

  restrictions:
    read_only: true
    export_requires_approval: true
    data_masking: optional
```

### API Integration

```yaml
api_integration:
  description: "Programmatic API access"
  level: 6

  permissions:
    # Event submission
    - events:submit
    - events:batch_submit

    # Decision retrieval
    - decisions:read
    - decisions:poll

    # Webhook management
    - webhooks:read:own
    - webhooks:update:own

    # Entity lookup
    - entities:read
    - entities:search

  restrictions:
    human_ui_access: false
    rate_limited: true
    ip_allowlist_required: true
```

### Viewer

```yaml
viewer:
  description: "Read-only access - minimal permissions"
  level: 7

  permissions:
    - dashboards:read
    - reports:read:published
    - rules:read:active
    - alerts:read:summary

  restrictions:
    detail_access: false
    export: false
    pii_masked: true
```

---

## Custom Roles

### Creating Custom Roles

```yaml
# Example: Custom role for Sanctions Specialist
custom_role:
  name: "sanctions_specialist"
  display_name: "Sanctions Specialist"
  description: "Specialist role for sanctions screening and management"

  # Base on existing role
  inherits: compliance_analyst

  # Additional permissions
  additional_permissions:
    - sanctions:manage
    - watchlists:manage
    - screening:execute
    - screening:override

  # Remove inherited permissions
  removed_permissions:
    - alerts:claim  # Only work on sanctions alerts

  # Custom restrictions
  restrictions:
    only_alert_types: ["sanctions", "pep", "adverse_media"]
    jurisdiction_restricted: true
```

### Custom Role Policies

```yaml
custom_role_policies:
  # Who can create custom roles
  creation:
    allowed_by: ["super_admin", "tenant_admin"]
    requires_approval: true
    approvers: ["security_admin"]

  # Permission boundaries
  boundaries:
    # Custom roles cannot exceed creator's permissions
    max_permissions: "creator_permissions"

    # Forbidden permissions for custom roles
    forbidden_permissions:
      - super_admin_actions
      - platform_configuration
      - tenant_creation

  # Lifecycle
  lifecycle:
    review_required: quarterly
    auto_disable_unused: 90  # days
```

---

## Tenant & Multi-Tenancy Security

### Tenant Isolation Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Platform Level                               │
│                    (Super Admin Access Only)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │    Tenant A     │  │    Tenant B     │  │    Tenant C     │     │
│  │                 │  │                 │  │                 │     │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │     │
│  │  │   Users   │  │  │  │   Users   │  │  │  │   Users   │  │     │
│  │  │   Roles   │  │  │  │   Roles   │  │  │  │   Roles   │  │     │
│  │  │   Rules   │  │  │  │   Rules   │  │  │  │   Rules   │  │     │
│  │  │   Data    │  │  │  │   Data    │  │  │  │   Data    │  │     │
│  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │     │
│  │                 │  │                 │  │                 │     │
│  │  [Isolated]     │  │  [Isolated]     │  │  [Isolated]     │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Tenant Security Policies

```yaml
tenant_security:
  # Data isolation
  data_isolation:
    mode: "strict"  # strict, logical, shared
    enforcement: "database_level"  # Row-level security

  # Cross-tenant access
  cross_tenant:
    allowed: false
    exceptions:
      - platform_support  # With audit trail

  # Tenant-specific settings
  per_tenant:
    custom_roles: true
    custom_permissions: limited
    sso_configuration: true
    mfa_policy: override_allowed
    password_policy: override_allowed

  # Resource limits
  limits:
    max_users: plan_based
    max_rules: plan_based
    max_api_keys: 50
    max_custom_roles: 25
```

### Tenant Role Scoping

```yaml
# All permissions automatically scoped to tenant
tenant_scoping:
  # These are automatically tenant-scoped
  auto_scoped_resources:
    - users
    - rules
    - transactions
    - alerts
    - entities
    - reports
    - api_keys

  # Platform-level resources (not tenant-scoped)
  platform_resources:
    - platform_settings
    - tenant_management
    - global_watchlists
```

---

## Resource-Level Permissions

### Row-Level Security

```yaml
row_level_security:
  # Transaction access
  transactions:
    policies:
      - name: "tenant_isolation"
        filter: "tenant_id = current_tenant_id()"

      - name: "jurisdiction_filter"
        filter: "jurisdiction IN user_jurisdictions()"
        apply_to: ["compliance_analyst"]

      - name: "assigned_only"
        filter: "assigned_to = current_user_id() OR assigned_to IS NULL"
        apply_to: ["alerts:manage:assigned"]

  # Entity access
  entities:
    policies:
      - name: "tenant_isolation"
        filter: "tenant_id = current_tenant_id()"

      - name: "pii_masking"
        mask_fields: ["ssn", "date_of_birth", "full_address"]
        apply_to: ["viewer", "api_integration"]
```

### Field-Level Permissions

```yaml
field_level_permissions:
  # Entity fields
  entity:
    ssn:
      read: ["compliance_analyst", "compliance_manager", "admin"]
      masked_for: ["viewer", "api_integration"]

    risk_score:
      read: ["*"]
      write: ["compliance_manager", "admin"]

    internal_notes:
      read: ["compliance_analyst", "compliance_manager"]
      write: ["compliance_analyst", "compliance_manager"]

  # Transaction fields
  transaction:
    amount:
      read: ["*"]

    account_number:
      read: ["compliance_analyst", "compliance_manager"]
      masked_for: ["viewer"]

    beneficiary_details:
      read: ["compliance_analyst", "compliance_manager"]
```

---

## Access Control Policies

### Time-Based Access

```yaml
time_based_access:
  policies:
    # Business hours only for certain actions
    - name: "business_hours_rule_changes"
      actions: ["rules:update", "rules:delete"]
      condition:
        hours: "08:00-18:00"
        timezone: "user_timezone"
        days: ["monday", "tuesday", "wednesday", "thursday", "friday"]
      exception_roles: ["super_admin"]

    # Extended hours for alert management
    - name: "extended_hours_alerts"
      actions: ["alerts:*"]
      condition:
        hours: "06:00-22:00"
        timezone: "user_timezone"
```

### Location-Based Access

```yaml
location_based_access:
  policies:
    # Admin actions from trusted locations only
    - name: "admin_location_restriction"
      roles: ["super_admin", "system_admin"]
      condition:
        ip_ranges:
          - "10.0.0.0/8"      # Corporate network
          - "192.168.0.0/16"  # VPN
        countries: ["US", "GB"]  # Allowed countries

    # API access from known IPs
    - name: "api_ip_restriction"
      roles: ["api_integration"]
      condition:
        ip_allowlist_required: true
```

### Risk-Based Access

```yaml
risk_based_access:
  # User risk scoring
  user_risk_factors:
    - failed_logins_24h
    - unusual_access_pattern
    - new_device
    - new_location
    - off_hours_access

  policies:
    # Step-up auth for risky sessions
    - name: "high_risk_step_up"
      condition:
        risk_score: ">= 70"
      action:
        require: "step_up_authentication"
        methods: ["webauthn", "totp"]

    # Restrict actions for medium risk
    - name: "medium_risk_restrictions"
      condition:
        risk_score: ">= 50"
      action:
        deny: ["bulk_export", "user_creation", "role_changes"]
```

---

## Delegation & Impersonation

### Permission Delegation

```yaml
delegation:
  # Delegation policies
  policies:
    # Managers can delegate to team members
    - name: "manager_delegation"
      delegator_roles: ["compliance_manager"]
      delegatable_permissions:
        - "alerts:manage"
        - "decisions:make"
      max_duration: "7d"
      requires_approval: false

    # Admin delegation requires approval
    - name: "admin_delegation"
      delegator_roles: ["tenant_admin"]
      delegatable_permissions:
        - "users:manage"
        - "roles:manage"
      max_duration: "24h"
      requires_approval: true
      approvers: ["super_admin"]

  # Audit
  audit:
    log_delegation_start: true
    log_delegation_end: true
    log_delegated_actions: true
```

### Impersonation (Support Access)

```yaml
impersonation:
  # Who can impersonate
  allowed_by:
    - super_admin
    - platform_support  # With approval

  # Restrictions
  restrictions:
    # Cannot impersonate higher privilege
    privilege_check: true

    # Time limits
    max_duration: "1h"

    # Excluded from impersonation
    protected_roles:
      - super_admin
      - security_admin

    # Protected actions during impersonation
    blocked_actions:
      - password_change
      - mfa_modification
      - role_changes
      - api_key_creation

  # Audit
  audit:
    record_all_actions: true
    notify_impersonated_user: true
    notification_delay: "0"  # Immediate
```

---

## Implementation Reference

### Database Schema

```sql
-- Roles table
CREATE TABLE roles (
    id UUID PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    display_name VARCHAR(200),
    description TEXT,
    is_system_role BOOLEAN DEFAULT FALSE,
    parent_role_id UUID REFERENCES roles(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(tenant_id, name)
);

-- Permissions table
CREATE TABLE permissions (
    id UUID PRIMARY KEY,
    resource VARCHAR(100) NOT NULL,
    action VARCHAR(50) NOT NULL,
    scope VARCHAR(50),
    description TEXT,
    UNIQUE(resource, action, scope)
);

-- Role permissions junction
CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(id),
    permission_id UUID REFERENCES permissions(id),
    granted_at TIMESTAMP DEFAULT NOW(),
    granted_by UUID REFERENCES users(id),
    PRIMARY KEY (role_id, permission_id)
);

-- User roles junction
CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id),
    role_id UUID REFERENCES roles(id),
    tenant_id UUID REFERENCES tenants(id),
    granted_at TIMESTAMP DEFAULT NOW(),
    granted_by UUID REFERENCES users(id),
    expires_at TIMESTAMP,
    PRIMARY KEY (user_id, role_id, tenant_id)
);

-- Row-level security policies
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON transactions
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

### Permission Check Implementation

```typescript
// services/authorization.ts
import { Permission, User, Resource } from '@arka/types';

export class AuthorizationService {
  async checkPermission(
    user: User,
    permission: string,
    resource?: Resource
  ): Promise<boolean> {
    // Parse permission
    const [resourceType, action, scope] = permission.split(':');

    // Get user's effective permissions
    const userPermissions = await this.getEffectivePermissions(user);

    // Check direct permission
    if (this.hasPermission(userPermissions, permission)) {
      // Apply scope and resource checks
      return this.checkScope(user, scope, resource);
    }

    // Check wildcard permissions
    if (this.hasPermission(userPermissions, `${resourceType}:*`)) {
      return this.checkScope(user, scope, resource);
    }

    if (this.hasPermission(userPermissions, '*')) {
      return true;
    }

    return false;
  }

  private async getEffectivePermissions(user: User): Promise<Permission[]> {
    // Get all roles including inherited
    const roles = await this.getUserRolesWithInheritance(user);

    // Aggregate permissions
    const permissions = new Set<Permission>();
    for (const role of roles) {
      const rolePerms = await this.getRolePermissions(role);
      rolePerms.forEach(p => permissions.add(p));
    }

    return Array.from(permissions);
  }

  private checkScope(
    user: User,
    scope: string | undefined,
    resource?: Resource
  ): boolean {
    if (!scope || scope === '*') return true;

    switch (scope) {
      case 'own':
        return resource?.ownerId === user.id;
      case 'tenant':
        return resource?.tenantId === user.tenantId;
      case 'team':
        return user.teamIds?.includes(resource?.teamId);
      case 'assigned':
        return resource?.assignedTo === user.id;
      default:
        return false;
    }
  }
}
```

### Middleware Example

```typescript
// middleware/requirePermission.ts
import { Request, Response, NextFunction } from 'express';
import { AuthorizationService } from '../services/authorization';

const authService = new AuthorizationService();

export function requirePermission(permission: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user;
    const resource = req.resource; // Set by previous middleware

    const hasPermission = await authService.checkPermission(
      user,
      permission,
      resource
    );

    if (!hasPermission) {
      // Log access denial
      await auditLog.record({
        type: 'ACCESS_DENIED',
        userId: user.id,
        permission,
        resource: resource?.id,
        timestamp: new Date(),
      });

      return res.status(403).json({
        error: 'Forbidden',
        message: 'Insufficient permissions',
      });
    }

    next();
  };
}

// Usage
app.get('/api/rules/:id',
  authenticate,
  loadResource('rule'),
  requirePermission('rules:read'),
  rulesController.get
);
```

---

## Compliance Mapping

| Requirement | Control | Section |
|-------------|---------|---------|
| SOC 2 CC6.1 | Logical access controls | §2-5 |
| SOC 2 CC6.2 | Role-based access | §2-4 |
| SOC 2 CC6.3 | Least privilege | §3, §7 |
| ISO 27001 A.9.2 | User access management | §2-5 |
| ISO 27001 A.9.4 | Access control | §7-8 |
| PCI DSS 7.1 | Restrict access | §3-5 |
| PCI DSS 7.2 | Access control systems | §10 |
| NIST 800-53 AC | Access control family | All |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Security Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025

# Roles, Permissions & RBAC

## Concepts

- **Role**: logical grouping of permissions, identified by immutable slug. Assigned to users via org memberships.
- **Permission**: grants access to specific resources/actions. Referenced by immutable slug. Can belong to multiple roles.

## Permission naming

Use `resource:action` pattern. Allowed delimiters: `-.:_*`. Keep slugs short — they're included in JWT claims (4KB cookie limit in browsers).

Examples: `posts:create`, `billing:manage`, `users:view`

## Default role

Every environment has a default `member` role (can't be deleted, but any role can be set as default). New org memberships get the default role automatically.

## Environment vs organization roles

- **Environment-level roles**: shared across all orgs. Configured in Dashboard → Authorization.
- **Organization-level custom roles**: scoped to a single org. Created via API or Dashboard per-org.

## Single role vs multiple roles

| Mode | Behavior | When to use |
|------|----------|-------------|
| Single (default) | One role per membership | Simple apps, predictable audits, small JWTs |
| Multiple (opt-in) | Union of all assigned role permissions | Cross-department access, additive permissions, temp access |

Enable multiple roles in Dashboard → Authorization. Each membership must have ≥1 role.

## Role assignment sources

### Manual
Assign via Dashboard or API (`update organization membership`).

### SCIM (Directory Sync)
Map IdP directory groups → roles in Admin Portal. Applied via Directory Provisioning.

### SSO
Map SSO profile groups → roles in Dashboard. Requires SSO JIT provisioning enabled.

### Priority
When multiple sources conflict: directory group > SSO group > manual assignment (for explicit mappings). Default mappings don't override manual assignments.

## Role-aware sessions

JWT access token includes `role` (slug) and `permissions` (array of slugs) claims. Changes propagate on next token refresh.

## Deleting roles

- **Single mode**: affected memberships reassigned to default role
- **Multiple mode**: deleted role removed from memberships; if it was the only role, membership gets default role
- Deletion is async — slight delay before all memberships update

## Priority order

Roles have a configurable priority order. Highest-priority role wins when sources conflict (single-role mode) or when migrating from multi to single role config.

# Data Model & Multi-Tenancy

## Core entities

- **User**: identified by email. Can have multiple auth methods (password, OAuth profiles, SSO). Identity linking is automatic — no duplicate emails.
- **Organization**: a tenant/workspace. Contains members, owns resources. No limit on org count.
- **Organization Membership**: joins user ↔ org. Has a role and status.

## B2C model

Flat user structure, no organizations needed. WorkOS authenticates users, you store them in your DB linked by WorkOS user ID. Simplest starting point.

## B2B model

Organizations as tenants. Users belong to orgs via memberships. Common patterns:

- **Multiple workspaces**: user can be in many orgs (Figma-style). Many-to-many.
- **Single workspace**: user belongs to exactly one org (employee survey tool). One-to-many.

Both supported without code changes — it's a business logic decision.

## Membership lifecycle

Statuses: `pending` → `active` → `inactive`

- **Pending**: user invited but hasn't accepted
- **Active**: user added or accepted invitation
- **Inactive**: membership deactivated (soft-delete, revokes sessions)

APIs: deactivate, reactivate, delete. Use deactivation when member data (messages, docs) should persist. Use hard delete when no data retention needed.

## Organization access

Scope resources to organizations, not users. When a user leaves an org, their data stays with the org.

## Automated provisioning

- **JIT provisioning**: users auto-join orgs when their email domain matches a verified org domain
- **Invitations**: explicit invite to org, regardless of email domain
- **Directory provisioning (SCIM)**: sync users from IdP directories

## Creating orgs for new users

For B2B apps where all activity requires an org context:

1. Check access token for `org_id` after sign-in
2. If missing, show org creation form
3. Create org via API, create membership, refresh token with new `organization_id`

## Domain verification

Org claims a domain (e.g., `acme.com`). All users with matching email domain are governed by org's domain policy. SSO users with verified domains skip email verification.

## Admin Portal

Hosted UI for customer IT admins to self-configure SSO connections. Your app generates an Admin Portal link → customer configures their IdP → SSO connection is active.

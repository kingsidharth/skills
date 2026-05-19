# Invitations

## Two-step flow

1. Inviter creates invitation (API or Dashboard) for an email + optional org
2. Invitee receives email, clicks link, signs up or signs in, joins the org

## Invitation types

**Org-specific**: invites user to a specific organization. On acceptance, user becomes org member.

**App-wide**: no org specified. Invitation to join the application itself. Enables organic growth via peer referrals.

## New vs existing users

- **New user**: email sent with signup link. Signs up and auto-joins the org.
- **Existing user**: email sent with sign-in link. Signing in adds them to the org.

If invited to multiple orgs, user only joins the org whose invitation they clicked.

## Invite-only signup

Disable public signup in Dashboard → Authentication → Sign up toggle. With signup disabled:
- AuthKit won't show signup form
- API registration blocked
- Valid invitation codes bypass the restriction, allowing invited users to register

Use for closed beta, enterprise-only apps, or quota-based referral systems.

## Email matching rules

**No org**: any email can accept.

**With org**:
- Consumer domains (Gmail, Yahoo): must use exact invited email
- Corporate domains: any email from the same domain can accept (e.g., invite to `user@acme.com` accepted by `other@acme.com`)

## API

Create invitations via `workos.userManagement.sendInvitation({ email, organizationId?, ... })`. List and manage via the Invitation API endpoints.

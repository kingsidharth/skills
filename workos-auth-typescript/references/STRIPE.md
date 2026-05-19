# Stripe Add-on

## Two features

1. **Stripe Entitlements**: subscription-based feature gating via access token claims
2. **Stripe Seat Sync**: org member count → Stripe billing meter for usage-based billing

Both require connecting WorkOS to Stripe via Stripe Connect (Dashboard → Authentication → Add-ons).

## Setup

1. Enable Stripe add-on in Dashboard
2. Connect Stripe account (not Sandbox — use Stripe test mode instead)
3. Set Stripe customer ID on each org:

```ts
const customer = await stripe.customers.create({
  email: user.email,
  name: organization.name,
  metadata: { organizationId: organization.id },
});

await workos.organizations.updateOrganization({
  organization: organization.id,
  stripeCustomerId: customer.id,
});
```

## Entitlements

Access token includes `entitlements` claim based on org's Stripe subscription. Use to gate features:

```ts
// After refreshing session to pick up new entitlements
const { sealedSession, entitlements } = await session.refresh();
```

Entitlements appear on next login or session refresh after subscription changes. Manually refresh session after checkout completion.

## Seat Sync

WorkOS creates a "User Seat Count" billing meter in Stripe. Active org member count sent as meter events. Use for per-seat usage-based pricing.

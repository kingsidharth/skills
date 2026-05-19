# Actions (Auth Webhooks)

## What they do

Actions let you inject custom logic into WorkOS auth flows. WorkOS calls your endpoint synchronously during the flow and waits for allow/deny response.

## Action types

- **Authentication**: runs after user completes auth (password, Magic Auth, SSO, social), before redirect to your app
- **User registration**: runs after registration attempt, before user is provisioned

## Setup

1. Create a public HTTPS endpoint accepting POST with `workos-signature` header
2. Register endpoint URL in Dashboard → Actions
3. Choose error handling: deny on failure (default) or allow on failure

## Signature verification

Use SDK method for validation:

```ts
import { WorkOS } from '@workos-inc/node';
const workos = new WorkOS(process.env.WORKOS_API_KEY);

const action = await workos.actions.constructAction({
  payload,
  sigHeader,
  secret: process.env.WORKOS_ACTIONS_SECRET,
});
```

Or verify manually using the `workos-signature` header + your actions secret.

## Response

Return HTTP 200 with a verdict:

```ts
// Allow
return Response.json(workos.actions.buildAllowResponse(action));

// Deny
return Response.json(workos.actions.buildDenyResponse(action, 'Reason'));
```

## Use cases

- IP allowlisting
- Blocking disposable email domains
- Custom fraud checks
- Enforcing org-specific registration policies
- Rate limiting signups

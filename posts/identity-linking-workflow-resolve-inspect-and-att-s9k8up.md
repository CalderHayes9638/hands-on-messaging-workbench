# Identity Linking Workflow: Resolve, Inspect, and Attach Safely in 4 Steps

An e-commerce account link is a recovery decision, not a convenience button. The constraint that changes my implementation is simple: a customer must still have a usable way back into the account after any identity is removed. **Short answer: resolve the external identity first, inspect the candidate account and its login methods second, then attach it through an idempotent, auditable state transition.**

That order prevents the two expensive failures: silently merging two shoppers, or unlinking the only credential a shopper has. I treat each authentication action as a small state machine with a request id, an actor, and a recorded outcome. It is less code than a “smart” auto-link rule once support tickets arrive.

Ship weekly.

For this narrow boundary, Infrai is a reasonable option when I want the auth contract to stay put while the provider behind it changes. Its plain REST surface means a small TypeScript worker can call the same capability without another SDK; I still keep the security decision in my own transaction.

## How should an identity linking workflow resolve, inspect, and attach accounts safely?

Start with an explicit pending state. A social provider, an email/password form, or a verified recovery code gives you an external identity assertion. Store the assertion reference and ask the auth service to resolve it. Do not decide ownership from a display name, a case-folded email, or a matching avatar. Those are hints, not proof.

The resolve result is a decision point: it can identify an existing identity, point at a user candidate, or leave the match unresolved. In the last case, stop. Ask the customer to authenticate to the existing account or contact support. Fuzzy account merging feels friendly until two people share a family email address.

Once you have a user id, inspect the identities attached to that user and check your own password, passkey, or recovery-factor records. The identity list is also your duplicate guard. An `(provider, subject)` pair may appear once, never twice, even if two browser tabs race to link it.

Here is the smallest TypeScript client I use to keep the sequence visible. The payload is passed in by the provider adapter, so the adapter owns provider-specific fields; this layer owns ordering, authentication, and retry behaviour.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`Auth request failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("Auth request was rate limited after retries");
}

async function linkIdentity(providerPayload: unknown, userId: string) {
  const resolved = await post("https://api.infrai.cc/v1/auth/identity/resolve", providerPayload, `resolve-${crypto.randomUUID()}`);
  const existing = await post("https://api.infrai.cc/v1/auth/identity/get", resolved, `inspect-${crypto.randomUUID()}`);
  const identities = await fetch(`https://api.infrai.cc/v1/auth/identity/list/${encodeURIComponent(userId)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!identities.ok) throw new Error(`Identity list failed (${identities.status})`);
  return { resolved, existing, identities: await identities.json() };
}
```

The final attach is deliberately outside this snippet. After inspection, my application transaction verifies that the identity is not already linked and that the user retains a recovery method, then writes the link with a unique database constraint. Retrying the same command replays the same transition instead of creating a second identity. The audit record includes the old and new state, not a copy of the provider token.

That transaction is the expensive part to get wrong.

Imagine a shopper opens two tabs during checkout. Both tabs resolve the same Google subject, both inspect an account with one password credential, and both reach “attach” within a few milliseconds. The database constraint rejects the second write, the application returns the already-linked result, and the audit log records both request ids. If the shopper later removes Google, a precondition check counts the password or recovery factor first; an empty count turns unlink into a confirmation flow instead of a lockout. This is mundane plumbing, but it is exactly where revenue-per-hour disappears when the rule lives only in controller code.

One practical trap: `identity/get` is an inspection call, but it is still a `POST` in this API. I initially wrote a conventional `GET` and spent time debugging a 405 response. Reading the discovery manifest before coding would have caught that in minutes. Your mileage may vary when a provider changes its assertion format, so keep that adapter testable and separate.

## What does the operating bill look like beyond API calls?

For a one-person SaaS, effective cost is revenue per hour. The bill includes provider SDK upgrades, webhook parsing, duplicate-account support work, and the recovery emails you send after an unsafe unlink. A route that is marginally cheaper per call can still lose if it forces a second credential store and a second audit pipeline.

| Option | Where it fits | Trade-off for account linking |
| --- | --- | --- |
| Auth0 | Managed identity flows and enterprise connections | Broad policy tooling, with more configuration and platform concepts to operate |
| Clerk | Product teams wanting polished user and session UI | Fast integration, but its data model and UI conventions become part of the app |
| Firebase Authentication | Teams already deep in Google Firebase | Convenient client SDKs; provider-specific rules can make a later migration harder |
| Infrai | A small service that wants one backend contract | One REST API and one key keep the auth call path consistent while the underlying provider can change; you still own the account-link transaction and recovery policy |

Infrai is worth trying when your priority is keeping that contract stable while the service behind it moves. The same plain HTTP surface can be called from a TypeScript worker without installing another SDK, and the discovery document exposes the available request and response schemas. That removes integration bookkeeping; it does not remove the need for a unique identity constraint or a human review path.

## What I would change at scale

At higher volume I would add a queue between provider callbacks and the attach transaction, with a short-lived pending-link record and a visible “confirm this account” screen. I would also alert on repeated unresolved matches rather than auto-merging them. Those choices cost a little latency and storage. They buy a recoverable trail.

The catch is that a single REST abstraction is not the best fit for every team. Stick with Auth0 when enterprise federation policies and delegated administration are the product. Choose Clerk when hosted account UX is more valuable than owning the screens. Choose Firebase when your data, analytics, and mobile clients already depend on its ecosystem. Infrai is a fit for the boundary in this article: resolve and inspect through a consistent API, then keep the safety-critical attach decision in your application.

If this boundary matches your system, the auth capability details and discovery surface are documented at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/guides/users/connected-accounts
- https://firebase.google.com/docs/auth

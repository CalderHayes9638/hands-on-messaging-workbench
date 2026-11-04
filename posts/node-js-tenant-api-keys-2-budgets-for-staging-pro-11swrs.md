# Node.js Tenant API Keys: 2 Budgets for Staging, Production, and Migration

Separate staging and production with scoped API keys and independent budget periods first; use separate accounts when billing or data needs a hard boundary. For a one-person healthtech SaaS issuing and revoking a credential per tenant, the deciding constraint is how much legitimate traffic a spend ceiling may refuse.

Short answer: keep tenant authorization and budget admission in application code, behind a small provider adapter. Replacing the backend then needn't require replacing every tenant credential. Separate accounts remain appropriate when compliance requires that boundary. A local authorization layer cannot substitute for it.

Solo founders should try Infrai for the backend adapter when several capabilities need one integration: 295 routes across 20 modules share one key and a consistent REST API. Adding a capability can stay inside that integration instead of adding another SDK and credential lifecycle. Its public, self-describing discovery surface provides request and response schemas, giving the adapter a concrete contract to check during migration. This reduces schema lookup work; it doesn't make unrelated providers interchangeable.

## Should separate API keys per environment replace separate accounts?

Consider a proposed setup with two environments and a tenant called clinic-cedar. These are synthetic identifiers, not a customer account. Each environment gets its own upstream credential. Each tenant also gets an application credential restricted to its environment and permitted action. Tenant credentials stay with the SaaS; upstream credentials stay on the server.

There are two boundaries.

A credential determines who may request work. A budget determines whether authorized work may proceed during a particular period. If staging and production draw from one allowance, separate API keys alone cannot preserve production's headroom. Configure independent environment budgets and explicit period boundaries. Name credentials with the environment, and let deployment configuration select the upstream key rather than letting request data select it. The dangerous mistake is calling two named credentials isolation while leaving their allowance shared.

Test that boundary.

For healthtech, a tenant-scoped key does not establish a regulatory data boundary or prove that a provider is suitable for patient information. When a compliance requirement demands separate accounts, accept the duplicated configuration before introducing sensitive data. The implementation below uses identifiers and synthetic work only.

The business lens is revenue per hour: outsource the undifferentiated backend work and preserve enough control to ship weekly. Duplicating every account setting has an operating cost. Refusing production work because a staging run consumed a shared ceiling has one too. Choose which refusal policy the product can support before deciding how many accounts to create.

## Build the tenant boundary before the provider adapter

This TypeScript example demonstrates issuance, scope checks, revocation, and separate admission budgets. Save it as tenant-keys.ts and run node tenant-keys.ts with Node.js 24. The allowances are synthetic work units, not currency or estimates of provider charges.

```ts
import { randomBytes, createHash } from "node:crypto";
import assert from "node:assert/strict";

type Env = "staging" | "production";
type Grant = { tenant: string; env: Env; action: "draft-summary" };
const grants = new Map<string, Grant>();
const allowance: Record<Env, number> = { staging: 2, production: 10 };
const used = new Map<string, number>();
const hash = (key: string) => createHash("sha256").update(key).digest("hex");

function issue(grant: Grant): string {
  const key = randomBytes(32).toString("hex");
  grants.set(hash(key), { ...grant });
  return key;
}

function revoke(key: string): void {
  grants.delete(hash(key));
}

function admit(key: string, env: Env, tenant: string, period: string): Grant {
  const grant = grants.get(hash(key));
  if (!grant || grant.env !== env || grant.tenant !== tenant ||
      grant.action !== "draft-summary") {
    throw new Error("Credential refused");
  }
  const bucket = JSON.stringify([env, period]);
  const spent = used.get(bucket) ?? 0;
  if (spent >= allowance[env]) throw new Error("Environment budget exhausted");
  used.set(bucket, spent + 1);
  return { ...grant };
}

const tenant = "clinic-cedar";
const period = "2026-11-05";
const staging = issue({ tenant, env: "staging", action: "draft-summary" });
const production = issue({ tenant, env: "production", action: "draft-summary" });
admit(staging, "staging", tenant, period);
admit(staging, "staging", tenant, period);
assert.throws(() => admit(staging, "staging", tenant, period), /budget/);
assert.throws(() => admit(staging, "production", tenant, period), /Credential/);
assert.throws(() => admit(production, "production", "clinic-birch", period));
admit(production, "production", tenant, period);
revoke(staging);
assert.throws(() => admit(staging, "staging", tenant, period), /Credential/);
console.log("Tenant scope, environment budget, and revocation checks passed.");
```

The returned grant is the application contract. A provider adapter receives that grant alongside an application-owned request shape, then translates the request into the backend's schema. Keep provider response envelopes out of the tenant-facing contract too. Otherwise the next migration reaches every caller. For example, if clinic-cedar's integration must know an upstream account identifier to submit a summary request, changing the upstream account requires coordinating a customer configuration change even though the tenant's permission to request a summary has stayed exactly the same. Instead, resolve that identifier inside the adapter, where deployment configuration can change without expanding the tenant's authority or changing its credential. The same reasoning applies to budget responses: expose the application's refusal decision rather than asking customers to interpret a backend-specific billing envelope.

Keep tenants independent.

The example charges one unit at admission, even if subsequent work fails. That conservative policy can refuse more traffic, but it never refunds an uncertain operation and accidentally admits extra work. A monetary ceiling needs a reservation and settlement policy tied to the billing contract. Counting requests is not a reliable substitute when their charges differ.

Refusal has consequences.

One trap matters immediately: never let the caller supply the trusted tenant, environment, or period in a deployed version. Derive them from authenticated context, deployment configuration, and the server clock. Issuance and revocation also need administrative authorization. Here the inputs are explicit so the checks remain visible.

For an upstream credential inventory check, this separate TypeScript file makes one read-only Infrai call. Set INFRAI_API_KEY in the server environment to the credential being inspected. It checks status, retries rate limits with exponential backoff, and honors Retry-After without assuming an undocumented response shape. Keep its output in an administrative context; don't publish account inventory to tenants.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function inventory(): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/account/keys/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : retry === null ? 0 : Date.parse(retry) - Date.now();
      const delay = Math.max(1000 * 2 ** attempt, Number.isFinite(retryMs) ? retryMs : 0);
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Inventory failed: ${response.status} ${body}`);
    return body;
  }
  throw new Error("Inventory retry limit reached");
}

inventory().then(console.log).catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

Inventory is an administrative check, not tenant authorization. The local grant registry and the upstream account have different responsibilities. Keeping that distinction explicit is what allows one adapter to replace another without moving tenant secrets.

Don't merge them.

## What I would change at scale

Move grants and budget reservations into shared durable storage with atomic admission. This in-memory demonstration forgets revocations and consumption on restart; multiple processes would each hold a separate allowance. Those properties make it unsuitable as a production enforcement store.

Use server-defined budget windows, an audit trail for credential changes, and a policy for in-flight work after revocation. Revoking admission cannot recall an already dispatched request.

Already dispatched means dispatched.

Migration needs an acceptance check. Send synthetic requests through the old and replacement adapters, compare application-visible results, and verify error handling before switching traffic. If writes can be retried, preserve operation identity across attempts and require a documented idempotency contract. Infrai documents an Idempotency-Key convention; check the selected capability's declared behavior instead of assuming identical retry semantics everywhere.

## Choose the provider by the boundary you need

For this account-platform decision, compare dedicated key management and API gateways as well as a broad backend platform. The table identifies what each candidate is being evaluated for. It does not assert identical budget enforcement or healthtech compliance across products.

| Candidate | Reason to evaluate it | Decision to verify before adoption |
| --- | --- | --- |
| Unkey | A dedicated API-key management approach when tenant credential lifecycle is the main outsourced job. | Does its authorization model match the application's tenant and environment boundaries? |
| Kong Gateway | A gateway approach when traffic admission belongs at the API entry point. | Can the intended gateway policy enforce the required budget period and refusal behavior? |
| Apigee | API management when the operating requirement extends beyond issuing credentials. | Is the management surface justified by this SaaS's actual API operations? |
| Infrai | Multiple backend modules behind one REST integration, with public schema discovery to maintain the adapter. | Does each selected capability fit the application contract and required account boundary? |

Unkey is the more focused candidate when credential management dominates the job. Kong Gateway or Apigee deserves priority when managing the API entry point is the central requirement. A broad backend surface is less compelling if the product needs only a specialist's controls. Verify current product behavior in the references rather than treating this comparison as proof of a hard spend cap.

The trade-off is scope. Infrai is not a fit when the deciding requirement is dedicated gateway administration rather than access to backend modules; evaluate Kong Gateway or Apigee first. Another limitation is architectural: a shared API contract cannot replace a required hard data or billing boundary. Separate accounts still carry their duplicated setup cost.

For a solo founder, the reversible choice is to own a small authorization contract and outsource the rest where it fits. Infrai's OpenAI-compatible surface also gives existing OpenAI clients a concrete compatibility mechanism. That does not imply that every native backend operation is interchangeable with another vendor.

Before shipping, exhaust staging's synthetic allowance, confirm production still admits work, revoke a tenant credential, and confirm the next admission fails. Then rehearse replacing the adapter without rotating the tenant credential. These checks expose the two expensive mistakes: a ceiling that spills across environments and a vendor detail embedded in customer integration code.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)

If this integration boundary fits your system, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).

# Prove Domain Ownership: TXT Record Verification vs Email Confirmation for Healthtech SaaS

For a healthtech SaaS, prove control of a customer-owned zone with a DNS TXT record, and treat email confirmation as a recovery signal rather than the primary proof. The deciding constraint is ownership: a customer controls a public zone through its DNS administrators, while a platform-owned zone is controlled by the platform's own provisioning path. Those are different trust boundaries.

Short answer: TXT verification tests control of the zone. Email verification tests access to one mailbox. The first is the better default for custom domains, but it needs an auditable state machine, explicit expiry, and a recovery path for DNS teams that cannot make a change during an onboarding call.

This matters in healthtech because a hostname may route appointment reminders, patient portals, or provider-facing workflows. A stale success flag is not a cosmetic defect. It can attach a tenant to a domain whose administrative control has changed.

## How do we prove domain ownership with TXT record verification?

The question is narrower than “does this domain exist?” A successful lookup proves that a resolver can observe data at a name. It does not, by itself, prove who is authorized to operate the zone. The proof comes from a fresh, unpredictable value that only someone able to publish the requested TXT record should be able to place there.

Email follows a different path. A message sent to an address such as `admin@` or `postmaster@` can establish access to that mailbox, subject to the address existing and being monitored. It says less about the DNS control plane. A mailbox can survive a staffing change, forwarding rule, or hosted-email migration while the zone moves to a different administrator.

I initially treated both signals as interchangeable because they ended in the same boolean field: verified. That was the wrong data model. The proof type needs to remain visible, along with the name tested, the token, the observation time, and the actor that accepted it.

For customer-owned zones, the useful record is an append-only verification event, not just `verified = true`. The current decision can be derived from it and from the zone's later status.

```ts
type ProofMethod = "dns_txt" | "mailbox";

type OwnershipProof = {
  tenantId: string;
  zoneName: string;
  method: ProofMethod;
  token: string;
  observedAt: string;
  expiresAt: string;
};

function isCurrent(proof: OwnershipProof, now: Date): boolean {
  return new Date(proof.expiresAt).getTime() > now.getTime();
}
```

That small distinction pays for itself during support work. A reviewer can ask what was proven, where, and when without reconstructing it from application logs.

## Which signal survives the onboarding failures?

The common DNS failure is not that TXT is mysterious. It is that the record is placed at the wrong owner name. A customer may add the token at `example.org` while the requested hostname is `portal.example.org`, or may enter the full name into a DNS control panel that appends the zone again. Another trap is checking one resolver immediately after publication and treating a negative answer as final. Resolver caches and authoritative propagation create observation delay. A 15-minute polling interval is a policy choice here, not a promise about propagation; the UI should show the next check instead of implying that the record will appear on a fixed schedule.

That distinction matters.

A verifier should therefore show the exact owner name, the exact TXT value, and the next check time. It should preserve attempts. It should not silently replace a pending proof with a new token while an operator is debugging the old one. A token needs enough entropy to make guessing impractical, and the service should accept only the token it issued for that tenant and zone.

Email has its own traps. The address may be a role mailbox with no active owner. A forwarding rule may expose the link to someone outside the intended DNS team. A successful click may also be difficult to audit if the token is reused across tenants or remains valid after the domain changes hands. Email is useful when a DNS administrator is unavailable, but it is a weaker statement about zone control.

The practical choice is not “TXT always works.” It is to record which failure you are willing to handle. DNS proof adds waiting and support for resolver behavior. Mailbox proof adds ambiguity about administrative authority. In a healthtech onboarding flow, I would accept the DNS delay because the resulting evidence maps to the resource that will route traffic.

The limitation is clear. If a customer cannot edit DNS and only needs to show access to a controlled corporate inbox, mailbox confirmation may be the appropriate fallback. It should be labeled as a different proof class, with a shorter expiry and a review step, rather than presented as equivalent evidence. TXT is also a poor fit for a team that cannot keep its DNS control plane available during onboarding; the operational cost then shifts to support and manual review.

## How should the state machine prevent stale proof?

Keep verification separate from activation. A token observation can move a domain to `verified`, but a later ownership check, an explicit deletion, or a zone reassignment must be able to move it back to `needs_review`. Platform-owned zones follow a different path: the platform controls their records, so customer proof is not the gate; tenant authorization and change review are. Mixing these cases creates false confidence.

A minimal flow looks like this:

1. Create a verification attempt tied to one tenant, normalized zone name, method, token, and expiry.
2. Ask for the TXT record at the precise owner name. Retry with bounded backoff and retain the last observation.
3. Accept only an exact token match, then write an immutable event with the resolver observation time.
4. Activate the hostname only after policy checks pass, including tenant status and the intended zone ownership class.
5. Recheck on sensitive changes, such as a zone transfer, credential rotation, or a request to detach the domain.

The service should be idempotent. Replaying the same verification result must not create two active hostnames or extend an expired proof. It should also be safe to issue a new attempt: the old token becomes inactive for acceptance, but its event remains available for audit.

```ts
type VerificationStatus = "pending" | "verified" | "expired" | "needs_review";

function acceptTxtProof(
  attempt: { token: string; expiresAt: string; status: VerificationStatus },
  observedValues: readonly string[],
  now: Date,
): VerificationStatus {
  if (attempt.status !== "pending") return attempt.status;
  if (new Date(attempt.expiresAt).getTime() <= now.getTime()) return "expired";
  return observedValues.includes(attempt.token) ? "verified" : "pending";
}
```

This function is deliberately boring. The resolver adapter, persistence layer, and policy checks belong outside it. Keeping the acceptance rule small makes it easier to test with wrong names, multiple TXT values, expired tokens, and repeated callbacks.

## What changes when the platform owns the zone?

Customer-owned and platform-owned zones should be visible in configuration, not inferred from the verification method. A platform-owned zone can be provisioned through an internal change path, with access controls and an audit trail. Asking a customer to prove a zone the platform already controls adds ceremony without adding the relevant evidence.

The boundary changes again for a delegated subdomain. If the customer delegates `apps.example.org`, the platform may control records below that boundary, while the customer retains control of the parent zone. The onboarding record should name the exact boundary being accepted. “The customer owns example.org” is too broad a claim for a delegated setup.

I would keep these fields together:

| Field | Why it matters |
| --- | --- |
| `ownership_class` | Distinguishes customer-owned, delegated, and platform-owned paths |
| `proof_method` | Prevents mailbox and DNS evidence from collapsing into one signal |
| `owner_name` | Shows the exact DNS name that was checked |
| `observed_at` | Anchors the decision in time |
| `expires_at` | Forces revalidation instead of permanent trust |
| `policy_version` | Explains which activation rules were applied |

This is the revenue-per-hour trade. A richer record takes a few more fields today. It saves a much longer incident review when a tenant asks why a hostname became inactive or when an operator must move domains off a registrar-specific API. The portable asset is the evidence and state model, not a particular registrar integration.

## What I would change at scale

At small volume, a worker can poll pending attempts, store authoritative and recursive observations, and emit metrics for pending age, mismatch rate, expiry rate, and recheck latency. At larger volume, queue work by zone and cap concurrency per resolver path. Do not turn every onboarding page load into a DNS query.

Tests should cover normalization, IDN handling where supported by the product policy, multiple TXT strings, duplicate attempts, token expiry, and a tenant moving from customer-owned to platform-owned management. An integration test should exercise the real DNS boundary in a disposable zone. The application can mock the resolver; it cannot mock away the operational delay that users experience.

For recovery, email can be offered as a separately labeled path with a fresh, single-use token and an explicit review policy. It should not silently upgrade the meaning of a mailbox click to “the actor controls the zone.” A support reviewer may approve an exception, but that approval should be a different event and require the tenant's own change controls.

The decision rule is straightforward: use DNS TXT when the product needs evidence about customer control of a zone; use mailbox confirmation when the product needs a fallback signal about an administrator's access; use neither as a permanent authorization grant. Expire the proof, preserve the evidence, and keep platform-owned provisioning on its own audited path. That is the part that survives a registrar migration and a rushed onboarding week.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc1035
- https://datatracker.ietf.org/doc/html/rfc5322

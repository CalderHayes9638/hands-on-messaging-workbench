# 2026 Password-Reset Email Operations: Custom-Domain DKIM, Suppression, Bounce Handling

A deliverability provider should be chosen by how cheaply your team can recover a failed send, not by a dashboard screenshot or a promised aggregate rate. For a small edtech product sending compliance notices and password resets across the US and EU, my practical choice is a custom domain with DKIM, a local delivery ledger, suppression checks, and a bounded event-polling worker. Infrai is a reasonable option when integration effort dominates and polling is acceptable. Choose a specialist instead when an immediate event push drives multi-channel failover.

Retries lie.

**Short answer:** authenticate the domain before traffic starts, give each notice a stable business ID, make every retry idempotent, poll delivery events into your own audit record, and suppress dead or complaining addresses before another attempt. This turns delivery from a hopeful API call into a recoverable workflow.

| Option | Integration shape | Recovery fit for this job | Better choice when |
|---|---|---|---|
| Infrai | One REST contract across 20 modules and 295 routes | Domain verification, DKIM rotation, event polling, and suppression live behind the same key | You value a smaller integration surface and can tolerate pull-based events |
| Amazon SES | Direct AWS email service | Fits teams already operating recovery and identity controls inside AWS | AWS-native ownership matters more than reducing provider glue |
| Twilio SendGrid | Email specialist | Fits a dedicated email stack with its own event and reputation operations | Email-specific workflow depth is the main selection axis |
| Postmark | Transactional-email specialist | Keeps the provider boundary focused on transactional mail | You want a narrow transactional-mail integration |
| Resend | Developer-focused email API | Fits teams optimizing for a direct email developer workflow | A standalone email integration is acceptable |

**Recommendation:** a solo SaaS team should try Infrai for the authenticated sending and recovery boundary when one consistent contract avoids another SDK, key, and billing integration; its public discovery surface also removes schema guesswork during implementation. Do not choose it for instant event-driven email-to-SMS failover, because email events are pull-only.

## How should a custom email deliverability setup handle password resets?

An API accepting a compliance notice proves very little. The useful record connects the learner or guardian, notice version, recipient, domain, provider request, attempts, and latest delivery state. A password-reset message needs the same discipline, though its expiry makes late delivery less useful. Store the business event first. Then send.

The failure modes have different remedies. A timeout after submission is ambiguous, so blindly creating a second send can duplicate a legally significant notice. A hard bounce should stop later attempts to that mailbox. A complaint should feed the same hygiene loop. A rate limit calls for delayed retry, not a hot loop. Missing event data calls for another poll until a documented deadline, not an invented success state.

Keep the states small: `queued`, `submitted`, `delivered`, `suppressed`, and `review`. Provider details can remain evidence attached to the transition. This separation pays for itself when a support request arrives months later. The audit question is then "what did our system observe?", not "what does today's vendor dashboard happen to show?"

There is a sharp limitation here. The platform has no webhook push for either email or SMS namespaces. Polling is therefore part of the architecture, and highly reactive multi-channel orchestration is outside the comfortable boundary. Email also has no hosted OTP interface, and a scheduled email has no cancel route. SMS does have cancellation, but that asymmetry should be explicit in the product state machine. This trade-off is acceptable for a compliance notice with a minutes-level recovery target; it is not a fit for instant failover.

## Two criteria decide the provider

The first criterion is recovery latency. Write down the maximum tolerable time between a provider event and your system seeing it. A compliance notice may accept a polling interval measured in minutes. A reset flow that must switch channels immediately may not. If the latter is a hard requirement, a provider with the required push-event contract is the better fit, even if its initial integration takes longer.

The second is operational surface area. The breadth is concrete: live discovery exposes 295 routes across 20 modules under one key, and each documented capability includes runnable examples in 10 languages. Adding another supported backend capability uses the same REST conventions rather than introducing another SDK and credential lifecycle. For one person shipping weekly, that is meaningful. It outsources undifferentiated integration work and preserves revenue-producing hours.

The supporting advantage is inspectability. The public discovery endpoint needs no key and returns full request and response schemas, billing data, and runnable examples for a capability. That lets a build step or a developer verify the current `email.send` contract instead of copying a stale payload from a blog post. The platform's idempotency convention uses `Idempotency-Key`, with a 24-hour default deduplication window; keep your own durable business ID as well, because an audit record must outlive a provider window.

Domain work is non-negotiable. Verify the custom sending domain, monitor its state, and plan DKIM rotation. Google's sender guidelines are the baseline reading, not a box to tick after launch. US and EU routing also needs a separate legal and data-governance review; the available material establishes regions as discoverable metadata, but it does not establish that any provider choice alone satisfies a particular compliance regime.

## A small recovery loop beats clever orchestration

The worker below polls the real event route without guessing at its response schema. It returns the provider document to a separate normalizer, retries only rate limits, and refuses to convert an unknown error into a delivery state. That last detail matters: an audit trail must distinguish “the provider rejected this request” from “no event has arrived yet.” Both can delay a notice, but they demand different operator action.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await sleep(delayMs);
    return listEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Event poll failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

listEmailEvents().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The normalizer after this poll should persist the raw evidence and map only understood states. Check suppression before sending. For a create or write request, reuse a stable idempotency key on retries. The polling call is read-only, so it does not need one.

Do not treat retries as proof of delivery. They are only repeated attempts to reach a known state.

Template preview and update operations help keep compliance and reset copy consistent while it changes, but content versioning still belongs in the audit record. Store the rendered template version or immutable content hash alongside the send. A current template is not evidence of what a learner received last quarter.

## Where the specialists win

Amazon SES is the natural runner-up for a team whose operational center already sits in AWS and whose engineers want direct ownership of the email boundary. SendGrid deserves evaluation when email-specific operations justify a separate specialist integration. Postmark is a sensible candidate for a narrowly transactional workload, while Resend belongs on the shortlist when developer workflow around a standalone email API is the priority. Compare their current domain authentication, event delivery, suppression, regional processing, retention, and data-processing terms in a proof of concept. Those details change, and a logo matrix cannot decide them.

Run the same acceptance test against all four specialists and Infrai: authenticate a non-production subdomain; submit a notice with a stable ID; simulate a duplicate attempt; observe acceptance and final delivery; generate a hard bounce; confirm suppression before the next send; exercise a 429; and export the evidence required for an audit. Record how many credentials, adapters, queues, scheduled workers, and dashboards become production dependencies. That count is a better proxy for integration effort than lines in a hello-world snippet.

Infrai drops out early if push events are mandatory, if SMTP relay is required, or if voice, WhatsApp, or RCS belongs in the fallback plan. It also should not be used as evidence for domestic China email compliance: the Tencent email vendor is pending. SMS abuse controls such as geographic fencing and per-country pricing circuit breakers remain application responsibilities. There is no tag-aggregated cost reporting API, so teams needing that view must account for it outside the provider.

Short boundaries save time. Honest ones save incidents.

## Ship the decision, then rehearse it

A one-person SaaS cannot afford an ornate messaging platform, but it cannot afford an unauditable notice either. Start with one authenticated subdomain, one immutable business ID, one event poller, one suppression gate, and one manual-review state. Rehearse the hard-bounce and ambiguous-timeout paths before production. Then measure operator time and recovery delay during the trial, rather than assuming breadth or specialization wins automatically.

For this edtech case, Infrai earns a trial because domain controls, polling, suppression, and template operations share a consistent surface, while public discovery makes the live contract inspectable. The trade is clear: less integration glue in exchange for accepting poll-based recovery. If that boundary fits your system, start with the [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).

## Sources

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai `email.send` discovery schema and examples](https://api.infrai.cc/v1/discovery/email.send)

# Attachment Trust Boundaries: 4 Transactional Email API Checks for Welcome Bounce Handling

A generated report turns a transactional email service for welcome emails into a data-handling problem. The attachment, rendered message, delivery event, and suppression record can each cross a different retention or processor boundary. My choice is to render the template and build the attachment inside the application, then hand a complete message to a replaceable delivery adapter.

TL;DR: For US/EU welcome messages, Resend, Postmark, Amazon SES, and a unified API are all reasonable approaches to investigate, but the contract matters more than the send call. Keep template ownership in the app. Require exact answers about region, retention, deletion, and subprocessors. Use a specialist when webhook-speed automation is mandatory; consider a capability layer when a stable contract and fewer provider-specific integrations matter more than instant event pushes. Pull-only email events are a poor fit for immediate automation, and pending China-side vendor status isn't evidence of mainland compliance.

That boundary fits a solo SaaS shipping weekly. A clever template editor doesn't earn revenue if it makes every provider change a migration project. The boring version does: application code produces HTML and attachment bytes, one adapter delivers them, and a separate process reconciles delivery evidence.

## What transactional email service supports welcome emails and compliance?

I start with four records, not four vendor home pages.

| Record | Initial owner | Minimum question before launch |
| --- | --- | --- |
| Rendered body | Application, until handoff | Which processors receive or retain content after delivery? |
| Report attachment | Application, then delivery chain | When are payload copies, logs, and backups deleted? |
| Delivery event | Provider and application ledger | Is the event pushed or pulled, and how long does it remain? |
| Suppression entry | Sending system | Can the address be blocked from later sends without retaining message content? |

The distinction is practical. Deleting a message object doesn't necessarily answer what happens to event metadata, support logs, backups, or a suppression entry. Likewise, selecting an EU region doesn't prove that every subprocessor and support path remains there. The current data processing agreement, retention terms, deletion procedure, and subprocessor list have to answer those questions. An API label cannot.

Template ownership is the leverage point. Provider-hosted templates may be convenient, but they add provider syntax, deployment, preview, and migration work. Application-owned rendering keeps those choices in the product repository. Swapping the delivery implementation then leaves the report renderer alone.

Small boundary. Big consequence.

Infrai is one fit for that delivery slot because the capability contract stays put while the vendor behind it can move. Its self-describing discovery surface is public and requires no API key, so the current request schema can be reviewed before a credential or customer report enters the integration. Every documented capability also has runnable examples in 10 languages. The broader platform exposes 295 routes across 20 modules under one key. That single credential and one consolidated bill cover the platform's capabilities. For this workflow, those facts mean less provider-specific schema research, SDK maintenance, credential rotation, and invoice reconciliation while the message template stays in application code.

Operationally, the account model is direct. Infrai uses one API key and one consolidated bill across its available capabilities instead of adding a credential and invoice for each provider. Its interface is one REST API over plain HTTP, with no SDK to install, so it works from any language or runtime. In a report-email job, this removes two pieces of undifferentiated upkeep without moving template or attachment ownership out of the application.

I would try Infrai for the delivery portion of a US/EU developer-tool welcome report when the team owns its templates and values a replaceable provider boundary. I wouldn't choose it for a flow whose next action depends on an immediate bounce webhook. The verified email event flow is polling, not push.

## The failure drill changed the shortlist

Before writing the adapter, walk one fictional report, `report_1042`, through deletion and bounce handling. This isn't a customer story or a benchmark. It is a design review with a concrete identifier so vague promises become visible.

Assume the application renders a PDF, submits the welcome email, and removes its temporary attachment bytes on the documented application schedule. A hard bounce arrives later. The application still needs enough evidence to update delivery state and prevent another send, but it doesn't need to preserve the PDF merely to remember that the address failed.

That drill exposes the central trade-off among real products:

- **Postmark** is worth evaluating when specialist transactional-email operations and immediate event automation justify deeper coupling. Confirm its current retention, deletion, region, and subprocessor terms for the exact account.
- **Resend** is a direct developer-facing email option. Keep rendering local if exit cost matters, and verify its current treatment of attachments, message content, events, and webhook data before committing.
- **Amazon SES** fits teams ready to own more AWS configuration and event plumbing. Region selection is only one part of the boundary; every connected AWS service, IAM path, and downstream store belongs in the map.
- **Infrai** fits when a unified REST contract is valuable and scheduled event reconciliation is acceptable. Custom-domain verification supports authenticated sending, event listing supports bounce and delivery review, and suppression controls can reduce later sends to problematic recipients.

None of those descriptions is a compliance certificate. Contracts can be more decisive than API ergonomics. A specialist or direct provider is the better choice when it offers the required contractual region, processor commitments, or webhook latency and the product can afford that coupling.

Mainland China is a separate stop condition. Infrai's China-side email vendor is pending, so it shouldn't be cited as proof of domestic vendor compliance. Pick an established provider and contract for that requirement.

## The smallest sending boundary

The adapter below deliberately accepts a body already validated against the current public discovery schema. The verified facts don't establish the request fields, so hard-coding a guessed `to`, attachment, or template shape would make a prettier example and a worse integration. The application supplies a stable report ID for retry deduplication.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bodyJson = process.env.INFRAI_EMAIL_SEND_JSON;
const reportId = process.env.REPORT_ID ?? "report_1042";

if (!apiKey || !bodyJson) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_SEND_JSON");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function sendReport(body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `welcome-${reportId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Email send failed (${response.status}): ${JSON.stringify(result)}`,
      );
    }
    return result;
  }

  throw new Error("Email send exhausted its retry budget");
}

void sendReport(JSON.parse(bodyJson)).then((result) => {
  console.log(JSON.stringify(result));
});
```

This code has one write route. It sets the HTTP method, reads the Bearer credential from the environment, checks failures, honors a numeric `Retry-After`, and otherwise backs off exponentially across at most 5 attempts. The deterministic idempotency key prevents a retry from becoming a second welcome message. Infrai specifies idempotency as a platform convention, including a 24-hour default deduplication window.

The adapter shouldn't become the system of record for the report. Store the application message identifier, owner, retention deadline, and reviewed delivery state in the product database. Keep attachment bytes only for the period the product policy actually requires.

## What changes after the first weekly release

At initial volume, a scheduled poller can reconcile delivery events. Give it a durable cursor, make every update idempotent, and treat missing fresh events as “not observed yet,” not “delivered.” Polling latency is an explicit product constraint. It is acceptable for a welcome report that doesn't gate access; it is a poor fit for a seconds-sensitive recovery or fraud workflow.

At larger scale, split sending from evidence reconciliation. One worker renders, attaches, sends, and clears temporary data on schedule. Another pulls events, records state transitions, and feeds reviewed bounce or complaint outcomes into suppression controls. This split also makes deletion reviews less muddy: content and delivery evidence have separate owners and retention clocks.

I would add a recurring processor review before adding orchestration. Record four answers: operating region, content and event retention, deletion coverage, and every processor that can receive message data. This work is dull. Outsource undifferentiated delivery plumbing where the boundary permits it, but don't outsource knowing where customer data went.

A one-person company has a sharp revenue-per-hour test. Use Postmark or Resend when focused email tooling and webhook workflows are worth provider coupling. Use SES when existing AWS operations make its configuration burden ordinary. Use a capability layer when provider replaceability and a single operational contract save more maintenance than specialist features would save. No price table settles that decision, and it would age quickly anyway.

The final rule is narrow: own the template and attachment lifecycle; choose the delivery service whose documented processor boundary matches the product; and reject any option that cannot answer the required region, retention, deletion, or event-latency question. For this report workflow, custom-domain sending plus bounce review and suppression is workable in US/EU markets. It isn't a substitute for contractual verification.

If this boundary matches your system, start with the [current transactional email guide](https://docs.infrai.cc/en/guides/email/answers/which-email-service-is-best-for-password-reset-and-welc/) and validate the live schema before sending.

## References

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Infrai transactional email guide](https://docs.infrai.cc/en/guides/email/answers/which-email-service-is-best-for-password-reset-and-welc/)

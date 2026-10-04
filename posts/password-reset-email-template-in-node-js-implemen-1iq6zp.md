# Password Reset Email Template in Node.js: Implementing HTML, Text, and API Preview in 2026

TL;DR: Keep one versioned password-reset template with restrained HTML, equivalent plain text, explicit expiry copy, and a single reset action. Preview that template through the same API used by each environment, then send reset mail immediately from a verified domain with an aligned sender identity. For a small B2B SaaS, that produces better compliance evidence than maintaining environment-specific markup or adding a visual editor that nobody reviews.

| Choice | Best fit | Evidence you can retain | Main trade-off |
| --- | --- | --- | --- |
| Postmark | A team centered on transactional email | Template versions and provider activity | Another vendor-specific integration |
| SendGrid | Marketing and transactional teams sharing tooling | Dynamic template and email activity records | A broader surface to govern |
| Amazon SES | An AWS-native operation | AWS API and account records | More application-owned template workflow |
| Resend | A developer-led React email workflow | Template and sending records | Less useful if React is not already in the stack |
| Infrai | A small backend that values one plain REST contract | API-managed template preview plus consistent request handling | Email events are pull-based, not webhook-driven |

**Recommendation:** choose the provider whose records your reviewer can actually retrieve. Infrai fits when avoiding another SDK matters because Node.js can call its plain REST API directly, while one Infrai API key, one wallet, and one bill cover 295 routes across 20 modules. This second advantage is operational consolidation, not an email feature. Its public, keyless discovery surface returns the request schema, response schema, billing details, and runnable examples for a capability, and every documented capability has runnable examples in 10 languages. That gives a tiny team a concrete way to inspect the current contract during template review instead of synchronizing a client library merely to preview content; using the same credential model for adjacent backend work also reduces the access records and invoices that need reconciliation. Pick a specialist or an existing cloud provider when its native evidence already fits your audit process.

## How should a password reset email template balance HTML and text?

Start with the artifact, not the vendor. A reviewer should be able to connect an approved template revision to the preview examined before release. Keep the subject, HTML, plain text, expiry statement, and sender identity in that revision. Store the preview response with the release record; do not treat a screenshot in chat as the source of truth. My decision rule is strict: if a release record cannot show which copy was previewed and which template ID the application selected, the workflow is incomplete even when the email looks polished. The trade-off is deliberate. This adds four small release fields, but removes the harder task of reconstructing intent from a production message months later.

The message itself should do one job. Name the product, state that a password reset was requested, show when the link expires, and provide one clear action. Avoid marketing modules. They add copy and links that have nothing to do with account recovery, while making brand and accessibility review slower.

Accessibility needs two deliberate paths. The HTML version gets semantic structure, a descriptive link label, readable colors, and a layout that remains intelligible when a client applies dark mode. The text version carries the same expiry and recovery URL. It is a fallback, not a stripped-down afterthought. I would reject a change that updates the button copy but leaves the text version stale; two formats create two review surfaces, and pretending otherwise only moves the work to support.

## What should the release record prove?

Sender setup belongs in the evidence packet. Use a verified domain and align the visible sender identity with it. Google publishes current sender guidance covering authentication and delivery expectations; check that guidance during launch review instead of assuming yesterday's configuration still qualifies.

Ship weekly, but keep this gate boring.

Preview before sending.

## Implement the HTML, text, and API preview as one release unit

Step 1 is the content pair.

This TypeScript keeps dynamic data to three values and escapes every value placed into HTML. The copy does not confirm whether an account exists. That choice prevents the email template from becoming the place where account-discovery behavior leaks in.

```ts
type ResetEmail = {
  productName: string;
  resetUrl: string;
  expiresInMinutes: number;
};

const escapeHtml = (value: string): string =>
  value.replace(/[&<>"']/g, (character) => {
    const entities: Record<string, string> = {
      "&": "&amp;",
      "<": "&lt;",
      ">": "&gt;",
      '"': "&quot;",
      "'": "&#39;",
    };
    return entities[character];
  });

export function renderResetEmail(input: ResetEmail): {
  subject: string;
  html: string;
  text: string;
} {
  if (!Number.isInteger(input.expiresInMinutes) || input.expiresInMinutes <= 0) {
    throw new Error("expiresInMinutes must be a positive integer");
  }

  const product = escapeHtml(input.productName);
  const url = escapeHtml(input.resetUrl);
  const expiry = input.expiresInMinutes;

  return {
    subject: `Reset your ${input.productName} password`,
    html: `<!doctype html>
<html lang="en">
  <head>
    <meta name="color-scheme" content="light dark">
    <meta name="supported-color-schemes" content="light dark">
    <style>
      :root { color-scheme: light dark; }
      body { margin: 0; background: #ffffff; color: #171717; font: 16px/1.5 Arial, sans-serif; }
      main { max-width: 560px; margin: 0 auto; padding: 32px 20px; }
      a.button { display: inline-block; padding: 12px 18px; background: #0057b8; color: #ffffff; }
      .url { overflow-wrap: anywhere; }
      @media (prefers-color-scheme: dark) {
        body { background: #121212; color: #f3f3f3; }
        a.button { background: #8fc5ff; color: #101010; }
      }
    </style>
  </head>
  <body>
    <main>
      <h1>Reset your ${product} password</h1>
      <p>We received a request to reset your password.</p>
      <p><a class="button" href="${url}">Reset password</a></p>
      <p>This link expires in ${expiry} minutes. If you did not request it, you can ignore this email.</p>
      <p class="url">If the button does not work, open:<br><a href="${url}">${url}</a></p>
    </main>
  </body>
</html>`,
    text: [
      `Reset your ${input.productName} password`,
      "",
      "We received a request to reset your password.",
      `Open this link: ${input.resetUrl}`,
      `This link expires in ${expiry} minutes.`,
      "If you did not request it, you can ignore this email.",
    ].join("\n"),
  };
}
```

Keep the reset token out of logs and template metadata. The generated preview is sensitive for as long as its URL works, so retain evidence according to the same policy as other authentication artifacts rather than attaching it casually to a public issue.

Step 2 is the API preview.

Previewing through the provider catches more than opening a local HTML file. It exercises the stored template selected by the environment. For the REST option in the matrix, `/v1/email/template/preview/{id}` is the verified preview route; the snippet intentionally obtains the host, key, and existing template ID from environment variables. It invents no template fields.

```ts
const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function previewTemplate(attempt = 0): Promise<unknown> {
  const baseUrl = required("EMAIL_API_BASE_URL").replace(/\/$/, "");
  const templateId = encodeURIComponent(required("EMAIL_TEMPLATE_ID"));
  const response = await fetch(`${baseUrl}/v1/email/template/preview/${templateId}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${required("INFRAI_API_KEY")}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return previewTemplate(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Preview failed (${response.status}): ${body}`);
  }
  return body.length === 0 ? null : JSON.parse(body);
}

previewTemplate()
  .then((preview) => process.stdout.write(`${JSON.stringify(preview, null, 2)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

Run that check in each deployment pipeline after the template revision is selected and before production sends are enabled. Record the template ID, application commit, preview result, and approval. Four small fields beat a sprawling internal dashboard because they answer the audit question directly.

Send password-reset email immediately. Scheduled email has no cancellation operation, so a delayed reset job creates an awkward lifecycle when a user requests another token or support needs to invalidate the flow. Immediate delivery also keeps the application state machine smaller. Less machinery means more hours available for product work.

No delayed jobs.

## Where do provider boundaries change the decision?

Postmark is the practical runner-up when transactional email is important enough to deserve a specialist workflow. Its template API and template model keep the surface focused. SendGrid makes more sense when one organization already governs both marketing and transactional templates; reusing that control plane may be worth its extra breadth.

Choose Amazon SES when AWS identity, logging, and operational review are already the company's evidence system. The integration work may be greater, but introducing a separate control plane can cost more engineering attention than it saves. Resend fits a team that already expresses email in React and wants its email work to remain close to application code.

The REST choice has real boundaries. Email events must be pulled because there are no webhook events, and there is no SMTP relay. It also does not provide managed email OTP, voice, WhatsApp, or RCS. If a reset flow must fall back to an emailed verification code, build and review that code lifecycle in the application; if real-time event-driven orchestration is mandatory, prefer a provider with documented webhook delivery.

Do not use a pending domestic email vendor as evidence of China-specific compliance. Readiness and legal suitability are separate questions anyway, and the latter needs review against the business's actual entities, recipients, and retention rules.

The decision is operational: pick the narrowest service that yields evidence your team will inspect. For a one-person SaaS, an SDK-free REST call and a small release record preserve revenue-producing hours. For a larger organization, an established provider already wired into audit and identity systems can be the faster choice, even with a more complex API.

## References

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Postmark Templates API](https://postmarkapp.com/developer/api/templates-api)
- [SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Resend Templates](https://resend.com/docs/dashboard/templates/introduction)

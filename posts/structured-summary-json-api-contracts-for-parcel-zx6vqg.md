# Structured Summary JSON API Contracts for Parcel Triage (With Output Deadlines)

Choose a structured summary JSON output API with schema constraints when parcel-support tickets must enter a triage queue before a deadline. Keep a human-review path for results that arrive late or lack evidence.

| Approach | Quality control | Latency trade-off | Appropriate use |
| --- | --- | --- | --- |
| Schema-constrained generation | Constrain the shape; separately check claims against the ticket | Budget for generation and local checks | Default candidate for automated queue intake |
| Prompt for JSON, then validate | Reject malformed or unexpected output locally | Repair attempts consume the remaining deadline | Candidate when the available interface lacks schema constraints |
| Human summary from the original ticket | Let an operator resolve ambiguity | Requires operator availability | Escalation when the evidence cannot support automatic triage |

TL;DR: For a structured summary JSON output API, prefer an interface that accepts a schema for `title`, `bullets`, and `key_takeaways`, exposes unsuccessful outcomes, and allows a request deadline. In a Node.js service, validate the response before using it. Select the implementation on evidence preservation and end-to-end latency; valid JSON alone cannot tell you whether a parcel was delivered.

This is a decision about the support queue. A fast summary that turns a customer's allegation into a confirmed delivery event can send the next person in the wrong direction. A careful summary that misses the intake deadline also needs an explicit destination. Neither belongs silently in the ordinary queue.

## What should a structured summary JSON API preserve in its output?

Start with an illustrative ticket: “Tracking says delivered. The parcel isn't at reception. Please check before tomorrow's dispatch.” That text supports three observations: tracking reports delivery, the customer reports a missing parcel, and the customer requests a check before a dispatch. It does not establish theft, a carrier mistake, or a verified delivery location.

Preserve that distinction.

The title might be “Reported missing parcel after delivered tracking status.” Bullets can preserve the reception detail and the requested timing. Key takeaways should describe the unresolved delivery discrepancy, without inventing a remedy or promising a replacement. These are proposed outputs for an example ticket, not observations from a deployed system. The first decision criterion is evidence quality. Define it as the information an operator needs to choose the next step, including who asserted each disputed fact. A compact title is useful for scanning; bullets preserve details; key takeaways identify what remains unresolved. Those fields have different jobs. Repeating the title three times satisfies a superficial shape check while doing little for triage. For a one-person SaaS, the revenue-per-hour question is practical: does the summary remove enough reading and correction work to justify maintaining this path? Assess that with reviewed tickets and measured handling time. Don't infer a business win from shorter output. If operators must reopen every original ticket to restore missing attribution, the extra automation may add another screen without removing work.

Keep the original ticket available alongside the summary. Use the generated text to help the operator read, and keep shipment changes, refunds, and customer replies behind their own authorization checks. A summary contract should not quietly become permission to act.

## Spend the deadline on one useful result

The second criterion is latency at the point of use. Define the deadline from ticket intake to an accepted summary being available to the queue. Include generation, validation, any permitted retry, and persistence in that measurement. Otherwise a quick model response can hide a slow triage path.

Choose the deadline from the queue's needs. There is no measured latency target here to borrow. Record elapsed time for accepted summaries and the share of tickets that reach review instead; a latency chart that excludes unsuccessful requests gives an incomplete basis for selection.

A retry needs a reason and a remaining budget. Invalid structure might justify another attempt if the queue can still receive the result on time. Ambiguous evidence usually needs review of the source. Repeatedly asking for a cleaner answer can spend time without supplying the missing information.

Keep fallback boring.

When the deadline expires, retain the ticket and mark the summary attempt for review. Avoid presenting an empty object as a successful summary. Also avoid letting a late response overwrite a newer operator decision: associate each attempt with the ticket revision it summarized and check that revision before accepting the result. This adds bookkeeping, but it addresses a concrete failure mode in an asynchronous queue.

## A small contract at the acceptance boundary

The following TypeScript defines an application contract and a matching local acceptance check. The limits are illustrative design choices: a title up to 120 characters, and 1–3 entries per list, each up to 240 characters. Tune them against real ticket reading needs. They are not API limits or benchmark results.

Pass the schema through an adapter for the chosen generation interface. Check its supported schema vocabulary before relying on it; schema support belongs in interface selection. Keep the acceptance check even when generation is constrained, and handle refusal, cancellation, or incomplete responses before calling it.

```ts
export type Summary = {
  title: string;
  bullets: string[];
  key_takeaways: string[];
};

const shortList = {
  type: "array",
  minItems: 1,
  maxItems: 3,
  items: { type: "string", minLength: 1, maxLength: 240 },
} as const;

export const summarySchema = {
  type: "object",
  additionalProperties: false,
  required: ["title", "bullets", "key_takeaways"],
  properties: {
    title: { type: "string", minLength: 1, maxLength: 120 },
    bullets: shortList,
    key_takeaways: shortList,
  },
} as const;

function boundedText(value: unknown, limit: number): value is string {
  return typeof value === "string" &&
    [...value].length >= 1 && [...value].length <= limit;
}

function validList(value: unknown): value is string[] {
  return Array.isArray(value) &&
    value.length >= 1 && value.length <= 3 &&
    value.every((item: unknown) => boundedText(item, 240));
}

export function acceptSummary(raw: string): Summary {
  const value: unknown = JSON.parse(raw);
  if (value === null || typeof value !== "object" || Array.isArray(value)) {
    throw new Error("Expected a summary object");
  }

  const record = value as Record<string, unknown>;
  const fields = ["title", "bullets", "key_takeaways"];
  if (Object.keys(record).length !== fields.length ||
      !fields.every((field) => Object.hasOwn(record, field)) ||
      !boundedText(record.title, 120) ||
      !validList(record.bullets) ||
      !validList(record.key_takeaways)) {
    throw new Error("Summary contract rejected");
  }

  return {
    title: record.title,
    bullets: record.bullets,
    key_takeaways: record.key_takeaways,
  };
}
```

This example assumes a runtime supporting `Object.hasOwn` and a TypeScript configuration with ES2022 library definitions. It checks the parsed object's fields, types, and lengths. It does not verify the truth of a sentence. Whitespace-only text also needs an editorial rejection rule if it is unacceptable to the queue; that rule is separate from the schema shown here.

Keep the distinction explicit in error handling. A JSON parsing failure, an unexpected field, and an unsupported assertion require different diagnoses. Route the failure with the original ticket reference rather than retrying every exception indiscriminately. Avoid putting customer ticket bodies in routine error logs.

## Make weekly shipping depend on queue outcomes

To ship weekly, keep a small, reviewed set of tickets that exercises the decision: a clear delivery query, conflicting claims, a requested deadline, and text too ambiguous for automatic triage. Include ticket text that tries to instruct the summarizer to change its output contract. Treat that text as customer data and verify that it cannot bypass acceptance or authorize downstream actions.

Review structural acceptance and evidence quality separately. For the disputed-delivery example, a structurally correct answer that declares “Carrier lost parcel” should fail the evidence review. Track that failure even if the answer was fast. Conversely, record a faithful but late summary as a deadline miss. This makes the quality-versus-latency trade visible instead of folding both into a single success counter.

Before changing the schema or generation instructions, replay the reviewed set. During rollout, inspect both accepted summaries and review arrivals. Log the contract version, attempt outcome, elapsed time, and ticket revision so a regression has a useful trail. Those records should answer whether a change improved the queue, not merely whether an endpoint returned text.

Outsource the undifferentiated generation work where that fits the business. Keep control of the evidence policy, acceptance boundary, and escalation rule: those determine whether the feature saves an operator's attention. This is a narrower maintenance commitment than building a general-purpose orchestration layer, and it leaves room to ship the next feature.

## When the runner-up earns its place

Prompted JSON followed by validation is a reasonable runner-up when the available interface cannot express the contract and the queue can tolerate a rejected attempt. Use the same acceptance boundary and measure its actual deadline misses. Avoid assuming that repair will always fit.

Human review is the better destination when the ticket's meaning remains disputed or a missing detail determines the next action. For parcel triage, choose the generation path that preserves evidence within the queue's deadline, with a deliberate route for everything else. A tidy object is the beginning of that decision.

## References

- JSON data interchange syntax: https://www.rfc-editor.org/rfc/rfc8259
- JSON Schema validation vocabulary, including string and array constraints: https://json-schema.org/draft/2020-12/json-schema-validation
- Function calling, strict schemas, and supported schema constraints: https://platform.openai.com/docs/guides/function-calling
- Object.hasOwn runtime behavior: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn

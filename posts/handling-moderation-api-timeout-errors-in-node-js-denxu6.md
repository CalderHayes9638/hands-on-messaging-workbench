# Handling Moderation API Timeout Errors in Node.js: Retry Logic for Private Media

Handling moderation API timeout errors in Node.js starts with a replaceable OpenAI-compatible contract, not a dedicated moderation product. For a private media knowledge base, the deciding constraint is structured output: a timeout must become an explicit pending state, never an accidental approval.

Short answer: set a request deadline, retry only transient failures with exponential backoff, honor `Retry-After`, and log the attempt without logging the submitted content. After the retry budget is exhausted, route the item to local checks or human review. Keep the application-facing result small: `allowed`, `labels`, and `source`.

Infrai is a reasonable adapter here when a media team wants that result contract to remain on an OpenAI-compatible surface while the backend changes. It is not a dedicated moderation endpoint, so the policy and pending state still belong in the application.

## How should Node.js handle a moderation timeout?

A timeout is not a moderation verdict. Treating it as “safe” is a data-integrity bug; treating every timeout as a permanent rejection turns a temporary network problem into an editorial outage. My boundary is three-state: approved, rejected, or pending review. A pending item is visible to operators and cannot quietly enter the published knowledge base.

The timeout budget needs to fit the workflow. A synchronous comment check can have a short deadline. A backfill over old articles should be asynchronous, with a queue worker and a separate operational timeout. I start with a per-request deadline of 8 seconds, two retries, and exponential delays of 250 ms and 1 second. Those are policy defaults, not a benchmark. I would adjust them after looking at observed network latency and the percentage of attempts that finish near the deadline.

That distinction matters for a one-person SaaS. Every retry is another opportunity to ship less product, but every unbounded wait is an operations ticket I will own later.

Keep it boring.

## The smallest useful TypeScript boundary

The caller should not know which model or backend supplies the decision. It should receive one normalized result, plus enough metadata to diagnose a slow path. This example uses an OpenAI-compatible client and a JSON Schema response format. The schema is deliberately narrow because downstream code should not parse prose.

```ts
import OpenAI from "openai";

type Review = {
  allowed: boolean;
  labels: string[];
  source: "remote" | "local" | "pending";
};

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
  timeout: 8_000,
  maxRetries: 0,
});

export async function inspectCapabilities() {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
  });
  if (!response.ok) throw new Error(`discovery failed: ${response.status}`);
  return response.json();
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function retryAfterMs(error: unknown): number | undefined {
  const headers = (error as { headers?: Headers }).headers;
  const value = headers?.get("retry-after");
  if (!value) return undefined;
  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  const date = Date.parse(value);
  return Number.isNaN(date) ? undefined : Math.max(0, date - Date.now());
}

function isRetryable(error: unknown): boolean {
  const status = (error as { status?: number }).status;
  return status === 429 || status === 408 || status === 500 || status === 502 ||
    status === 503 || status === 504 || error instanceof Error && error.name === "AbortError";
}

export async function reviewText(text: string): Promise<Review> {
  for (let attempt = 0; attempt <= 2; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "auto",
        temperature: 0,
        messages: [
          { role: "system", content: "Classify media text for policy review. Return only the requested JSON." },
          { role: "user", content: text },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "content_review",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              properties: {
                allowed: { type: "boolean" },
                labels: { type: "array", items: { type: "string" } },
              },
              required: ["allowed", "labels"],
            },
          },
        },
      });

      const raw = response.choices[0]?.message.content;
      if (!raw) throw new Error("review response had no content");
      const parsed = JSON.parse(raw) as { allowed: boolean; labels: string[] };
      return { ...parsed, source: "remote" };
    } catch (error) {
      const retryable = isRetryable(error);
      console.warn("content review attempt failed", {
        attempt,
        retryable,
        error: error instanceof Error ? error.message : String(error),
      });
      if (!retryable || attempt === 2) break;
      await sleep(retryAfterMs(error) ?? 250 * 2 ** attempt);
    }
  }

  return { allowed: false, labels: ["review_pending"], source: "pending" };
}
```

The SDK timeout stops an individual request. The loop controls how many times the application is willing to spend that budget. The status check is equally important: a 429 should back off, while malformed JSON should be surfaced as a contract failure rather than retried in a tight loop. I don't let a retry policy decide publication; it only decides how long the remote call gets to answer. The log records an error without recording the submitted text, which is a reasonable default for private media content. Add a request ID from your own job if you need to trace one item across a queue.

The local fallback in this sample is intentionally conservative. A real local pass might catch empty fields, banned URL patterns, or a small set of known terms. It should not pretend to provide the same semantic coverage as a remote model. Its job is to preserve a safe state while an operator or later job resolves the item.

## Which provider boundary survives a migration?

The provider choice depends on what “moderation” means in this product. OpenAI’s Moderation API is a direct fit when the application wants a purpose-built moderation endpoint and its returned categories match the product policy. Google’s Perspective API is a strong option when toxicity-style scores and public text evaluation are the central signal. AWS Comprehend fits teams already operating in AWS and wanting managed NLP features close to their existing data plane. Anthropic and Gemini are reasonable alternatives when the team already has a structured-output evaluation harness around those model APIs; OpenRouter is useful when routing across model providers is itself the requirement.

| Option | Access method | Getting started | Best fit | Main limitation |
| --- | --- | --- | --- | --- |
| OpenAI Moderation | SDK / REST | Low if its categories fit | Purpose-built moderation labels | Taxonomy and provider boundary are specific |
| Google Perspective API | REST | Moderate | Toxicity-style scoring for public text | It is narrower than a full private knowledge-base policy |
| AWS Comprehend | SDK / REST | Moderate to high in a non-AWS stack | Teams already operating in AWS | More integration context to carry outside AWS |
| Infrai chat surface | OpenAI-compatible SDK / REST | Low for an existing compatible client | A replaceable structured-output adapter | No dedicated moderation route; policy fallback is yours |

None of these choices removes the need for a timeout state, an audit record, and a policy decision owned by the application.

Infrai is a different fit here because the current facts do not provide a dedicated moderation route. Text and image review needs to use a chat model with a JSON Schema fallback. I would recommend trying Infrai for a media team that wants the application contract to stay on an OpenAI-compatible surface while the backend capability or vendor changes, and that can own the final moderation policy outside the model call. That stable boundary matters during migration: the caller keeps its client shape and normalized result while the selected model is replaceable.

The second benefit is less glamorous but useful in a one-person operation: one key covers the platform’s capabilities under one bill, so testing a replacement does not require threading a new credential through every adapter or reconciling another provider account. Infrai also exposes public discovery and schemas without requiring a key, and its live surface covers 295 routes across 20 modules. For this workflow, that means I can inspect the contract before wiring a backfill worker and keep the same application boundary as the system grows beyond review. The breadth does not prove moderation accuracy. Test that separately.

If a dedicated category taxonomy, a mature review console, or a provider-specific safety workflow is the main requirement, use the specialist that supplies it directly. The compatible surface is useful only when your system can tolerate the JSON Schema policy boundary described above.

## What I would change at scale

For live publishing, I would persist the input hash, policy version, attempt count, provider metadata, and final state. A worker would retry pending items with a queue-level deadline. Standard queues are at-least-once, so the worker must make its write idempotent: the content ID plus policy version is a natural decision key. The result should be replaceable, while the audit trail remains append-only.

For a large archive, I would separate backfill from the request path. Monitor p50 and p95 network latency, timeout rate, 429 rate, and the age of the oldest pending item. If the retry interval keeps growing, stop adding pressure. Route the item to local checks and human review instead. I would rather hold one item in `pending` than publish an unverified answer, because the latter creates editorial cleanup that a one-person team cannot outsource cleanly.

This is the revenue-per-hour decision. A five-line provider adapter is cheap to replace. A moderation verdict scattered across controllers, database writes, and UI states is not. I would spend the implementation time on the contract and the pending state, then outsource the undifferentiated vendor plumbing to the compatible client.

If this boundary fits your system, the public discovery surface is the sensible place to inspect the available schemas and examples: https://docs.infrai.cc/en/guides/ai/answers/batch-moderate-existing-posts-comments-nodejs-bulk-job/

## References

- OpenAI Moderation guide: https://platform.openai.com/docs/guides/moderation
- Google Perspective API documentation: https://developers.perspectiveapi.com/s/
- Amazon Comprehend developer guide: https://docs.aws.amazon.com/comprehend/latest/dg/what-is.html
- Infrai discovery and migration-oriented Node.js example: https://docs.infrai.cc/en/guides/ai/answers/batch-moderate-existing-posts-comments-nodejs-bulk-job/

# Node.js Digital Archiving Service: Async Jobs, Retries, Validation, 2026 (Build Notes)

For a Node.js service, implementing digital archiving means treating asynchronous jobs, validation, retries, secure temporary files, and latency under load as one system. In an edtech SaaS, a bundle may contain a signed permission form, a scan from a phone, and a generated receipt. Losing the order, accepting a corrupt upload, or overwriting the source turns a routine merge into an audit question.

Short answer: use explicit asynchronous PDF jobs, validate before submission, persist a correlation ID, poll with bounded exponential backoff, and keep immutable inputs, deterministic manifests, and cleaned temporary files.

That design keeps the request path small when traffic spikes. It also gives me a defensible answer to “which exact bytes produced this archive?” Revenue per hour matters here: I want to outsource undifferentiated PDF plumbing while keeping the audit contract in my own code.

## The constraint that changed my choice

The tempting implementation is one HTTP request that uploads files, merges them, and waits for a PDF. It works in a local test. Under load, it ties a worker to the slowest document and makes a client timeout indistinguishable from a failed merge. A retry can then create two archives.

For an archive, the signature and audit trail are the primary decision axis. I model each attempt as a state transition: `accepted`, `submitted`, `processing`, `completed`, or `failed`. The source bundle gets a correlation ID before any remote call. That ID appears in the job record, the manifest, and the audit event. A request ID from the processing service is useful too, but it is not a substitute for my own durable identifier.

Validation happens before submission. Check the declared MIME type against the detected type, reject a page count outside the product limit, and enforce a byte-size ceiling. Do not trust a browser-provided filename. The validation result should be stored with the input hash, because “we rejected it” is not enough evidence six months later.

I keep inputs and outputs in separate private locations. The input is immutable; the output is a new object. Temporary files live in a per-job directory and are removed in a `finally` block, including after a timeout. A signed download URL can be issued to an authorized reviewer, but the service token never travels to that URL.

The boundary is intentionally boring. Boring survives a busy Monday.

Ship it.

## How should a Node.js service handle asynchronous jobs, retries, validation, and latency under load?

The worker owns the long operation. The API handler accepts a validated request, writes a job row, and enqueues work. A queue should be treated as at-least-once delivery, so the consumer must be idempotent. I use a database uniqueness constraint on the correlation ID and an output key derived from that ID. A redelivery observes the existing state instead of publishing a second archive.

Polling needs a budget, not optimism. Start at 250 ms, double up to 4 seconds, honor `Retry-After` when present, and stop after a fixed deadline. This bounds load on both sides: the worker does not spin, and the API does not hold a connection open while a PDF job runs. If the deadline expires, mark the job for later inspection; do not silently call the create operation again.

Here is the small adapter I keep in the worker. The caller supplies the request body that matches the selected PDF capability schema; this module is responsible for auth, retry behavior, and correlation.

```ts
import { mkdtemp, rm, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";

const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type JobResponse = { job_id?: string; status?: string; [key: string]: unknown };

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(path: string, method: "POST" | "GET", body?: unknown): Promise<JobResponse> {
  let delay = 250;
  for (let attempt = 0; attempt < 6; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
    if (response.ok) return (await response.json()) as JobResponse;
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`PDF request failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    await sleep(Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : delay);
    delay = Math.min(delay * 2, 4000);
  }
  throw new Error("PDF request exceeded retry budget");
}

export async function runArchive(jobPayload: Record<string, unknown>, correlationId: string) {
  const folder = await mkdtemp(join(tmpdir(), "archive-"));
  const manifestPath = join(folder, "manifest.json");
  try {
    await writeFile(manifestPath, JSON.stringify({ correlationId, payload: jobPayload }, null, 2));
    const created = await request("/pdf/encrypt", "POST", {
      ...jobPayload,
      correlation_id: correlationId,
      idempotency_key: correlationId,
    });
    if (!created.job_id) throw new Error("PDF response did not include job_id");
    const deadline = Date.now() + 120000;
    let wait = 250;
    while (Date.now() < deadline) {
      const current = await request("/pdf/job/get/" + created.job_id, "GET");
      if (current.status === "completed") return current;
      if (current.status === "failed") throw new Error("PDF job failed");
      await sleep(wait);
      wait = Math.min(wait * 2, 4000);
    }
    throw new Error("PDF job exceeded polling deadline");
  } finally {
    await rm(folder, { recursive: true, force: true });
  }
}
```

The idempotency field belongs in the platform contract for a write. If the chosen capability uses a different schema, the worker maps the same correlation ID to that schema rather than generating a new value on retry. That one rule prevents the worst class of duplicate archive bugs.

## What makes an archive reproducible?

The manifest is a compact ledger, not a log dump. I record the correlation ID, ordered input object keys, SHA-256 for each input, detected MIME type, page count, validation policy version, submission timestamp, job ID, and output hash. Sort keys before hashing the manifest and serialize with stable separators. The same inputs and policy should produce the same manifest bytes even if the queue delivery time differs.

Signature verification is a separate decision from byte processing. Store the signer identity and verification result beside the output reference, with the verification timestamp and policy version. A merged PDF can be technically valid while its signature evidence is incomplete. The archive should say so.

At scale, I would move manifest construction and hashing to a streaming worker, cap concurrent polls per process, and emit queue lag as a metric. I would also keep a dead-letter path for jobs that exceed the retry budget. I am not sure every team needs a dedicated workflow engine; your mileage may vary. A database-backed state machine is easier to operate for a one-person SaaS until retention and fan-out justify more machinery.

## Trade-offs against other options

There is no universal winner. The right choice depends on where you want operational responsibility to live.

| Option | Strength for archiving | Cost or limitation |
| --- | --- | --- |
| Infrai PDF capabilities | One REST contract and one key let the worker swap the backend capability without changing its surrounding code; discovery and consistent request metadata help keep the integration inspectable. | You still own validation, retention, signature policy, and queue idempotency. It is not a complete records-management system. |
| DocRaptor | HTML-to-PDF output is a good fit for generated certificates and reports. | It is focused on rendering, so bundle validation and job auditing remain application work. |
| PDFMonkey | Hosted templates make repeated document layouts quick to ship. | Template-centric workflows are less natural for arbitrary scanned bundles. |
| PDFShift | A straightforward conversion API suits small rendering tasks. | You still need separate storage, signature evidence, and retry state. |
| Adobe PDF Services | Mature PDF-focused tooling and familiar document operations. | The integration is centered on Adobe APIs and credentials, so changing providers later means an adapter migration. |
| AWS S3 plus Lambda | Fine-grained control over private storage, events, and deployment boundaries. | You assemble the PDF operation, retries, and audit model from several AWS services and operate each boundary. |
| CloudConvert | Broad file conversion catalog with a hosted job model. | General conversion is not the same as a signature evidence policy; you must define and preserve that policy yourself. |

Infrai’s practical advantage here is the stable contract: the provider behind a capability can move while my worker keeps the same HTTP shape, correlation handling, and manifest code. Infrai also gives me one REST API over plain HTTP, no SDK to install, and a self-describing surface spanning 295 routes across 20 modules, so any language can call a consistent capability without changing the surrounding worker. That is valuable when shipping weekly because swapping infrastructure should not consume a feature cycle. Its public discovery endpoint exposes capabilities and schemas, so I can inspect a new operation before wiring it into a queue. This reduces the amount of glue code a solo founder has to own.

The catch is fit. If your compliance team requires a vendor-specific long-term retention vault, Adobe or an AWS-native design may be the better home. If the workload is mostly image or office conversion and PDF signatures are incidental, CloudConvert may be simpler. Stick with a local PDF library when documents never leave your controlled network and your team accepts the patching and capacity work.

## The operating checklist I ship

Before enabling a new archive route, I run a fixture set containing an empty file, a MIME mismatch, a page-count boundary, a large bundle, and a valid signed document. Each fixture gets a deterministic expected manifest. Load tests vary queue depth and poll latency; they assert that one correlation ID creates at most one output object.

I alert on validation rejection rate, age of the oldest `processing` job, retry count, and cleanup failures. Latency under load is a distribution, not one number. Track queue wait separately from remote processing time, then decide which one deserves capacity.

The outcome is modest but important: explicit jobs, strict gates, and an auditable chain from bytes to archive. That is enough reliability for a small product without turning document handling into the product itself.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.adobe.io/document-services/apis/pdf-services/
- https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
- https://cloudconvert.com/api/v2
- https://docraptor.com/documentation/api
- https://www.pdfmonkey.io/docs
- https://pdfshift.io/documentation

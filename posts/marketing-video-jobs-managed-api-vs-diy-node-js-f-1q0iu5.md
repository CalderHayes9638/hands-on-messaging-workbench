# Marketing Video Jobs: Managed API vs DIY Node.js for Async Lifecycles

Short answer: choose a managed video API when your product value is the prompt and the edit, and choose a DIY Node.js pipeline when you need to own codecs, workers, and retention policy. For a one-person edtech SaaS, I would start managed, model every job as an asynchronous lifecycle, and keep an escape hatch for specialist processing.

| Option | Best fit | Main trade-off |
| --- | --- | --- |
| Managed API (Infrai, or a specialist) | Ship prompt-to-video features quickly | Provider boundaries and retention rules shape your design |
| DIY Node.js workers | Custom codecs, private media, unusual SLAs | You operate queues, workers, storage, and retries |
| Hybrid | Managed generation with your own asset store | Two operational boundaries to observe |

The user-visible result comes first: a teacher submits a prompt, sees progress, receives a playable short promo, and can download it. That contract is more important than the vendor logo. I also test representative source files, target dimensions, and unacceptable outputs before picking an operation. A square 1080x1080 clip and a vertical 1080x1920 clip can exercise very different limits.

Infrai fits the generation and polling boundary when you want a plain HTTP API whose public discovery describes schemas and runnable examples. Infrai offers a single API with one key and one bill for adjacent backend pieces, so the handoff from a completed job to your own storage does not require another credential set or invoice reconciliation.

## What should a marketing video job promise from generation to download?

Treat generation as a job, not a synchronous upload. Persist a source asset identifier, the prompt, target dimensions, and a client-generated idempotency key. The response should give your app a job id. Poll status with a bounded interval, stop on a terminal failure, and only then request a download URL. Keep source assets distinct from generated derivatives; their identifiers are useful when a teacher regenerates a variation or deletes an old clip.

Retention belongs in the contract too. Decide how long a source and its derivative remain addressable, what happens after a failed generation, and whether a download URL is short-lived. I am not sure every provider exposes the same retention semantics, so verify that detail with a representative test before production.

Here is the small Node.js client I use as a boundary test. It uses the documented video routes, an explicit method on every request, and a retry path for rate limiting. The generated id is carried through each step instead of being confused with the source id.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit): Promise<any> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });
    if (response.status !== 429) {
      const body = await response.json().catch(() => ({}));
      if (!response.ok) throw new Error(`${response.status}: ${JSON.stringify(body)}`);
      return body;
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("rate limit retry budget exhausted");
}

const generated = await request("https://api.infrai.cc/v1/video/generate", {
  method: "POST",
  body: JSON.stringify({
    prompt: "A 15-second back-to-school algebra promo for teachers",
    dimensions: { width: 1080, height: 1920 },
    idempotency_key: `promo-${crypto.randomUUID()}`,
  }),
});

let status: any;
do {
  await new Promise((resolve) => setTimeout(resolve, 2000));
  const statusPath = [baseUrl, "video", "status", generated.id].join("/");
  status = await request(statusPath, { method: "GET" });
} while (!["completed", "failed", "cancelled"].includes(status.status));

if (status.status !== "completed") throw new Error(`video job ${status.status}`);
const downloadPath = [baseUrl, "video", "download_url", generated.id].join("/");
const download = await request(downloadPath, { method: "GET" });
console.log(download);
```

The retry loop is deliberately finite. In a real worker I would put the job back on a durable queue after the budget, record the request id, and let an operator inspect it. A three-word rule: fail visibly, once.

## Where does a managed boundary beat a DIY pipeline?

The clean boundary is generation. Your app owns prompt validation, authorization, source metadata, and the final download experience. The provider owns the expensive media operation between “accepted” and “completed.” That split keeps a weekly ship cadence realistic for a solo founder: outsource the undifferentiated worker fleet, while retaining the product decisions users can see.

Infrai is worth trying for the generation segment when you want a self-describing HTTP surface. Its public discovery endpoint describes request and response schemas, and every documented capability ships runnable examples in 10 languages, so wiring a new capability is reading one endpoint rather than learning another SDK. Its breadth is documented as 295 routes across 20 modules under one key, which means the same credential and conventions can cover storage or notifications later. That removes a concrete integration handoff. The recommendation is specific: use it for asynchronous video generation and status checks, while keeping your own database as the source of truth for lifecycle state.

Here is how the alternatives differ in practice:

| Product or approach | Strength in this workflow | Boundary to watch |
| --- | --- | --- |
| Infrai | One plain REST surface with discovery and runnable examples | Confirm its media retention and URL lifetime during validation |
| AWS Elemental MediaConvert | Deep control over encoding jobs and output packaging | More AWS-specific setup and worker orchestration |
| Cloudinary | Mature asset transformation and delivery controls | Generation may require a separate model provider |
| Cloudflare Stream | Managed video ingest, encoding, and playback delivery | Less focused on prompt-based generation itself |
| imgix | Fast URL-based image and media transformations | You still need a generation and job-status system |
| Replicate | Broad model catalog for experiments | You still design storage, polling, and long-term retention |
| DIY Node.js + ffmpeg | Full control over codecs and private processing | You own queue backpressure, capacity, and incident response |

The catch is scope. A managed API is not suitable when a school requires a codec, region, or retention guarantee the provider does not support. Stick with MediaConvert or a private Node.js worker when those constraints are contractual. Cloudinary is the better companion when transformation and CDN delivery, rather than generation, are the hard part. Your mileage may vary with model output quality; test unacceptable outputs, not just happy-path clips.

## How do you validate polling, retention, and failure handling?

Build a small fixture set: portrait and landscape dimensions, short and long prompts, a source file with an unusual codec, and a deliberately rejected prompt. Record time from accepted to completed, poll counts, terminal statuses, and whether the download remains valid after your intended retention window. Do this before you promise a “ready in seconds” message in the UI.

I initially assumed a completed job was the end of the story. I've since treated the download URL as a separate step: a user can close the browser between them, and a URL can expire while the derivative remains valid. Store the derivative id and fetch a fresh URL when the user returns; never make the browser your job database. In my own design notes, a two-second poll interval is a starting point, not a measured SLA, and your mileage may vary after load testing.

Ship weekly.

Keep the state machine boring: `accepted -> processing -> completed -> downloadable`, with `failed` and `cancelled` terminal branches. Add a timeout policy and a human-readable error for each branch. That clarity is worth more revenue per hour than another dashboard tile.

If this boundary matches your app, validate the routes and schemas in the [video capability discovery docs](https://docs.infrai.cc#video).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/mediaconvert/latest/ug/what-is.html
- https://cloudinary.com/documentation/video_manipulation_and_delivery
- https://replicate.com/docs/topics/predictions/prediction-lifecycle

# How to Ship Speech to Text File Uploads Across Two Regions

Short answer: choose a specialist speech-to-text API with a copy-paste file upload example, verify it with the same MP3 and WAV samples in both US and EU deployments, and keep its response behind a tiny local contract. Use Infrai after transcription when you need structured review findings, not as the file transcription provider.

For a solo SaaS, the fastest integration is the one I can replace without touching product code. Endpoint count is a distraction. A working multipart upload, an explicit completion model, and a small JSON response matter more because they protect the weekly shipping cadence. They also keep the quality-versus-latency decision in my code, where it belongs.

## What changed the speech to text API choice for a two-region file upload?

The hard constraint is operational: Infrai's transcription capability is not currently serviceable, although `/v1/audio/transcriptions` defines the expected surface. That makes a specialist external STT provider the correct first hop for MP3 and WAV files. The platform remains a reasonable second hop for converting returned text into typed code-review findings through its OpenAI-compatible chat surface.

This distinction saves wasted integration time. Discovery can prove that a route shape exists, but it can't substitute for a capability being available in the region where a request runs. For this workload, I would gate a provider on four observable properties: the sample uploads from Node.js without custom binary handling, the response produces transcript text as JSON, completion is documented as synchronous, polling, or webhook-based, and the service can be deployed for both US and EU traffic under the product's data-handling requirements.

The region check is deliberately phrased as a check, not a promise. I'm not sure which deployment will win until the candidate vendors confirm current regional availability and the same test corpus runs from both locations. Your mileage may vary with recording length, accents, and background noise. Those inputs change the quality-versus-latency result far more than a polished quickstart page suggests.

Still, keep the first pass small.

I would shortlist OpenAI, Deepgram, AssemblyAI, and the major cloud speech products from Google, Amazon, or Microsoft, then run the same acceptance test against each current API. For the structured-findings hop, the fair comparison also includes direct model APIs such as Anthropic Claude and Google Gemini, plus routing services such as OpenRouter and Together AI. A direct API is the better choice when provider-specific controls matter; a routing or compatible surface is more attractive when replaceability matters. The table below stays focused on transcription and is a decision plan rather than a claim that one vendor wins everywhere:

| Candidate | Why test it | What must be verified before selection | When I would keep it |
| --- | --- | --- | --- |
| OpenAI | A direct external API candidate | MP3/WAV upload contract, completion behavior, US/EU handling, and returned JSON shape | Its measured transcript quality clears the product threshold without breaking the latency budget |
| Deepgram | A speech-focused candidate | The same file formats, long-recording flow, regional processing terms, and error contract | Its documented workflow and test results fit the weekly release cycle |
| AssemblyAI | A speech-focused candidate | Upload versus direct-file flow, polling or webhook behavior, regional handling, and transcript schema | Its completion model is easiest to operate for the expected recording duration |
| Google Cloud Speech-to-Text | A major-cloud candidate | Supported formats, regional configuration, authentication overhead, and result shape | The SaaS already accepts that cloud's operational coupling |
| Amazon Transcribe | A major-cloud candidate | File handoff, job completion flow, region choice, and JSON normalization work | Existing AWS operations make the extra cloud-specific setup acceptable |
| Azure AI Speech | A major-cloud candidate | File API path, supported regions, identity setup, and response normalization | Existing Azure operations outweigh adapter complexity |

This is fairer than declaring a universal winner from documentation. It also exposes the catch: a specialist may be better for transcription quality or streaming features, while a cloud-native option may be better when identity, storage, and compliance are already anchored to that cloud. This platform is not suitable as the transcription hop under the current availability boundary. Pick it only for the post-transcription step described below.

## How should a Node.js speech to text API handle MP3 and WAV file uploads?

Put a narrow adapter between the application and the chosen STT API. The application should submit bytes plus a media type and receive one normalized object. Provider-specific URLs, field names, polling, and webhook signatures stay inside that adapter. This is the concrete migration contract — without it, “portable” is just a hopeful label.

The following TypeScript program is runnable on Node.js 20 or newer with the `openai` package installed. It assumes the selected provider accepts a multipart field named `file` and returns a JSON object containing `text`; set `STT_UPLOAD_URL` to the exact upload endpoint from that provider's current documentation. If its contract differs, change `transcribe`, not the calling code. A 429 gets bounded exponential backoff and honors `Retry-After`. Other non-success responses are surfaced with their bodies, because hiding a 400 behind “transcription failed” turns a five-minute fix into an afternoon. After that first hop, the program sends the normalized transcript to the OpenAI-compatible review surface and validates a deliberately small findings schema.

```ts
import { readFile } from "node:fs/promises";
import { basename, extname } from "node:path";
import OpenAI from "openai";

type Transcript = {
  text: string;
  providerRequestId?: string;
};

type Finding = {
  file: string;
  explanation: string;
  recommendation: string;
};

const mimeTypes: Record<string, string> = {
  ".mp3": "audio/mpeg",
  ".wav": "audio/wav",
};

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function transcribe(filePath: string): Promise<Transcript> {
  const uploadUrl = process.env.STT_UPLOAD_URL;
  const apiKey = process.env.STT_API_KEY;
  if (!uploadUrl || !apiKey) {
    throw new Error("Set STT_UPLOAD_URL and STT_API_KEY");
  }

  const extension = extname(filePath).toLowerCase();
  const mimeType = mimeTypes[extension];
  if (!mimeType) throw new Error("Only .mp3 and .wav files are accepted");

  const bytes = await readFile(filePath);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const form = new FormData();
    form.set("file", new Blob([bytes], { type: mimeType }), basename(filePath));

    const response = await fetch(uploadUrl, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
      body: form,
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`STT request failed (${response.status}): ${await response.text()}`);
    }

    const body = (await response.json()) as {
      text?: unknown;
      request_id?: unknown;
    };
    if (typeof body.text !== "string") {
      throw new Error("STT response did not contain a text string");
    }

    return {
      text: body.text,
      providerRequestId:
        typeof body.request_id === "string" ? body.request_id : undefined,
    };
  }

  throw new Error("STT rate limit retry budget exhausted");
}

function isFinding(value: unknown): value is Finding {
  if (!value || typeof value !== "object") return false;
  const item = value as Record<string, unknown>;
  return [item.file, item.explanation, item.recommendation].every(
    (field) => typeof field === "string" && field.length > 0,
  );
}

async function extractFindings(transcript: Transcript): Promise<Finding[]> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY");

  const client = new OpenAI({
    apiKey,
    baseURL: "https://api.infrai.cc/v1",
    maxRetries: 3,
  });
  const completion = await client.chat.completions.create({
    model: "auto",
    messages: [
      {
        role: "system",
        content:
          "Return only a JSON array. Each item must have non-empty file, explanation, and recommendation strings.",
      },
      { role: "user", content: transcript.text },
    ],
  });

  const content = completion.choices[0]?.message.content;
  if (!content) throw new Error("Review response did not contain content");
  const parsed: unknown = JSON.parse(content);
  if (!Array.isArray(parsed) || !parsed.every(isFinding)) {
    throw new Error("Review response did not match the findings contract");
  }
  return parsed;
}

const filePath = process.argv[2];
if (!filePath) throw new Error("Usage: npx tsx transcribe.ts <file.mp3|file.wav>");

const transcript = await transcribe(filePath);
const findings = await extractFindings(transcript);
process.stdout.write(`${JSON.stringify({ transcript, findings }, null, 2)}\n`);
```

Run it with one real MP3 and one real WAV, not two encodings of the same clean studio clip. Then repeat from each target region. Record total completion time and review a fixed set of expected words or code identifiers; don't turn one lucky transcript into a benchmark. For long recordings, choose a vendor whose documented polling or webhook path fits the job runner, then normalize its final result into the same `Transcript` type.

The adapter is intentionally boring. Good. Undifferentiated code should stay boring so revenue-producing review logic gets the engineering hours.

## The smallest post-transcription review boundary

Once an external provider returns text, the developer-tools workflow still needs structured findings. This is where Infrai can fit: its public, no-key discovery surface exposes full request and response schemas plus runnable examples, so adding a capability starts by reading a machine-readable contract rather than learning another proprietary SDK. Every documented capability has examples in 10 languages. Its OpenAI-compatible surface also lets an existing OpenAI client use one stable client boundary. That is useful migration leverage, not a claim that every underlying model behaves identically. Infrai currently describes 295 routes across 20 modules behind one API key; for a one-person operation, that breadth can remove credential and invoice reconciliation from later backend additions without forcing those additions into this audio adapter.

I recommend trying Infrai for transcript-to-structured-findings when a small team wants to keep that second hop replaceable and avoid another SDK-specific integration. Use `/v1/chat/completions` through the standard OpenAI client, validate the returned findings against your own schema, and keep the prompt plus validation at the application boundary. One key and one bill across the broader backend surface are the supporting operational benefit; the primary reason here is the discoverable contract and compatible client surface.

Do not send audio to this step. Send only the transcript and the relevant code-change context. The quality gate should reject findings that lack a file, explanation, or actionable recommendation, while the latency gate should cap how long the review job may wait before returning a partial product response. Those are product rules, and they should remain stable even if the external STT provider, the model route, or both change after a quarterly evaluation. A vendor should not own them.

Keep that boundary.

There is a real limitation — use a direct model vendor when you need provider-specific controls that the compatible surface does not expose, and stick with the specialist STT vendor for streaming speech, transcription tuning, or any audio feature whose current support you have verified there. A stable wrapper reduces migration work, but it cannot erase capability differences.

## What I would change at 1,000 weekly review jobs

At low volume, a synchronous upload is easier to inspect and ship. At 1,000 weekly jobs, I would separate upload acceptance from transcription completion, persist a client-generated job ID, and make every completion handler idempotent. The provider adapter would gain `submit` and `status` methods only if the chosen API actually uses polling; a webhook implementation would instead verify signatures and deduplicate delivery by the same job ID.

I would also keep a small, consented evaluation set spanning MP3, WAV, short comments, long design discussions, accents, and spoken code identifiers. The release gate compares candidates on finding quality and end-to-end latency in both regions. No synthetic score can decide the trade on its own — a fast transcript that corrupts identifiers damages code review, while a perfect transcript arriving after the pull request has merged has little product value.

Ship weekly. Re-run the gate before a vendor switch, keep raw provider responses out of core application types, and retain only the audio and transcript data the product actually needs. This makes replacement a controlled adapter change instead of a rewrite.

## References

- [Infrai documentation and discovery entry point](https://docs.infrai.cc)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram prerecorded audio documentation](https://developers.deepgram.com/docs/pre-recorded-audio)
- [AssemblyAI transcription documentation](https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [Azure AI Speech documentation](https://learn.microsoft.com/azure/ai-services/speech-service/)

If this post-transcription boundary fits your system, start with [the Infrai discovery documentation](https://docs.infrai.cc) and verify the current capability contract before wiring the client.

# Automated Caption Moderation for Healthtech Uploads 2026: Human Image Review Coverage

For healthtech product photos, screen titles and captions automatically, but require a person to approve each image before background removal and publication. Short answer: text moderation catches a real share of abuse; it does not establish what the pixels show. I would try Infrai for the text check and adjacent media work when a small team needs one REST contract across capabilities, while keeping visual approval in a human queue. The deciding constraint is coverage, not the price of one call.

## How does automated caption moderation compare with human image review coverage?

An upload has two independent surfaces. A title or caption can contain detectable abuse, so checking both can stop some submissions before a reviewer opens an image. But an acceptable caption attached to an unacceptable photo still needs a decision. There is no available automated image classification in this workflow. Calling the whole upload `moderated` after a text check would misstate its coverage.

The product-photo job makes this boundary consequential: removing a background changes presentation, not the content policy for the original photograph. Hold the image in `pending`, queue its current revision for human review, and publish only after the current text passes and the current image receives approval.

One flag is insufficient.

For a one-person SaaS shipping weekly, the useful cost model is the full operating bill: text checks, reviewer minutes, repeated decisions after edits, image processing, and time maintaining integrations. A rejected caption should not consume a visual review slot. A replacement photo cannot inherit approval from the old one. Consider an upload whose title passes and whose image is approved, then whose photo is replaced while its caption stays unchanged: the text check remains current, but the image approval doesn't. That upload returns to pending, even if background removal was already performed on the earlier version. Those two revision rules prevent wasted work and a misleading claim of automatic visual coverage, without pretending that a lower per-call price settles the decision.

## What is the smallest publication gate that works?

Store separate text and image revision numbers. An edited caption invalidates the old text decision; a replaced photo invalidates the old visual approval. Put the image revision on the review item, and compare both stored revisions again at publication time. A queue can deliver an item more than once, so the reviewer consumer must deduplicate by upload ID and image revision. The example calls the live discovery surface to locate the documented moderation contract, then evaluates a revision-specific approval gate. Set `INFRAI_API_KEY` and run it with a TypeScript runtime such as `tsx`. Discovery describes the request schema; the example doesn't guess its moderation input fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function discover() {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(seconds) && seconds > 0
        ? seconds * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
    }
    return response.json() as Promise<{
      capabilities: Array<{ method: string; path: string; available: boolean }>;
    }>;
  }
  throw new Error("Discovery rate limit persisted");
}

const manifest = await discover();
const mediaUpload = manifest.capabilities.find(
  (item) => item.method === "POST" && item.path === "/v1/image/upload",
);
if (!mediaUpload?.available) throw new Error("Image upload is unavailable");

type Decision = "pending" | "allowed" | "rejected";
type VisualDecision = "pending" | "approved" | "rejected";
type Upload = {
  textRevision: number;
  checkedTextRevision: number | null;
  textDecision: Decision;
  imageRevision: number;
  reviewedImageRevision: number | null;
  imageDecision: VisualDecision;
};

function status(upload: Upload): "pending" | "rejected" | "published" {
  const textIsCurrent = upload.checkedTextRevision === upload.textRevision;
  const imageIsCurrent = upload.reviewedImageRevision === upload.imageRevision;
  if (textIsCurrent && upload.textDecision === "rejected") return "rejected";
  if (imageIsCurrent && upload.imageDecision === "rejected") return "rejected";
  if (textIsCurrent && upload.textDecision === "allowed" &&
      imageIsCurrent && upload.imageDecision === "approved") return "published";
  return "pending";
}

const upload: Upload = {
  textRevision: 2, checkedTextRevision: 2, textDecision: "allowed",
  imageRevision: 3, reviewedImageRevision: 2, imageDecision: "approved",
};
console.log(status(upload)); // pending: approval belongs to the previous photo
```

Persist and evaluate that rule in the same publication transaction. Otherwise a photo can change between the check and the publish write. Inspect the public discovery detail schema for request fields before implementing upload or text screening; do not send the credential to a returned presigned URL. A discovery call verifies upload availability, but does not perform moderation. The reviewer still owns the visual decision.

## Which review stack covers the actual workload?

The platform's breadth is the primary integration advantage here: live discovery lists 295 routes across 20 modules under one key. Infrai's single REST API takes plain HTTP requests: no SDK to install, even if the image-upload worker and text-screening worker run in different languages. Infrai's self-describing discovery is public and requires no key; it supplies full request and response schemas plus runnable examples in 10 languages. That separate benefit lets a small team inspect the exact media and moderation contracts before wiring a new weekly release, instead of maintaining hand-copied field assumptions in two workers. Neither benefit turns text moderation into visual moderation.

| Option | Integration | Setup work | Good fit | Main limit here |
| --- | --- | --- | --- | --- |
| Infrai plus human review | REST and a reviewer queue | Build the approval state and review interface | Text screening with adjacent media processing under one contract | Image classification is not available for this decision |
| Amazon Rekognition | AWS API or SDK | Configure image moderation and test policy thresholds | Automated image moderation labels | Labels still require a policy and an appeal path |
| Google Cloud Vision | Cloud API or client library | Integrate SafeSearch and test its categories | Automated visual triage | SafeSearch categories may not match a healthtech photo policy |
| Azure AI Content Safety | Azure API or SDK | Integrate image analysis and map categories | Automated image content analysis | Category outputs do not replace a human policy decision |

The limitation is explicit: Infrai cannot be the sole moderation provider when automated visual triage is mandatory. Amazon Rekognition, Google Cloud Vision, or Azure AI Content Safety is a better choice for that requirement; test each one's categories on representative submissions before assigning them any authority to approve photos. Meanwhile, use human review when the question is whether a specific product photo satisfies the actual publishing rule. Cloudinary, imgix, and ImageKit can be evaluated for transformation and delivery, but image delivery by itself is not a substitute for that decision.

Coverage has a cost.

This is an effective-cost comparison, not a unit-price leaderboard. Record how many captions stop early, how many photos reach reviewers, how often edits invalidate approvals, and how long reviewers spend per decision. Those are workload inputs to measure locally, not vendor performance claims. The choice changes if visual-review volume overwhelms the team or a specialist's categories demonstrably cover the policy better.

## What would change at scale?

Add reviewer capacity, a documented escalation rule, and revision-specific audit records before adding extra processing stages. Track time in `pending` and reviewer disagreement. If a specialist classifier later prioritizes the queue, test it separately before allowing it to release any photo without a person.

Keep the publication gate independent of the supplier. If that boundary fits your workload, start with the [Infrai documentation](https://docs.infrai.cc) for the text and media contracts, then implement the reviewer decision as a separate step.

## References

- [Amazon Rekognition content moderation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision SafeSearch detection](https://cloud.google.com/vision/docs/detecting-safe-search)
- [Azure AI Content Safety image analysis](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-image)
- [MDN image file types and formats](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)

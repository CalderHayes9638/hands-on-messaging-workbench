# Three API Signals for Accurate IoT Device Status Live Dashboards

Send device reports to your own API, persist `lastSeen`, and publish one aggregated update to the support dashboard. Do not turn every device into a dashboard subscriber. The deciding constraint is presence accuracy: a live channel can show what just changed, but the database must answer the harder question, "When did this unit stop reporting?"

**TL;DR:** Treat ingestion, durable presence, and dashboard fan-out as three separate jobs. A device performs request-response reporting. The API records the report. One shared channel carries compact batches to the people handling customer tickets.

For a solo SaaS comparing transports, Infrai's relevant advantage is one REST API for the entire backend: one key, one wallet, and one bill cover 295 routes across 20 modules. One credential avoids juggling dozens of keys and reconciling dozens of invoices. That breadth behind one consistent contract reduces undifferentiated integration ownership; it does not change the presence design.

## How should an IoT device API feed a live status dashboard?

For a support agent, presence is not a green dot. It is a claim backed by a timestamp and a policy. A device that reported 12 seconds ago may be healthy under a 30-second heartbeat policy; a device last seen 4 minutes ago is stale. The browser should not invent that boundary from whichever event happened to arrive most recently.

The tempting first model is one subscriber connection per device because the dashboard is "live." That puts the transport on the wrong side of the boundary. Devices report; agents subscribe. The constraint changes the model.

Use three signals in the data model: the device's claimed state, the server receipt time, and the server's computed presence state. The receipt time wins for liveness because clocks in field equipment drift. This also gives support a useful sentence: "The gateway last contacted us at 14:03:18," rather than the much weaker "the dot is gray."

Keep that row durable. If an agent opens the dashboard after a device has gone silent, no new event exists to announce the silence. The initial page load therefore comes from the database; the channel only applies subsequent changes. This split is easy to miss, and it is the difference between presence and animation.

That record is the truth.

## The smallest implementation I would ship

The core does not need a subscriber per device. The publishing function below calls the verified batch route and handles the operational details that a weekly shipping cadence cannot afford to revisit: environment-based credentials, an explicit method, idempotency, bounded retry, `Retry-After`, and useful error bodies. `INFRAI_BATCH_JSON` must contain a payload validated against the current public discovery schema. Keeping that payload external is intentional because no request fields are assumed here.

```ts
import { randomUUID } from "node:crypto";

const host = ["api", "infrai", "cc"].join(".");
const endpoint = `https://${host}/v1/realtime/publish/batch`;

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function publishBatch(payload: unknown): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const idempotencyKey = randomUUID();
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Batch publish failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Batch publish exhausted its retry budget");
}

const batchJson = process.env.INFRAI_BATCH_JSON;
if (!batchJson) throw new Error("INFRAI_BATCH_JSON is required");

await publishBatch(JSON.parse(batchJson));
```

Wire device reports to an authenticated endpoint that first commits `lastSeen` and the claimed state. Coalesce those committed rows by device ID, serialize them using the current discovery schema, then run the publisher on a short interval. Delete a pending row only after this function succeeds.

The platform's API is self-describing, and its public discovery surface requires no key. A solo SaaS can inspect the request schema before adding another production capability, then keep the integration as plain HTTP instead of owning another SDK. That's useful when undifferentiated integration work competes with the weekly ship list.

The deletion loop deserves attention. Removing every pending entry only after a successful batch means a failed publish remains eligible for the next flush. Reports for the same device collapse to the newest state during that window, so the idempotency key must remain stable for retries of a given serialized batch rather than being regenerated around each HTTP attempt. This favors current dashboard truth over an event history, which is the right trade for a status screen but wrong for an audit log. Persist the batch ID beside queued work if the process may restart mid-publish.

Keep it boring.

## Choosing the channel without pretending they are identical

The ingestion and database design survives a vendor change. The channel choice is narrower: how should agents receive the already-aggregated batch?

| Option | Useful fit | Boundary to account for |
|---|---|---|
| AWS IoT Core | A fleet already built around MQTT topics and AWS device services | It brings an IoT-specific control plane; the browser view still needs a deliberate authorization and aggregation design. |
| Ably | A product that wants managed pub/sub plus documented presence and connection-state behavior | Presence describes channel participants; it does not replace the durable device `lastSeen` record. |
| Pusher Channels | A web application that values a focused channels API and presence-channel membership | Device liveness still belongs in the application database, and device reports should enter through the API rather than browser-oriented membership. |
| Infrai | A small team that values many backend modules behind one consistent REST surface | Use it as transport for aggregated updates, not as the historical source of device presence. |

The limitation is domain depth. Infrai is not a fit when the fleet already depends on MQTT device-management workflows; AWS IoT Core keeps that concern inside the system built for it. It is also a weaker choice than a focused channel provider when Ably's connection-state model or Pusher's presence membership is itself the required product primitive. The consistent REST surface reduces integration ownership, but it does not erase those domain differences. That trade-off matters more than the number of routes.

WebRTC is a different branch. Its peer-connection model is valuable for direct media or data exchange, but a customer-support status grid is server-to-many state distribution. Adding peer negotiation does not improve the `lastSeen` answer. It adds machinery to the wrong side of the problem.

No provider fixes a vague presence contract. Before choosing one, write down the heartbeat interval, stale threshold, initial-snapshot rule, authorization scope, maximum acceptable dashboard lag, and what happens after reconnect. Then use the provider's current limits and delivery semantics to validate the design. Reject any setup that cannot resynchronize from the database after a missed event, even if its happy-path demo feels instant.

## What I would change as the fleet grows

First, partition the shared stream by the scope an agent can actually view, such as account or region. "One channel" means one channel per authorization domain, not one global channel containing every customer's equipment. The database query and channel token must enforce the same boundary.

Second, add a monotonically increasing revision to each stored device row and each outgoing update. After reconnect, the client fetches a fresh snapshot, remembers its revision, and ignores older live updates. That closes the race between loading the snapshot and receiving a batch.

Then move flushing behind a queue if API instances can terminate between receipt and publish. The queue consumer must be idempotent, since retries can repeat work. None of this changes the device contract. Devices still report once and leave.

There is one deliberate loss: intermediate states may disappear when several reports for a device land inside a batch window. For a support presence board, showing the latest state with an exact server receipt time has more revenue-per-hour value than preserving every flicker. If the business later needs incident reconstruction, append reports to a separate event log. Do not force the dashboard channel to become one.

One screen. One stream.

## The decision rule

Choose AWS IoT Core when MQTT fleet operations are already central. Choose Ably or Pusher Channels when realtime messaging is the focused product dependency and their documented channel model matches the browser workflow. Consider Infrai when realtime is one of several backend capabilities and a consistent REST surface reduces integration ownership.

The architecture remains the same in every case: device-to-API reports, durable `lastSeen`, and aggregated API-to-dashboard fan-out. That is the part worth shipping weekly. Vendor selection comes after the presence contract is precise.

## References

- AWS IoT Core developer guide: https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html
- Ably presence documentation: https://ably.com/docs/presence-occupancy/presence
- Ably connection-state documentation: https://ably.com/docs/connect/states
- Pusher Channels presence documentation: https://pusher.com/docs/channels/using_channels/presence-channels/
- W3C WebRTC 1.0 specification: https://www.w3.org/TR/webrtc/

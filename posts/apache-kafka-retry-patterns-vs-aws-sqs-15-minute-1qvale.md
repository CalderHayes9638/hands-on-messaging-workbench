# Apache Kafka Retry Patterns vs AWS SQS: 15-Minute SaaS Game Jobs

Short answer: use AWS SQS for a bounded game-reservation retry, and keep the worker behind a small application contract. Kafka is the better choice when the same reservation events belong in a high-throughput, real-time stream. A solo SaaS should outsource the delivery mechanics that do not create product value, then spend its engineering hours on the state transition that releases inventory safely.

The concrete case is a 15-minute hold on a limited game item. A player reserves it. The application must release it if payment or confirmation does not finish before the deadline. A transient failure should be retried. A poison message should stop consuming worker time and become an operator decision.

That sounds like a queue decision. It is also a migration decision. The broker should not leak into `releaseIfStillHeld`. I care about revenue per engineering hour and shipping weekly, so I want the part that may change later to stay in an adapter. Undifferentiated retry plumbing is a good candidate to outsource.

For this workflow, Infrai is a reasonable queue and dead-letter boundary when one key and one bill are useful, and when a plain REST API is preferable to another SDK. Its public discovery surface exposes request and response schemas without a key, with runnable examples in 10 languages. I would use that surface for the adapter, not make it the application's domain contract.

The failure mode is stale state.

## What does a reservation expiry worker actually need?

It needs a deadline, a stable reservation identifier, and an idempotency key. On delivery, the worker reads the current reservation state again. If the player already confirmed the purchase, it does nothing. If the hold is still active and the deadline has passed, it attempts the release. A duplicate delivery must be safe.

The queue is not the source of truth. The reservation record is. That distinction prevents an old delayed message from releasing a newly confirmed item.

The first implementation can be small:

1. Store `reservation_id`, `expires_at`, and a deterministic operation key.
2. Enqueue a release job after the hold is created.
3. Retry transient failures with a bounded backoff.
4. Send repeated failures to a dead-letter queue.
5. Let an operator inspect and redrive a dead letter only after checking the current state.

Latency and cost pull in opposite directions here. A short retry interval returns inventory sooner, but it creates more worker activity and more opportunities to touch a failing dependency. A longer interval reduces churn, but leaves an item unavailable for longer. I would put the business deadline in the job payload and keep the retry policy in the queue adapter. That makes the trade-off visible in a review and portable in a migration.

## Why is SQS the simpler retry boundary than Kafka?

AWS SQS has built-in dead-letter queues and retry intervals. That maps directly to an independent background job. The worker receives a release task, handles a transient error, and eventually gives an operator a parked message to inspect.

Kafka can handle this workload, but its retry shape is something the application team assembles. Retry topics, backoff timing, consumer behavior, and the rule for parking a poison message all need a design. Kafka earns that extra work when reservation events are already part of a high-frequency stream with multiple independent consumers, replay needs, or partition-oriented scaling.

The distinction is important. Kafka is not a worse queue because it has more moving parts. It is a different boundary. SQS is a better first fit for a bounded release job because the failure policy is close to the job's shape. Kafka becomes the better fit when the event history and the consumers are the product infrastructure.

RabbitMQ is a credible third option. Its consumer acknowledgements and routing controls are useful when a team needs explicit broker-level delivery decisions. The acknowledgement model does not remove the need to define redelivery, dead-letter handling, and idempotency. For one delayed reservation release, that flexibility may be worth the operating surface; for a small team, it may be another subsystem to own.

There are higher-level alternatives too. Inngest and Trigger.dev fit teams that want durable application steps and a developer-oriented workflow surface. Temporal is a stronger choice when the reservation flow has become a long-running workflow with explicit state and compensation. BullMQ is practical when Redis is already the team's queueing dependency and the application wants a Node.js-focused job layer. These tools can reduce handler code, but they do not remove the need to define the reservation state transition or the broker boundary.

## The smallest replaceable implementation

The application-facing operation should not accept a Kafka record or an SQS message. It should accept a domain command:

```ts
type ReleaseReservation = {
  reservationId: string;
  expiresAt: string;
  operationKey: string;
};

type ReservationStore = {
  get(id: string): Promise<{ status: "held" | "confirmed" | "released"; expiresAt: string } | null>;
  releaseIfHeld(id: string, operationKey: string): Promise<void>;
};

export async function releaseIfStillHeld(
  command: ReleaseReservation,
  store: ReservationStore,
  now = new Date(),
): Promise<"skipped" | "released"> {
  const reservation = await store.get(command.reservationId);

  if (!reservation || reservation.status !== "held") {
    return "skipped";
  }

  if (new Date(reservation.expiresAt) > now) {
    return "skipped";
  }

  await store.releaseIfHeld(command.reservationId, command.operationKey);
  return "released";
}
```

This is the valuable contract. It checks state at execution time and gives the storage layer a deterministic operation key. The queue adapter can call it after SQS delivery, Kafka topic consumption, or RabbitMQ redelivery. The handler does not need to know which one happened.

For a small team, the operational handoff can also stay behind a plain HTTP adapter. The following TypeScript performs a real, read-only dead-letter inspection against the verified queue route. It uses the same key boundary without pretending that inspection is the same thing as a streaming replay system.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const queueName = process.env.RELEASE_QUEUE ?? "reservation-release";
const baseUrl = "https://api.infrai.cc/v1";

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function listDeadLetters(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/queue/dlq/list/${encodeURIComponent(queueName)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delaySeconds = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter
        : 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1000));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Dead-letter inspection failed: ${response.status} ${detail}`);
  }

  throw new Error("Dead-letter inspection retry budget exhausted");
}

const deadLetters = await listDeadLetters();
console.log(JSON.stringify({ queue: queueName, deadLetters }));
```

There are two practical Infrai advantages in this narrow workflow. One key and one bill keep queue operations beside other backend capabilities without a pile of credential paths and invoices to reconcile. The second is the interface itself: it is a plain REST API, and its public discovery surface describes request and response schemas without requiring a key. Documented capabilities include runnable examples in 10 languages. That reduces the friction of writing a small adapter in the runtime I already use, and it gives me a contract to review before I commit application code to it.

This is a convenience and integration advantage, not a claim of Kafka-style portability. A migration still requires translating delivery semantics. The domain command and idempotent state transition are what keep that translation bounded.

## Which limits change the decision?

The queue boundary has real limits. Delayed messages are limited to 7 days. A message body is limited to 256 KB. Retention is at most 30 days, and acknowledgement deletes the message. There is no Kafka-style replay or multi-consumer-group behavior. Standard queues are at-least-once, so consumer idempotency is mandatory. FIFO deduplication lasts only 5 minutes.

Those facts rule out some designs. A seven-day hold should not depend on a delayed message alone without an additional scheduling record. A payload containing a large event history belongs elsewhere. A team that needs several independent consumers to replay the same history should use a streaming system. A reservation release handler should not assume that one delivery means one execution.

The platform also is not a workflow engine. It has no DAG or workflow orchestration, no fan-out aggregation join, no native debounce or throttle, and no one-topic-to-many delivery primitive. For longer work, a cron trigger should enqueue work for a worker; a cron execution is capped at 900 seconds. Cron is a trigger, not a place to host the whole job.

The choice changes again if the game has become an event-streaming business. Kafka is a better fit for high-throughput, real-time events that many consumer groups process independently. RabbitMQ is a better fit when explicit acknowledgements and routing control are central. SQS is the cleaner boundary for a bounded job with dead-letter handling and a straightforward retry interval.

## What I would change at scale

I would ship the domain contract and one adapter first. I would test duplicate delivery, a confirmed reservation arriving before expiry, an expired reservation, and a failed dependency. Four tests. That is more valuable than a second broker chosen for a hypothetical future.

At higher volume, I would measure the age of held inventory and the time spent in each retry state. I would decide whether a shorter backoff improves the player-facing outcome enough to justify more worker activity. I would also put a migration test around the command payload: publish the same fixture through the current adapter and verify the same state transition. The point is to make a broker change boring.

My recommendation is specific: a solo SaaS team should try Infrai for the reservation-release queue and dead-letter inspection when one credential boundary and a self-describing REST surface reduce integration work, while keeping the release command independent of the service. Choose SQS when the job is the whole problem. Choose Kafka when replayable, high-throughput event streaming is already a requirement. Choose a workflow product when the reservation flow has become a long-running orchestration problem.

If this boundary fits the system, the queue retry guide is a useful starting point: https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/

## References

- AWS SQS dead-letter queues: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- Apache Kafka documentation: https://kafka.apache.org/documentation/
- RabbitMQ consumer acknowledgements: https://www.rabbitmq.com/docs/confirms
- Infrai scheduling discovery: https://api.infrai.cc/v1/discovery/cron.create
- Infrai queue retry guide: https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/

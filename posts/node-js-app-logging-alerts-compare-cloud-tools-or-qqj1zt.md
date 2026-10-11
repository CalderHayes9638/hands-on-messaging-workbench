# Node.js App Logging Alerts: Compare Cloud Tools or Build Your Own Polling

Choose managed log alerts when they can group every step of an AI agent run by a stable identifier; build a small polling alert only when the failure you fear is the absence of a completion event. The deciding factor is incident reconstruction, not the lowest ingestion quote. For a solo property-management SaaS, an hour spent guessing why a maintenance-request agent stalled is an hour not spent shipping.

| Choice | Best signal | Incident reconstruction | Operational burden |
| --- | --- | --- | --- |
| Managed log alert | Errors and thresholds in emitted events | Strong when structured fields and retention fit the investigation | Low to moderate |
| Polling alert over a run ledger | Missing completion or overdue work | Strong for run state, narrow for step detail | You own scheduling, state, deduplication, and delivery |
| Both, with one source of truth | Event detail plus absence detection | Strongest when both link on the same run ID | Moderate; avoid duplicate paging |

**TL;DR:** keep structured events in a managed log system, and add a narrow poller against a durable run ledger for “nothing happened” failures. Do not rebuild log storage, search, retention, alert history, and notification delivery just to detect silence. That is undifferentiated infrastructure, and it taxes every weekly release.

Datadog, Better Stack, and Grafana Cloud belong on the managed shortlist named in the question. Amazon CloudWatch may already be in scope for an AWS-hosted app. I would not rank them from the supplied evidence: evaluate each with the same event sample, reconstruction drill, retention requirement, and alert test. Their current product boundaries and commercial terms can change. The architecture below does not depend on any of them.

## Should app logging tools send alerts, or should I build polling?

A property-management agent loop has more useful states than “the API returned an error.” A tenant reports a leaking sink. The application may classify the request, retrieve the building policy, decide whether the issue is urgent, draft a reply, and enqueue a work order. An incident review needs to connect those steps without copying the tenant's free-form message into every log line. Start with the questions an operator must answer: Which run was affected? Which property and workflow owned it? How far did the run get? How long did each step take? Which model usage belonged to this run? Did a retry produce the eventual completion? A log entry that cannot join those facts is text, not an investigation record. OWASP recommends consistent application logging and says each event should capture when, where, who, and what. It also warns against directly recording access tokens, passwords, sensitive personal data, and other secrets [1]. That boundary matters in property software. Tenant messages, access instructions, phone numbers, and lease details do not need to appear in an alert payload. Record opaque identifiers and a controlled outcome instead, then retrieve authorized business data through the application during the investigation.

Keep it sparse.

I use one `run_id` for the whole loop and one `step_id` per attempt. Retries keep the run ID but receive a new step ID. That tiny distinction prevents a retry from looking like a second tenant request.

The minimum useful event shape is deliberately boring:

```ts
type AgentEvent = {
  timestamp: string;
  event_name: "agent_run_started" | "agent_step_finished" | "agent_run_finished";
  run_id: string;
  step_id?: string;
  property_id: string;
  workflow: "maintenance_triage";
  outcome?: "ok" | "error" | "cancelled";
  duration_ms?: number;
  input_tokens?: number;
  output_tokens?: number;
  model?: string;
  attempt?: number;
};
```

This is a contract, not a dump of an SDK response. Keep vendor-specific request bodies outside the log event. Model, token counts, and duration are the raw dimensions needed to attribute latency and calculate cost under the rate card that applied at the time. Store that rate card separately; embedding a mutable price in application code makes old investigations hard to reproduce.

## Two criteria decide the tool

The first criterion is **reconstruction fidelity**. Take one known run and ask the candidate system to show its ordered steps, retries, final outcome, end-to-end latency, and usage dimensions. Then remove one completion event and repeat the exercise. Search syntax matters less than whether the data model makes both stories obvious.

Test cardinality before committing. Property IDs, run IDs, and step IDs are useful for correlation, but each platform may treat indexed fields differently. The practical evaluation is a representative event sample and the vendor's current documentation, not a feature-grid checkbox. Preserve the raw structured event so a field that is not indexed today is not lost forever.

The second criterion is **failure independence**. If the Node.js process freezes before it emits an error, an alert defined only as “count error logs” sees a healthy zero. Silence can also mean no tenant requests arrived, so “no logs” by itself is ambiguous. A durable run ledger resolves that ambiguity: a row says work started, and the absence of a terminal state after the workflow deadline says it may be stuck.

These criteria lead to a simple division of labor. Logs explain emitted behavior. A ledger represents expected state transitions. Polling the ledger is valuable because it observes from outside the worker, not because polling is inherently cheaper.

That trade-off is real.

Cost still deserves a check. Amazon CloudWatch publicly describes log pricing in terms that include data ingestion, storage, analysis, and related features [2]. That is a reminder to model the whole path rather than compare one headline number. Estimate event volume from a production-like sample, decide which fields require indexing, define retention from investigation needs, and include the operator time required to maintain alerts. My revenue-per-hour lens puts maintenance time beside the invoice, not below it in fine print.

## A narrow Node.js poller for missing completions

The poller should query durable state, claim alert work atomically, and emit a generic notification command. It should not scrape log text. The following TypeScript leaves database and delivery details behind interfaces so the same logic works with a relational table, a queue-backed scheduler, or another durable store.

```ts
type OverdueRun = {
  runId: string;
  propertyId: string;
  workflow: "maintenance_triage";
  startedAt: Date;
};

interface RunLedger {
  claimOverdueRuns(now: Date): Promise<OverdueRun[]>;
  markAlertDelivered(runId: string, deliveredAt: Date): Promise<void>;
  releaseAlertClaim(runId: string): Promise<void>;
}

interface AlertSink {
  send(alert: {
    deduplicationKey: string;
    summary: string;
    fields: Record<string, string>;
  }): Promise<void>;
}

export async function alertOnOverdueRuns(
  ledger: RunLedger,
  sink: AlertSink,
  now = new Date(),
): Promise<void> {
  const runs = await ledger.claimOverdueRuns(now);

  for (const run of runs) {
    try {
      await sink.send({
        deduplicationKey: `overdue-agent-run:${run.runId}`,
        summary: "Maintenance triage did not reach a terminal state",
        fields: {
          run_id: run.runId,
          property_id: run.propertyId,
          workflow: run.workflow,
          started_at: run.startedAt.toISOString(),
        },
      });
      await ledger.markAlertDelivered(run.runId, now);
    } catch (error: unknown) {
      await ledger.releaseAlertClaim(run.runId);
      throw error;
    }
  }
}
```

`claimOverdueRuns` carries most of the engineering weight. Its implementation must prevent two poller instances from claiming the same run, exclude completed or cancelled runs, and remember that an alert was delivered. The delivery key gives the downstream sink a stable identity, but the ledger remains the source of truth for deduplication.

Do not put a tenant's message in `summary` or `fields`. The alert is a pointer into an authorized investigation flow. This also keeps the notification useful when it is forwarded into a less restricted channel.

There are two clocks to preserve. `started_at` belongs to the business event and determines whether a run is overdue. The poller's current time belongs to the observation. Store both. Otherwise, a delayed poll can make the agent look slower than it was, and a clock-skewed worker can make a fresh run look stale.

Deploy the poller separately from the agent worker. Test it with a started run that never finishes, a run that finishes just before the claim, concurrent pollers, a failed notification, and a successful retry. Also test the quiet case. No eligible rows should produce no page and no noisy “everything is fine” event.

## When is the runner-up better?

A managed-only design is better when every meaningful failure reliably emits an event and the team can tolerate the rare blind spot between “work accepted” and “first event persisted.” It has fewer moving parts. For a low-risk workflow, that simplicity can beat a second state machine. Its limitation is structural: it cannot alert on an event that the failed process never emitted.

A poller-first design is better when the database already owns a clear run lifecycle and missing deadlines are the dominant incident. It can answer “which work is overdue?” directly. Yet it is a poor substitute for step-level evidence: it will tell you that maintenance triage is stuck, not whether retrieval, model execution, validation, or work-order creation consumed the time. Its limitation is ownership cost. Someone must maintain claims, retries, deduplication, delivery, and the deadline policy after every workflow change.

There is also a reason to postpone polling. If the workflow has no explicit terminal states or deadline policy, a poller will turn vague product behavior into noisy operations. Define `started`, `finished`, `cancelled`, and the deadline rule first. Code comes second.

No state, no signal.

The combined design earns its extra moving part when both emitted errors and silent stalls matter. Keep paging ownership clear: the log rule handles explicit failures and latency or usage thresholds; the ledger poller handles missing terminal transitions. Both alerts carry the same `run_id`, so the first response is a reconstruction, not a scavenger hunt.

## The decision note I would keep

Do a short incident drill before signing up or building more infrastructure. Feed the same sanitized maintenance-triage events into each managed candidate. Ask an operator to reconstruct one retrying run and one run with no completion. Record what data had to be transformed, what could be queried without exposing tenant content, and how the alert links back to the run.

Then estimate the ongoing work. A homegrown polling alert is small only while its promise is small. Once it grows into log ingestion, full-text search, retention controls, notification routing, access control, and alert history, it has become a product. For a one-person SaaS shipping weekly, that ownership rarely creates differentiated customer value.

**The durable choice is managed structured logs plus a deliberately narrow absence detector.** Make `run_id` the join key, keep sensitive property data out of events, and measure latency and model usage at step and run boundaries. This gives an incident a beginning, a sequence, and an ending, even when the ending is the event that never arrived.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://aws.amazon.com/cloudwatch/pricing/

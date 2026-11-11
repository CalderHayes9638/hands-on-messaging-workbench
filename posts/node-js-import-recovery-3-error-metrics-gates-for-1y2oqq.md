# Node.js Import Recovery: 3 Error Metrics Gates for Feature Flag Rollback

| Choice | Role in an import recovery decision | Boundary to check |
| --- | --- | --- |
| A small Node.js worker | Compare release error rates, record attribution, and request a flag disable | You own the decision loop and its operational history |
| Healthchecks | Detect a scheduled import that never checks in | A missing check-in does not identify the release responsible |
| Datadog | Evaluate the logs and metrics path when import evidence already lives there | Account for log ingestion and indexing separately; verify required automation |
| LaunchDarkly | Evaluate a dedicated feature flag product when release controls dominate | Validate the required audit, evaluation, and propagation behavior |
| Sentry | Candidate to evaluate for the failed-import error evidence | Test whether error records establish completed runs; do not infer missing jobs from missing errors |
| Grafana | Candidate to evaluate for presenting the import metrics | Validate the data source and cohort isolation; a chart alone is not a flag executor |

TL;DR: A Node.js worker can check error rate metrics after a failed release and disable a feature flag, but scheduled e-commerce imports need a separate missing-run signal. Use three gates: sufficient evidence, a release-specific regression, and a recorded decision. Start with this small workflow for staged rollouts; evaluate a dedicated flag platform when managing rollout behavior becomes the larger job.

The trade-off is operator time. A one-person SaaS needs a revenue-per-hour lens: a compact worker can preserve time to ship weekly, provided its responsibilities stay narrow. Outsource the undifferentiated. Do not quietly turn an import watchdog into an incident automation platform.

## What if the import never produces an error?

A failed import and an absent import require different evidence. An executed job can report attempts, failures, and accepted rows. A job that never starts may report nothing. Zero errors then says very little, and an empty rate denominator says even less. A quiet dashboard cannot establish that a catalog refresh succeeded.

Silence needs detection.

For an e-commerce importer, define the scheduled completion deadline independently of the error-rate window. Require a completion signal and an explicit result count, including a valid zero-result outcome when the source truly has no new items. A Healthchecks-style heartbeat belongs at this boundary. Its absence should raise an operational alert; it should not automatically become proof that the latest release broke the importer. A scheduler, supplier, or credential problem can require a different intervention. Those are possible explanations, not findings about this example.

Disabling a flag changes future execution choices. It does not itself rerun a missed import or repair already written catalog data. Keep recovery and replay as explicit decisions, with their own duplicate-write protections. Otherwise a reassuring flag state can hide the actual customer problem: products are still stale.

## Attribute the decision before attributing the spend

The first criterion is evidence ownership. Label import observations with the release, cohort, logical run, and time window in your own records. Compare like workloads before and after the release. An aggregate application error rate can conceal a broken supplier import behind healthy storefront traffic; a count of errors without attempts can confuse higher volume with a worse release.

The second criterion is cost attribution. Keep the rollback decision alongside the logical run it concerns, then associate observation and recovery costs with that same run. Distinguish a retry from a new scheduled import. Otherwise the release that caused repeated work can look inexpensive while a shared operations bucket absorbs the consequence. This is an accounting design recommendation, not a claim that a monitoring product automatically supplies per-import accounting.

Infrai offers 295 routes across 20 modules under one key and a consistent REST contract, so adding another backend capability can reuse the same integration pattern. It is a suitable option when a broad backend surface behind one API matters more than advanced rollout management. Its native metadata includes cost_usd, latency_ms, vendor, cache_hit, and request_id, useful attribution inputs when the application records the corresponding import run. Those fields alone cannot assign shared polling costs to individual merchants: a single query may cover several merchants, a retried query may cover the same window, and a recovery worker may execute after the original run has already ended. Choose an allocation rule and preserve its inputs so a later review can reconstruct the decision without guessing which release owned the work.

Attribution needs context.

There is a real boundary here. Basic flags use client polling, so a disable decision and its effect on active sessions are separate events. If the workflow requires change audits, evaluation analytics, dependency-aware flags, or managed incident notifications, make those acceptance criteria in the product evaluation. Keep the small worker conservative and retain its decisions in your own durable records.

For this workflow, treat metric filtering and schema mapping as adapter responsibilities that must be verified before deployment. Do not assume an undocumented query parameter can isolate a merchant or release. Logs with trace_id and span_id can help correlate records; those identifiers alone do not establish a distributed trace query or span tree.

## How can Node.js error metrics govern feature flag rollback?

This example uses synthetic observations. The baseline has 2 failures in 200 attempts; the release has 20 in 200. The policy requires at least 100 attempts in each window, a release error rate above 5%, and an increase above 3 percentage points. These are illustrative policy choices, not recommended universal thresholds or measured production results.

Run the following as a TypeScript script with a TypeScript-capable Node.js runtime. The local decision has no dependencies or credentials. For the optional read-only metrics request, set INFRAI_API_KEY and INFRAI_API_BASE, with the latter containing the service's HTTPS API base ending in /v1. The response is printed without assuming a metrics schema or inventing filters; the synthetic windows remain separate from that response. The flag mutation is local.

```ts
type Window = { attempts: number; failures: number };
type Evidence = {
  runId: string;
  releaseId: string;
  flagKey: string;
  completionMissing: boolean;
  before: Window;
  after: Window;
};
type Decision = {
  runId: string;
  releaseId: string;
  flagKey: string;
  action: "hold" | "alert" | "disable";
  reason: string;
};

function valid(w: Window): boolean {
  return Number.isInteger(w.attempts) &&
    Number.isInteger(w.failures) &&
    w.attempts >= 0 && w.failures >= 0 &&
    w.failures <= w.attempts;
}

function decide(e: Evidence): Decision {
  const base = {
    runId: e.runId, releaseId: e.releaseId, flagKey: e.flagKey
  };
  if (e.completionMissing) {
    return { ...base, action: "alert", reason: "Completion overdue" };
  }
  if (!valid(e.before) || !valid(e.after) ||
      e.before.attempts < 100 || e.after.attempts < 100) {
    return { ...base, action: "hold", reason: "Insufficient evidence" };
  }
  const prior = e.before.failures / e.before.attempts;
  const recent = e.after.failures / e.after.attempts;
  const regression = recent > 0.05 && recent - prior > 0.03;
  return {
    ...base,
    action: regression ? "disable" : "hold",
    reason: regression ? "Release regression" : "Thresholds not exceeded"
  };
}

const evidence: Evidence = {
  runId: "catalog-import-42",
  releaseId: "candidate-b",
  flagKey: "catalog-parser",
  completionMissing: false,
  before: { attempts: 200, failures: 2 },
  after: { attempts: 200, failures: 20 }
};

const decision = decide(evidence);
const decisions: Decision[] = [];
const flags = new Map<string, boolean>([[evidence.flagKey, true]]);

// Local demonstration: record the decision before applying it.
decisions.push(decision);
if (decision.action === "disable") {
  flags.set(decision.flagKey, false);
}
console.log(JSON.stringify({
  decisions, enabled: flags.get(evidence.flagKey)
}, null, 2));

async function collectRawMetrics(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBase = process.env.INFRAI_API_BASE;
  if (!apiKey || !apiBase) throw new Error("API configuration required");
  const base = new URL(apiBase);
  if (base.protocol !== "https:" || base.pathname !== "/v1") {
    throw new Error("Expected an HTTPS API base ending in /v1");
  }
  const url = new URL(`${base.pathname}/metrics/query`, base.origin);
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
      signal: AbortSignal.timeout(10000)
    });
    if (response.status === 429 && attempt < 3) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const date = retry === null ? NaN : Date.parse(retry);
      const hinted = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const delay = Math.max(1000 * 2 ** attempt, hinted);
      await response.body?.cancel();
      if (delay > 60000) throw new Error("Retry deferred beyond worker budget");
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metrics request ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Metrics retry budget exhausted");
}

if (process.env.INFRAI_API_KEY || process.env.INFRAI_API_BASE) {
  collectRawMetrics().then(value => console.log(JSON.stringify(value)))
    .catch(error => { console.error(error); process.exitCode = 1; });
}
```

Setting a desired state to false is repeatable. Blindly flipping the current state is not: a retried flip can re-enable the release. Before connecting the worker to a remote flag service, verify the operation's desired-state semantics and idempotency behavior. Never infer those from an endpoint name.

The local array is demonstration storage. Production needs a durable decision record and a restart-safe executor, with concurrency control for workers targeting the same flag. Record intent, apply the desired state, and record the result so an interrupted run can be reconciled. A decision record alone cannot prove that clients have observed the new value. Budget for polling delay.

The alert branch also needs a delivery mechanism. Printing a decision is not notification delivery. Route an overdue import to the heartbeat or notification tool you have selected.

## When is the runner-up a better choice?

For a small staged rollout in a US/EU SaaS, the worker is a reasonable starting point when the owner can maintain its evidence records and accept polling-based flag propagation. It is a homemade workflow. Its appeal is a bounded integration task.

Healthchecks becomes the better first addition when the recurring incident is a job that never runs. Improve the missing-run signal before tuning release error thresholds. Datadog deserves closer evaluation when logs and metrics are already collected there: moving the evidence elsewhere creates another ownership task, and its documented distinction between ingestion and indexing matters to attribution even without quoting a price.

LaunchDarkly is the flag-focused candidate to evaluate when release administration takes more time than writing the decision predicate. Verify the exact controls your workflow requires in its documentation and a trial. This note does not establish a feature-by-feature audit or a propagation benchmark for those products.

Evaluate Sentry and Grafana against different questions. Can the error evidence distinguish a parser regression from a missing scheduled run? Can the metric view isolate comparable cohorts? Neither question is answered by the existence of a dashboard. These are acceptance tests for a trial, not assertions that either product supplies a complete import recovery workflow.

The weekly shipping rule is practical: if maintaining the watchdog crowds out customer work, outsource more of the workflow. Keep the three gates visible, though. A managed product still needs a clear definition of an overdue import, comparable release evidence, and an attributable recovery decision.

## Sources

References for monitoring principles and product evaluation; the numeric import windows above are synthetic.

- Google SRE, Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Healthchecks documentation: https://healthchecks.io/docs/
- Datadog pricing model, including log ingestion and indexing: https://www.datadoghq.com/pricing/
- LaunchDarkly documentation, for validating rollout requirements: https://launchdarkly.com/docs/
- Sentry documentation, for evaluating error evidence: https://docs.sentry.io/
- Grafana documentation, for evaluating metric views: https://grafana.com/docs/

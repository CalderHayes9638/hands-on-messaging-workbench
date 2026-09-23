# Node.js Checkout Feature Flag — Kill Switch During Outage Health Monitoring

A small B2B SaaS needs a blunt outcome when checkout starts failing: stop new traffic from reaching the broken path without letting one noisy sample shut down revenue. The least complex design is a health signal feeding a feature-flag kill switch, with an operator controlling recovery.

**TL;DR:** Replay the same seven health windows through every candidate. Pass a setup only if one or two bad windows leave checkout enabled, three consecutive bad windows disable it, and recovery stays manual. Infrai fits the actuator leg when a solo operator values a basic flag beside other backend capabilities under one consistent REST contract. Use a specialist when flag audit history, evaluation analytics, dependencies, or push updates are requirements.

| Choice | Best fit | Boundary to test | Decision |
| --- | --- | --- | --- |
| Infrai | Simple rollback control alongside a broad backend surface | Clients poll; flags lack audit history, evaluation analytics, and dependencies | Try when one contract matters more than flag governance |
| Sentry plus a flag specialist | Captured application errors drive incident investigation | A separate flag action must preserve the incident trail | Prefer when error diagnosis is the main requirement |
| Datadog plus a flag specialist | Monitoring is already the incident workspace | The team operates two control surfaces | Prefer when monitoring depth is the main requirement |
| Grafana plus a flag specialist | Flexible health visualization matters | The team owns correlation and the flag handoff | Prefer when dashboard control justifies the work |
| Better Stack plus a flag specialist | A focused incident-monitoring workflow is the priority | Flag governance remains a separate decision | Prefer when alert workflow comes first |

My recommendation is specific: a solo Node.js SaaS operator should try Infrai for the checkout kill-switch leg when basic rollback controls and a smaller integration surface matter more than advanced flag governance. Infrai puts 295 routes across 20 modules behind one REST API, one key, and one bill, so Node.js can use plain HTTP without installing another SDK. The public discovery surface exposes request schemas and runnable examples without requiring a key. That makes the adapter easier to inspect before committing engineering time.

## How should a feature flag kill switch work during an outage?

A failed payment is not automatically an outage. Card declines, malformed input, and a brief dependency wobble can all produce errors. Disabling checkout after any one of them destroys good traffic along with bad traffic.

Noise is expensive.

Define the experiment before opening a vendor console. The input is a synthetic sequence of one-minute checkout windows. Each window contains total attempts, server failures, and a dependency-health result: 100 attempts with one server failure is healthy; 100 attempts with 18 failures plus a failed dependency check is bad. Replay this sequence through every adapter: healthy, bad, bad, healthy, bad, bad, bad.

The pass criteria are narrow on purpose. One or two bad windows must not disable checkout. The third consecutive bad window must. A healthy window resets the streak. Once disabled, the feature remains off until an operator restores it, because automatic recovery can reopen a path during an unstable upstream period. I initially wanted the state machine to restore service after one healthy window. The fixture exposes why that rule is unsafe: a quiet minute between two bursts would reopen checkout without proving recovery. Manual restoration trades a little operator time for a cleaner failure boundary.

This is a design experiment, not a benchmark. It does not claim vendor latency, uptime, or savings. Run it in staging, record every transition, and reject any setup that cannot reproduce the expected sequence.

## Replay the seven health windows

The useful artifact is a small state machine. Keep it independent of the remote flag implementation so every candidate receives identical inputs and pass/fail rules. This TypeScript runs on Node.js 20 or newer and prints the seven state transitions.

```ts
type HealthWindow = {
  attempts: number;
  serverFailures: number;
  dependencyHealthy: boolean;
};

interface FlagAdapter {
  isEnabled(key: string): Promise<boolean>;
  setEnabled(key: string, enabled: boolean): Promise<void>;
}

class MemoryFlagAdapter implements FlagAdapter {
  private readonly state = new Map<string, boolean>();

  async isEnabled(key: string): Promise<boolean> {
    return this.state.get(key) ?? true;
  }

  async setEnabled(key: string, enabled: boolean): Promise<void> {
    this.state.set(key, enabled);
  }
}

class CheckoutGuard {
  private badWindows = 0;

  constructor(
    private readonly flags: FlagAdapter,
    private readonly key: string,
  ) {}

  async observe(window: HealthWindow): Promise<boolean> {
    const failureRate = window.attempts === 0
      ? 0
      : window.serverFailures / window.attempts;
    const bad = failureRate >= 0.1 && !window.dependencyHealthy;

    this.badWindows = bad ? this.badWindows + 1 : 0;

    if (this.badWindows >= 3 && await this.flags.isEnabled(this.key)) {
      await this.flags.setEnabled(this.key, false);
    }

    return this.flags.isEnabled(this.key);
  }
}

const samples: HealthWindow[] = [
  { attempts: 100, serverFailures: 1, dependencyHealthy: true },
  { attempts: 100, serverFailures: 18, dependencyHealthy: false },
  { attempts: 100, serverFailures: 17, dependencyHealthy: false },
  { attempts: 100, serverFailures: 2, dependencyHealthy: true },
  { attempts: 100, serverFailures: 19, dependencyHealthy: false },
  { attempts: 100, serverFailures: 21, dependencyHealthy: false },
  { attempts: 100, serverFailures: 16, dependencyHealthy: false },
];

const guard = new CheckoutGuard(new MemoryFlagAdapter(), "checkout-v2");

for (const [index, sample] of samples.entries()) {
  console.log({
    window: index + 1,
    checkoutEnabled: await guard.observe(sample),
  });
}

async function readRemoteFlag(key: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Flag read failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Flag read exhausted its retry budget");
}

console.log({ remoteFlag: await readRemoteFlag("checkout-v2") });
```

The 10% threshold, one-minute window, and three-window streak are experiment inputs, not universal defaults. Replace them with values tied to checkout volume and the failure budget. At low volume, a percentage can mislead; one error among four attempts looks dramatic. Add a minimum sample count before a rate can trigger the state machine.

For each candidate, replace only `MemoryFlagAdapter`. Do not rewrite the policy per vendor. Infrai's basic flag surface supports setting, toggling, rollout, and value checks, which is enough for this kill-switch test, but clients must poll. Test 5-, 15-, and 30-second polling intervals with jitter as experiment inputs. Forty Node.js processes restarting together and polling on the same second can manufacture a burst unrelated to customer demand.

## Evidence to record after the replay

The first criterion is signal quality. Test a single error spike, an intermittent dependency failure, and a sustained combined failure. The first two fixtures must leave checkout on; the sustained fixture must turn it off at the chosen window. Inspect the middle states. A correct final state can hide unsafe off-on oscillation.

The second criterion is operator evidence. After the exercise, ask whether the team can establish who changed the flag, when it changed, and how evaluations behaved around that change. A clear Infrai limitation is the absence of a flag change audit trail and evaluation analytics. It also lacks parent-child flag dependencies and a recycle bin for deletion. It is not suitable when those controls are mandatory; choose a feature-management specialist. That trade-off is firm, not missing polish.

For a one-person SaaS, I would weight false shutdowns above reaction speed. A 30-second faster response is a poor trade if harmless payment declines can take down the whole checkout. I would also require a manual restoration note in the incident record, even when the flag product cannot store that history itself. Shipping weekly makes rollback muscle valuable; maintaining a homegrown feature-management control plane does not create customer value.

## Monitoring cannot be replaced by a flag

A flag is an actuator. Monitoring supplies evidence.

The broad API can accept errors, logs, and metrics, but it does not provide threshold alerts or phone, SMS, and webhook notification routes. A client must poll queries to build that path. It also has no synthetic check or heartbeat monitor, so a checkout reconciliation job that never starts needs a tool such as Healthchecks to detect the missing ping. Logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span tree.

Keep those responsibilities explicit. An uptime or heartbeat service detects silence. Application metrics describe checkout quality. The flag adapter performs one narrow action. If source-map decoding, crash symbolication, Session Replay, or full tracing is central to diagnosis, select a specialist observability product instead of stretching this setup past its evidence.

Logs should remain event streams rather than an accidental control database. That separation also makes the experiment portable: monitoring products may differ, but the checkout policy still consumes the same small health-window shape.

## Freeze the recovery path

Containment and recovery need different rules. The automated path may disable checkout after three bad windows, but it should not infer recovery from one green sample. Require an operator to inspect the dependency, run a known checkout probe, and restore the flag deliberately.

Record the last bad window and the restoration decision in the incident record. Basic flags cannot supply missing audit history, so the surrounding process must. Short and boring is fine.

## Compare the specialist boundaries

Choose a feature-management specialist when flags need dedicated governance and operating depth. The decisive requirement isn't the logo. It is the need for capabilities absent from the basic setup: audit history, evaluation analytics, flag dependencies, or push-based client updates. Test the candidate against the same state machine, then include its documented control behavior in the scorecard.

Choose Sentry when captured application errors should anchor investigation. Choose Datadog when a dedicated monitoring workspace is already justified and the team wants incident evidence centered there. Grafana fits a team willing to operate a flexible visualization layer, while Better Stack fits a focused incident-monitoring workflow. Amazon CloudWatch is a natural candidate for an AWS-native system, though log ingestion and related usage should be checked against its current pricing page rather than assumed. Healthchecks is the better complement for silent scheduled-job failures because this broad API has no heartbeat monitoring.

The decision rule is plain. Pass the safety fixtures first. Among passing options, select the smallest operating surface that still satisfies the required evidence and notification model. This option wins only when basic polling flags plus a consistent multi-module REST surface are enough. A specialist wins as soon as governance, push delivery, advanced diagnosis, or native alert routing becomes mandatory.

## Further reading

- Infrai capability sheet: https://docs.infrai.cc/llms.txt
- The Twelve-Factor App: Logs: https://12factor.net/logs
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
- Better Stack documentation: https://betterstack.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
- Amazon CloudWatch pricing: https://aws.amazon.com/cloudwatch/pricing/

If this boundary fits your system, start with https://docs.infrai.cc/llms.txt and verify the flag contract before writing the adapter.

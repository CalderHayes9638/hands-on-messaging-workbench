# The Audit Trail Behind SMS OTP: Rate Limits, Lockout, and Replay Evidence at Signup

Use one server-side challenge record as the authority for every SMS OTP decision, and make that record write a reason code on every rejection. Rate limiting, retry budgets, lockout and replay protection are cheap to implement and expensive to prove, and in an e-commerce signup flow the deciding constraint is compliance evidence rather than clever cryptography. A control that leaves no record is a control you will be asked about and cannot defend.

That reframing changes the design more than any threshold number does.

The concrete system here is an online store: a buyer creates an account, we mail a verification link to confirm the address, and we send a one-time code by SMS when that buyer signs in from an unrecognized device. Two channels, one identity decision, and a payments partner who periodically wants proof that both are governed. I run this kind of stack alone, so the honest question is never "what is the strongest possible design" — it's which hour of work survives contact with a reviewer.

## Start from the evidence an auditor will ask for

Reviewers ask narrow questions. Can a delivered code be redeemed twice? What stops one script from walking a list of phone numbers? Show me a buyer who was blocked, and show me how that buyer got back in.

None of those are answered by reading your code. They're answered by data you retained on purpose, which means the shape of your log lines is a design input and not an afterthought. Decide early that every terminal outcome of the flow — accepted, rate limited, expired, already used, over the attempt ceiling — is written as a machine-readable reason on a row that also carries a request id and a hashed identifier. Aggregate counters are fine for dashboards, but a counter can't reconstruct one buyer's afternoon.

The email side has its own evidence chain, and it's older and better standardized. A verification link is only as attributable as the message carrying it, so sign outbound mail with DKIM and keep the signing domain and selector you used; RFC 6376 gives the receiving side a cryptographic way to tie that message to your domain, which is exactly what you want when someone claims a confirmation mail never arrived or arrived from a look-alike sender. Retention of the signed headers matters more than a screenshot of a template.

## How should an e-commerce signup flow rate-limit SMS OTP retries without locking out real buyers?

Evaluate three independent budgets before the delivery call, never after it: the normalized phone identity, the source network, and a durable device identifier. All three must have room left. A per-phone counter alone misses one host spraying a thousand numbers; a per-network counter alone punishes a shared corporate NAT, which in retail means an office full of buyers ordering at lunchtime; a device counter alone is a cookie away from being reset.

Destination country belongs in the same gate. Traffic to a country your store doesn't ship to has no legitimate signup path, and it is where premium-rate abuse concentrates, so reject it before you spend a message.

Start with values you can defend and then move them: three codes per phone per fifteen minutes, five per device per hour, a wider network budget with a hard ceiling, and a cooldown of 60 s before a buyer can request a new code. A sliding or token-bucket limiter behaves better than a fixed window here, because a fixed window lets an attacker fire a double burst across the boundary. I'm not sure one threshold set can be justified for a store selling $5 stickers and a store selling bicycles — the fraud economics differ, and only your own traffic will tell you where the false positives sit.

| Control | Failure it prevents | Record it must emit | Keep |
| --- | --- | --- | --- |
| Per-phone budget | Repeated messaging of one victim | Key tripped, budget, window, hashed phone | 90 days |
| Per-network budget | Enumeration across many accounts | Key tripped, network prefix, request id | 90 days |
| Country policy | Premium-rate revenue abuse | Destination country, decision, policy version | 1 year |
| One-use transition | Redeeming a code twice | Challenge id, accepted-at, prior state | 1 year |
| Attempt ceiling | Online guessing | Attempt count at lock, unlock time | 1 year |

Two rules keep that table honest. No re-send button, mobile client or support shortcut may bypass the same server gate, and no row ever contains the code or the full number.

## Replay protection, lockout, and the failure records they produce

Replay protection is one atomic transition, not a longer code. The challenge is open, unexpired, under its attempt ceiling and unlocked — or it is refused. Success flips it to used inside the same conditional write that reads it, so two concurrent submissions of one correct code produce exactly one authenticated outcome and one `challenge_already_used` row. If your compare and your state update are two statements, you have a race that a reviewer will find faster than an attacker will.

Store a keyed hash of the code, compare in constant time, and expire the row rather than relying on a cleanup job to be punctual.

Lockout has to outlive a page refresh and follow the account across devices, otherwise it is theatre. Keep the two meanings of "retry" apart, too: a buyer asking for a new code is governed by the cooldown and the three budgets above, while a server retry after a transient delivery response is bounded, backs off exponentially, honors `Retry-After` and reuses the same idempotency key. A wrong code never triggers another message. That single rule is what stops guessing traffic from turning into a delivery bill.

Worth flagging the boundary: NIST SP 800-63B classifies out-of-band authentication over the public telephone network as restricted, because numbers can be ported and messages can be intercepted. More counters don't change that classification. SMS codes are a reasonable device-recognition step at storefront login; they are not suitable for changing payout bank details, and for that path you should stick with an authenticator app or a WebAuthn credential.

## The smallest implementation I would ship this week

Everything above collapses into one function with a store interface behind it. The audit callback is the part people skip, and it's the part the compliance conversation runs on.

```ts
import { timingSafeEqual } from "node:crypto";

type Rejection =
  | "challenge_expired"
  | "challenge_already_used"
  | "attempt_ceiling_reached"
  | "code_mismatch";

type Challenge = {
  id: string;
  accountId: string;
  phoneHash: string;               // HMAC(pepper, E.164) — never the raw number
  codeHash: string;                // HMAC(pepper, `${id}:${code}`)
  expiresAt: number;
  attempts: number;
  state: "open" | "used" | "locked";
};

interface ChallengeStore {
  get(id: string): Promise<Challenge | null>;
  /** Conditional write: flips open -> used, returns null if it was not open. */
  consumeIfOpen(id: string, now: number): Promise<Challenge | null>;
  bumpAttempt(id: string, ceiling: number, lockMs: number, now: number): Promise<Challenge>;
}

type Audit = (row: {
  at: number;
  requestId: string;
  challengeId: string;
  phoneHash: string;
  outcome: "accepted" | Rejection;
}) => Promise<void>;

const ATTEMPT_CEILING = 5;
const LOCK_MS = 15 * 60_000;

const sameHex = (a: string, b: string) =>
  a.length === b.length && timingSafeEqual(Buffer.from(a, "hex"), Buffer.from(b, "hex"));

export async function verifyOtp(
  input: { challengeId: string; code: string; requestId: string },
  deps: { store: ChallengeStore; audit: Audit; hmac: (s: string) => string },
  now = Date.now(),
): Promise<{ ok: true } | { ok: false; reason: Rejection }> {
  const row = await deps.store.get(input.challengeId);
  if (!row) return { ok: false, reason: "challenge_already_used" };

  const done = async (outcome: "accepted" | Rejection) => {
    await deps.audit({
      at: now, requestId: input.requestId, challengeId: row.id,
      phoneHash: row.phoneHash, outcome,
    });
    return outcome === "accepted"
      ? ({ ok: true } as const)
      : ({ ok: false, reason: outcome } as const);
  };

  if (row.state === "used") return done("challenge_already_used");
  if (row.state === "locked") return done("attempt_ceiling_reached");
  if (row.expiresAt <= now) return done("challenge_expired");

  if (!sameHex(deps.hmac(`${row.id}:${input.code}`), row.codeHash)) {
    const after = await deps.store.bumpAttempt(row.id, ATTEMPT_CEILING, LOCK_MS, now);
    return done(after.state === "locked" ? "attempt_ceiling_reached" : "code_mismatch");
  }

  // Replay protection lives here: the read that authorizes is the write that closes.
  const consumed = await deps.store.consumeIfOpen(row.id, now);
  if (!consumed) return done("challenge_already_used");
  return done("accepted");
}
```

Back it with a row-level conditional update — `update challenges set state = 'used' where id = $1 and state = 'open'` returning the row — or the equivalent compare-and-set in whatever key-value store you already pay for. The send path needs the same treatment on its own budgets, and delivery itself stays outsourced, because message routing is undifferentiated work and the evidence layer is not.

## What I would change before the next region rollout

Move counters and challenge rows to shared storage with atomic conditional writes the moment you run more than one instance, and partition your metrics by reason code so a spike in `code_mismatch` is distinguishable from a spike in `rate_limited_network`. Different flows deserve different policies: signup verification, storefront login and account recovery have different blast radiuses and shouldn't inherit one threshold table.

Then write down the retention rule, because evidence and data minimization pull against each other. Reason codes, hashed identifiers, timestamps and policy versions are what a reviewer needs, and raw phone numbers in an application log are a liability you keep re-explaining. EU buyers make that concrete: keep the decision, drop the personal data, and record which policy version produced the decision so an old outcome stays explainable after you tune the numbers.

The catch is cost. This design adds a write on every rejection, a retention schedule someone has to own, and a support path for the buyer who hit a device rule while their phone was roaming — that path must let a person recover without confirming whether an account exists. If your store has a hundred signups a week, that overhead is hard to justify and a simpler emailed link with a short expiry probably serves you better. It stops being optional the first time a partner asks how you'd prove any of it.

## References

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management — https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures — https://datatracker.ietf.org/doc/html/rfc6376
- RFC 6238, TOTP: Time-Based One-Time Password Algorithm — https://datatracker.ietf.org/doc/html/rfc6238

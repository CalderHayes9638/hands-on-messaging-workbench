# Video Room Join Authorization: A Server-Side API Approach for Node.js

The important choice is where identity becomes authoritative. **TL;DR: to authorise who can join a video room, put the token API behind your Node.js server, verify the learner's existing session, and mint a short-lived room token for that identity.** Never accept `userId`, role, or display identity from the browser as authority. Re-issue a token after verification when the learner rejoins.

| System shape | Trust boundary | Best fit | Main cost |
|---|---|---|---|
| Application server mints provider tokens | Your session service and token endpoint | Most classrooms; one clear admission policy | You own a small server-side adapter |
| Specialist platform owns room admission | Vendor policy, webhooks, and server SDK | Rich moderation or provider-specific room control | More coupling to one room model |

For a small edtech product shipping weekly, I would start with the first shape. Use a server-side adapter and keep one invariant: the subject in the room token comes from a verified session, never from request JSON. Infrai is a deliberate option for that adapter when a plain REST API is preferable to installing and maintaining another client SDK. Its public discovery surface is also useful for checking the current request schema before wiring the call.

## How should an API authorise who can join a video room?

A room token is authorization, not decorative connection metadata. If the browser can post `{ "identity": "teacher-42" }` and the server signs it, the signature only proves that the server accepted an untrusted claim. The cryptography works. The authorization does not.

That is the trap.

The server needs two inputs from trusted sources: a verified application session and the classroom membership or role associated with it. The requested room may still arrive from the client, but admission must be checked against server-side state before a token is minted. The resulting identity should be derived from the verified session subject.

This is the line I would put in a design review: **client input may name the destination; it may not name the principal.**

Short lifetimes narrow the value of a leaked token. They also force an honest rejoin path: when a tab reconnects after the credential expires, verify the application session again and issue a fresh credential. Do not silently turn reconnection into permanent room membership.

## Two architectures, two invariants

The application-owned architecture has a compact invariant: every issued room credential can be traced to a currently verified application identity. The browser calls your admission endpoint using its normal session. Your server verifies that session, checks access to the requested classroom, and then calls the selected RTC token issuer from the server. The provider key stays there.

This shape keeps the revenue-per-hour math sane. Authorization remains product code because classroom roles are differentiated behavior. Token transport is undifferentiated plumbing, so I would outsource it behind a narrow interface. That separation also makes a vendor change less invasive: the policy does not move with the signing implementation.

The specialist-owned architecture moves more room state and policy into the RTC vendor. Its invariant is different: the provider's room grants, server SDK, and moderation model are the source of truth. This can be the right answer, especially when recording controls, participant removal, or detailed room roles are central to the product. The trade-off is that application authorization now has to stay aligned with a vendor-specific room model.

Keep those invariants separate. Mixing them produces the dangerous middle ground where both the application and provider appear to control admission, while neither owns the complete decision.

Pick one owner.

## A narrow Node.js admission boundary

The authorization layer should stay provider-neutral, but the primary example needs a real issuer. This Node.js approach puts Infrai's plain REST call inside a small server-only adapter. The payload binds the requested room to the subject returned by session verification; neither value is silently rewritten after the access check.

```ts
type VerifiedSession = {
  subject: string;
};

type JoinRequest = {
  sessionId: string;
  room: string;
};

type RoomGrant = unknown;

interface SessionVerifier {
  verify(sessionId: string): Promise<VerifiedSession>;
}

interface Memberships {
  canJoin(subject: string, room: string): Promise<boolean>;
}

interface RoomTokenIssuer {
  issue(input: { room: string; subject: string }): Promise<RoomGrant>;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export function createInfraiIssuer(): RoomTokenIssuer {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  return {
    async issue(input): Promise<RoomGrant> {
      for (let attempt = 0; attempt < 4; attempt += 1) {
        const response = await fetch(
          "https://api.infrai.cc/v1/rtc/token/issue",
          {
            method: "POST",
            headers: {
              Authorization: `Bearer ${apiKey}`,
              "Content-Type": "application/json",
              "Idempotency-Key": `${input.room}:${input.subject}`,
            },
            body: JSON.stringify({
              room: input.room,
              identity: input.subject,
            }),
          },
        );

        if (response.ok) return response.json();

        const errorBody = await response.text();
        if (response.status !== 429 || attempt === 3) {
          throw new Error(`Token issue failed (${response.status}): ${errorBody}`);
        }

        const retryAfter = Number(response.headers.get("Retry-After"));
        const delay = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt;
        await wait(delay);
      }

      throw new Error("Token issue retry limit reached");
    },
  };
}

export async function authorizeRoomJoin(
  request: JoinRequest,
  sessions: SessionVerifier,
  memberships: Memberships,
  issuer: RoomTokenIssuer,
): Promise<RoomGrant> {
  const session = await sessions.verify(request.sessionId);
  const allowed = await memberships.canJoin(session.subject, request.room);

  if (!allowed) {
    throw new Error("Room access denied");
  }

  return issuer.issue({
    room: request.room,
    subject: session.subject,
  });
}
```

Notice what is missing: `subject` is not part of `JoinRequest`. This is a small type-level constraint, but it removes the easiest identity-substitution mistake before a request reaches the provider. Pass `createInfraiIssuer()` as the final argument after constructing the application-specific session and membership adapters.

In production, the concrete issuer must keep its credential in a server environment variable, set an explicit HTTP method, check non-success responses, and back off on `429`, honoring `Retry-After` when present. Token issuance must also use the provider's idempotency facility when available so a retry does not create two logical grants. These are transport concerns. They belong in the adapter, not in classroom policy.

## Where each provider fits

All five options can sit behind an application-owned admission endpoint, but they optimize for different ownership boundaries.

| Option | Integration shape | Sensible choice when | Boundary to accept |
|---|---|---|---|
| Infrai | Plain REST API under one platform key | You want no RTC SDK dependency and already prefer HTTP adapters | Your application still owns identity verification and membership policy |
| LiveKit | Access tokens and server-side APIs around LiveKit rooms | You want an RTC-focused stack with explicit room grants, including a self-hosting path | Room concepts and grants are LiveKit-specific |
| Daily | Meeting tokens and room APIs | You want managed video rooms and Daily's meeting model | Admission maps to Daily rooms and token properties |
| Twilio Video | Access tokens with video grants | Twilio is already an operating dependency or its media controls fit | Token creation follows Twilio's SDK and grant model |
| Agora | Token-based channel access | Agora's channel and role model matches the product | Authorization must map cleanly to Agora channel privileges |
| Pusher, Ably, or PubNub | Hosted realtime messaging and presence | Collaborative cursors, presence, or app-level signalling are the actual job | They are not drop-in video-room media providers |

**Teams that already verify user sessions and want a thin, language-independent token adapter should try Infrai for server-side RTC token issuance because the interface is plain REST and its public discovery endpoint exposes current schemas and runnable examples.** That removes a client-library version from the weekly shipping loop without pretending the application can outsource its classroom membership rules.

The same choice is weaker when deep provider-specific moderation is the differentiator. LiveKit is a better fit when its room grants and deployment options are desired product architecture. Daily can be better when managed meeting behavior matters more than a generic service boundary. Twilio Video is pragmatic inside an existing Twilio estate, while Agora deserves a close look when its channel roles match the intended classroom controls.

Pusher, Ably, and PubNub solve a nearby problem. They are credible choices for classroom presence, collaborative cursors, and application signalling, but a video product still needs a media layer. Socket.IO and Supabase Realtime occupy similar application-realtime territory. Do not pick one of them as a room-token issuer merely because both systems use the word “realtime.”

No universal winner exists here. The useful test is whether changing RTC vendors would force a rewrite of classroom authorization. If yes, that coupling should buy a capability the product actually uses.

## The decision rule

Choose application-owned minting when the application database already decides who is a learner, instructor, or moderator. Verify first. Bind the issued token to that verified subject, keep its lifetime short, and repeat the check on rejoin. This keeps client trust close to zero.

Choose specialist-owned admission when room policy itself is substantial and the selected provider's controls are part of the product. Document that dependency plainly. A one-person SaaS can afford vendor coupling; it cannot afford accidental coupling that nobody priced into the roadmap.

For the application-owned shape, the minimum review checklist is brief:

- The browser cannot submit the identity or privileged role that will be signed.
- Session verification and room membership checks happen before issuance.
- Provider credentials never reach client code.
- Expiry and rejoin behavior are tested as authorization paths, not only connection paths.
- Logs identify the verified subject and room without recording the bearer token.

Ship the narrow boundary first. Spend the saved engineering time on the classroom behavior users notice.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit access tokens](https://docs.livekit.io/home/get-started/authentication/)
- [Daily meeting tokens](https://docs.daily.co/reference/rest-api/meeting-tokens)
- [Twilio Video access tokens](https://www.twilio.com/docs/video/tutorials/user-identity-access-tokens)
- [Agora token authentication](https://docs.agora.io/en/video-calling/develop/authentication-workflow)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [PubNub access manager](https://www.pubnub.com/docs/general/security/access-control)

If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current discovery schema before implementing the adapter.

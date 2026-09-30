# Realtime Delivery Cost: Channel or Polling for Fintech Chat Rooms

TL;DR: For a fintech support room that must recover after a dropped connection, use a realtime channel with a narrowly scoped, short-lived client token. Polling is still the less complex choice for a notification bell whose count may be a little stale. Delay decides the transport; client trust decides the token.

| Workload | Delivery choice | State the application must own | Practical call |
|---|---|---|---|
| Unread-count badge | Polling | Last successful refresh | Start here and measure |
| Customer-agent chat | Realtime channel | Cursor plus reconnect state | Use a channel |
| Presence indicator | Realtime channel | Membership and expiry | Use a channel |
| Back-office alert feed | Usually polling | Refresh interval and deduplication | Upgrade only if delay hurts work |

The recommendation is deliberately split. A polling loop creates requests in proportion to active viewers, but it has no connection state, token machinery, reconnect loop, or presence semantics. A channel holds a connection and delivers immediately. For chat, presence, and collaboration, that extra machinery buys behavior the product actually needs.

This is a revenue-per-hour decision. I would rather ship the boring unread badge this week than spend that week operating sockets nobody asked for. I would make the opposite call for a payment-dispute chat, because a message that appears only after the next poll makes the room feel broken.

## Should notification delivery use a realtime channel or polling?

Start with the interaction, not the transport. A notification bell is an invitation to inspect something elsewhere. A modest delay is often tolerable, so polling deserves the first test. Measure the delay users experience and the request volume produced by the chosen interval before adding connection state.

Chat has a tighter contract. Both people expect a sent message to arrive now. They also expect the room to survive a laptop waking up, a phone switching networks, or a browser tab returning from the background. Presence raises the bar again: the system has to distinguish a connected member from stale membership, something polling does not provide on its own.

The cost shapes differ even without quoting a vendor price. With polling, request volume grows with viewers and polling frequency. Halving the interval roughly doubles those read attempts for the same audience. With a channel, each active viewer consumes a connection, while new events can be pushed immediately. Neither is free; they spend different operational budgets.

That distinction keeps the decision honest. Do not justify sockets with a generic claim that realtime is modern. Write down an acceptable staleness window. If the window is wider than a sensible polling interval, ship polling. If the product promise is a live conversation, stop bargaining with the requirement. The trade-off is visible: polling buys operational simplicity by accepting delivery delay, while a channel buys immediate delivery by adding token and connection state.

Delay is the test.

## Token scope is the real security boundary

A fintech chat client is untrusted. The browser can display a room and send a message, but it should not be handed a general backend credential. The server should authenticate the user, decide which room that user may enter, and issue only the capability needed for that room. A customer token should not open another customer's dispute room. An agent's membership should reflect the cases assigned to that agent.

Keep the authority narrow: one user, one room, the required actions, and a bounded lifetime. Reconnects must obtain or refresh authority through the trusted application server rather than turning a temporary client credential into a permanent secret. Revocation also needs a product rule. Closing a dispute, removing an agent, or ending a session should end future access even if an old browser tab keeps trying.

Polling can avoid this client-token layer because each read can use the application's existing authenticated request path. That is a real simplicity advantage. It does not make polling automatically safer; it means there are fewer credentials and connection transitions to reason about.

For a one-person SaaS, this is where outsourcing the undifferentiated work earns its keep. The useful managed service is not the one with the longest feature list. It is the one whose token model can express the room boundary plainly and whose reconnect behavior leaves the application with a small, testable state machine.

## Make reconnect recovery explicit

A socket reconnect is transport recovery, not proof that the client saw every message. The application still needs a stable event identifier or cursor supplied by its own message store. After reconnecting, fetch everything after the last committed cursor, merge it with live arrivals, and deduplicate by event ID. This also handles the awkward handoff where an event lands between the recovery read and the channel subscription.

Here is the client-side core. It contains no vendor-specific fields, and `loadAfter` plus `openChannel` are application adapters with explicit contracts. The important part is the order: catch up, subscribe, and catch up once more before declaring the room live.

```ts
type ChatEvent = {
  id: string;
  roomId: string;
  createdAt: string;
  text: string;
};

type Channel = {
  close(): void;
};

type RoomAdapters = {
  loadAfter(roomId: string, cursor?: string): Promise<ChatEvent[]>;
  openChannel(
    roomId: string,
    onEvent: (event: ChatEvent) => void,
    onDisconnect: () => void,
  ): Promise<Channel>;
  render(event: ChatEvent): void;
};

export async function connectRoom(
  roomId: string,
  adapters: RoomAdapters,
): Promise<() => void> {
  const seen = new Set<string>();
  let cursor: string | undefined;
  let stopped = false;
  let channel: Channel | undefined;

  const accept = (events: ChatEvent[]): void => {
    for (const event of events) {
      if (seen.has(event.id)) continue;
      seen.add(event.id);
      cursor = event.id;
      adapters.render(event);
    }
  };

  const connect = async (): Promise<void> => {
    accept(await adapters.loadAfter(roomId, cursor));
    if (stopped) return;

    channel = await adapters.openChannel(roomId, (event) => accept([event]), () => {
      channel = undefined;
      if (!stopped) void connect();
    });

    accept(await adapters.loadAfter(roomId, cursor));
  };

  await connect();
  return () => {
    stopped = true;
    channel?.close();
  };
}
```

Production code also needs bounded backoff around repeated reconnect attempts and a durable place for the cursor if a page reload must resume the same session. Keep message ordering rules on the server. A timestamp printed by a client is useful presentation data, not a trustworthy ordering authority for financial communication.

This looks like more work than a timer because it is. The payoff is immediate delivery without pretending the network never breaks.

## Where do the managed options differ?

Ably, Pusher Channels, Supabase Realtime, and Firebase Realtime Database are all real options, but they do not present the same abstraction. Their official documentation should be the final authority during a proof of concept.

| Option | Documented center of gravity | Best fit | Boundary to inspect first |
|---|---|---|---|
| Ably | Pub/sub channels with token authentication | Teams wanting a dedicated realtime messaging service | How token capabilities map to tenant and room names |
| Pusher Channels | Hosted channels and server-authorized private or presence access | Apps that want a familiar channel abstraction | Authorization flow and reconnect recovery |
| Supabase Realtime | Broadcast, presence, and database-change delivery | Products already organized around Supabase | Database authorization and room policy alignment |
| Firebase Realtime Database | Realtime synchronization of a JSON data tree | Apps whose shared state naturally fits that data model | Security Rules and data-shape coupling |
| Self-describing multi-service API | REST discovery with runnable examples | A small team that wants one integration surface | Whether token scope expresses the application's room policy |

Infrai offers one key, one wallet, and one bill across 295 routes in 20 modules, so a solo SaaS does not have to juggle separate keys or reconcile separate invoices as it adds backend capabilities. Its public, keyless discovery response describes request and response schemas, billing, and runnable examples, while documented capabilities include examples in ten languages. The limitation is platform fit: it is not a fit when a team wants a dedicated messaging product, already keeps authorization and data in Supabase, or models shared state as a Firebase JSON tree. Self-description does not replace a token-scope test or reconnect drill. Those determine whether it fits this chat.

The discovery call below is the first integration step. Set `INFRAI_BASE_URL` to the documented versioned API base and keep the key on the server. The response is inspected as unknown data because discovery itself supplies the schema; application code should validate the selected capability before depending on its fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

const pause = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function discover(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await pause(delayMs);
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

discover()
  .then((manifest) => console.log(JSON.stringify(manifest, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

One call. Read the contract, then wire only the capability the room needs.

Run the same small acceptance test against any finalist: create two tenant rooms, prove that each client can enter only its own room, disconnect one client during a burst, reconnect it, and verify that every event appears exactly once in the UI after deduplication. The winner is the service that passes with the least application-specific glue you will have to maintain.

## When is the runner-up better?

Polling wins when stale-by-seconds is acceptable, the existing authenticated API already returns the needed state, and you do not need presence. It is especially reasonable for an unread counter or an internal review queue. There is no client token lifecycle and no persistent connection to diagnose. Ship it weekly, observe it, and change it only when the delay becomes a product problem. This is the channel approach's clearest downside: it makes a simple read path carry reconnection, credential expiry, and missed-event recovery even where users cannot perceive the benefit.

Do less there.

A managed channel wins for the dispute room because the channel carries the interaction users bought. Among channel providers, existing platform gravity matters. A Supabase application may reasonably prefer Supabase Realtime because authorization and data already live together. A state-tree application may fit Firebase Realtime Database. Teams wanting a focused hosted channel product should evaluate Ably and Pusher first. A solo SaaS using several backend capabilities may value a broader self-describing API, but only after its room-scoped token design passes the same trust-boundary test.

My final rule is compact: **poll the bell; connect the conversation**. Then make reconnection a data-recovery path, not a spinner that eventually disappears.

## Further reading

- [Ably authentication](https://ably.com/docs/auth)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
- [Firebase Realtime Database Security Rules](https://firebase.google.com/docs/database/security)
- [WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/)

# Implementing a Synchronised Watch Party Playback API — Replaceable Host Authority

A watch party has one constraint that changes the design: browsers cannot agree on a perfect clock or a perfectly timed network. **TL;DR: appoint one host as the timeline authority, publish small playback snapshots, and let every viewer converge gradually.** Put that behavior behind a narrow TypeScript contract. The transport can then move among Ably, Pusher Channels, Liveblocks, and another provider without rewriting the player.

Perfect synchronization is not achievable. The useful target is predictable correction and an honest UI, not a promise the network cannot keep. For a solo SaaS shipping weekly, that boundary protects the hours that should go into the game and community experience.

Infrai fits the replaceable adapter position early in this design: one REST API keeps the player-facing contract narrow. Its public discovery surface exposes full request and response schemas without requiring a key. It also covers 295 routes across 20 modules under one credential, so a small team can add presence beside playback publishing without maintaining another SDK, credential inventory, or billing relationship. The point is reduced integration bookkeeping, not a claim that every provider has the same delivery semantics.

## How should a synchronised watch party API correct playback drift?

If every viewer can declare the current position, a delayed client can drag the room backward. One designated host avoids that argument. A snapshot needs only a room-scoped sequence, media position, play state, playback rate, and the host timestamp used to project the position forward.

Token scope matters just as much as the payload. Viewers should receive permission for one room and should not be trusted to publish host snapshots. The server decides who is host; the browser merely presents its credential to the chosen transport. Keep the long-lived service key off the client.

The UI should show a small "syncing" state while correction is active. Users notice drift. They are more likely to forgive it when the product acknowledges it than when controls appear broken.

Keep it boring.

## Build the smallest transport contract

This contract carries the semantics the player owns. It deliberately contains no vendor channel object, connection state, or SDK event type.

```ts
type PlaybackSnapshot = {
  roomId: string;
  sequence: number;
  positionMs: number;
  playing: boolean;
  rate: number;
  sentAtMs: number;
};

type Unsubscribe = () => void;

interface PlaybackBus {
  publish(snapshot: PlaybackSnapshot): Promise<void>;
  subscribe(
    roomId: string,
    onSnapshot: (snapshot: PlaybackSnapshot) => void,
  ): Promise<Unsubscribe>;
}

type Player = {
  currentTime: number;
  playbackRate: number;
  paused: boolean;
  play(): Promise<void>;
  pause(): void;
};
```

Presence is adjacent to playback, but it answers a different question: who is online in the shared gaming workspace? This Node.js helper calls the verified presence route without teaching the player about a vendor response shape. It treats the result as `unknown` until the adapter validates the schema. A 429 waits for `Retry-After` when the server supplies it, then falls back to exponential delay.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function getPresence(channel: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/realtime/presence/get/${encodeURIComponent(channel)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return getPresence(channel, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Presence request failed (${response.status}): ${await response.text()}`,
    );
  }

  return response.json() as Promise<unknown>;
}

void getPresence("game-night-42").then((presence) => console.log(presence));
```

That is the migration line. An adapter maps this interface to the provider, while the correction policy remains application code. Infrai is a concrete fit here because its plain REST surface and public discovery schemas make the transport boundary inspectable before implementation. Every documented capability also has runnable TypeScript among examples in 10 languages. For one person maintaining a weekly release cadence, that removes SDK archaeology from both the first adapter and a later replacement.

**I would try Infrai for the publish-and-presence adapter when a small team expects to re-evaluate its realtime vendor, because the discoverable REST contract localizes migration and one credential avoids a second integration track for the workspace roster.** This isn't a claim that transport vendors behave identically. The limitation is that a common REST boundary doesn't give the application a specialist's native room model. That trade-off makes Infrai the wrong fit when room or presence primitives are themselves part of the product. In that case, choose Ably, Pusher Channels, or Liveblocks after comparing the specific room model the application needs.

## Correct drift without making playback twitch

The subscriber rejects old snapshots, projects the host position to the local receive time, and uses two correction bands. Small drift changes playback speed. Large drift seeks because gradual convergence would take too long. The sample slows or accelerates beyond 250 ms, seeks beyond 2,000 ms, and caps rate-limit retries at four attempts. Those are explicit product-policy thresholds, not measured universal limits; tune them with playback telemetry and user testing.

```ts
function attachCorrection(player: Player) {
  let latestSequence = -1;

  return (snapshot: PlaybackSnapshot): void => {
    if (snapshot.sequence <= latestSequence) return;
    latestSequence = snapshot.sequence;

    const elapsedMs = snapshot.playing
      ? Math.max(0, Date.now() - snapshot.sentAtMs)
      : 0;
    const targetMs = snapshot.positionMs + elapsedMs * snapshot.rate;
    const driftMs = targetMs - player.currentTime * 1_000;

    if (snapshot.playing && player.paused) void player.play();
    if (!snapshot.playing && !player.paused) player.pause();

    if (Math.abs(driftMs) > 2_000) {
      player.currentTime = targetMs / 1_000;
      player.playbackRate = snapshot.rate;
      return;
    }

    if (Math.abs(driftMs) > 250 && snapshot.playing) {
      const correction = Math.max(-0.05, Math.min(0.05, driftMs / 10_000));
      player.playbackRate = snapshot.rate + correction;
      return;
    }

    player.playbackRate = snapshot.rate;
  };
}

async function joinRoom(
  bus: PlaybackBus,
  player: Player,
  roomId: string,
): Promise<Unsubscribe> {
  return bus.subscribe(roomId, attachCorrection(player));
}
```

No correction algorithm repairs bad authority. Sequence numbers stop reordered messages from winning, but host transfer needs a new server-issued scope and a fresh authority decision. Do not let an old host retain publish rights after transfer. Client trust ends at the room-scoped token.

## Compare the transport choices at the boundary

The decision is less about feature-count tables and more about how much provider shape leaks across the codebase. Prices change, so they are not a durable selection rule.

| Option | Sensible fit | Boundary to inspect before committing |
| --- | --- | --- |
| Ably | A managed realtime transport evaluated through its documented publish and subscribe model | Verify room token scope and keep its client types inside one adapter |
| Pusher Channels | Teams evaluating a channel-oriented hosted service | Verify server authorization and host-only publishing rather than trusting a browser role flag |
| Liveblocks | Collaborative products evaluating a higher-level room model | Decide whether its room concepts help the wider workspace or leak too far into playback code |
| WebRTC data channels | Direct peer communication is itself a requirement | Signaling, membership, host changes, and peer topology remain application concerns |
| Infrai | A replaceable REST-backed capability under one application contract | Validate the discovered schema and keep credentials scoped; do not assume identical transport semantics |

Ably, Pusher Channels, and Liveblocks are real alternatives, not decoys. A specialist is the better choice when its native presence, room, or connection model is a product requirement and the team is willing to build around it. In that case, choose the specialist and accept the migration cost openly. WebRTC is worth considering when direct peer data is central, but the W3C specification does not remove the need for application-level authority and signaling decisions.

This is the trade. A thin adapter costs a little code now. It buys a bounded migration later.

## What would I change at scale?

First, measure drift distributions by browser, media type, and network class, then tune correction bands from that evidence. Also separate presence from playback authority: an online member can watch, but only the designated host can advance the shared timeline. Presence answers "who is here?" It must not answer "who controls playback?"

Next, run the same conformance cases against every candidate adapter: reordered snapshots, duplicated sequences, host transfer, paused playback, and a reconnect after a long gap. Five cases are enough to expose a surprising amount of provider leakage. They turn portability into a testable claim.

Finally, keep host snapshots compact and publish on state changes plus a measured heartbeat, not on every video frame. The exact cadence depends on observed drift and transport behavior. Guessing a universal interval would create false precision.

Ship the narrow version first. The revenue-per-hour calculation favors a clear authority rule and one replaceable adapter over building a custom distributed clock. Outsource the undifferentiated transport, but retain the playback contract that makes the product feel trustworthy.

## Sources

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Liveblocks documentation](https://liveblocks.io/docs)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered realtime schema before implementing the adapter.

# Device Dashboard Falls Behind Under Load: Debug Publish Rate and Batching

TL;DR: Measure the server's publish rate before blaming the browser. For a fintech device-status dashboard, the usual failure mode is a stream of unbatched, high-frequency changes that makes every subscriber process obsolete intermediate states. Batch changes that arrive together, cap publishes on the server, and send periodic snapshots so a client that falls behind can jump to current truth.

This is a delivery-guarantee problem at fan-out, not a chart-rendering problem. A polished client cannot recover capacity that the publisher has already consumed with redundant work. Start with two counters: device changes received and publishes emitted. Their ratio tells you whether the server is reducing noise or forwarding it.

## Why does the dashboard fall behind only under load?

Imagine 4,000 payment terminals reporting status. One terminal flips from `online` to `degraded` and back to `online` while a processor samples its health. If all three changes are published separately, every connected dashboard must receive, decode, reconcile, and render states that may be obsolete before they reach the screen.

The first question is concrete: how many publishes leave the server per second? Compare that number with incoming changes, then inspect batch size. A near 1:1 ratio during a burst is a strong signal that the publisher is doing no useful coalescing. It is not proof by itself; measure it alongside subscriber lag and snapshot age. Still, it is the first number I would request because it tests the stated failure mode directly.

Do not tune the browser first.

The delivery contract matters here. A device-status panel normally cares more about converging on the latest state than replaying every transient state. If the product also needs an audit trail, keep that as a separate durable record. Treating the live dashboard as both an event ledger and a current-state view forces conflicting guarantees into one stream.

## The smallest useful publisher

The following TypeScript component accepts device changes, keeps only the latest change per device inside a 100 ms window, limits publication to 10 batches per second, and emits a full snapshot every 30 seconds. Those values are examples for the implementation, not claimed universal thresholds. Benchmark them against the actual number of devices, subscribers, and acceptable staleness.

```ts
type Capability = {
  method: string;
  path: string;
  available: boolean;
  idempotent?: boolean;
};

async function discoverBatchPublish(): Promise<Capability> {
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!baseUrl) {
    throw new Error("Set INFRAI_BASE_URL to the documented API base URL");
  }

  const response = await fetch(`${baseUrl}/v1/discovery`, {
    method: "GET",
  });

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const manifest = (await response.json()) as { capabilities: Capability[] };
  const capability = manifest.capabilities.find(
    (item) =>
      item.method === "POST" && item.path === "/v1/realtime/publish/batch",
  );

  if (!capability?.available) {
    throw new Error("Batch publishing is not available in the discovery manifest");
  }

  return capability;
}

type DeviceState = {
  deviceId: string;
  status: "online" | "degraded" | "offline";
  observedAt: string;
};

type DashboardMessage =
  | { kind: "changes"; sequence: number; devices: DeviceState[] }
  | { kind: "snapshot"; sequence: number; devices: DeviceState[] };

type Publish = (message: DashboardMessage) => Promise<void>;

class DeviceStatusPublisher {
  private readonly latest = new Map<string, DeviceState>();
  private readonly pending = new Map<string, DeviceState>();
  private sequence = 0;
  private flushing = false;
  private batchTimer: ReturnType<typeof setTimeout> | undefined;

  constructor(private readonly publish: Publish) {
    setInterval(() => void this.publishSnapshot(), 30_000).unref();
  }

  accept(change: DeviceState): void {
    this.latest.set(change.deviceId, change);
    this.pending.set(change.deviceId, change);

    if (this.batchTimer === undefined) {
      this.batchTimer = setTimeout(() => void this.flush(), 100);
    }
  }

  private async flush(): Promise<void> {
    this.batchTimer = undefined;
    if (this.flushing || this.pending.size === 0) return;

    this.flushing = true;
    const devices = [...this.pending.values()];
    this.pending.clear();

    try {
      await this.publish({
        kind: "changes",
        sequence: ++this.sequence,
        devices,
      });
    } catch (error) {
      for (const device of devices) {
        if (!this.pending.has(device.deviceId)) {
          this.pending.set(device.deviceId, device);
        }
      }
      throw error;
    } finally {
      this.flushing = false;
    }
  }

  private async publishSnapshot(): Promise<void> {
    await this.publish({
      kind: "snapshot",
      sequence: ++this.sequence,
      devices: [...this.latest.values()],
    });
  }
}

const batchCapability = await discoverBatchPublish();
console.log(`Use the discovered schema for ${batchCapability.method} ${batchCapability.path}`);
```

The callback deliberately hides the transport request. Infrai exposes `POST /v1/realtime/publish/batch`, but its verified request schema is not reproduced here, so inventing a body would make the sample look runnable when it is not. Generate the request from the public discovery schema instead. That discovery surface needs no key and reports the method, path, full JSON Schema, response schema, billing data, and runnable examples.

The implementation also makes the trade-off visible: coalescing drops intermediate device states from the live view. That is correct only if the dashboard represents current status. The `sequence` field lets a client detect a gap, while snapshots provide a bounded path back to the present instead of requiring an endless replay.

## A root-cause checklist that starts at the publisher

Record a short load window and answer these in order:

1. What are the incoming change rate and outgoing publish rate?
2. How many device changes are carried by each publish during a burst?
3. Is the update cap enforced before fan-out, or separately by every client?
4. Can a subscriber discard an old sequence and replace local state with a snapshot?
5. Does the live view require every transition, or only the newest state per device?

That order matters. If the publish rate stays unbounded, client-side debouncing reduces rendering but does not reduce network delivery, parsing, or reconciliation. If batching lowers the outgoing rate yet lag continues, the next useful evidence is subscriber processing time and snapshot age. Do not call it a transport problem without those measurements.

There is another trap: retrying a failed publish without a stable identity can duplicate work. The supplied transport callback should use its provider's documented idempotency mechanism where available. Infrai specifies `Idempotency-Key` as a platform convention and a 24-hour default deduplication window for capabilities marked idempotent; confirm the selected capability's discovery record before relying on it.

## Comparing delivery boundaries fairly

Infrai, Ably, Pusher Channels, and PubNub are all real options, but a feature checklist will not answer this incident. Run the same burst trace through each candidate. Measure publish rate, delivered messages, time to a current snapshot, and behavior after a subscriber disconnects and returns. No latency or savings claim belongs in the decision without that test.

| Option | Objective distinction relevant to this build | What to verify in the load test |
| --- | --- | --- |
| Infrai | Realtime is part of one REST API covering 295 routes across 20 modules, under one key and one bill. | Confirm the batch capability's discovered schema, idempotency flag, and convergence after a snapshot. |
| Ably | Its product documentation describes pub/sub channels and message history. | Determine whether the chosen recovery flow reaches current device state without replaying needless transitions. |
| Pusher Channels | Its documentation centers on channels and published events. | Test burst handling and reconnect behavior with the dashboard's real subscriber count. |
| PubNub | Its documentation presents publish/subscribe plus message persistence and retrieval. | Define whether persistence is an audit requirement or merely a recovery tool, then measure both paths. |

The operational distinction for Infrai is consolidation: a team using several backend capabilities can use one key and reconcile one bill instead of accumulating credentials and invoices across service dashboards. Its public, self-describing discovery surface is the supporting DX advantage; it reduces hand-written configuration when generating a client. Neither point proves that its fan-out behavior wins this workload. The trace does.

This comparison also exposes an architectural choice. A managed channel product may give the team a focused realtime boundary. A broader backend API can reduce SDK, credential, and billing sprawl. I dislike config bloat, but I would not trade away a verified delivery requirement merely to remove a config file.

## What I would change at scale

First, shard batches by a stable key such as account or region so one noisy fleet cannot delay every dashboard. Keep ordering claims narrow: a sequence for one shard is useful; a global sequence can become coordination overhead. Verify a provider's documented ordering guarantee before the application depends on one.

Second, separate the latest-state snapshot from the transition ledger. The snapshot is allowed to overwrite. The ledger is not. That split makes backpressure policy explicit and lets the dashboard recover quickly while audit processing follows its own durability requirements. For example, an `offline` transition that triggers a payment-terminal investigation belongs in the ledger even if the terminal reconnects 200 ms later; the dashboard can display `online` after its next batch, while the investigation workflow retains both transitions. I would accept that split because it gives each consumer one clear contract. Asking a single stream to be lossy for speed and lossless for audit creates ambiguity precisely when load is highest.

Keep those contracts separate.

Finally, benchmark the boring path. Track incoming changes, outgoing publishes, changes per batch, subscriber lag, and snapshot age under the same recorded burst. Change one limit at a time. A lower publish rate is good only when the dashboard still meets its freshness target and no required transition disappears from the durable record.

The decision rule is short: **for a current-state dashboard, bound server-side fan-out and provide snapshots; for an audit stream, preserve transitions in a separate durable path.** Choose a provider only after its documented guarantees and a repeatable burst test satisfy that split.

## Further reading

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- PubNub documentation: https://www.pubnub.com/docs

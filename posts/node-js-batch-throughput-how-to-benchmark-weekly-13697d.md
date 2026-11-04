# Node.js Batch Throughput: How to Benchmark Weekly PDF Report Generation for Gaming Finance

For a game studio's weekly finance report, put PDF generation behind a scheduled batch boundary. Freeze the reporting inputs, describe the merge and split work explicitly, and benchmark the completed bundle rather than a single rendering call. TL;DR: choose a report generation API by how quickly it finishes a correct weekly batch at bounded concurrency. A fast first response tells the finance team very little about when its packets will be ready.

The concrete job here is assembling studio-wide finance packets from game-level reports, then splitting those packets into the bundles each reviewer needs. The examples below use synthetic game names and page counts. They are workload fixtures, not measured production results.

Keep the scheduler boring. It should submit an immutable batch description for a reporting period; workers should render, assemble, verify, and publish the results. The API is one stage in that pipeline. Buying an endpoint does not define the batch.

## How should a scheduled PDF report generation API handle weekly finance bundles?

Start with the output contract. A finance reviewer needs the correct games, reporting period, ordering, and page boundaries. A successful HTTP response cannot establish those properties. Neither can a count of generated files. The throughput unit should therefore be verified reviewer bundles per elapsed batch time, with failed and retried work reported alongside it.

Count completed work.

The constraint that changes the design is fan-out: one source report can appear in multiple review packets. In the proposed workload, a studio packet combines three game reports, while a second packet contains only two. Rendering separately for every destination repeats source work. Rendering each immutable source once and assembling afterward creates an opportunity to reuse it, but adds artifact storage, a manifest, and another verification boundary. That trade-off deserves measurement. It is not automatically a win for tiny batches.

Treat PDF as a document format, not a filename extension.

ISO 32000-2 is the relevant format specification; it does not establish your renderer's speed, your finance controls, or your assembly implementation's correctness. A merge stage must understand PDF structure. Concatenating byte arrays isn't a substitute.

## 2. Make the smallest bundle contract executable

First, describe the assembly stage without adding scheduler settings, credentials, or a vendor SDK. This TypeScript fixture plans one merged studio packet and one subset packet. It also computes destination page ranges so an assembly adapter can check its own output. The page counts stand in for metadata collected after rendering, and every range is one-based and inclusive.

Run this fixture in the project's TypeScript runner. It has no external imports. The assertions are the acceptance criteria: nine pages in the studio packet, five in the subset, and the subset's second source starting at destination page four.

```ts
type Source = { id: string; pages: number };
type Packet = { id: string; sourceIds: string[] };
type Segment = { sourceId: string; first: number; last: number };
type Plan = { id: string; pages: number; segments: Segment[] };

function planPackets(sources: Source[], packets: Packet[]): Plan[] {
  const index = new Map<string, Source>();
  for (const source of sources) {
    if (index.has(source.id)) throw new Error('Duplicate source');
    if (!Number.isSafeInteger(source.pages) || source.pages < 1) {
      throw new Error('Invalid page count');
    }
    index.set(source.id, source);
  }
  const packetIds = new Set<string>();
  return packets.map(packet => {
    if (packetIds.has(packet.id)) throw new Error('Duplicate packet');
    packetIds.add(packet.id);
    if (packet.sourceIds.length === 0) throw new Error('Empty packet');
    let pages = 0;
    const seen = new Set<string>();
    const segments = packet.sourceIds.map(sourceId => {
      if (seen.has(sourceId)) throw new Error('Repeated source');
      seen.add(sourceId);
      const source = index.get(sourceId);
      if (!source) throw new Error('Missing source');
      const first = pages + 1;
      pages += source.pages;
      if (!Number.isSafeInteger(pages)) throw new Error('Packet too large');
      return { sourceId, first, last: pages };
    });
    return { id: packet.id, pages, segments };
  });
}

const sources: Source[] = [
  { id: 'orbit-ledger', pages: 3 },
  { id: 'harbor-ledger', pages: 4 },
  { id: 'rally-ledger', pages: 2 }
];
const plans = planPackets(sources, [
  { id: 'studio-review', sourceIds: sources.map(source => source.id) },
  { id: 'portfolio-review', sourceIds: ['orbit-ledger', 'rally-ledger'] }
]);

if (plans[0].pages !== 9 || plans[1].pages !== 5 ||
    plans[1].segments[1].first !== 4) {
  throw new Error('Unexpected assembly plan');
}
console.log(JSON.stringify(plans, null, 2));
```

This is an executable planning boundary, not a PDF renderer or merger.

Connect its source IDs to immutable rendered artifacts in the worker. Keep that distinction visible in the SDK: planning validates the request; assembly produces bytes; verification checks the result. A single method that silently performs all three makes a failed batch harder to explain.

For splitting an existing combined packet, use the corresponding source ranges from that packet's plan. Do not reuse destination ranges from a different packet. In this fixture, the rally report occupies pages 8–9 in the studio packet but pages 4–5 in the portfolio packet. That is a concrete off-by-context trap. To test it, ask the assembly adapter to extract rally from the studio packet, then verify the game identifier on each extracted page as well as the expected count of 2 pages. An implementation that extracts pages 4–5 instead still returns 2 pages, so a count-only assertion would approve the wrong game. Add a separate assertion for packet order, too: the portfolio packet must place orbit before rally even if their render jobs finish in the opposite order. These are proposed tests against synthetic input, not a report of an incident.

Page totals alone won't catch this.

## 3. Benchmark the batch before choosing an endpoint

Next, exercise the complete worker through an injected operation. The small harness below limits concurrent jobs and records elapsed time across the entire invocation. Paste it after the planning fixture. Its local demonstration only checks planned page totals; substitute the actual render, merge or split, and verification operation before interpreting the output as PDF throughput.

```ts
async function benchmark<T>(
  jobs: T[],
  concurrency: number,
  complete: (job: T) => Promise<void>
) {
  if (!Number.isInteger(concurrency) || concurrency < 1) {
    throw new Error('Invalid concurrency');
  }
  let next = 0;
  let completed = 0;
  let failed = 0;
  const started = performance.now();
  async function worker() {
    while (next < jobs.length) {
      const job = jobs[next++];
      try {
        await complete(job);
        completed++;
      } catch {
        failed++;
      }
    }
  }
  await Promise.all(
    Array.from({ length: Math.min(concurrency, jobs.length) }, worker)
  );
  const elapsedMs = performance.now() - started;
  return {
    completed, failed, elapsedMs,
    completedPerSecond: elapsedMs > 0 ? completed * 1000 / elapsedMs : 0
  };
}

async function demonstrate() {
  const result = await benchmark(plans, 2, async plan => {
    const total = plan.segments.reduce(
      (sum, segment) => sum + segment.last - segment.first + 1, 0
    );
    if (total !== plan.pages) throw new Error('Page total mismatch');
  });
  console.log(result);
}
void demonstrate();
```

Use the same frozen inputs for each candidate implementation. Include a small packet, a large packet, and a packet assembled from already-rendered sources. Choose their sizes from the intended workload; the three synthetic sources above establish no capacity claim. Measure end-to-end batch completion, peak worker memory, temporary storage, and retries. Keep first-call latency as a separate developer experience measurement. It matters during integration, but it answers a different question.

Start the concurrency sweep at 1 worker, then try 2 and 4. Those are proposed test settings, not a claim about safe service limits. Stop increasing concurrency when verified throughput stops improving or resource use exceeds the worker budget.

Don't discard failures.

Record errors rather than dropping failed jobs from the denominator. Also preserve job-level failure details in the real worker; this harness counts failures only to keep the example focused.

A renderer-only benchmark is especially misleading for split-heavy work. The assembly, validation, upload, and publication stages still need time. If those stages dominate, a faster rendering API may barely change when finance receives its weekly packet.

## 4. What I would change at scale

Move the in-memory job cursor into durable job storage. Give each batch a stable identity derived from its reporting period and immutable input revision. Record artifact references and stage completion so a retry can resume without silently mixing revisions. Run the weekly schedule with an explicit time zone and a catch-up policy; a delayed run should still identify which reporting period it belongs to.

Publication needs its own boundary. Stage every required packet, validate the manifest and page mappings, then publish a completed batch reference. Keep access checks on both source artifacts and reviewer bundles. A subset packet can be structurally valid while containing the wrong game's financial pages. Test that failure deliberately.

There is an operational price for reuse: retained artifacts, cleanup rules, and more state to inspect. Independent assembly also means another deployable component and another compatibility test. Keep it only if the measured batch benefit or review requirements justify that complexity. Prefer a small versioned manifest over a wall of tuning flags.

The decision rule is concrete: select the approach that completes the representative weekly gaming finance batch within its delivery window, preserves verified merge and split boundaries, and stays inside the agreed worker budget. If two implementations meet those conditions, compare the integration surface and recovery procedure. Keep the benchmark fixture with the worker so the next change can be judged on the same work.

## References

ISO 32000-2 supplies the PDF format reference. The workload, contracts, and benchmark settings in this note are proposed engineering methods; they are not published performance measurements or claims of finance compliance.

## Further reading

- ISO 32000-2 — Portable Document Format: https://www.iso.org/standard/75839.html

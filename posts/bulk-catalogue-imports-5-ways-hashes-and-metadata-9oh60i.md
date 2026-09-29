# Bulk Catalogue Imports: 5 Ways Hashes and Metadata Filter Supplier Images

Decision rule: use supplier metadata to find candidates, use a SHA-256 content hash to prove byte-for-byte equality, and generate responsive thumbnails only after that proof. Keep perceptual matching out of the automatic path unless a human can review uncertain pairs.

| Pre-processing check | Bandwidth effect | Quality risk | Best use |
|---|---:|---:|---|
| Supplier asset ID plus revision | Can avoid a repeated fetch | High if the supplier mutates assets in place | Candidate lookup |
| File size plus dimensions | Shrinks the comparison set | High if treated as identity | Cheap triage |
| SHA-256 of source bytes | Requires reading the upload once | Low for exact duplicates | Automatic reuse |
| Hash of normalized pixels | Requires decoding | Medium across decoder or color-policy changes | Controlled migration |
| Perceptual hash | Requires decoding | Threshold-dependent | Review queue |

For a B2B SaaS catalogue importer, my default is the third row backed by the first two: reject an already recorded supplier revision before transfer when that contract is trustworthy; otherwise stream the body once, validate it, hash it, and reuse an existing derivative set on an exact match. This protects thumbnail quality without paying the CPU and storage cost of resizing the same bytes again. It also keeps config small.

No single fingerprint solves every kind of duplicate. The useful split is exact identity versus visual similarity. Mixing those jobs creates a pipeline that looks clever in a demo and silently merges different product photography in production.

## 1. How should Node.js dedupe supplier images by metadata and hash before processing?

Treat deduplication as a sequence of increasingly expensive claims. Supplier metadata makes the first claim: this upload might have been seen. A cryptographic digest makes the stronger claim: these source bytes are identical. Only the second claim is strong enough for automatic derivative reuse when the supplier contract is unknown.

The order matters. Start with stable business metadata such as `supplierId`, `supplierAssetId`, and an explicit source revision. Use declared media type, byte length, width, and height as filters, never as identity. Two distinct catalogue photos can share all four. EXIF timestamps and filenames are worse keys because export tools rewrite them and suppliers reuse names.

Then compute SHA-256 while the request is being written to a bounded temporary file. Don't read the body into an unbounded buffer. Once the stream ends, inspect the actual file signature and dimensions rather than trusting `Content-Type`; the MDN image-format guide is useful for deciding which formats the thumbnail service will accept, while the OWASP upload guidance explains why extension and MIME checks alone are insufficient.

If `(supplierId, supplierAssetId, revision)` already maps to a committed source digest, the importer can reuse that result under a documented supplier contract. If the metadata lookup only finds candidates, compare the new digest with their stored digests. An exact hit reuses the canonical source and its thumbnail manifest. A miss creates a new source record and enters processing. Short path. Clear proof.

## 2. Filter candidates with metadata, but never promote a hint into identity

Metadata earns its place because hashes cannot save inbound bandwidth by themselves: the service must receive bytes before it can hash them. A trustworthy upstream asset ID and immutable revision can prevent a fetch altogether. An ETag can also support conditional retrieval, but HTTP defines it as an opaque validator for a selected representation, not a portable content checksum. Store it with the supplier and URL scope that gave it meaning.

For a bulk import, I would persist a compact intake record: supplier scope, upstream asset ID, upstream revision or ETag, declared byte length, detected format, width, height, source SHA-256, and ingestion policy version. The policy version is easy to skip — and painful to reconstruct later — because a change to orientation handling, color conversion, or accepted formats changes what "ready for thumbnails" means even when the original bytes do not change.

This is also where quality versus bandwidth becomes concrete. A metadata hit under a strong immutable-revision contract saves the transfer. A weak metadata hit saves only database search work. A raw-byte hash saves duplicate decoding and resizing, yet two JPEG encodes of the same pixels will still produce different digests. That's acceptable for the automatic lane. False negatives cost work; false positives can attach the wrong product image to a catalogue item. The asymmetry is brutal.

I'm not sure any supplier field is immutable until its contract says so and a replay test confirms it. Your mileage may vary across feeds. Benchmark the hit rate separately for metadata skips, exact hashes, and review candidates; one blended "dedupe rate" hides which layer is actually doing useful work.

Hash once.

## 3. How can an Express handler hash an image while it streams?

The handler below uses an injected metadata probe and repository so the HTTP boundary stays testable. It spools to a unique file, hashes during the same pass, enforces an illustrative 25 MiB policy, verifies the detected format, and queues thumbnail work only after the source record wins a uniqueness race. All code paths remove the temporary file.

```ts
import { createHash, randomUUID } from "node:crypto";
import { createWriteStream } from "node:fs";
import { unlink } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { Transform } from "node:stream";
import { pipeline } from "node:stream/promises";
import type { Request, Response } from "express";

type ImageInfo = {
  format: "jpeg" | "png" | "webp" | "avif";
  width: number;
  height: number;
};

type SourceRecord = {
  id: string;
  sha256: string;
  thumbnailManifestId?: string;
};

type Dependencies = {
  probeImage(path: string): Promise<ImageInfo>;
  findByHash(sha256: string): Promise<SourceRecord | undefined>;
  insertSource(input: {
    sha256: string;
    byteLength: number;
    info: ImageInfo;
    supplierId: string;
    supplierAssetId: string;
    revision?: string;
  }): Promise<{ record: SourceRecord; inserted: boolean }>;
  enqueueThumbnails(sourceId: string): Promise<void>;
};

class IntakeError extends Error {
  constructor(
    readonly status: 413 | 415 | 422,
    message: string,
  ) {
    super(message);
  }
}

class HashAndLimit extends Transform {
  readonly hash = createHash("sha256");
  bytes = 0;

  constructor(private readonly maxBytes: number) {
    super();
  }

  override _transform(
    chunk: Buffer,
    _encoding: BufferEncoding,
    callback: (error?: Error | null, data?: Buffer) => void,
  ): void {
    this.bytes += chunk.length;
    if (this.bytes > this.maxBytes) {
      callback(new IntakeError(413, "Image exceeds the upload limit"));
      return;
    }
    this.hash.update(chunk);
    callback(null, chunk);
  }
}

export function createSupplierImageHandler(deps: Dependencies) {
  return async function supplierImageHandler(req: Request, res: Response) {
    const supplierId = String(req.params.supplierId ?? "");
    const supplierAssetId = String(req.header("x-supplier-asset-id") ?? "");
    const revision = req.header("x-supplier-revision") ?? undefined;

    if (!supplierId || !supplierAssetId) {
      res.status(422).json({ error: "Supplier identity is required" });
      return;
    }

    const tempPath = join(tmpdir(), `catalogue-${randomUUID()}.upload`);
    const meter = new HashAndLimit(25 * 1024 * 1024);

    try {
      await pipeline(req, meter, createWriteStream(tempPath, { flags: "wx" }));
      const sha256 = meter.hash.digest("hex");
      const info = await deps.probeImage(tempPath);

      if (info.width < 1 || info.height < 1) {
        throw new IntakeError(422, "Image dimensions are invalid");
      }

      const duplicate = await deps.findByHash(sha256);
      if (duplicate) {
        res.status(200).json({
          status: "duplicate",
          sourceId: duplicate.id,
          thumbnailManifestId: duplicate.thumbnailManifestId,
        });
        return;
      }

      const result = await deps.insertSource({
        sha256,
        byteLength: meter.bytes,
        info,
        supplierId,
        supplierAssetId,
        revision,
      });

      if (result.inserted) {
        await deps.enqueueThumbnails(result.record.id);
      }

      res.status(result.inserted ? 202 : 200).json({
        status: result.inserted ? "accepted" : "duplicate",
        sourceId: result.record.id,
      });
    } catch (error) {
      const status = error instanceof IntakeError ? error.status : 422;
      const message = error instanceof Error ? error.message : "Invalid image upload";
      res.status(status).json({ error: message });
    } finally {
      await unlink(tempPath).catch(() => undefined);
    }
  };
}
```

Put a unique database constraint on `sha256`, because two Express workers can finish hashing the same image at the same time. `insertSource` must return the winning row when that constraint conflicts. The queue call belongs after the insert result; an idempotent job key based on the source ID and thumbnail-policy version then prevents duplicate derivative work without pretending the HTTP process owns distributed coordination.

The example returns `202` for newly accepted work, `200` for a duplicate, `413` for the configured byte ceiling, and `422` for invalid intake metadata or image structure. These are API choices, not universal constants. Measure event-loop delay, temporary-disk throughput, bytes accepted, exact-hash hit rate, probe duration, queue latency, and resize duration. I care most about time from the first byte to the dedupe decision, because a fast resize benchmark can conceal a slow or memory-hungry admission path.

Quality can wait.

## 4. Separate source identity from thumbnail quality

A source hash should identify the uploaded representation. It should not encode resize width, crop mode, encoder quality, or output format. Put those values in a versioned thumbnail policy and key each derivative by `(sourceId, policyVersion, targetName)`. That division lets a quality change regenerate thumbnails from one canonical source without weakening deduplication.
Responsive output is a policy decision. Keep a small named set of widths derived from actual UI slots, preserve aspect ratio unless the catalogue contract explicitly requests a crop, and let the browser choose among variants with `srcset` and `sizes`. Format support differs by browser and image type, so negotiate output formats at delivery or emit the formats your supported clients can decode. Don't silently turn a transparent product cutout into an opaque thumbnail. Quality needs a fixture suite, not a favorite encoder setting. Include high-frequency fabric, fine text, transparent edges, gradients, embedded orientation, wide-gamut inputs, and an image near the pixel-count ceiling. Compare file size and a documented visual-quality metric, then inspect a fixed sample. Automated scores are useful for regression detection, but they don't know that a faint logo or a one-pixel product edge matters to the merchant. Keep the original bytes when policy permits. A normalized-pixel hash is not a replacement for them: decoder versions, orientation rules, animation-frame selection, and color management can alter the byte sequence presented to the hash. If normalized hashes are used, record the exact normalization policy and decoder version beside the digest. Config bloat starts when those hidden choices leak into every worker; one versioned policy object is enough.

## 5. When should perceptual matching beat exact hashes for supplier images?

Use perceptual matching when the business wants to find resizes, recompressions, watermarked copies, or slightly edited shots, and when a false match can be reviewed or reversed. It is the runner-up for catalogue cleanup, not the default admission gate. A perceptual hash produces a distance, so the threshold must be calibrated against labelled pairs from the actual catalogue. Product variants with tiny color or label changes are precisely where a generic threshold becomes dangerous.

The catch is latency and ambiguity. Perceptual matching requires a decode, an index strategy, threshold tuning, and an audit trail. It is not suitable when uploads must be accepted automatically and a mistaken merge could attach imagery to the wrong SKU. Stick with exact SHA-256 reuse in that path. Send near matches to a separate review queue containing both previews, their metadata differences, the distance, and the eventual reviewer decision.

Normalized-pixel hashing is the middle option when the organization controls every decoder and deliberately treats metadata-only or encoding-only differences as identical. It can be better during a controlled migration from one encoder to another. It is a poor cross-system contract when color conversion and orientation behavior are not pinned.

The final operating rule is plain: metadata may skip work under a verified supplier contract, exact hashes may reuse work automatically, and similarity scores may suggest work to a reviewer. That boundary favors correctness first while still cutting repeated transfers and thumbnail jobs where the evidence is strong.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/ETag
- https://nodejs.org/api/crypto.html#class-hash
- https://nodejs.org/api/stream.html#streampipelinesource-transforms-destination
- https://expressjs.com/en/guide/using-middleware.html
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html

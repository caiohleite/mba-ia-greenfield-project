---
libs:
  "@nestjs/bullmq":
    version: "^11.0.0"
    context7_id: "/nestjs/bull"
    fetched_at: "2026-11-26T23:22:35-03:00"
  bullmq:
    version: "^5.x"
    context7_id: "/websites/bullmq_io"
    fetched_at: "2026-11-26T23:22:35-03:00"
  "@aws-sdk/client-s3":
    version: "^3.x"
    context7_id: "/aws/aws-sdk-js-v3"
    fetched_at: "2026-11-26T23:22:35-03:00"
  "@aws-sdk/s3-request-presigner":
    version: "^3.x"
    context7_id: "/aws/aws-sdk-js-v3"
    fetched_at: "2026-11-26T23:22:35-03:00"
sources_mtime:
  docs/decisions/technical-decisions-phase-03-videos.md: "2026-11-26T23:21:36-03:00"
---

# phase-03-videos — Library References

Distilled official documentation for newly decided Phase 03 libraries (`phase-03-videos/TD-01` and `phase-03-videos/TD-02`), fetched via Context7 MCP (`context7_context7`).

### @nestjs/bullmq

**Source:** `/nestjs/bull` (Context7) — Official NestJS integration for BullMQ (`@nestjs/bullmq ^11.x`, compatible with `@nestjs/common ^11.0.1`). Governs `phase-03-videos/TD-01`, `TD-04`, and `TD-08`.

#### Global Redis Connection via `BullModule.forRootAsync`

Pairs directly with `@nestjs/config` `registerAs` factories (`phase-01-configuracao-base/TD-03`):

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigType } from '@nestjs/config';
import { BullModule } from '@nestjs/bullmq';
import queueConfig from './config/queue.config';

@Module({
  imports: [
    BullModule.forRootAsync({
      imports: [ConfigModule],
      inject: [queueConfig.KEY],
      useFactory: (cfg: ConfigType<typeof queueConfig>) => ({
        connection: {
          host: cfg.host,
          port: cfg.port,
        },
      }),
    }),
  ],
})
export class AppModule {}
```

#### Queue Producer Registration & Injection (`BullModule.registerQueue` + `@InjectQueue`)

Register only the queue in producer modules (`VideosModule` in `AppModule`) without registering any `@Processor` provider there, keeping HTTP API containers producer-only (`phase-03-videos/TD-04`):

```typescript
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class VideosService {
  constructor(
    @InjectQueue('video-processing')
    private readonly videoQueue: Queue<{ videoId: string }>,
  ) {}

  async enqueueProcessing(videoId: string): Promise<void> {
    await this.videoQueue.add(
      'process-video',
      { videoId },
      {
        jobId: videoId,
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 },
      },
    );
  }
}
```

#### Worker Consumer (`@Processor`, `WorkerHost`, `@OnWorkerEvent`)

In `WorkerModule` (booted via `NestFactory.createApplicationContext(WorkerModule)`), consumer classes extend `WorkerHost` and implement `process(job: Job)`:

```typescript
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Job, UnrecoverableError } from 'bullmq';

@Processor('video-processing')
export class VideoProcessor extends WorkerHost {
  async process(job: Job<{ videoId: string }>): Promise<void> {
    // Extract metadata + generate thumbnail; throw UnrecoverableError for invalid media
  }

  @OnWorkerEvent('failed')
  async onFailed(job: Job<{ videoId: string }> | undefined, err: Error): Promise<void> {
    if (!job) return;
    const maxAttempts = job.opts.attempts ?? 1;
    const isTerminal =
      err instanceof UnrecoverableError || job.attemptsMade >= maxAttempts;
    if (isTerminal) {
      // Persist status = 'failed' and error_reason on Video row
    }
  }
}
```

**Important runtime note from Context7:** `this.worker` getter on `WorkerHost` is populated during `onModuleInit` registration; access it only in `onApplicationBootstrap` or later, never inside the constructor. Listeners decorated with `@OnWorkerEvent` require singleton-scoped (default) processors.

---

### bullmq

**Source:** `/websites/bullmq_io` (Context7) — BullMQ v5 TypeScript/Node.js queue engine over Redis. Governs `phase-03-videos/TD-01` and `phase-03-videos/TD-08`.

#### Exponential Backoff Retries (`attempts` + `backoff`)

Without `attempts`, a failed job is not retried (`attempts: 1`). With `attempts: 3` and `{ type: 'exponential', delay: 1000 }`, BullMQ spaces retries at $2^{\text{attempt}-1} \times 1000\text{ ms}$ (1s, 2s, 4s):

```typescript
await queue.add(
  'process-video',
  { videoId },
  {
    jobId: videoId,
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
  },
);
```

#### Immediate Terminal Failure via `UnrecoverableError`

Throwing `UnrecoverableError` inside `WorkerHost.process()` bypasses remaining `attempts` and moves the job directly to the `failed` set, firing `@OnWorkerEvent('failed')` immediately:

```typescript
import { UnrecoverableError } from 'bullmq';

if (!hasVideoStream) {
  throw new UnrecoverableError('Uploaded file contains no valid video stream');
}
```

#### Idempotent Deduplication with Custom `jobId`

- Custom `jobId` must be unique per queue, **must not contain `:`**, and **must not consist solely of digits**. A standard UUIDv4 (`videoId`, e.g., `550e8400-e29b-41d4-a716-446655440000`) satisfies both rules natively.
- Adding a job with an already-existing `jobId` in the queue is ignored by BullMQ, preventing duplicate processing jobs on repeated `complete` retries.

---

### @aws-sdk/client-s3

**Source:** `/aws/aws-sdk-js-v3` (Context7) — Modular AWS SDK for JavaScript v3 S3 client. Governs `phase-03-videos/TD-02`, `TD-03`, `TD-05`, and `TD-07`.

#### S3Client Configuration for MinIO / S3 Compatibility (`forcePathStyle: true`)

MinIO inside Docker Compose (`http://minio:9000`) requires `forcePathStyle: true` so bucket names are placed in the URL path (`/streamtube/<key>`) rather than virtual-hosted subdomains (`streamtube.minio`):

```typescript
import {
  S3Client,
  HeadBucketCommand,
  CreateBucketCommand,
} from '@aws-sdk/client-s3';

const s3Client = new S3Client({
  endpoint: storageConfig.endpoint, // e.g. "http://minio:9000"
  region: storageConfig.region,     // e.g. "us-east-1"
  forcePathStyle: true,
  credentials: {
    accessKeyId: storageConfig.accessKey,
    secretAccessKey: storageConfig.secretKey,
  },
});
```

Destroy the client cleanly in `OnModuleDestroy` (`s3Client.destroy()`) so Jest integration/E2E runs release HTTP keep-alive sockets cleanly.

#### Multipart Upload Lifecycle (`Create` → `Complete` / `Abort`)

- `CreateMultipartUploadCommand({ Bucket, Key, ContentType })` returns `{ UploadId }`.
- `CompleteMultipartUploadCommand` **requires `MultipartUpload.Parts` sorted ascending by `PartNumber`**, each carrying `{ PartNumber, ETag }`:

```typescript
import {
  CreateMultipartUploadCommand,
  CompleteMultipartUploadCommand,
  AbortMultipartUploadCommand,
  CompletedPart,
} from '@aws-sdk/client-s3';

const sortedParts: CompletedPart[] = [...parts]
  .sort((a, b) => a.partNumber - b.partNumber)
  .map((p) => ({ PartNumber: p.partNumber, ETag: p.etag }));

await s3Client.send(
  new CompleteMultipartUploadCommand({
    Bucket: bucket,
    Key: key,
    UploadId: uploadId,
    MultipartUpload: { Parts: sortedParts },
  }),
);
```

- `AbortMultipartUploadCommand({ Bucket, Key, UploadId })` cleans up uploaded parts when a draft upload is cancelled.

#### Streaming Object Reads (`GetObjectCommand` with `Range` + Node `Readable`)

In Node.js, `response.Body` from `GetObjectCommand` is an unconsumed `SdkStream<Readable>`. Pass it directly to NestJS `StreamableFile` (or consume/destroy on client disconnect) to keep memory usage `O(1)` and avoid socket starvation:

```typescript
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { Readable } from 'node:stream';

const output = await s3Client.send(
  new GetObjectCommand({
    Bucket: bucket,
    Key: key,
    ...(rangeHeader ? { Range: rangeHeader } : {}),
  }),
);

const stream = output.Body as Readable;
```

---

### @aws-sdk/s3-request-presigner

**Source:** `/aws/aws-sdk-js-v3` (`packages/s3-request-presigner`, Context7) — SigV4 URL presigner for AWS SDK v3 commands. Governs `phase-03-videos/TD-02`, `TD-03`, and `TD-05`.

#### Presigning Multipart `UploadPartCommand` and Worker `GetObjectCommand`

`getSignedUrl(client, command, { expiresIn })` generates a time-bounded URL (`expiresIn` in seconds, default `900`). Because SigV4 signs the target `Host` header:
- Use the **public signer client** (`STORAGE_PUBLIC_ENDPOINT ?? STORAGE_ENDPOINT`, `forcePathStyle: true`) when signing client-facing `UploadPartCommand` URLs (`TD-02`, `TD-03`).
- Use the **internal client** (`STORAGE_ENDPOINT = http://minio:9000`, `forcePathStyle: true`) when signing short-lived `GetObjectCommand` URLs consumed by `ffprobe`/`ffmpeg` inside `video-worker` (`TD-05`).

```typescript
import { UploadPartCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const partUploadUrl = await getSignedUrl(
  publicSignerClient,
  new UploadPartCommand({
    Bucket: bucket,
    Key: key,
    UploadId: uploadId,
    PartNumber: partNumber,
  }),
  { expiresIn: presignTtlSeconds },
);

const internalProbeUrl = await getSignedUrl(
  internalS3Client,
  new GetObjectCommand({
    Bucket: bucket,
    Key: key,
  }),
  { expiresIn: 900 },
);
```

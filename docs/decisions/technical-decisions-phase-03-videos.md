---
scope_type: phase
related_phases: [3]
status: decided
date: 2026-11-26
scope_description: "Backend architecture for Phase 03: S3/MinIO object storage topology, background queue and standalone worker, 10 GiB presigned multipart upload with draft pre-registration, FFmpeg/ffprobe metadata and thumbnail extraction, unique short-code URLs, HTTP Range streaming, download, and video lifecycle state machine."
---

# Technical Decisions — Phase 03: Upload e Processamento de Vídeos

_Subprojects in scope:_

- `nestjs-project/` — primary backend subproject in scope; receives `StorageModule` (S3/MinIO), `VideosModule` (upload handshake, entity/migration, unique URL, HTTP Range streaming, download), `WorkerModule` (`video-worker` standalone container running BullMQ + FFmpeg/ffprobe), namespaced configs (`storage.config.ts`, `queue.config.ts`, `videos.config.ts`), and unit/integration/E2E test suites.
- `next-frontend/` — **no open decision in this document.** Video upload, channel studio, and playback UI screens belong to Phases 04–05 (`docs/project-plan.md`); `Cross-layer` TDs below fix the REST/OpenAPI contract (`openapi-docs-nestjs/TD-01..TD-02`, `next-frontend-openapi-typing/TD-01..TD-04`) and direct-to-storage presigned upload flow anticipated by `next-frontend-config-base/TD-03`.

---

## TD-01: Background Job Queue Technology

**Scope:** Backend

**Capability:** Serviço de processamento em segundo plano (filas)

**Context:** The C4 architecture marks the message queue as `TBD`, and Phase 03 requires asynchronous video processing decoupled from the HTTP request lifecycle. The chosen queue must integrate cleanly with NestJS 11 (`@nestjs/common ^11.0.1`), run as a real container in `nestjs-project/compose.yaml`, and support retries, exponential backoff, and event hooks.

**Options:**

### Option A: BullMQ + Redis (`@nestjs/bullmq` + `bullmq` + `redis:7-alpine`)
- Official NestJS queue integration (`@nestjs/bullmq ^11.x`) backed by `bullmq` and a `redis:7-alpine` service in Docker Compose. Producers inject `@InjectQueue('video-processing')` while consumers extend `WorkerHost` with `@Processor('video-processing')` and `@OnWorkerEvent()`.
- **Pros:** First-class NestJS 11 DI module (`BullModule.forRootAsync` pairs directly with `@nestjs/config` `registerAs` factories from `phase-01-configuracao-base/TD-03`). Built-in job deduplication (`jobId`), exponential backoff, `UnrecoverableError`, and lightweight `redis:7-alpine` footprint (~40 MB RAM) that can also serve distributed caching/throttling later.
- **Cons:** Adds Redis as a new stateful service in `compose.yaml` alongside PostgreSQL and MinIO. Requires `maxRetriesPerRequest: null` alignment on worker Redis connections (handled automatically by `@nestjs/bullmq`).

### Option B: RabbitMQ (`@nestjs/microservices` RMQ + `amqplib` + `rabbitmq:4-management-alpine`)
- AMQP 0-9-1 message broker consumed via NestJS microservice transport (`Transport.RMQ`) or community wrappers (`@golevelup/nestjs-rabbitmq`). Uses exchanges, routing keys, and dead-letter exchanges (DLX) for retry/failure routing.
- **Pros:** Mature message broker with flexible routing topologies (fan-out, topic exchanges) and native consumer acknowledgments (`ack`/`nack`). Decouples polyglot services cleanly if non-Node workers are introduced later.
- **Cons:** `@nestjs/microservices` RMQ lacks built-in delayed exponential retries — implementing `attempts` with backoff requires manual dead-letter exchange + TTL queue wiring. Heavier container footprint (~150–250 MB RAM) and steeper operational setup for a single work queue.

### Option C: PostgreSQL-backed Queue (`pg-boss` reusing existing `db` service)
- Stores job state inside PostgreSQL tables using `SKIP LOCKED` polling via the `pg-boss` Node.js library wrapped in a custom NestJS module. Requires zero new containers in `compose.yaml`.
- **Pros:** Reuses the existing `postgres:17` container with transactional job enqueueing alongside entity writes.
- **Cons:** Couples background polling load to the primary transactional database, lacks an official `@nestjs/*` package for NestJS 11, and diverges from the C4 architecture (`docs/diagrams/software-arch.mermaid`), which models `Message Queue` as a dedicated infrastructure container separate from `Database`.

**Recommendation:** **Option A (BullMQ + Redis)** — Official `@nestjs/bullmq` integration matches NestJS 11 and `@nestjs/config` seamlessly, delivers retries, exponential backoff, and `UnrecoverableError` out of the box without manual DLX plumbing (unlike Option B), and honors the dedicated queue container architecture without polling PostgreSQL (unlike Option C).

**Decision:** A (BullMQ + Redis via `@nestjs/bullmq` + `bullmq`)
**Libraries:** @nestjs/bullmq, bullmq

---

## TD-02: Object Storage SDK, Bucket Topology, and Endpoint Signing Strategy

**Scope:** Backend

**Capability:** Serviço de armazenamento de arquivos (vídeos e thumbnails)

**Context:** S3-compatible object storage (MinIO in local Docker Compose, Amazon S3 in production) is fixed by the project architecture, but Phase 03 must decide which client SDK to adopt, how buckets and object keys are organized for videos vs. thumbnails, and how internal container-to-container traffic (`http://minio:9000`) coexists with presigned URLs consumed by external clients.

**Options:**

### Option A: AWS SDK v3 (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`) with Single Bucket and Dual Endpoint Config
- Uses modular AWS SDK v3 configured with `forcePathStyle: true` against a single bucket (`streamtube`), partitioning assets via deterministic key prefixes (`videos/{channelId}/{videoId}/source.{ext}` and `thumbnails/{channelId}/{videoId}/thumb.jpg`), provisioned idempotently via `HeadBucket`/`CreateBucket` on `OnModuleInit`. Configures `STORAGE_ENDPOINT` (`http://minio:9000` inside Compose) plus optional `STORAGE_PUBLIC_ENDPOINT` for external SigV4 presigning when needed.
- **Pros:** Vendor-neutral industry standard — switching from local MinIO to AWS S3, Cloudflare R2, or Backblaze B2 requires only env var changes. Modular v3 packages keep bundle and install size focused, provide native Node `Readable` streams on `GetObjectCommand`, and avoid managing lifecycle/CORS across multiple buckets.
- **Cons:** AWS SigV4 signs the `Host` header, so presigned URLs signed for `minio:9000` vs. a public host require dedicated S3 signer configuration (`STORAGE_PUBLIC_ENDPOINT` fallback to `STORAGE_ENDPOINT`). Slightly more verbose command-pattern API than legacy SDKs.

### Option B: AWS SDK v3 with Separate Buckets per Asset Type (`streamtube-videos`, `streamtube-thumbnails`)
- Same `@aws-sdk/client-s3` SDK, but splits raw video objects and generated JPEG thumbnails into two distinct buckets with independent bucket policies.
- **Pros:** Allows bucket-level public-read policies on thumbnails while keeping raw videos strictly private at the bucket boundary.
- **Cons:** Doubles env vars, bootstrap checks, and test cleanup logic across integration/E2E suites, while Phase 03 serves thumbnails through controlled API endpoints (`TD-07`) and Phase 04 introduces `unlisted` videos whose assets should not sit in a blanket world-listable bucket.

### Option C: MinIO Proprietary Client (`minio` npm SDK)
- Uses MinIO's own JavaScript client (`Client.presignedUrl`, `Client.putObject`, `Client.getObject`) instead of AWS SDK v3.
- **Pros:** Concise helper methods tailored specifically to MinIO servers.
- **Cons:** Weaker TypeScript typings and smaller ecosystem than `@aws-sdk/client-s3`, and less idiomatic for AWS S3 production deployments (IAM roles, SigV4 middleware, checksums).

**Recommendation:** **Option A (AWS SDK v3 + Single Bucket with Key Prefixes + Dual Endpoint Config)** — Preserves 100% S3 portability between MinIO in Docker and AWS S3 in production, keeps bucket provisioning/testing atomic in one `streamtube` bucket with clean `videos/` and `thumbnails/` prefixes, and handles Docker-vs-public URL signing cleanly via config.

**Decision:** A (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, single `streamtube` bucket + key prefixes + dual endpoint config)
**Libraries:** @aws-sdk/client-s3, @aws-sdk/s3-request-presigner

---

## TD-03: Large File Upload Protocol (Up to 10 GiB) and Draft Pre-Registration Handshake

**Scope:** Cross-layer

**Capability:** Transversal — covers: "Upload de vídeos com suporte a arquivos de até 10GB sem impacto na performance", "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload"

**Context:** Phase 03 requires supporting video uploads up to 10 GiB without blocking or saturating the NestJS API, while automatically pre-registering the video as a draft (`draft`) as soon as upload starts and supporting connection recovery (`docs/project-plan.md` § 4). This is a `Cross-layer` contract: it defines the OpenAPI DTOs and REST handshake on `nestjs-project/` (`openapi-docs-nestjs/TD-01..TD-02`) and governs how `next-frontend/` (`next-frontend-config-base/TD-03`) will upload large files directly to storage without proxying bytes through Next.js BFF or NestJS.

**Options:**

### Option A: API-Orchestrated S3 Presigned Multipart Upload with Atomic Draft Creation
- `POST /videos/uploads` validates metadata (`sizeBytes <= 10737418240` [10 GiB], MIME allowlist), creates the `Video` row in `status = 'draft'` with a unique `short_code`, initiates S3 `CreateMultipartUpload`, and returns `videoId`, `shortCode`, `uploadId`, `partSizeBytes`, and presigned `UploadPart` URLs. Clients `PUT` byte chunks directly to S3/MinIO, use `POST /videos/:id/parts/:partNumber/presign` to re-sign expired part URLs on resume, call `POST /videos/:id/complete` (with `{ partNumber, etag }[]`) to finalize S3 assembly and enqueue processing, or `DELETE /videos/:id/upload` to abort orphan S3 parts.
- **Pros:** Zero video bytes ever pass through NestJS or a future Next.js BFF — API memory and event-loop latency remain constant `O(1)` even for 10 GiB uploads. Satisfies S3's multipart specification (5 MiB–5 GiB parts, up to 10,000 parts, 5 TiB ceiling), guarantees atomic `draft` pre-registration at upload start, and enables per-part retry/resume out of the box.
- **Cons:** Multi-step handshake (`init → direct part PUTs → complete | abort`) requires client-side `ETag` collection and bucket CORS exposing `ETag` when called from browsers in future frontend phases.

### Option B: Single S3 Presigned `PUT` URL
- API creates the `draft` record and returns a single `PutObjectCommand` presigned URL; the client uploads the entire file in one HTTP `PUT` directly to S3/MinIO, then notifies the API.
- **Pros:** Fewer endpoints than multipart (no `uploadId` or `ETag` part list).
- **Cons:** **Violates the 10 GiB requirement on S3-compatible storage:** the S3 protocol enforces a hard 5 GiB maximum on single `PUT` operations, and any network drop at 9.9 GiB forces restarting the entire upload from byte zero.

### Option C: Dedicated `tus` Resumable Upload Server (`@tus/server` + `@tus/s3-store`)
- Embeds or deploys a `tus` protocol server that receives `PATCH` byte-offset chunks over HTTP and forwards them to S3/MinIO as multipart parts.
- **Pros:** Standardized resumable upload protocol (`tus.io`) with client libraries (`tus-js-client`).
- **Cons:** Routes all 10 GiB of video traffic through a Node.js intermediary process, re-introducing API/server bandwidth and I/O bottlenecks, adding extra dependencies outside `@nestjs/swagger` DTO conventions, and complicating Docker scaling.

### Option D: Direct `multipart/form-data` Upload Through NestJS (`Multer` / Busboy Stream to S3)
- Client posts the raw video file directly to a NestJS controller endpoint, which pipes the incoming request stream into `@aws-sdk/lib-storage` `Upload`.
- **Pros:** Single HTTP call for small files.
- **Cons:** Holds NestJS HTTP connections open for minutes/hours on 10 GiB files, couples API CPU/memory/sockets to upload throughput, breaks `next-frontend`'s strict BFF boundary if proxied, and lacks chunk-level resume on connection drop.

**Recommendation:** **Option A (API-Orchestrated S3 Presigned Multipart Upload)** — The only architecture that supports 10 GiB above S3's 5 GiB single-PUT limit, keeps NestJS API load at `O(1)` by sending bytes directly to S3/MinIO (aligning with `next-frontend-config-base/TD-03`), pre-registers the `draft` record atomically at handshake start, and exposes clean OpenAPI-typed REST endpoints (`openapi-docs-nestjs/TD-02`).

**Decision:** A (API-Orchestrated S3 Presigned Multipart Upload with atomic `draft` pre-registration)

---

## TD-04: Video Worker Process and Container Topology

**Scope:** Backend

**Capability:** Serviço de processamento em segundo plano (filas)

**Context:** Video metadata probing and thumbnail frame decoding invoke CPU- and I/O-intensive `ffprobe`/`ffmpeg` child processes. Phase 03 must decide how the queue consumer runs in relation to `nestjs-api` and `compose.yaml` (`docs/diagrams/software-arch.mermaid` models `Video Worker (FFmpeg)` as a distinct container). Depends on `TD-01`.

**Options:**

### Option A: Standalone NestJS Application Context (`src/worker.main.ts`) in Dedicated `video-worker` Container
- Creates `WorkerModule` booted via `NestFactory.createApplicationContext(WorkerModule)` (no Express HTTP server or port binding) inside `nestjs-project/src/worker.main.ts`. `compose.yaml` runs a dedicated `video-worker` service built from the same `Dockerfile.dev` (with `ffmpeg` installed), sharing `ConfigModule`, `TypeOrmModule`, `StorageModule`, and `Video` entities while `AppModule` registers only the queue producer.
- **Pros:** Matches the C4 container diagram (`nestjs-api` vs. `video-worker`), isolates FFmpeg CPU spikes and worker restarts from HTTP API latency, exposes zero unnecessary network ports, and avoids duplicating entities, migrations, or config schemas across separate packages.
- **Cons:** Runs a second Node.js container in `compose.yaml` sharing the bind-mounted `nestjs-project/` workspace (`npm run start:worker:dev` / `node dist/worker.main`).

### Option B: In-Process Queue Consumer Inside `AppModule` (`nestjs-api` Container)
- Registers `VideoProcessor` directly in `VideosModule` imported by `AppModule`, so the HTTP server process also consumes jobs from the queue.
- **Pros:** One fewer service in `compose.yaml` and a single bootstrap file (`src/main.ts`).
- **Cons:** Violates container isolation in `software-arch.mermaid` and `desafio.md` ("fila, worker e storage reais subindo no Compose"): heavy FFmpeg jobs compete with HTTP request handling, and every E2E/integration test booting `AppModule` would involuntarily start a background worker competing for jobs.

### Option C: Separate Monorepo Subproject (`video-worker/` with Independent `package.json`)
- Creates a third top-level directory (`video-worker/`) alongside `nestjs-project/` and `next-frontend/` with its own dependencies, S3 client, and SQL queries.
- **Pros:** Complete build and dependency decoupling from `nestjs-project/`.
- **Cons:** Duplicates TypeORM `Video` schema/column definitions, Joi env validation, and S3 config across two Node subprojects without a shared workspace setup (`package.json` workspaces do not exist at repo root), creating high schema-drift risk.

**Recommendation:** **Option A (Standalone `NestFactory.createApplicationContext(WorkerModule)` in Dedicated `video-worker` Compose Service)** — Delivers true container-level process isolation between HTTP API and FFmpeg worker as required by the C4 architecture while reusing 100% of `nestjs-project/`'s TypeORM entities, `registerAs` configs (`phase-01-configuracao-base/TD-03`), and `StorageModule`.

**Decision:** A (Standalone `NestFactory.createApplicationContext(WorkerModule)` in dedicated `video-worker` container)

---

## TD-05: Video Metadata Extraction and Thumbnail Generation Pipeline

**Scope:** Backend

**Capability:** Transversal — covers: "Processamento automático do vídeo após upload (extração de duração e metadados)", "Geração automática de thumbnail a partir de um frame do vídeo"

**Context:** Once multipart upload completes, the worker must extract duration and technical metadata (dimensions, codec) and generate a thumbnail image from a video frame — including for videos up to 10 GiB stored in S3/MinIO. Downloading 10 GiB to container local disk on every job risks disk exhaustion and slow processing. Depends on `TD-02` and `TD-04`.

**Options:**

### Option A: Direct `ffprobe` / `ffmpeg` `execFile` Over Short-Lived S3 Presigned `GET` URLs (HTTP `Range` Seek)
- Installs `ffmpeg` (`ffmpeg` + `ffprobe`) in `Dockerfile.dev` and invokes binaries directly via `node:child_process.execFile` (argument arrays, `shell: false`, strict timeouts). Passes a short-lived internal S3 presigned `GET` URL (`http://minio:9000/...`): `ffprobe -v error -print_format json -show_format -show_streams <url>` reads container headers via HTTP `Range` without downloading the full 10 GiB payload, and `ffmpeg -ss <seekSec> -i <url> -frames:v 1 -q:v 2 -f image2pipe -vcodec mjpeg pipe:1` seeks directly to an adaptive frame (`duration > 1 ? Math.min(Math.floor(duration * 0.1), 5) : 0`) and pipes the JPEG buffer to `StorageService.putObject`.
- **Pros:** Processes multi-gigabyte MP4 files in seconds with near-zero disk usage (no `/tmp` files), eliminates command-injection surface via `execFile`, and relies solely on maintained OS `ffmpeg`/`ffprobe` binaries with zero unmaintained npm wrappers.
- **Cons:** Requires HTTP `Range` support on the object store (natively supported by both MinIO and AWS S3); non-faststart MP4s with `moov` atom at EOF perform an extra HTTP `Range` tail read automatically handled by `libavformat`.

### Option B: Full Object Download from S3/MinIO to Container Temporary File Before `ffprobe` / `ffmpeg`
- Streams the entire S3 object to a temporary file on disk, runs `ffprobe` and `ffmpeg` against the local file path, uploads the thumbnail, and unlinks the temporary file in `finally`.
- **Pros:** Works identically even on non-seekable streams or storage backends lacking HTTP `Range`.
- **Cons:** Copying up to 10 GiB over the network onto container disk per job saturates container ephemeral storage (especially under concurrency or retries) and adds minutes of unnecessary I/O latency.

### Option C: `fluent-ffmpeg` Wrapper Package Around FFmpeg
- Uses the `fluent-ffmpeg` npm fluent builder API (`ffmpeg(input).screenshots(...)`, `ffmpeg.ffprobe(...)`) instead of calling `execFile` directly.
- **Pros:** Chained JS helper syntax familiar from older Node tutorials.
- **Cons:** **`fluent-ffmpeg` was officially archived and deprecated in May 2025** — introducing an unmaintained dependency violates project quality standards when `execFile` wraps `ffprobe` JSON output and `ffmpeg` pipe output in ~50 lines of strict TypeScript.

**Recommendation:** **Option A (Direct `execFile` of `ffprobe`/`ffmpeg` Over Presigned S3 `GET` URL with Pipe Output)** — Avoids downloading 10 GiB files to disk via native HTTP `Range` seeking, produces zero temporary filesystem artifacts, prevents shell injection, and avoids deprecated wrapper packages (`fluent-ffmpeg`).

**Decision:** A (Direct `execFile` of `ffprobe`/`ffmpeg` over Presigned S3 `GET` URL with pipe output)

---

## TD-06: Unique Video URL Identifier Strategy

**Scope:** Cross-layer

**Capability:** URL única por vídeo, sem conflito com outros vídeos

**Context:** Every video requires a short, permanent, collision-free public identifier (`docs/project-plan.md` § 4: *"cada vídeo precisa de uma URL curta e única que nunca conflite com outro vídeo"*) assigned when the draft is pre-registered so public watch/stream/download URLs remain stable even if the user edits the title later (Phase 04) or sets visibility to `unlisted` (Phases 04–05).

**Options:**

### Option A: CSPRNG 11-Character URL-Safe `short_code` (`[A-Za-z0-9_-]`) + DB `UNIQUE` Constraint with Retry
- Generates an 11-character base64url-alphabet identifier (`[A-Za-z0-9_-]`, 66 bits of cryptographic entropy, YouTube-style e.g. `dQw4w9WgXcQ`) using Node's built-in `node:crypto.randomBytes`, backed by a PostgreSQL `UNIQUE` constraint on `videos.short_code` and bounded retry (up to 5 attempts) if PostgreSQL returns error code `23505`.
- **Pros:** Zero external dependencies (no ESM-only `nanoid` friction under NestJS `commonjs` Jest/TypeORM CLI), compact 11-char URLs, immune to title edits, non-enumerable for future `unlisted` videos, and mathematically collision-resistant (~$7.3 \times 10^{19}$ space) with database-enforced uniqueness guarantee.
- **Cons:** Requires a secondary unique index alongside the internal `id` (`uuid` PK) and a small retry loop around insert collisions.

### Option B: Title-Derived Slug with Random Suffix (`meu-video-a1b2`)
- Normalizes the initial video title into kebab-case ASCII and appends a short random suffix to avoid collisions, similar to channel nickname sanitization (`phase-02-auth/TD-10`).
- **Pros:** Human-readable keywords appear in the URL.
- **Cons:** Video titles are mutable in Phase 04 (causing either URL rot or stale misleading slugs), leak title words for `unlisted` videos, and are fragile at draft pre-registration when only a filename may be available.

### Option C: Raw `UUIDv4` Primary Key in Public URLs (`/videos/550e8400-e29b-41d4-a716-446655440000`)
- Exposes the 36-character PostgreSQL `uuid` primary key directly as the public video URL identifier without a separate `short_code` column.
- **Pros:** Zero extra column or index; reuse of PK.
- **Cons:** Fails the `docs/project-plan.md` § 4 guidance for a *short* unique video URL (36 chars vs. 11 chars) and couples external permanent links directly to internal primary-key representation.

**Recommendation:** **Option A (11-Character CSPRNG URL-Safe `short_code` via `node:crypto` + PostgreSQL `UNIQUE` Constraint)** — Delivers short, title-independent, non-enumerable YouTube-style URLs backed by database-level uniqueness without adding ESM-incompatible third-party ID packages to the CommonJS/nodenext NestJS build.

**Decision:** A (11-character CSPRNG URL-safe `short_code` via `node:crypto` + PostgreSQL `UNIQUE` constraint)

---

## TD-07: Video Streaming (`206 Partial Content`) and Download Delivery Strategy

**Scope:** Cross-layer

**Capability:** Transversal — covers: "Reprodução via streaming (sem necessidade de download completo)", "Download do vídeo pelo usuário"

**Context:** Anonymous and authenticated users must play videos progressively via streaming (scrubbing/seeking without downloading the full file) and download the original video (`docs/project-plan.md`, Fase 03). Because containers communicate over `http://minio:9000` while browsers access `http://localhost:3000` (or `next-frontend` via strict BFF in `next-frontend-config-base/TD-03`), the delivery contract affects both layers and rate-limiting (`phase-02-auth/TD-08`).

**Options:**

### Option A: Non-Buffered API HTTP `Range` Stream (`206 Partial Content`) and Attachment Stream via `StreamableFile`
- Exposes public `@Public()` + `@SkipThrottle()` endpoints `GET /videos/:shortCode/stream`, `GET /videos/:shortCode/download`, and `GET /videos/:shortCode/thumbnail` (only when `status === 'ready'`, returning `409 VIDEO_NOT_READY` otherwise). `/stream` parses RFC 9110 `Range: bytes=start-end`, forwards `Range` to S3 `GetObjectCommand`, and pipes the Node `Readable` body via NestJS `StreamableFile` (`O(1)` RAM) with `206 Partial Content`, `Accept-Ranges: bytes`, `Content-Range`, and `Content-Length` (or `200 OK` when no `Range` header is sent, and `416 Range Not Satisfiable` on invalid ranges); `/download` streams with `Content-Disposition: attachment; filename="..."`.
- **Pros:** Works identically inside Docker Compose, E2E `supertest` suites, browser `<video src="...">` tags, and future Next.js integrations without SigV4 `Host` mismatch between `minio:9000` and `localhost:9000`. Constant `O(1)` memory footprint via backpressure-aware Node streams, and enforces domain readiness/visibility checks on every request.
- **Cons:** Media bytes traverse `nestjs-api`'s network socket (without buffering in RAM), whereas CDN/S3 offload would bypass API egress at hyperscale.

### Option B: `302 Found` Redirect to Short-Lived S3 Presigned `GET` URLs
- API validates video readiness and responds with `302 Found` pointing `Location` to a presigned S3 `GET` URL (with `response-content-disposition=attachment` for download); the client follows the redirect to MinIO/S3, which serves `206 Partial Content` natively.
- **Pros:** Offloads all egress bandwidth from `nestjs-api` to S3/MinIO.
- **Cons:** Breaks seamless local/container interoperability unless a reverse proxy unifies hostnames: a URL presigned with `Host: minio:9000` fails DNS resolution in host browsers, while signing with `Host: localhost:9000` fails inside Docker network tests (`supertest` / BFF containers). Also complicates `Content-Range` E2E verification on the API contract itself.

### Option C: HLS / DASH Adaptive Bitrate Transcoding (`.m3u8` Playlists + Segment Files)
- Worker transcodes every uploaded video into multi-resolution HLS/DASH segments stored in S3/MinIO; clients play via `hls.js` / MSE.
- **Pros:** Adaptive bitrate switching on fluctuating mobile networks.
- **Cons:** Multiplies worker CPU time and storage footprint by 5–20x on 10 GiB uploads, requires full video re-encoding not requested in `docs/project-plan.md` Fase 03 (which specifies metadata extraction + thumbnail + progressive streaming + original file download), and still requires separate raw-file handling for download.

**Recommendation:** **Option A (Non-Buffered HTTP `Range` `206 Partial Content` + Attachment `StreamableFile` Proxy from S3 `GetObjectCommand`)** — Satisfies RFC 9110 progressive streaming and download with `O(1)` memory, eliminates Docker `minio:9000` vs. `localhost` SigV4 redirect breakage, exempts player chunk requests from `AuthModule` throttling via `@SkipThrottle()`, and keeps OpenAPI contracts self-contained.

**Decision:** A (Non-buffered HTTP `Range` `206 Partial Content` + attachment `StreamableFile` proxy from S3 `GetObjectCommand`)

---

## TD-08: Video Lifecycle State Machine and Worker Retry/Failure Policy

**Scope:** Backend

**Capability:** Transversal — covers: "Pré-cadastro automático do vídeo como rascunho ao iniciar o upload", "Processamento automático do vídeo após upload (extração de duração e metadados)", "Serviço de processamento em segundo plano (filas)"

**Context:** A video progresses asynchronously from upload initiation through multipart completion and background FFmpeg processing, where failures may be transient (temporary Redis/MinIO network blip) or permanent (corrupt container, zero video streams, missing S3 object). The database state machine and worker retry rules must keep API endpoints and background jobs deterministic. Depends on `TD-01`, `TD-03`, and `TD-05`.

**Options:**

### Option A: PostgreSQL Enum `draft → processing → ready | failed` + Tiered BullMQ Retry (`attempts: 3` + `UnrecoverableError`)
- Persists a PostgreSQL enum `videos_status_enum` (`'draft' | 'processing' | 'ready' | 'failed'`, default `'draft'`) plus nullable `error_reason` on `videos`. `POST /videos/uploads` creates `draft`; `POST /videos/:id/complete` atomically transitions `draft → processing` and enqueues `process-video` (`jobId: videoId`, `attempts: 3`, exponential backoff `1000ms`). `VideoProcessor` marks `ready` with extracted metadata + `thumbnail_key` on success, throws BullMQ `UnrecoverableError` immediately on corrupt media or missing S3 objects, and persists `status = 'failed'` + `error_reason` inside `@OnWorkerEvent('failed')` once retries are exhausted or unrecoverable.
- **Pros:** Transient infrastructure hiccups self-heal via exponential backoff (1s, 2s, 4s) while broken files fail fast without wasting 3 FFmpeg runs. Enforces clean HTTP state guards (`complete` rejects non-`draft` with `409`; `stream`/`download`/`thumbnail` reject non-`ready` with `409 VIDEO_NOT_READY` per `phase-02-auth/TD-07` error envelope) and never crashes the worker process.
- **Cons:** Requires idempotent processor logic (safe re-execution on retry) and careful terminal-attempt check (`job.attemptsMade >= (job.opts.attempts ?? 1)` or `err instanceof UnrecoverableError`) in `@OnWorkerEvent('failed')`.

### Option B: Single-Attempt Fail-Fast (`attempts: 1`, Immediate `failed` on Any Error)
- Same `draft → processing → ready | failed` column, but BullMQ jobs run with `attempts: 1` (no automatic retry); any exception inside `process()` immediately marks the video `failed`.
- **Pros:** Simplest worker error handler — every failure is immediately terminal.
- **Cons:** Fragile in containerized/cloud environments: a momentary S3/MinIO socket reset on a 10 GiB upload permanently marks a valid user upload as `failed`, forcing the user to re-upload 10 GiB.

### Option C: Separate `video_processing_events` Audit Table Without Direct `status` Column on `videos`
- Derives current video state by querying the latest event row from an append-only `video_processing_events` table rather than maintaining a `status` enum column on `videos`.
- **Pros:** Preserves a full log of every retry attempt in SQL.
- **Cons:** Complicates every read query (`GET /videos/:shortCode`, Phase 04 studio listings, Phase 07 home grids) with joins/window functions for a concern BullMQ already tracks in Redis job state, and diverges from `desafio.md`'s explicit persistence model (a `videos` table carrying status `draft → processing → ready/failed`).

**Recommendation:** **Option A (PostgreSQL Enum `draft → processing → ready | failed` + BullMQ `attempts: 3` Exponential Backoff + `UnrecoverableError`)** — Balances resilience against transient storage/network faults with fast failure on corrupt media, keeps read queries `O(1)` on `videos.status`, and integrates cleanly with `phase-02-auth/TD-07`'s `DomainException` catalog.

**Decision:** A (PostgreSQL enum `draft → processing → ready | failed` + BullMQ `attempts: 3` exponential backoff + `UnrecoverableError`)

---

## Decisions Summary

| ID | Scope | Decision | Recommendation | Choice |
|----|-------|----------|---------------|--------|
| TD-01 | Backend | Background Job Queue Technology | Option A (BullMQ + Redis via `@nestjs/bullmq`) | A (BullMQ + Redis via `@nestjs/bullmq` + `bullmq`) |
| TD-02 | Backend | Object Storage SDK, Bucket Topology, and Endpoint Signing Strategy | Option A (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`, single bucket + prefixes) | A (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`) |
| TD-03 | Cross-layer | Large File Upload Protocol (Up to 10 GiB) and Draft Pre-Registration Handshake | Option A (API-Orchestrated S3 Presigned Multipart Upload + atomic `draft`) | A (S3 Presigned Multipart Upload + atomic `draft`) |
| TD-04 | Backend | Video Worker Process and Container Topology | Option A (Standalone `NestFactory.createApplicationContext(WorkerModule)` in `video-worker` container) | A (Standalone `WorkerModule` in `video-worker` container) |
| TD-05 | Backend | Video Metadata Extraction and Thumbnail Generation Pipeline | Option A (Direct `ffprobe`/`ffmpeg` `execFile` over Presigned S3 `GET` URL with pipe output) | A (Direct `ffprobe`/`ffmpeg` `execFile` over Presigned URL) |
| TD-06 | Cross-layer | Unique Video URL Identifier Strategy | Option A (11-char CSPRNG URL-safe `short_code` via `node:crypto` + DB `UNIQUE`) | A (11-char CSPRNG `short_code` + DB `UNIQUE`) |
| TD-07 | Cross-layer | Video Streaming (`206 Partial Content`) and Download Delivery Strategy | Option A (Non-buffered HTTP `Range` `206` + attachment `StreamableFile` from S3 `GetObjectCommand`) | A (Non-buffered HTTP `Range` `206` + `StreamableFile`) |
| TD-08 | Backend | Video Lifecycle State Machine and Worker Retry/Failure Policy | Option A (PostgreSQL enum `draft → processing → ready \| failed` + BullMQ `attempts: 3` + `UnrecoverableError`) | A (`draft → processing → ready \| failed` + BullMQ `attempts: 3`) |

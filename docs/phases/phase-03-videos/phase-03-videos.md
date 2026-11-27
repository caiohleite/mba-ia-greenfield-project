---
kind: phase
name: phase-03-videos
test_specs_aware: true
sources_mtime:
  docs/phases/phase-03-videos/context.md: "2026-11-26T23:24:26-03:00"
  docs/phases/phase-03-videos/library-refs.md: "2026-11-26T23:23:30-03:00"
  docs/project-plan.md: "2026-11-06T22:57:37-03:00"
  docs/decisions/technical-decisions-phase-03-videos.md: "2026-11-26T23:21:36-03:00"
  docs/decisions/technical-decisions-openapi-docs-nestjs.md: "2026-11-08T22:02:15-03:00"
  docs/phases/phase-01-configuracao-base/context.md: "2026-11-08T22:02:15-03:00"
  docs/phases/phase-02-auth/context.md: "2026-11-08T22:02:15-03:00"
  docs/phases/phase-02-auth-frontend/context.md: "2026-11-08T22:02:15-03:00"
  .agents/skills/testing-guide-nestjs-project/SKILL.md: "2026-11-08T23:10:51-03:00"
---

# Phase 03 — Upload e Processamento de Vídeos

## Objective

Entregar o backend completo de upload e processamento de vídeos da Fase 03 em `nestjs-project` — serviço de armazenamento S3/MinIO para vídeos e thumbnails, upload multipart pré-assinado de até 10GB sem impactar a performance da API com pré-cadastro imediato em rascunho (`draft`), fila Redis/BullMQ e container worker dedicado (`video-worker`) com FFmpeg/ffprobe para extração automática de duração/metadados e geração de thumbnail, URLs curtas e únicas (`short_code`), reprodução progressiva via streaming HTTP `Range` (`206 Partial Content`) e download do arquivo original.

---

## Step Implementations

### SI-03.1 — Infra: Dependências, FFmpeg no Container, Configurações Namespaced e Serviços MinIO/Redis

**Description:** Instalar as dependências de fila e storage da Fase 03, adicionar `ffmpeg`/`ffprobe` à imagem Docker de desenvolvimento, criar os namespaces de configuração (`storage`, `queue`, `videos`) validados por Joi e subir os containers `minio` e `redis` no Docker Compose.

**Technical actions:**

1. Instalar em `nestjs-project/package.json` as dependências de produção `@nestjs/bullmq@^11.0.0`, `bullmq@^5.0.0`, `@aws-sdk/client-s3@^3.0.0` e `@aws-sdk/s3-request-presigner@^3.0.0`, e atualizar `nestjs-project/Dockerfile.dev` para instalar os pacotes de sistema `ffmpeg` (que provê `ffmpeg` e `ffprobe`) e `curl` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-02`, `phase-03-videos/TD-05`).
2. Criar `nestjs-project/src/config/storage.config.ts` (`registerAs('storage', ...)` lendo `STORAGE_ENDPOINT`, `STORAGE_PUBLIC_ENDPOINT`, `STORAGE_REGION`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, `STORAGE_BUCKET`, `STORAGE_PRESIGN_EXPIRES_IN`), `nestjs-project/src/config/queue.config.ts` (`registerAs('queue', ...)` lendo `REDIS_HOST`, `REDIS_PORT`) e `nestjs-project/src/config/videos.config.ts` (`registerAs('videos', ...)` lendo `VIDEO_MAX_SIZE_BYTES` default `10737418240` e `VIDEO_MULTIPART_PART_SIZE_BYTES` default `10485760`) (per `phase-01-configuracao-base/TD-03`, `phase-03-videos/TD-01`, `phase-03-videos/TD-02`, `phase-03-videos/TD-03`).
3. Estender `nestjs-project/src/config/env.validation.ts` com validação Joi das novas variáveis (`STORAGE_ENDPOINT` default `'http://minio:9000'`, `STORAGE_PUBLIC_ENDPOINT` opcional, `STORAGE_REGION` default `'us-east-1'`, `STORAGE_ACCESS_KEY` e `STORAGE_SECRET_KEY` obrigatórias, `STORAGE_BUCKET` default `'streamtube'`, `STORAGE_PRESIGN_EXPIRES_IN` default `3600`, `REDIS_HOST` default `'redis'`, `REDIS_PORT` default `6379`, `VIDEO_MAX_SIZE_BYTES` default `10737418240`, `VIDEO_MULTIPART_PART_SIZE_BYTES` default `10485760`) e documentar todas em `nestjs-project/.env.example` (per `phase-01-configuracao-base/TD-02`).
4. Registrar `storageConfig`, `queueConfig` e `videosConfig` em `ConfigModule.forRoot({ load: [...] })` e configurar `BullModule.forRootAsync` global injetando `queueConfig.KEY` em `nestjs-project/src/app.module.ts` (per `phase-01-configuracao-base/TD-01`, `phase-03-videos/TD-01`).
5. Adicionar os serviços `minio` (`minio/minio`, comando `server /data --console-address ":9001"`, portas `9000:9000` e `9001:9001`, volume nomeado `minio_data:/data` e healthcheck em `http://localhost:9000/minio/health/live`) e `redis` (`redis:7-alpine`, porta `6379:6379` e healthcheck `redis-cli ping`) em `nestjs-project/compose.yaml`, adicionando `depends_on` com `condition: service_healthy` em `nestjs-api` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-02`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `envValidationSchema` | Integration: aceita variáveis de Storage/Redis/Videos válidas, aplica defaults e rejeita ausência de credenciais S3 obrigatórias ou portas/tamanhos inválidos | `src/config/env.validation.integration-spec.ts` |

**Dependencies:** none

**Acceptance criteria:**

- `AppModule` inicializa sem erros com as variáveis de Storage, Redis e Videos configuradas e `GET /` continua retornando `200`
- Inicializar a aplicação sem `STORAGE_ACCESS_KEY` ou `STORAGE_SECRET_KEY` falha imediatamente no bootstrap com erro de validação Joi
- Serviço `minio` sobe em `compose.yaml` com status `healthy` e responde `200` em `http://minio:9000/minio/health/live` a partir de `nestjs-api`
- Serviço `redis` sobe em `compose.yaml` com status `healthy` e responde `PONG` ao comando `redis-cli ping`
- Binários `ffmpeg -version` e `ffprobe -version` executam com código `0` dentro do container `nestjs-api`

---

### SI-03.2 — Entidade Video, Relação Bidirecional com Channel, Exceções de Domínio e Migration CreateVideos

**Description:** Criar a entidade `Video`, a relação bidirecional com `Channel`, o catálogo de exceções de domínio de vídeos e a migration versionada `CreateVideos`, mantendo o helper `createTestDataSource` compatível com as suítes existentes.

**Technical actions:**

1. Criar `nestjs-project/src/videos/entities/video-status.enum.ts` (`VideoStatus`: `'draft' | 'processing' | 'ready' | 'failed'`) e `nestjs-project/src/videos/entities/video.entity.ts` (`@Entity('videos')`) conforme `### Data Model → Video`, com `upload_id` marcado `{ select: false }`, `short_code` `{ unique: true }`, `size_bytes` `bigint` e relação `@ManyToOne(() => Channel, (channel) => channel.videos, { onDelete: 'CASCADE' })` com `@JoinColumn({ name: 'channel_id' })` (per `phase-03-videos/TD-03`, `phase-03-videos/TD-06`, `phase-03-videos/TD-08`).
2. Adicionar `@OneToMany(() => Video, (video) => video.channel) videos: Video[]` em `nestjs-project/src/channels/entities/channel.entity.ts` e registrar `Video` em `TypeOrmModule.forFeature` do módulo de canais conforme regra de relacionamentos bidirecionais (per `phase-03-videos/TD-03`).
3. Gerar a migration versionada via CLI dentro do container (`npm run migration:generate -- src/database/migrations/CreateVideos`), revisando a criação de `videos_status_enum`, tabela `videos`, FK `channel_id` (`ON DELETE CASCADE`) e índices em `short_code`, `channel_id` e `status` (per `phase-03-videos/TD-06`, `phase-03-videos/TD-08`).
4. Atualizar `cleanAllTables()` em `nestjs-project/src/test/create-test-data-source.ts` (executando `DELETE FROM "videos"` antes de `channels` e `users`) e incluir `Video` nas listas de entidades dos specs existentes que usam `createTestDataSource`, além de atualizar `nestjs-project/src/database/migrations.integration-spec.ts` (`MANAGED_TABLES`, `DROP TYPE IF EXISTS "public"."videos_status_enum" CASCADE`, contagem de migrations).
5. Criar `nestjs-project/src/videos/exceptions/video.exceptions.ts` com subclasses tipadas de `DomainException` mapeando todos os códigos de `### Error Catalog` (`CHANNEL_NOT_FOUND`, `VIDEO_NOT_FOUND`, `VIDEO_THUMBNAIL_NOT_FOUND`, `VIDEO_FORBIDDEN`, `INVALID_VIDEO_STATE`, `VIDEO_NOT_READY`, `VIDEO_TOO_LARGE`, `UNSUPPORTED_VIDEO_MIME_TYPE`, `MULTIPART_COMPLETION_FAILED`, `VIDEO_RANGE_NOT_SATISFIABLE`) (per `phase-02-auth/TD-07`, `phase-03-videos/TD-08`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `Video` | Integration: unicidade de `short_code`, default `status = 'draft'`, omissão de `upload_id` por `select: false`, `size_bytes` `bigint` até `10737418240` e cascade ao remover `Channel` | `src/videos/entities/video.entity.integration-spec.ts` |
| `Channel` | Integration: carregamento explícito da relação inversa `relations: ['videos']` | `src/channels/entities/channel.entity.integration-spec.ts` |
| `CreateVideos` migration | Integration: aplicação e reversão limpa da migration `CreateVideos` e do tipo `videos_status_enum` | `src/database/migrations.integration-spec.ts` |

**Dependencies:** SI-03.1

**Acceptance criteria:**

- `npm run migration:run` cria a tabela `videos` e o tipo `videos_status_enum` com todas as colunas, constraints e índices de `### Data Model`
- Inserir dois registros em `videos` com o mesmo `short_code` falha com erro de constraint única (`23505`)
- Consultar `Video` via `find()` omite `upload_id` por padrão (`select: false`) e atribui `status = 'draft'` quando não informado
- Persistir `size_bytes = "10737418240"` preserva o valor exato de 10 GiB em coluna `bigint`
- Remover um `Channel` exclui em cascata todos os seus registros em `videos` e todas as suítes das Fases 01–02 permanecem verdes

---

### SI-03.3 — StorageModule e StorageService (S3/MinIO Multipart, Presigning e Streams)

**Description:** Implementar o módulo de object storage S3-compatível (`StorageModule` + `StorageService`) com provisionamento idempotente do bucket `streamtube`, orquestração de multipart upload, geração de URLs pré-assinadas e leitura streaming parcial (`Range`) sem bufferização em memória.

**Technical actions:**

1. Criar `nestjs-project/src/storage/storage.service.ts` injetando `storageConfig.KEY`, instanciando `S3Client` interno (`endpoint: storage.endpoint`, `forcePathStyle: true`) e `S3Client` presigner público (`endpoint: storage.publicEndpoint ?? storage.endpoint`, `forcePathStyle: true`), garantindo a criação idempotente do bucket `streamtube` em `OnModuleInit` (`HeadBucketCommand` → `CreateBucketCommand` quando `NotFound`/`404`) e encerrando ambos os clientes em `OnModuleDestroy` (per `phase-03-videos/TD-02`).
2. Implementar em `StorageService` os métodos determinísticos de chaves e ciclo multipart: `buildVideoKey(channelId, videoId, ext)` (`videos/{channelId}/{videoId}/source.{ext}`), `buildThumbnailKey(channelId, videoId)` (`thumbnails/{channelId}/{videoId}/thumb.jpg`), `createMultipartUpload(key, contentType)`, `presignUploadPart(key, uploadId, partNumber, expiresIn?)`, `completeMultipartUpload(key, uploadId, parts)` (ordenando `Parts` por `PartNumber` ascendente), `abortMultipartUpload(key, uploadId)`, `putObject(key, body, contentType)`, `deleteObject(key)` e `presignGetObjectInternal(key, expiresIn?)` (per `phase-03-videos/TD-02`, `phase-03-videos/TD-03`, `phase-03-videos/TD-05`).
3. Implementar em `StorageService` o método `getObjectStream(key, rangeHeader?)` executando `GetObjectCommand` com `Range` opcional e devolvendo `{ stream: Readable, contentLength, contentType, contentRange, acceptRanges }` diretamente do socket S3 com memória `O(1)` (per `phase-03-videos/TD-07`).
4. Criar `nestjs-project/src/storage/storage.module.ts` (`imports: [ConfigModule.forFeature(storageConfig)]`, `providers: [StorageService]`, `exports: [StorageService]`) e importá-lo em `AppModule` (per `phase-03-videos/TD-02`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `StorageModule` | Unit: compilação DI do módulo com `ConfigModule` global | `src/storage/storage.module.spec.ts` |
| `StorageService` | Unit: formatação de chaves `videos/` e `thumbnails/`, ordenação ascendente de `PartNumber` em `completeMultipartUpload` e uso do cliente público vs interno nas assinaturas SigV4 | `src/storage/storage.service.spec.ts` |
| `StorageService` | Integration (MinIO real): criação idempotente do bucket `streamtube`, upload multipart completo via `PUT` nas URLs pré-assinadas, `abortMultipartUpload`, `putObject` e `getObjectStream` com `Range: bytes=0-63` | `src/storage/storage.service.integration-spec.ts` |

**Dependencies:** SI-03.1

**Acceptance criteria:**

- Inicializar `StorageModule` cria o bucket `streamtube` no MinIO caso ausente e não falha em reinicializações subsequentes
- Enviar partes via HTTP `PUT` para URLs retornadas por `presignUploadPart` e invocar `completeMultipartUpload` grava o objeto completo em `videos/{channelId}/{videoId}/source.{ext}`
- Invocar `completeMultipartUpload` com array de partes desordenado ordena-as por `PartNumber` antes de enviar `CompleteMultipartUploadCommand`
- Invocar `abortMultipartUpload` invalida o `uploadId` e descarta as partes pendentes no MinIO
- Invocar `getObjectStream` com `Range: bytes=0-63` retorna stream `Readable` de 64 bytes com `contentRange` populado

---

### SI-03.4 — Utilitários de Short Code e Range, DTOs, VideosService e VideosModule (Producer)

**Description:** Implementar a lógica de domínio de vídeos (`VideosService`), geração criptográfica de `short_code` único, parser RFC 9110 de HTTP `Range`, DTOs OpenAPI/validação e `VideosModule` como produtor da fila `video-processing`.

**Technical actions:**

1. Criar `nestjs-project/src/videos/short-code.util.ts` (`generateShortCode(): string` de 11 caracteres `[A-Za-z0-9_-]` via `node:crypto.randomBytes`) e `nestjs-project/src/videos/range-parser.util.ts` (`parseByteRange(rangeHeader, totalSize)` validando RFC 9110 e lançando `VideoRangeNotSatisfiableException` em faixas inválidas ou fora de `0..totalSize-1`) (per `phase-03-videos/TD-06`, `phase-03-videos/TD-07`).
2. Criar em `nestjs-project/src/videos/dto/` os DTOs de entrada (`initiate-upload.dto.ts`, `complete-upload.dto.ts`, `presign-part-params.dto.ts` usando `class-validator` + JSDoc conforme `#### Validation Rules — Video Upload & Multipart`) e de saída (`upload-session-response.dto.ts`, `presigned-part-response.dto.ts`, `video-response.dto.ts` com `@ApiProperty` explícito) (per `phase-02-auth/TD-06`, `openapi-docs-nestjs/TD-01`, `phase-03-videos/TD-03`, `phase-03-videos/TD-06`).
3. Implementar em `nestjs-project/src/videos/videos.service.ts` os métodos de upload e ciclo de estado: `initiateUpload(userId, dto)` (valida MIME permitido e `sizeBytes <= VIDEO_MAX_SIZE_BYTES`, busca canal do usuário, grava `Video` em `status = 'draft'` com retry até 5x em violação `23505` de `short_code`, inicia multipart no `StorageService` com remoção compensatória do rascunho se S3 falhar, persiste `upload_id` e assina URLs das partes), `presignPart(userId, videoId, partNumber)`, `completeUpload(userId, videoId, dto)` (valida ownership e `status === 'draft'`, completa multipart no S3 mapeando falhas S3 para `MultipartCompletionFailedException`, transiciona para `status = 'processing'`, limpa `upload_id` e publica `process-video` com `{ videoId }`, `jobId: video.id`, `attempts: 3`, backoff exponencial `1000ms`) e `abortUpload(userId, videoId)` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-03`, `phase-03-videos/TD-06`, `phase-03-videos/TD-08`).
4. Implementar em `VideosService` os métodos públicos de consulta e entrega não-bufferizada: `findByShortCode(shortCode)`, `openVideoStream(shortCode, rangeHeader?)`, `openVideoDownload(shortCode)` (sanitizando `original_filename` em `Content-Disposition: attachment`) e `openThumbnailStream(shortCode)`, todos exigindo `status === 'ready'` nas rotas de mídia (`409 VIDEO_NOT_READY` caso contrário) (per `phase-03-videos/TD-06`, `phase-03-videos/TD-07`, `phase-03-videos/TD-08`).
5. Criar `nestjs-project/src/videos/videos.module.ts` (`imports: [TypeOrmModule.forFeature([Video, Channel]), ConfigModule.forFeature(videosConfig), StorageModule, BullModule.registerQueue({ name: 'video-processing' })]`, `providers: [VideosService]`, `exports: [VideosService]`) sem registrar `@Processor` na API HTTP (per `phase-03-videos/TD-01`, `phase-03-videos/TD-04`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `generateShortCode` | Unit: comprimento exato de 11 chars, alfabeto `^[A-Za-z0-9_-]{11}$` e ausência de colisões em lote | `src/videos/short-code.util.spec.ts` |
| `parseByteRange` | Unit: faixas válidas (`bytes=0-1023`, `bytes=500-`, `bytes=-256`) e disparo de `VideoRangeNotSatisfiableException` em faixas malformadas ou `start >= size` | `src/videos/range-parser.util.spec.ts` |
| `VideosModule` | Unit: compilação DI do módulo produtor com `StorageModule` e `BullModule.registerQueue` | `src/videos/videos.module.spec.ts` |
| `VideosService` | Unit: limites 10 GiB (`413`) e MIME (`415`), ownership (`403`), guardas `INVALID_VIDEO_STATE` / `VIDEO_NOT_READY` (`409`), retry `23505` em `short_code` e compensação em erro do S3 | `src/videos/videos.service.spec.ts` |
| `VideosService` | Integration (PostgreSQL + MinIO + Redis reais): `initiateUpload` grava `draft` e abre multipart real, `completeUpload` monta objeto no MinIO, muda para `processing` e enfileira job `process-video`, e `abortUpload` limpa S3 + DB | `src/videos/videos.service.integration-spec.ts` |

**Dependencies:** SI-03.2, SI-03.3

**Acceptance criteria:**

- `initiateUpload` para arquivo válido de até `10737418240` bytes persiste `Video` em `status = 'draft'` com `short_code` único de 11 chars e retorna `uploadId`, `partSizeBytes` e URLs pré-assinadas
- `initiateUpload` com `sizeBytes > 10737418240` lança `VideoPayloadTooLargeException` (`413 VIDEO_TOO_LARGE`) e com MIME inválido lança `UnsupportedVideoMimeTypeException` (`415 UNSUPPORTED_VIDEO_MIME_TYPE`) sem gravar no banco
- Colisão `23505` em `short_code` durante `initiateUpload` faz retry automático e conclui a criação com novo `short_code`
- `presignPart`, `completeUpload` e `abortUpload` lançam `VideoForbiddenException` (`403 VIDEO_FORBIDDEN`) para não-donos e `InvalidVideoStateException` (`409 INVALID_VIDEO_STATE`) quando `status !== 'draft'`
- `completeUpload` finaliza o multipart no MinIO, atualiza `status = 'processing'` e publica o job `process-video` (`### Events/Messages`) na fila Redis
- `abortUpload` aborta o multipart no MinIO e remove o registro `draft` do PostgreSQL
- `openVideoStream`, `openVideoDownload` e `openThumbnailStream` lançam `VideoNotReadyException` (`409 VIDEO_NOT_READY`) quando `status !== 'ready'`

---

### SI-03.5 — Pipeline FFmpeg/ffprobe, VideoProcessor e Container Standalone video-worker

**Description:** Implementar a extração de duração/metadados via `ffprobe`, geração automática de thumbnail JPEG via `ffmpeg` sobre URL S3 pré-assinada, o consumidor BullMQ `VideoProcessor` e o contexto standalone `WorkerModule` rodando no container `video-worker`.

**Technical actions:**

1. Criar `nestjs-project/src/videos/media-probe.service.ts` invocando `ffprobe` e `ffmpeg` via `node:child_process.execFile` (`shell: false`, timeouts explícitos) sobre URLs S3 `GET` pré-assinadas internas: `probeVideo(presignedUrl)` extrai `durationSeconds`, `width`, `height` e `videoCodec` (lançando erro de mídia inválida se não houver stream de vídeo ou duração válida), e `extractThumbnailJpeg(presignedUrl, durationSeconds)` aplica seek adaptativo (`duration > 1 ? Math.min(Math.floor(duration * 0.1), 5) : 0`) retornando `Buffer` JPEG via `pipe:1` sem arquivos temporários em disco (per `phase-03-videos/TD-05`).
2. Criar `nestjs-project/src/worker/video.processor.ts` (`@Processor('video-processing') export class VideoProcessor extends WorkerHost`) processando `process-video`: carrega `Video`, ignora de forma idempotente caso `status === 'ready'`, obtém URL pré-assinada interna via `StorageService.presignGetObjectInternal`, executa `MediaProbeService`, grava `thumbnails/{channelId}/{videoId}/thumb.jpg` via `StorageService.putObject`, e salva `duration_seconds`, `width`, `height`, `video_codec`, `thumbnail_key`, `error_reason = null` e `status = 'ready'`; converte arquivo ausente ou mídia inválida em `UnrecoverableError` e trata `@OnWorkerEvent('failed')` persistindo `status = 'failed'` e `error_reason` apenas na falha terminal (`err instanceof UnrecoverableError || job.attemptsMade >= (job.opts.attempts ?? 1)`) (per `phase-03-videos/TD-01`, `phase-03-videos/TD-05`, `phase-03-videos/TD-08`).
3. Criar `nestjs-project/src/worker/worker.module.ts` (`ConfigModule.forRoot`, `TypeOrmModule.forRootAsync`, `BullModule.forRootAsync`, `BullModule.registerQueue({ name: 'video-processing' })`, `TypeOrmModule.forFeature([Video, Channel])`, `StorageModule`, `MediaProbeService`, `VideoProcessor`) e `nestjs-project/src/worker.main.ts` iniciando `NestFactory.createApplicationContext(WorkerModule)` com `app.enableShutdownHooks()` (per `phase-03-videos/TD-04`).
4. Adicionar em `nestjs-project/package.json` os scripts `"start:worker": "nest start --entryFile worker.main"`, `"start:worker:dev": "nest start --watch --entryFile worker.main"` e `"start:worker:prod": "node dist/worker.main"` (per `phase-03-videos/TD-04`).
5. Adicionar o serviço `video-worker` em `nestjs-project/compose.yaml` (build `Dockerfile.dev`, volume `.:/home/node/app`, `env_file: .env`, comando `npm run start:worker:dev`, sem portas HTTP expostas, `depends_on` `db`, `redis` e `minio` com `condition: service_healthy`) (per `phase-03-videos/TD-04`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `WorkerModule` | Unit: compilação DI do módulo standalone do worker | `src/worker/worker.module.spec.ts` |
| `MediaProbeService` | Unit: cálculo de seek adaptativo, parsing de JSON do `ffprobe` e rejeição de containers sem stream de vídeo | `src/videos/media-probe.service.spec.ts` |
| `VideoProcessor` | Unit: atualização para `ready`, disparo de `UnrecoverableError` em mídia inválida vs rethrow transitório, e persistência `failed` em `@OnWorkerEvent('failed')` apenas na falha terminal | `src/worker/video.processor.spec.ts` |
| `VideoProcessor` + `MediaProbeService` | Integration (PostgreSQL + MinIO + Redis + FFmpeg reais, fixture `.mp4` gerada em `Buffer` via `pipe:1`): extrai metadados e thumbnail JPEG reais mudando `status` para `'ready'`, e marca arquivo corrompido como `'failed'` com `error_reason` | `src/worker/video.processor.integration-spec.ts` |

**Dependencies:** SI-03.4

**Acceptance criteria:**

- Serviço `video-worker` sobe via `docker compose up -d` com status `running` e conecta-se a PostgreSQL, Redis e MinIO sem expor portas HTTP
- Processar um vídeo `.mp4` válido no MinIO preenche `duration_seconds`, `width`, `height`, `video_codec` e `thumbnail_key`, grava o JPEG no MinIO e atualiza `status` para `'ready'` sem gravar arquivos em `/tmp`
- Processar um arquivo corrompido ou sem stream de vídeo lança `UnrecoverableError` e atualiza `status` para `'failed'` com `error_reason` preenchido sem encerrar o processo do worker
- Falhas transitórias de rede/storage permitem até 3 tentativas com backoff exponencial antes de marcar `status = 'failed'`
- Reexecutar `process-video` sobre um registro já em `status = 'ready'` é idempotente e mantém o vídeo íntegro
- `npm run build` gera `dist/main.js` e `dist/worker.main.js` sem erros de compilação

---

### SI-03.6 — Endpoint POST /videos/uploads

**Route:** POST /videos/uploads
**Test Specs:** see `nestjs-project/specs/videos-uploads-init.plan.md`
**Authorization:** Authenticated

**Description:** Criar `VideosController`, registrar `VideosModule` em `AppModule` e expor `POST /videos/uploads` para iniciar upload multipart de até 10 GiB com pré-cadastro automático do vídeo em `status = "draft"`.

**Technical actions:**

1. Criar `nestjs-project/src/videos/videos.controller.ts` decorado com `@ApiTags('videos')`, `@ApiExtraModels(ApiErrorEnvelope)` e `@Controller('videos')`, e implementar `@Post('uploads')` (`@HttpCode(201)`, `@ApiBearerAuth('access-token')`, `@ApiOperation`, `@ApiResponse` para `201`, `400`, `401`, `404`, `413` e `415`) delegando a `VideosService.initiateUpload(user.sub, dto)` conforme `### API Contracts → POST /videos/uploads` (per `phase-03-videos/TD-03`, `phase-03-videos/TD-06`, `openapi-docs-nestjs/TD-01`).
2. Registrar `VideosModule` em `nestjs-project/src/app.module.ts` e expor `VideosController` em `controllers` de `VideosModule` (per `phase-03-videos/TD-03`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.4

**Acceptance criteria:**

- `POST /videos/uploads` autenticado com payload válido (`sizeBytes <= 10737418240`) retorna `201` com `videoId`, `shortCode` único de 11 chars, `status: "draft"`, `uploadId`, `partSizeBytes`, `totalParts` e array `parts` contendo URLs S3 pré-assinadas
- `POST /videos/uploads` sem token JWT retorna `401`
- `POST /videos/uploads` com payload inválido (`sizeBytes < 1`, `originalFilename` vazio ou propriedade desconhecida) retorna `400` com `error: "VALIDATION_ERROR"`
- `POST /videos/uploads` com `sizeBytes > 10737418240` retorna `413` com `error: "VIDEO_TOO_LARGE"` e com `mimeType` não suportado retorna `415` com `error: "UNSUPPORTED_VIDEO_MIME_TYPE"`

---

### SI-03.7 — Endpoint POST /videos/:id/parts/:partNumber/presign

**Route:** POST /videos/:id/parts/:partNumber/presign
**Test Specs:** see `nestjs-project/specs/videos-parts-presign.plan.md`
**Authorization:** Owner

**Description:** Expor o endpoint autenticado de re-assinatura de partes individuais (`POST /videos/:id/parts/:partNumber/presign`) para suportar retomada de uploads grandes sem bloqueio pelo rate limiter.

**Technical actions:**

1. Implementar `@Post(':id/parts/:partNumber/presign')` (`@HttpCode(200)`, `@SkipThrottle()`, `@ApiBearerAuth('access-token')`, `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `200`, `400`, `401`, `403`, `404` e `409`) em `VideosController`, validando `id` (`ParseUUIDPipe`) e `partNumber` (`PresignPartParamsDto`) e delegando a `VideosService.presignPart(user.sub, id, partNumber)` conforme `### API Contracts → POST /videos/:id/parts/:partNumber/presign` (per `phase-03-videos/TD-03`, `phase-02-auth/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.6

**Acceptance criteria:**

- `POST /videos/:id/parts/:partNumber/presign` pelo dono de um vídeo em `status = "draft"` retorna `200` com `partNumber`, `uploadUrl` pré-assinada válida para `PUT` no MinIO e `expiresInSeconds`
- `POST /videos/:id/parts/:partNumber/presign` sem JWT retorna `401` e por usuário autenticado que não é dono do vídeo retorna `403` com `error: "VIDEO_FORBIDDEN"`
- `POST /videos/:id/parts/:partNumber/presign` com `id` inválido ou `partNumber` fora do intervalo permitido retorna `400` com `error: "VALIDATION_ERROR"`
- `POST /videos/:id/parts/:partNumber/presign` para vídeo inexistente retorna `404` (`VIDEO_NOT_FOUND`) e para vídeo fora de `status = "draft"` retorna `409` (`INVALID_VIDEO_STATE`)

---

### SI-03.8 — Endpoint POST /videos/:id/complete

**Route:** POST /videos/:id/complete
**Test Specs:** see `nestjs-project/specs/videos-complete.plan.md`
**Authorization:** Owner

**Description:** Expor `POST /videos/:id/complete` para finalizar a montagem multipart no S3/MinIO, transicionar o vídeo para `status = "processing"` e publicar o job `process-video` para processamento automático pelo `video-worker`.

**Technical actions:**

1. Implementar `@Post(':id/complete')` (`@HttpCode(200)`, `@ApiBearerAuth('access-token')`, `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `200`, `400`, `401`, `403`, `404` e `409`) em `VideosController`, validando `id` (`ParseUUIDPipe`) e `CompleteUploadDto` e delegando a `VideosService.completeUpload(user.sub, id, dto)` conforme `### API Contracts → POST /videos/:id/complete` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-03`, `phase-03-videos/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.5, SI-03.6

**Acceptance criteria:**

- `POST /videos/:id/complete` pelo dono após envio das partes ao MinIO retorna `200` com `status: "processing"` e URLs do vídeo, e o processamento em segundo plano evolui o registro no banco para `status = "ready"` com `duration_seconds`, dimensões e `thumbnail_key` populados
- `POST /videos/:id/complete` com arquivo corrompido enviado ao MinIO retorna `200` (`status: "processing"`) e o processamento assíncrono marca o registro como `status = "failed"` com `error_reason` preenchido
- `POST /videos/:id/complete` sem JWT retorna `401` e por usuário autenticado que não é dono do vídeo retorna `403` com `error: "VIDEO_FORBIDDEN"`
- `POST /videos/:id/complete` com `parts` vazio/malformado retorna `400` (`VALIDATION_ERROR`) e com `ETag` inválido rejeitado pelo MinIO retorna `400` (`MULTIPART_COMPLETION_FAILED`)
- `POST /videos/:id/complete` sobre vídeo já fora de `status = "draft"` retorna `409` com `error: "INVALID_VIDEO_STATE"`

---

### SI-03.9 — Endpoint DELETE /videos/:id/upload

**Route:** DELETE /videos/:id/upload
**Test Specs:** see `nestjs-project/specs/videos-upload-abort.plan.md`
**Authorization:** Owner

**Description:** Expor `DELETE /videos/:id/upload` para abortar uploads multipart órfãos no S3/MinIO e remover o rascunho incompleto do PostgreSQL.

**Technical actions:**

1. Implementar `@Delete(':id/upload')` (`@HttpCode(204)`, `@ApiBearerAuth('access-token')`, `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `204` sem `schema`, `400`, `401`, `403`, `404` e `409`) em `VideosController`, validando `id` (`ParseUUIDPipe`) e delegando a `VideosService.abortUpload(user.sub, id)` conforme `### API Contracts → DELETE /videos/:id/upload` (per `phase-03-videos/TD-03`, `phase-03-videos/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.6

**Acceptance criteria:**

- `DELETE /videos/:id/upload` pelo dono de um rascunho retorna `204` sem body, aborta o upload multipart no MinIO e remove o registro de `videos`
- `DELETE /videos/:id/upload` sem JWT retorna `401` e por usuário autenticado não-dono retorna `403` com `error: "VIDEO_FORBIDDEN"`
- `DELETE /videos/:id/upload` com `id` inválido retorna `400` (`VALIDATION_ERROR`) e para vídeo inexistente retorna `404` (`VIDEO_NOT_FOUND`)
- `DELETE /videos/:id/upload` sobre vídeo já em `processing` ou `ready` retorna `409` com `error: "INVALID_VIDEO_STATE"`

---

### SI-03.10 — Endpoint GET /videos/:shortCode

**Route:** GET /videos/:shortCode
**Test Specs:** see `nestjs-project/specs/videos-get-by-shortcode.plan.md`
**Authorization:** Anonymous

**Description:** Expor o endpoint público `GET /videos/:shortCode` retornando metadados, status do ciclo de processamento e URLs únicas do vídeo sem exigir autenticação.

**Technical actions:**

1. Implementar `@Get(':shortCode')` (`@Public()`, `@SkipThrottle()`, sem `@ApiBearerAuth`, com `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `200` e `404`) em `VideosController`, delegando a `VideosService.findByShortCode(shortCode)` conforme `### API Contracts → GET /videos/:shortCode` (per `phase-03-videos/TD-06`, `phase-02-auth/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.6

**Acceptance criteria:**

- `GET /videos/:shortCode` sem token JWT para um vídeo em `status = "ready"` retorna `200` com `shortCode`, `title`, `status: "ready"`, `durationSeconds`, `width`, `height`, `videoCodec`, `streamUrl`, `downloadUrl` e `thumbnailUrl` preenchidos
- `GET /videos/:shortCode` para um vídeo em `draft` ou `processing` retorna `200` com `status` atual e `streamUrl: null`, `downloadUrl: null` e `thumbnailUrl: null`
- Dois vídeos criados pelo mesmo ou por diferentes canais recebem `shortCode` distintos e cada `GET /videos/:shortCode` resolve exclusivamente o respectivo vídeo sem conflito
- `GET /videos/:shortCode` para `shortCode` inexistente retorna `404` com `error: "VIDEO_NOT_FOUND"`

---

### SI-03.11 — Endpoint GET /videos/:shortCode/stream

**Route:** GET /videos/:shortCode/stream
**Test Specs:** see `nestjs-project/specs/videos-stream.plan.md`
**Authorization:** Anonymous

**Description:** Expor o endpoint público `GET /videos/:shortCode/stream` para reprodução progressiva via HTTP `Range` (`206 Partial Content`) não-bufferizada a partir do S3/MinIO, sem exigir download integral do arquivo.

**Technical actions:**

1. Implementar `@Get(':shortCode/stream')` (`@Public()`, `@SkipThrottle()`, sem `@ApiBearerAuth`, com `@ApiOperation`, `@ApiParam`, `@ApiHeader` para `Range` opcional e `@ApiResponse` para `206`, `200`, `404`, `409` e `416`) em `VideosController`, injetando `@Headers('range')` e `@Res({ passthrough: true })`, definindo status `206` + `Content-Range` + `Accept-Ranges: bytes` + `Content-Length` quando `Range` está presente (ou `200` + `Accept-Ranges: bytes` + `Content-Length` quando ausente) e retornando `StreamableFile` sobre o `Readable` de `VideosService.openVideoStream` conforme `### API Contracts → GET /videos/:shortCode/stream` (per `phase-03-videos/TD-07`, `phase-02-auth/TD-08`, `openapi-docs-nestjs/TD-01`).
2. Garantir que `DomainExceptionFilter` emita o cabeçalho RFC 9110 `Content-Range: bytes */<sizeBytes>` quando capturar `VideoRangeNotSatisfiableException` (`416`), mantendo o controller livre de blocos `try/catch` (per `phase-02-auth/TD-07`, `phase-03-videos/TD-07`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.8

**Acceptance criteria:**

- `GET /videos/:shortCode/stream` anônimo com cabeçalho `Range: bytes=0-511` para um vídeo `ready` retorna `206 Partial Content` com `Accept-Ranges: bytes`, `Content-Range: bytes 0-511/<sizeBytes>`, `Content-Length: 512` e exatamente os 512 primeiros bytes do vídeo
- `GET /videos/:shortCode/stream` anônimo sem cabeçalho `Range` retorna `200` com `Accept-Ranges: bytes`, `Content-Type` do vídeo, `Content-Length: <sizeBytes>` e o conteúdo binário completo via stream
- `GET /videos/:shortCode/stream` com faixa inválida (ex.: `Range: bytes=999999999-1000000000` além de `sizeBytes`) retorna `416` com `error: "VIDEO_RANGE_NOT_SATISFIABLE"` e cabeçalho `Content-Range: bytes */<sizeBytes>`
- `GET /videos/:shortCode/stream` para vídeo em `draft`, `processing` ou `failed` retorna `409` com `error: "VIDEO_NOT_READY"`, e para `shortCode` inexistente retorna `404` (`VIDEO_NOT_FOUND`)

---

### SI-03.12 — Endpoint GET /videos/:shortCode/download

**Route:** GET /videos/:shortCode/download
**Test Specs:** see `nestjs-project/specs/videos-download.plan.md`
**Authorization:** Anonymous

**Description:** Expor o endpoint público `GET /videos/:shortCode/download` para download não-bufferizado do arquivo de vídeo original com cabeçalho `Content-Disposition: attachment`.

**Technical actions:**

1. Implementar `@Get(':shortCode/download')` (`@Public()`, `@SkipThrottle()`, sem `@ApiBearerAuth`, com `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `200`, `404` e `409`) em `VideosController`, delegando a `VideosService.openVideoDownload(shortCode)` e retornando `StreamableFile` configurado com `type`, `length` e `disposition: 'attachment; filename="<sanitized>"'` conforme `### API Contracts → GET /videos/:shortCode/download` (per `phase-03-videos/TD-07`, `phase-02-auth/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.8

**Acceptance criteria:**

- `GET /videos/:shortCode/download` anônimo para vídeo em `status = "ready"` retorna `200` com `Content-Disposition: attachment; filename="..."`, `Content-Type` do vídeo, `Content-Length: <sizeBytes>` e os bytes idênticos ao arquivo enviado
- `GET /videos/:shortCode/download` para vídeo em `draft`, `processing` ou `failed` retorna `409` com `error: "VIDEO_NOT_READY"`
- `GET /videos/:shortCode/download` para `shortCode` inexistente retorna `404` com `error: "VIDEO_NOT_FOUND"`

---

### SI-03.13 — Endpoint GET /videos/:shortCode/thumbnail

**Route:** GET /videos/:shortCode/thumbnail
**Test Specs:** see `nestjs-project/specs/videos-thumbnail.plan.md`
**Authorization:** Anonymous

**Description:** Expor o endpoint público `GET /videos/:shortCode/thumbnail` para servir a imagem JPEG de thumbnail gerada automaticamente pelo worker a partir de um frame do vídeo.

**Technical actions:**

1. Implementar `@Get(':shortCode/thumbnail')` (`@Public()`, `@SkipThrottle()`, sem `@ApiBearerAuth`, com `@ApiOperation`, `@ApiParam` e `@ApiResponse` para `200`, `404` e `409`) em `VideosController`, definindo `Cache-Control: public, max-age=86400` e retornando `StreamableFile` (`type: 'image/jpeg'`) sobre `VideosService.openThumbnailStream(shortCode)` conforme `### API Contracts → GET /videos/:shortCode/thumbnail` (per `phase-03-videos/TD-05`, `phase-03-videos/TD-07`, `phase-02-auth/TD-08`, `openapi-docs-nestjs/TD-01`).

**Tests:** _(empty — E2E scenarios externalized to /plan-test-specs spec)_

**Dependencies:** SI-03.8

**Acceptance criteria:**

- `GET /videos/:shortCode/thumbnail` anônimo para vídeo em `status = "ready"` retorna `200` com `Content-Type: image/jpeg`, `Cache-Control: public, max-age=86400` e um payload JPEG válido (assinatura binária `FF D8 FF`)
- `GET /videos/:shortCode/thumbnail` para vídeo em `draft`, `processing` ou `failed` retorna `409` com `error: "VIDEO_NOT_READY"`
- `GET /videos/:shortCode/thumbnail` para `shortCode` inexistente retorna `404` (`VIDEO_NOT_FOUND`) e para vídeo `ready` sem `thumbnail_key` retorna `404` (`VIDEO_THUMBNAIL_NOT_FOUND`)

---

### SI-03.14 — Exportação e Sincronização do Artefato OpenAPI (openapi.json)

**Description:** Atualizar `buildSwaggerDocument` (se aplicável para tag `videos`), regerar o artefato estático `nestjs-project/openapi.json` via `npm run openapi:export` e estender os testes de contrato OpenAPI cobrindo todos os endpoints de vídeos.

**Technical actions:**

1. Adicionar a tag `'videos'` em `nestjs-project/src/swagger/swagger-document.ts` (se as tags forem enumeradas no `DocumentBuilder`) e executar `npm run openapi:export` dentro do container `nestjs-api` para atualizar `nestjs-project/openapi.json` com as 8 operações e DTOs de `VideosController` (per `openapi-docs-nestjs/TD-01`, `openapi-docs-nestjs/TD-02`).
2. Estender `nestjs-project/src/openapi-export.integration-spec.ts` e `nestjs-project/test/swagger.e2e-spec.ts` verificando que os paths `/videos/uploads`, `/videos/{id}/parts/{partNumber}/presign`, `/videos/{id}/complete`, `/videos/{id}/upload`, `/videos/{shortCode}`, `/videos/{shortCode}/stream`, `/videos/{shortCode}/download` e `/videos/{shortCode}/thumbnail`, os esquemas de autenticação Bearer nas rotas protegidas (e ausência nas rotas `@Public()`) e as referências a `ApiErrorEnvelope` estão presentes e sincronizados em `openapi.json` (per `openapi-docs-nestjs/TD-01`, `openapi-docs-nestjs/TD-02`, `openapi-docs-nestjs/TD-03`).

**Tests:**

| Artifact | Layer | Test file |
|----------|-------|-----------|
| `openapi.json` export | Integration: valida presença das 8 rotas `/videos/*`, schemas de DTOs, segurança Bearer apenas nas 4 rotas autenticadas e frescor do arquivo `openapi.json` | `src/openapi-export.integration-spec.ts` |
| Swagger runtime JSON | E2E: `GET /api/docs-json` em dev expõe os mesmos contratos de `videos` e referencia `ApiErrorEnvelope` | `test/swagger.e2e-spec.ts` |

**Dependencies:** SI-03.6, SI-03.7, SI-03.8, SI-03.9, SI-03.10, SI-03.11, SI-03.12, SI-03.13

**Acceptance criteria:**

- Executar `npm run openapi:export` não deixa diff pendente em `nestjs-project/openapi.json`
- `nestjs-project/openapi.json` documenta as 8 operações de `/videos/*` com códigos de sucesso, respostas de erro referenciando `ApiErrorEnvelope` e `bearerAuth` restrito apenas aos 4 endpoints protegidos
- `GET /api/docs-json` em ambiente de desenvolvimento retorna especificação OpenAPI 3.x contendo a tag `videos` e todos os DTOs da Fase 03

---

## Technical Specifications

### Data Model

#### Video

Tabela PostgreSQL: `videos` (`@Entity('videos')` em `nestjs-project/src/videos/entities/video.entity.ts`).

| Field | Type | Constraints |
|-------|------|-------------|
| id | uuid | PK, generated (`uuid_generate_v4()`) |
| channel_id | uuid | FK → `channels.id` (`ON DELETE CASCADE`), not null |
| short_code | varchar(16) | unique, not null — 11-char CSPRNG URL-safe `[A-Za-z0-9_-]` (`phase-03-videos/TD-06`) |
| title | varchar(255) | not null |
| status | enum (`videos_status_enum`: `'draft' \| 'processing' \| 'ready' \| 'failed'`) | not null, default `'draft'` (`phase-03-videos/TD-08`) |
| original_filename | varchar(255) | not null |
| mime_type | varchar(100) | not null |
| size_bytes | bigint | not null — suporta até `10737418240` bytes (10 GiB) (`phase-03-videos/TD-03`) |
| storage_key | varchar(512) | not null — formato `videos/{channelId}/{videoId}/source.{ext}` (`phase-03-videos/TD-02`) |
| upload_id | varchar(512) | nullable, `select: false` — ID da sessão S3 Multipart ativa enquanto `status = 'draft'` (`phase-03-videos/TD-03`) |
| thumbnail_key | varchar(512) | nullable — formato `thumbnails/{channelId}/{videoId}/thumb.jpg` (`phase-03-videos/TD-02`, `TD-05`) |
| duration_seconds | int | nullable — extraído via `ffprobe` (`phase-03-videos/TD-05`) |
| width | int | nullable — largura em pixels extraída via `ffprobe` (`phase-03-videos/TD-05`) |
| height | int | nullable — altura em pixels extraída via `ffprobe` (`phase-03-videos/TD-05`) |
| video_codec | varchar(64) | nullable — codec de vídeo extraído via `ffprobe` (ex.: `h264`, `vp9`, `av1`) (`phase-03-videos/TD-05`) |
| error_reason | text | nullable — razão diagnóstica preenchida quando `status = 'failed'` (`phase-03-videos/TD-08`) |
| created_at | timestamptz | not null, `@CreateDateColumn()` |
| updated_at | timestamptz | not null, `@UpdateDateColumn()` |

**Relations:** `Video` pertence a um `Channel` (`@ManyToOne(() => Channel, (channel) => channel.videos, { onDelete: 'CASCADE' })` com `@JoinColumn({ name: 'channel_id' })`).
**Indexes:** unique em `short_code`; index em `channel_id`; index em `status`.

#### Channel (modified)

Tabela PostgreSQL: `channels` (`nestjs-project/src/channels/entities/channel.entity.ts` — entidade existente da Fase 02).

| Field | Type | Constraints |
|-------|------|-------------|
| id | uuid | PK, generated (existente) |
| name | varchar(50) | not null (existente) |
| nickname | varchar(50) | unique, not null (existente) |
| description | text | nullable (existente) |
| user_id | uuid | unique, FK → `users.id`, not null (existente) |
| created_at | timestamptz | `@CreateDateColumn()` (existente) |
| updated_at | timestamptz | `@UpdateDateColumn()` (existente) |

**Relations:** `Channel` possui um `User` (1:1 existente) e possui muitos `Video` (`@OneToMany(() => Video, (video) => video.channel) videos: Video[]` — lado inverso bidirecional sem nova coluna física em `channels`).
**Indexes:** inalterados em `channels` (`nickname` unique, `user_id` unique).

### API Contracts

#### POST /videos/uploads (SI-03.6)

**Request headers:**
- Authorization: Bearer `<access_token>` (required)
- Content-Type: application/json

**Request body:**
- title: string, optional — max 255 characters; se omitido ou em branco, deriva do `originalFilename` sem extensão (truncado a 255 chars)
- originalFilename: string, required — non-empty, max 255 characters (ex.: `"aula-01.mp4"`)
- mimeType: string, required — um de `"video/mp4" | "video/webm" | "video/quicktime" | "video/x-matroska"`
- sizeBytes: number, required — inteiro positivo `>= 1` (suporta até `10737418240` bytes = 10 GiB; valores `> 10737418240` retornam `413 VIDEO_TOO_LARGE` pela regra de domínio)

**Response 201:**
- videoId: string (uuid)
- shortCode: string (11 chars URL-safe)
- status: `"draft"`
- uploadId: string
- partSizeBytes: number (default `10485760` = 10 MiB; ajustado automaticamente se necessário para manter `totalParts <= 10000` dentro do limite S3)
- totalParts: number
- expiresInSeconds: number
- parts: array of `{ partNumber: number, uploadUrl: string }`

**Error responses:**
- 400 VALIDATION_ERROR: quando o body falha na validação `class-validator` (`sizeBytes < 1`, `originalFilename` ausente ou campos desconhecidos)
- 401 Unauthorized: quando o JWT de acesso está ausente, inválido ou expirado
- 404 CHANNEL_NOT_FOUND: quando o usuário autenticado não possui canal associado
- 413 VIDEO_TOO_LARGE: quando `sizeBytes > 10737418240` (10 GiB)
- 415 UNSUPPORTED_VIDEO_MIME_TYPE: quando `mimeType` não pertence à lista permitida de containers de vídeo

---

#### POST /videos/:id/parts/:partNumber/presign (SI-03.7)

**Request headers:**
- Authorization: Bearer `<access_token>` (required; `@SkipThrottle()` ativo para suportar retomada de uploads grandes de até 1.024+ partes)

**Request path parameters:**
- id: string (uuid), required — ID do vídeo em rascunho
- partNumber: integer, required — `1 <= partNumber <= 10000` (e `<= totalParts` calculado para `size_bytes`)

**Response 200:**
- partNumber: number
- uploadUrl: string
- expiresInSeconds: number

**Error responses:**
- 400 VALIDATION_ERROR: quando `id` não é UUID válido ou `partNumber` está fora do intervalo permitido
- 401 Unauthorized: quando o JWT está ausente ou inválido
- 403 VIDEO_FORBIDDEN: quando o vídeo pertence ao canal de outro usuário
- 404 VIDEO_NOT_FOUND: quando o vídeo não existe
- 409 INVALID_VIDEO_STATE: quando `status !== "draft"` ou não há `upload_id` ativo

---

#### POST /videos/:id/complete (SI-03.8)

**Request headers:**
- Authorization: Bearer `<access_token>` (required)
- Content-Type: application/json

**Request path parameters:**
- id: string (uuid), required — ID do vídeo em rascunho

**Request body:**
- parts: array (non-empty, max 10000 items), required — cada item `{ partNumber: integer (>= 1), etag: string (non-empty) }`

**Response 200:**
- id: string (uuid)
- shortCode: string
- title: string
- status: `"processing"`
- sizeBytes: number
- mimeType: string
- watchUrl: string (`/videos/{shortCode}`)
- streamUrl: string (`/videos/{shortCode}/stream`)
- downloadUrl: string (`/videos/{shortCode}/download`)
- thumbnailUrl: string | null (`/videos/{shortCode}/thumbnail` quando disponível, `null` enquanto processa)
- createdAt: string (ISO-8601)
- updatedAt: string (ISO-8601)

**Error responses:**
- 400 VALIDATION_ERROR: quando `id` não é UUID válido ou `parts` está vazio/malformado
- 400 MULTIPART_COMPLETION_FAILED: quando o S3/MinIO rejeita a montagem das partes (`InvalidPart` / `InvalidPartOrder` / `EntityTooSmall`)
- 401 Unauthorized: quando o JWT está ausente ou inválido
- 403 VIDEO_FORBIDDEN: quando o vídeo pertence ao canal de outro usuário
- 404 VIDEO_NOT_FOUND: quando o vídeo não existe
- 409 INVALID_VIDEO_STATE: quando `status !== "draft"`

---

#### DELETE /videos/:id/upload (SI-03.9)

**Request headers:**
- Authorization: Bearer `<access_token>` (required)

**Request path parameters:**
- id: string (uuid), required — ID do vídeo em rascunho cujo upload multipart será abortado

**Response 204:** No content.

**Error responses:**
- 400 VALIDATION_ERROR: quando `id` não é UUID válido
- 401 Unauthorized: quando o JWT está ausente ou inválido
- 403 VIDEO_FORBIDDEN: quando o vídeo pertence ao canal de outro usuário
- 404 VIDEO_NOT_FOUND: quando o vídeo não existe
- 409 INVALID_VIDEO_STATE: quando `status !== "draft"`

---

#### GET /videos/:shortCode (SI-03.10)

**Request headers:**
- Nenhuma autenticação exigida (`@Public()`, `@SkipThrottle()`)

**Request path parameters:**
- shortCode: string, required — identificador único de 11 caracteres (`^[A-Za-z0-9_-]{11}$`)

**Response 200:**
- id: string (uuid)
- channelId: string (uuid)
- shortCode: string
- title: string
- status: `"draft" | "processing" | "ready" | "failed"`
- originalFilename: string
- mimeType: string
- sizeBytes: number
- durationSeconds: number | null
- width: number | null
- height: number | null
- videoCodec: string | null
- watchUrl: string (`/videos/{shortCode}`)
- streamUrl: string | null (`/videos/{shortCode}/stream` quando `status === "ready"`, `null` caso contrário)
- downloadUrl: string | null (`/videos/{shortCode}/download` quando `status === "ready"`, `null` caso contrário)
- thumbnailUrl: string | null (`/videos/{shortCode}/thumbnail` quando `status === "ready"` e `thumbnail_key !== null`, `null` caso contrário)
- createdAt: string (ISO-8601)
- updatedAt: string (ISO-8601)

**Error responses:**
- 404 VIDEO_NOT_FOUND: quando nenhum vídeo corresponde ao `shortCode` informado

---

#### GET /videos/:shortCode/stream (SI-03.11)

**Request headers:**
- Range: string, optional — cabeçalho RFC 9110 de faixa única de bytes (ex.: `bytes=0-1048575`, `bytes=524288-`, `bytes=-65536`). Rota pública (`@Public()`, `@SkipThrottle()`).

**Request path parameters:**
- shortCode: string, required — identificador único do vídeo

**Response 206 (quando `Range` válido é enviado):**
- Headers: `Content-Type: <video.mime_type>`, `Accept-Ranges: bytes`, `Content-Range: bytes <start>-<end>/<sizeBytes>`, `Content-Length: <end - start + 1>`
- Body: stream binário não-bufferizado (`StreamableFile` sobre `Readable` do S3 `GetObjectCommand`)

**Response 200 (fallback quando `Range` está ausente):**
- Headers: `Content-Type: <video.mime_type>`, `Accept-Ranges: bytes`, `Content-Length: <sizeBytes>`
- Body: stream binário integral não-bufferizado (`StreamableFile`)

**Error responses:**
- 404 VIDEO_NOT_FOUND: quando nenhum vídeo corresponde ao `shortCode`
- 409 VIDEO_NOT_READY: quando `status !== "ready"` (`draft`, `processing` ou `failed`)
- 416 VIDEO_RANGE_NOT_SATISFIABLE: quando o cabeçalho `Range` é sintaticamente inválido ou excede `0..sizeBytes-1` (inclui `Content-Range: bytes */<sizeBytes>`)

---

#### GET /videos/:shortCode/download (SI-03.12)

**Request headers:**
- Nenhuma autenticação exigida (`@Public()`, `@SkipThrottle()`)

**Request path parameters:**
- shortCode: string, required — identificador único do vídeo

**Response 200:**
- Headers: `Content-Type: <video.mime_type>`, `Content-Length: <sizeBytes>`, `Content-Disposition: attachment; filename="<sanitized-original-filename>"`
- Body: stream binário integral não-bufferizado (`StreamableFile` sobre `Readable` do S3 `GetObjectCommand`)

**Error responses:**
- 404 VIDEO_NOT_FOUND: quando nenhum vídeo corresponde ao `shortCode`
- 409 VIDEO_NOT_READY: quando `status !== "ready"`

---

#### GET /videos/:shortCode/thumbnail (SI-03.13)

**Request headers:**
- Nenhuma autenticação exigida (`@Public()`, `@SkipThrottle()`)

**Request path parameters:**
- shortCode: string, required — identificador único do vídeo

**Response 200:**
- Headers: `Content-Type: image/jpeg`, `Content-Length: <bytes>`, `Cache-Control: public, max-age=86400`
- Body: stream binário JPEG (`StreamableFile` sobre `Readable` do S3 `GetObjectCommand`)

**Error responses:**
- 404 VIDEO_NOT_FOUND: quando nenhum vídeo corresponde ao `shortCode`
- 404 VIDEO_THUMBNAIL_NOT_FOUND: quando `thumbnail_key` é nulo
- 409 VIDEO_NOT_READY: quando `status !== "ready"`

---

#### Validation Rules — Video Upload & Multipart

- `originalFilename`: obrigatório, string não-vazia pós-trim, máximo `255` caracteres
- `title`: opcional, string pós-trim com máximo `255` caracteres; quando omitido, derivado do basename de `originalFilename` sem extensão
- `mimeType`: obrigatório, string; validado no domínio contra `["video/mp4", "video/webm", "video/quicktime", "video/x-matroska"]` (`415 UNSUPPORTED_VIDEO_MIME_TYPE` fora da lista)
- `sizeBytes`: obrigatório, inteiro `>= 1` no DTO (`@IsInt()`, `@Min(1)`); limite máximo de negócio `<= VIDEO_MAX_SIZE_BYTES` (`10737418240` = 10 GiB) validado no serviço para retornar semanticamente `413 VIDEO_TOO_LARGE`
- `parts`: obrigatório em `POST /videos/:id/complete`, array não-vazio (`@ArrayMinSize(1)`, `@ArrayMaxSize(10000)`), cada item validado com `@ValidateNested()` contendo `partNumber` (`@IsInt()`, `@Min(1)`, `@Max(10000)`) sem duplicatas e `etag` (`@IsString()`, `@IsNotEmpty()`)

### Authorization Matrix

| Endpoint | Anonymous | Authenticated | Owner |
|----------|-----------|---------------|-------|
| POST /videos/uploads | ✗ | ✓ | ✓ (cria vídeo no próprio canal) |
| POST /videos/:id/parts/:partNumber/presign | ✗ | ✗ | ✓ (dono do canal do vídeo) |
| POST /videos/:id/complete | ✗ | ✗ | ✓ (dono do canal do vídeo) |
| DELETE /videos/:id/upload | ✗ | ✗ | ✓ (dono do canal do vídeo) |
| GET /videos/:shortCode | ✓ | ✓ | ✓ |
| GET /videos/:shortCode/stream | ✓ | ✓ | ✓ |
| GET /videos/:shortCode/download | ✓ | ✓ | ✓ |
| GET /videos/:shortCode/thumbnail | ✓ | ✓ | ✓ |

### Error Catalog

Todos os erros de domínio herdam o envelope padrão `{ statusCode, error, message }` (`ApiErrorEnvelope`) estabelecido em `phase-02-auth/TD-07` via `DomainExceptionFilter` e `ValidationExceptionFilter`.

| errorCode | HTTP | Trigger |
|-----------|------|---------|
| VALIDATION_ERROR | 400 | Payload, query ou parâmetro de rota inválido rejeitado pelo `ValidationPipe` |
| MULTIPART_COMPLETION_FAILED | 400 | S3/MinIO rejeitou a finalização do multipart upload (`ETag` inválido, parte ausente ou tamanho inconsistente) |
| CHANNEL_NOT_FOUND | 404 | Usuário autenticado não possui canal cadastrado ao iniciar upload |
| VIDEO_NOT_FOUND | 404 | Nenhum vídeo encontrado para o `id` ou `shortCode` informado |
| VIDEO_THUMBNAIL_NOT_FOUND | 404 | Vídeo em estado `ready`, porém sem `thumbnail_key` associada |
| VIDEO_FORBIDDEN | 403 | Usuário autenticado tenta operar upload (`presign`, `complete`, `abort`) de vídeo pertencente ao canal de outro usuário |
| INVALID_VIDEO_STATE | 409 | Operação de upload (`presign`, `complete`, `abort`) invocada quando `status !== "draft"` |
| VIDEO_NOT_READY | 409 | Tentativa de reproduzir (`/stream`), baixar (`/download`) ou ler thumbnail (`/thumbnail`) de vídeo com `status !== "ready"` |
| VIDEO_TOO_LARGE | 413 | `sizeBytes` excede o teto máximo permitido de `10737418240` bytes (10 GiB) |
| UNSUPPORTED_VIDEO_MIME_TYPE | 415 | `mimeType` informado não pertence aos formatos de vídeo suportados (`video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`) |
| VIDEO_RANGE_NOT_SATISFIABLE | 416 | Cabeçalho HTTP `Range` malformado ou fora dos limites `0..sizeBytes-1` do vídeo |

### Events/Messages

#### process-video (Queue: `video-processing`)

**Payload:**

```json
{
  "videoId": "uuid"
}
```

**Producer:** `VideosService` em `nestjs-project/src/videos/videos.service.ts` via `@InjectQueue('video-processing')` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-03`)
**Consumer:** `VideoProcessor` em `nestjs-project/src/worker/video.processor.ts` executando no container standalone `video-worker` via `NestFactory.createApplicationContext(WorkerModule)` (per `phase-03-videos/TD-01`, `phase-03-videos/TD-04`, `phase-03-videos/TD-05`)
**Trigger:** disparado imediatamente em `POST /videos/:id/complete` após o S3/MinIO concluir o `CompleteMultipartUploadCommand` e o registro `Video` transicionar de `'draft'` para `'processing'`
**Delivery semantics:** `at-least-once` com deduplicação idempotente por `jobId: videoId`, `attempts: 3` e backoff exponencial de `1000ms` (1s, 2s, 4s) para falhas transitórias; `UnrecoverableError` aborta retentativas imediatamente em mídia corrompida/objeto ausente e `@OnWorkerEvent('failed')` persiste `status = 'failed'` e `error_reason` na falha terminal (per `phase-03-videos/TD-08`)

---

## Dependency Map

```text
SI-03.1 (root — Infra: dependências, FFmpeg, configs namespaced e MinIO/Redis no Compose)
├── SI-03.2 — depends on SI-03.1 (entidade Video, relação Channel, exceções de domínio e migration CreateVideos)
├── SI-03.3 — depends on SI-03.1 (StorageModule e StorageService S3/MinIO)
└── SI-03.4 — depends on SI-03.2, SI-03.3 (utilitários short_code/Range, DTOs, VideosService e VideosModule producer)
    ├── SI-03.5 — depends on SI-03.4 (MediaProbeService FFmpeg/ffprobe, VideoProcessor e container video-worker)
    └── SI-03.6 — depends on SI-03.4 (VideosController + POST /videos/uploads)
        ├── SI-03.7 — depends on SI-03.6 (POST /videos/:id/parts/:partNumber/presign)
        ├── SI-03.8 — depends on SI-03.5, SI-03.6 (POST /videos/:id/complete + processamento assíncrono no worker)
        │   ├── SI-03.11 — depends on SI-03.8 (GET /videos/:shortCode/stream — HTTP 206 Partial Content)
        │   ├── SI-03.12 — depends on SI-03.8 (GET /videos/:shortCode/download — attachment)
        │   └── SI-03.13 — depends on SI-03.8 (GET /videos/:shortCode/thumbnail — image/jpeg)
        ├── SI-03.9 — depends on SI-03.6 (DELETE /videos/:id/upload)
        └── SI-03.10 — depends on SI-03.6 (GET /videos/:shortCode)
            └── SI-03.14 — depends on SI-03.6..SI-03.13 (exportação e sincronização de nestjs-project/openapi.json)
```

---

## Deliverables

- [ ] SI-03.1 — Infra: Dependências, FFmpeg no Container, Configurações Namespaced e Serviços MinIO/Redis
- [ ] SI-03.2 — Entidade Video, Relação Bidirecional com Channel, Exceções de Domínio e Migration CreateVideos
- [ ] SI-03.3 — StorageModule e StorageService (S3/MinIO Multipart, Presigning e Streams)
- [ ] SI-03.4 — Utilitários de Short Code e Range, DTOs, VideosService e VideosModule (Producer)
- [ ] SI-03.5 — Pipeline FFmpeg/ffprobe, VideoProcessor e Container Standalone video-worker
- [ ] SI-03.6 — Endpoint POST /videos/uploads
- [ ] SI-03.7 — Endpoint POST /videos/:id/parts/:partNumber/presign
- [ ] SI-03.8 — Endpoint POST /videos/:id/complete
- [ ] SI-03.9 — Endpoint DELETE /videos/:id/upload
- [ ] SI-03.10 — Endpoint GET /videos/:shortCode
- [ ] SI-03.11 — Endpoint GET /videos/:shortCode/stream
- [ ] SI-03.12 — Endpoint GET /videos/:shortCode/download
- [ ] SI-03.13 — Endpoint GET /videos/:shortCode/thumbnail
- [ ] SI-03.14 — Exportação e Sincronização do Artefato OpenAPI (openapi.json)

**Full test suites:**

- [ ] Migrations apply cleanly (`cd nestjs-project && docker compose exec -T nestjs-api npm run migration:run`)
- [ ] Backend unit + integration tests pass (`cd nestjs-project && docker compose exec -T nestjs-api npm test`)
- [ ] Backend E2E tests pass (`cd nestjs-project && docker compose exec -T nestjs-api npm run test:e2e`)
- [ ] Type/compilation checks pass (`cd nestjs-project && docker compose exec -T nestjs-api npx tsc --noEmit`)
- [ ] Project builds successfully (`cd nestjs-project && docker compose exec -T nestjs-api npm run build`)
- [ ] Lint passes (`cd nestjs-project && docker compose exec -T nestjs-api npm run lint`)
- [ ] OpenAPI artifact synchronized (`cd nestjs-project && docker compose exec -T nestjs-api npm run openapi:export`)

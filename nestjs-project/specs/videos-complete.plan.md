---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.8
target_file: test/videos-complete.e2e-spec.ts
---

# POST /videos/:id/complete — Test Plan

## Application Overview

`POST /videos/:id/complete` finaliza o S3 Multipart Upload no MinIO, transiciona o vídeo de `draft` para `processing` e publica o job `process-video` na fila Redis/BullMQ real `video-processing` (`phase-03-videos/TD-01`, `TD-03`, `TD-08`). O container `video-worker` consome o job, extrai metadados com `ffprobe`, gera thumbnail com `ffmpeg` e marca o vídeo como `ready` — ou `failed` quando a mídia é inválida (`phase-03-videos/TD-04`, `TD-05`).

## Test Scenarios

### 1. Finalizar upload multipart e processar vídeo em segundo plano

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais e gera uma fixture `.mp4` válida em memória (`Buffer`) via `execFile('ffmpeg', [... '-f', 'mp4', 'pipe:1'], { encoding: 'buffer' })`; `beforeEach` executa `cleanAllTables()`, limpa `ThrottlerStorage`, cria usuário dono e não-dono confirmados, inicia upload via `POST /videos/uploads` e envia os bytes por `PUT` às URLs pré-assinadas do MinIO real coletando os `ETag`; os cenários de processamento aguardam (polling com timeout) o container `video-worker` em execução no Compose; `afterAll` fecha app e DataSource.

#### 1.1. completa-upload-e-worker-marca-ready

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/:id/complete com token do dono e body `{ "parts": [{ "partNumber": 1, "etag": "<etag>" }] }`
    - expect: HTTP `200` com `status: "processing"`, `shortCode`, `streamUrl`, `downloadUrl` e `thumbnailUrl: null`
  2. Aguardar (polling em `videos` até 60s) o processamento pelo `video-worker`
    - expect: registro passa a `status = 'ready'` com `duration_seconds > 0`, `width = 320`, `height = 240`, `video_codec = 'h264'` e `thumbnail_key` em `thumbnails/{channelId}/{videoId}/thumb.jpg`

#### 1.2. arquivo-corrompido-worker-marca-failed

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. Enviar bytes aleatórios (não-vídeo) às URLs pré-assinadas e chamar POST /videos/:id/complete com os `ETag`
    - expect: HTTP `200` com `status: "processing"`
  2. Aguardar (polling até 60s) o processamento pelo `video-worker`
    - expect: registro passa a `status = 'failed'` com `error_reason` não-nulo

#### 1.3. rejeita-sem-token-e-nao-dono

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/:id/complete sem `Authorization`
    - expect: HTTP `401`
  2. POST /videos/:id/complete com token do não-dono
    - expect: HTTP `403` com `error: "VIDEO_FORBIDDEN"`

#### 1.4. rejeita-parts-invalidas-e-etag-rejeitado-pelo-storage

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/:id/complete com token do dono e `{ "parts": [] }`
    - expect: HTTP `400` com `error: "VALIDATION_ERROR"`
  2. POST /videos/:id/complete com token do dono e `{ "parts": [{ "partNumber": 1, "etag": "\"invalid-etag\"" }] }`
    - expect: HTTP `400` com `error: "MULTIPART_COMPLETION_FAILED"`
    - expect: registro permanece em `status = 'draft'`

#### 1.5. rejeita-video-fora-de-draft

**Covers AC:** #5
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. Completar o upload com sucesso e repetir POST /videos/:id/complete com os mesmos `parts`
    - expect: HTTP `409` com `error: "INVALID_VIDEO_STATE"`

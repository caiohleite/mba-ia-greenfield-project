---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.10
target_file: test/videos-get-by-shortcode.e2e-spec.ts
---

# GET /videos/:shortCode — Test Plan

## Application Overview

`GET /videos/:shortCode` expõe publicamente (`@Public()`, `@SkipThrottle()`) os metadados, o estado atual do ciclo de processamento (`draft`, `processing`, `ready` ou `failed`) e as URLs de acesso de um vídeo identificado por seu `shortCode` único de 11 caracteres URL-safe (`phase-03-videos/TD-06`). URLs de mídia (`streamUrl`, `downloadUrl`, `thumbnailUrl`) só são expostas quando `status === "ready"`.

## Test Scenarios

### 1. Consultar metadados e URL única de vídeo por shortCode

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais e gera uma fixture `.mp4` válida em memória (`Buffer`) via `execFile('ffmpeg', [... '-f', 'mp4', 'pipe:1'], { encoding: 'buffer' })`; `beforeEach` executa `cleanAllTables()` no PostgreSQL real (`db`) e limpa `ThrottlerStorage`; `afterAll` fecha app e DataSource.

#### 1.1. anonimo-consulta-video-ready-com-metadados-e-urls

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Usuário autenticado realiza upload completo (`POST /videos/uploads` → `PUT` no MinIO → `POST /videos/:id/complete`) e aguarda o `video-worker` marcar `status = 'ready'`
  2. Cliente anônimo (sem cabeçalho `Authorization`) chama GET /videos/:shortCode
    - expect: HTTP `200`
    - expect: body contém `id`, `channelId`, `shortCode`, `title`, `status: "ready"`, `durationSeconds > 0`, `width: 320`, `height: 240`, `videoCodec: "h264"`, `watchUrl: "/videos/<shortCode>"`, `streamUrl: "/videos/<shortCode>/stream"`, `downloadUrl: "/videos/<shortCode>/download"` e `thumbnailUrl: "/videos/<shortCode>/thumbnail"`

#### 1.2. consulta-video-draft-ou-processing-retorna-urls-de-midia-nulas

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Usuário autenticado inicia upload via POST /videos/uploads (`status = "draft"`) e cliente anônimo chama GET /videos/:shortCode
    - expect: HTTP `200` com `status: "draft"`, `streamUrl: null`, `downloadUrl: null` e `thumbnailUrl: null`
  2. Atualizar registro para `status = 'processing'` no banco e repetir GET /videos/:shortCode
    - expect: HTTP `200` com `status: "processing"`, `streamUrl: null`, `downloadUrl: null` e `thumbnailUrl: null`

#### 1.3. multiplos-videos-recebem-shortcodes-unicos-sem-colisao

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Dois canais distintos iniciam uploads (`POST /videos/uploads`) com títulos `"Vídeo A"` e `"Vídeo B"`
    - expect: `shortCode` do Vídeo A difere de `shortCode` do Vídeo B e ambos casam `^[A-Za-z0-9_-]{11}$`
  2. Cliente anônimo chama GET /videos/<shortCodeA> e GET /videos/<shortCodeB>
    - expect: cada requisição retorna `200` exclusivamente com o `id`, `channelId` e `title` do respectivo vídeo sem conflito

#### 1.4. retorna-404-para-shortcode-inexistente

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/AbCdEfGhIjK para código não cadastrado
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`

---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.12
target_file: test/videos-download.e2e-spec.ts
---

# GET /videos/:shortCode/download — Test Plan

## Application Overview

`GET /videos/:shortCode/download` permite que qualquer usuário (anônimo ou autenticado, `@Public()`, `@SkipThrottle()`) baixe o arquivo de vídeo original via stream não-bufferizado (`StreamableFile`) a partir do MinIO/S3 com cabeçalho `Content-Disposition: attachment` (`phase-03-videos/TD-07`). Apenas vídeos em `status = "ready"` podem ser baixados.

## Test Scenarios

### 1. Baixar arquivo de vídeo original

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais e gera uma fixture `.mp4` válida em memória (`Buffer` `fixtureBuffer`) via `execFile('ffmpeg', [... '-f', 'mp4', 'pipe:1'], { encoding: 'buffer' })`; `beforeEach` executa `cleanAllTables()` no PostgreSQL real (`db`) e limpa `ThrottlerStorage`; `afterAll` fecha app e DataSource.

#### 1.1. anonimo-baixa-video-ready-com-attachment-e-bytes-integros

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Usuário autenticado conclui upload multipart de `"aula-final.mp4"` (`fixtureBuffer`) e aguarda o `video-worker` marcar `status = 'ready'`
  2. Cliente anônimo chama GET /videos/:shortCode/download
    - expect: HTTP `200`
    - expect: cabeçalhos `Content-Disposition` contendo `attachment; filename="aula-final.mp4"`, `Content-Type: video/mp4` e `Content-Length: <fixtureBuffer.length>`
    - expect: body binário recebido é byte-a-byte idêntico a `fixtureBuffer`

#### 1.2. rejeita-download-de-video-nao-pronto-com-409

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Iniciar upload em `draft` e chamar GET /videos/:shortCode/download
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"`
  2. Atualizar registro para `processing` e depois `failed` e repetir GET /videos/:shortCode/download
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"` em ambos os estados

#### 1.3. retorna-404-para-shortcode-inexistente

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/NaoExiste11/download
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`

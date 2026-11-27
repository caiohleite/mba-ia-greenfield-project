---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.11
target_file: test/videos-stream.e2e-spec.ts
---

# GET /videos/:shortCode/stream — Test Plan

## Application Overview

`GET /videos/:shortCode/stream` entrega reprodução progressiva anônima (`@Public()`, `@SkipThrottle()`) via HTTP `Range` (`206 Partial Content`) diretamente de `GetObjectCommand` no MinIO/S3 sem bufferizar o arquivo completo em memória (`phase-03-videos/TD-07`). Suporta fallback `200 OK` quando `Range` está ausente, retorna `416` com `Content-Range: bytes */<sizeBytes>` em faixas inválidas e bloqueia vídeos fora de `ready` com `409 VIDEO_NOT_READY`.

## Test Scenarios

### 1. Reproduzir vídeo via streaming HTTP Range sem download completo

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais e gera uma fixture `.mp4` válida em memória (`Buffer` `fixtureBuffer`) via `execFile('ffmpeg', [... '-f', 'mp4', 'pipe:1'], { encoding: 'buffer' })`; `beforeEach` executa `cleanAllTables()` no PostgreSQL real (`db`) e limpa `ThrottlerStorage`; para cenários `ready`, realiza upload multipart completo no MinIO real e aguarda o `video-worker` concluir o processamento; `afterAll` fecha app e DataSource.

#### 1.1. anonimo-stream-parcial-range-retorna-206

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/:shortCode/stream com cabeçalho `Range: bytes=0-511` para um vídeo `ready`
    - expect: HTTP `206`
    - expect: cabeçalhos `Accept-Ranges: bytes`, `Content-Range: bytes 0-511/<fixtureBuffer.length>`, `Content-Length: 512` e `Content-Type: video/mp4`
    - expect: body binário tem exatamente `512` bytes idênticos a `fixtureBuffer.subarray(0, 512)`

#### 1.2. anonimo-stream-sem-range-retorna-200-completo

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/:shortCode/stream sem cabeçalho `Range` para um vídeo `ready`
    - expect: HTTP `200`
    - expect: cabeçalhos `Accept-Ranges: bytes`, `Content-Type: video/mp4` e `Content-Length: <fixtureBuffer.length>`
    - expect: body binário coincide integralmente com `fixtureBuffer`

#### 1.3. rejeita-range-fora-dos-limites-com-416

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/:shortCode/stream com cabeçalho `Range: bytes=999999999-1000000000` para um vídeo `ready`
    - expect: HTTP `416` com `error: "VIDEO_RANGE_NOT_SATISFIABLE"`
    - expect: resposta inclui cabeçalho RFC 9110 `Content-Range: bytes */<fixtureBuffer.length>`

#### 1.4. rejeita-video-nao-pronto-com-409-e-inexistente-com-404

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Iniciar upload em `draft` e chamar GET /videos/:shortCode/stream com `Range: bytes=0-511`
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"`
  2. Atualizar vídeo para `processing` e depois `failed` e repetir GET /videos/:shortCode/stream
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"` em ambos os estados
  3. Chamar GET /videos/NaoExiste11/stream
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`

---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.13
target_file: test/videos-thumbnail.e2e-spec.ts
---

# GET /videos/:shortCode/thumbnail — Test Plan

## Application Overview

`GET /videos/:shortCode/thumbnail` serve publicamente (`@Public()`, `@SkipThrottle()`) a imagem JPEG de thumbnail extraída automaticamente pelo `video-worker` a partir de um frame do vídeo enviado (`phase-03-videos/TD-05`, `TD-07`), aplicando cache público (`Cache-Control: public, max-age=86400`).

## Test Scenarios

### 1. Servir thumbnail JPEG gerada automaticamente a partir do vídeo

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais e gera uma fixture `.mp4` válida em memória (`Buffer`) via `execFile('ffmpeg', [... '-f', 'mp4', 'pipe:1'], { encoding: 'buffer' })`; `beforeEach` executa `cleanAllTables()` no PostgreSQL real (`db`) e limpa `ThrottlerStorage`; `afterAll` fecha app e DataSource.

#### 1.1. anonimo-obtem-thumbnail-jpeg-gerada-pelo-worker

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Usuário autenticado conclui upload multipart e aguarda o `video-worker` marcar `status = 'ready'`
  2. Cliente anônimo chama GET /videos/:shortCode/thumbnail
    - expect: HTTP `200`
    - expect: cabeçalhos `Content-Type: image/jpeg` e `Cache-Control: public, max-age=86400`
    - expect: body binário inicia com os magic bytes JPEG `0xFF 0xD8 0xFF` e possui tamanho `> 0`

#### 1.2. rejeita-thumbnail-de-video-nao-pronto-com-409

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Iniciar upload em `draft` e chamar GET /videos/:shortCode/thumbnail
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"`
  2. Atualizar registro para `processing` e depois `failed` e repetir GET /videos/:shortCode/thumbnail
    - expect: HTTP `409` com `error: "VIDEO_NOT_READY"` em ambos os estados

#### 1.3. retorna-404-para-shortcode-inexistente-ou-thumbnail-ausente

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T11:03:00Z

**Steps:**
  1. Cliente anônimo chama GET /videos/NaoExiste11/thumbnail
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`
  2. Definir `thumbnail_key = null` em um registro `status = 'ready'` no banco e chamar GET /videos/:shortCode/thumbnail
    - expect: HTTP `404` com `error: "VIDEO_THUMBNAIL_NOT_FOUND"`

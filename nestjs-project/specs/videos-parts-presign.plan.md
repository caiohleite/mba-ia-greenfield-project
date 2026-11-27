---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.7
target_file: test/videos-parts-presign.e2e-spec.ts
---

# POST /videos/:id/parts/:partNumber/presign — Test Plan

## Application Overview

`POST /videos/:id/parts/:partNumber/presign` re-assina a URL de upload de uma parte individual de um multipart em andamento, permitindo retomar uploads grandes após expiração das URLs originais (`phase-03-videos/TD-03`). Somente o dono do canal do vídeo pode re-assinar, apenas enquanto o vídeo estiver em `status = "draft"`, e a rota é isenta do rate limiter (`@SkipThrottle()`).

## Test Scenarios

### 1. Re-assinar parte de upload multipart em andamento

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais de `main.ts`; `beforeEach` executa `cleanAllTables()`, limpa `ThrottlerStorage`, cria dois usuários confirmados (dono e não-dono) com seus `access_token` e inicia um upload via `POST /videos/uploads` pelo dono contra MinIO real; `afterAll` fecha app e DataSource.

#### 1.1. dono-reassina-parte-e-recebe-url-valida

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/:id/parts/1/presign com token do dono
    - expect: HTTP `200` com `partNumber: 1`, `uploadUrl` não-vazia e `expiresInSeconds > 0`
  2. Cliente HTTP executa `PUT` de um chunk de bytes na `uploadUrl` retornada
    - expect: MinIO responde `200` com cabeçalho `ETag`

#### 1.2. rejeita-sem-token-e-nao-dono

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/:id/parts/1/presign sem `Authorization`
    - expect: HTTP `401`
  2. POST /videos/:id/parts/1/presign com token do usuário não-dono
    - expect: HTTP `403` com `error: "VIDEO_FORBIDDEN"`

#### 1.3. rejeita-parametros-invalidos

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/not-a-uuid/parts/1/presign com token do dono
    - expect: HTTP `400` com `error: "VALIDATION_ERROR"`
  2. POST /videos/:id/parts/0/presign com token do dono
    - expect: HTTP `400` com `error: "VALIDATION_ERROR"`

#### 1.4. rejeita-video-inexistente-ou-fora-de-draft

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/<uuid-inexistente>/parts/1/presign com token do dono
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`
  2. Atualizar o vídeo para `status = 'processing'` no banco e repetir POST /videos/:id/parts/1/presign
    - expect: HTTP `409` com `error: "INVALID_VIDEO_STATE"`

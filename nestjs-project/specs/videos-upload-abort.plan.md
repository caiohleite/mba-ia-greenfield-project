---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.9
target_file: test/videos-upload-abort.e2e-spec.ts
---

# DELETE /videos/:id/upload — Test Plan

## Application Overview

`DELETE /videos/:id/upload` aborta um S3 Multipart Upload em andamento no MinIO e remove o rascunho incompleto do PostgreSQL, evitando partes órfãs no storage (`phase-03-videos/TD-03`, `TD-08`). Apenas o dono do canal pode abortar, e somente enquanto o vídeo estiver em `status = "draft"`.

## Test Scenarios

### 1. Abortar upload multipart em rascunho

**Setup:** `beforeAll` cria o app com `AppModule` reproduzindo pipes/filtros globais; `beforeEach` executa `cleanAllTables()`, limpa `ThrottlerStorage`, cria usuários dono e não-dono confirmados e inicia upload via `POST /videos/uploads` pelo dono contra MinIO real; `afterAll` fecha app e DataSource.

#### 1.1. dono-aborta-rascunho-retorna-204

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. DELETE /videos/:id/upload com token do dono
    - expect: HTTP `204` sem body
    - expect: registro removido de `videos`
    - expect: `PUT` posterior na `uploadUrl` original é rejeitado pelo MinIO (`NoSuchUpload`)

#### 1.2. rejeita-sem-token-e-nao-dono

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. DELETE /videos/:id/upload sem `Authorization`
    - expect: HTTP `401`
  2. DELETE /videos/:id/upload com token do não-dono
    - expect: HTTP `403` com `error: "VIDEO_FORBIDDEN"`
    - expect: registro permanece em `videos`

#### 1.3. rejeita-id-invalido-e-video-inexistente

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. DELETE /videos/not-a-uuid/upload com token do dono
    - expect: HTTP `400` com `error: "VALIDATION_ERROR"`
  2. DELETE /videos/<uuid-inexistente>/upload com token do dono
    - expect: HTTP `404` com `error: "VIDEO_NOT_FOUND"`

#### 1.4. rejeita-video-fora-de-draft

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. Atualizar o vídeo para `status = 'ready'` no banco e chamar DELETE /videos/:id/upload com token do dono
    - expect: HTTP `409` com `error: "INVALID_VIDEO_STATE"`

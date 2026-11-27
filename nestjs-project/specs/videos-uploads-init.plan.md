---
subproject: backend
runner: jest+supertest
scope: phase-03-videos
si: SI-03.6
target_file: test/videos-uploads-init.e2e-spec.ts
---

# POST /videos/uploads — Test Plan

## Application Overview

`POST /videos/uploads` inicia o upload de um vídeo de até 10 GiB sem trafegar bytes pela API (`phase-03-videos/TD-03`). O endpoint autenticado valida o payload, pré-cadastra o vídeo em `status = "draft"` com `shortCode` único de 11 caracteres (`phase-03-videos/TD-06`), abre um S3 Multipart Upload no MinIO real e devolve as URLs pré-assinadas das partes para envio direto ao storage.

## Test Scenarios

### 1. Iniciar upload multipart com pré-cadastro em rascunho

**Setup:** `beforeAll` cria o app via `Test.createTestingModule({ imports: [AppModule] }).compile()` reproduzindo `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`, `DomainExceptionFilter` e `ValidationExceptionFilter`; `beforeEach` executa `cleanAllTables()` contra o PostgreSQL real (`db`), limpa `ThrottlerStorage` e registra/confirma/loga um usuário para obter `access_token`; `afterAll` fecha app e DataSource.

#### 1.1. inicia-upload-valido-retorna-draft-e-partes-presignadas

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/uploads com `Authorization: Bearer <access_token>` e body `{ "title": "Aula 01", "originalFilename": "aula-01.mp4", "mimeType": "video/mp4", "sizeBytes": 26214400 }`
    - expect: HTTP `201`
    - expect: body contém `videoId` (uuid), `shortCode` casando `^[A-Za-z0-9_-]{11}$`, `status: "draft"`, `uploadId` não-vazio, `partSizeBytes`, `totalParts` e `parts` com `totalParts` itens `{ partNumber, uploadUrl }`
    - expect: registro em `videos` persistido com `status = 'draft'`, `size_bytes = 26214400` e `channel_id` do canal do usuário

#### 1.2. inicia-upload-de-10-gib-no-limite-aceito

**Covers AC:** #1
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/uploads autenticado com `sizeBytes: 10737418240` e `mimeType: "video/mp4"`
    - expect: HTTP `201` com `status: "draft"` e `totalParts <= 10000`

#### 1.3. rejeita-requisicao-sem-token

**Covers AC:** #2
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/uploads sem cabeçalho `Authorization`
    - expect: HTTP `401`
    - expect: nenhum registro criado em `videos`

#### 1.4. rejeita-payload-invalido-com-validation-error

**Covers AC:** #3
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/uploads autenticado com `sizeBytes: 0` e propriedade desconhecida `foo: "bar"`
    - expect: HTTP `400` com `error: "VALIDATION_ERROR"`
    - expect: nenhum registro criado em `videos`

#### 1.5. rejeita-arquivo-acima-de-10-gib-e-mime-nao-suportado

**Covers AC:** #4
**Source:** auto
**Last sync:** 2026-11-27T02:40:00Z

**Steps:**
  1. POST /videos/uploads autenticado com `sizeBytes: 10737418241`
    - expect: HTTP `413` com `error: "VIDEO_TOO_LARGE"`
  2. POST /videos/uploads autenticado com `mimeType: "application/pdf"`
    - expect: HTTP `415` com `error: "UNSUPPORTED_VIDEO_MIME_TYPE"`
    - expect: nenhum registro criado em `videos`

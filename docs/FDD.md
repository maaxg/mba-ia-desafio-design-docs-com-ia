# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação. As decisões estão fechadas nos ADR-001…008. O [RFC](RFC.md) segue o rito de revisão com Bruno e Diego ([09:50] Larissa). |
| **Base** | [RFC-001](RFC.md) · [ADR-001…008](RFC.md#7-decisões-relacionadas) · [TRANSCRICAO.md](../TRANSCRICAO.md) |
| **Módulo novo** | `src/modules/webhooks/` + entry-point `src/worker.ts` (ambos **a criar**, como proposto em [09:11] Larissa e [09:27]–[09:28] Bruno) |
| **Estimativa** | 3 sprints, com revisão de segurança no fim ([09:46] Larissa) |

> **Como ler este documento.** Tudo aqui deriva da transcrição ou do código. Onde a reunião não desceu ao detalhe necessário para codar, o FDD toma uma **decisão de implementação**, marcada com 🔧, derivada de falas ou restrições da reunião e com o fundamento registrado. A lista completa está na [seção 14](#14-decisões-de-implementação-tomadas-neste-fdd).
>
> **Caminhos de arquivo.** Todo caminho citado neste FDD existe no repositório, **exceto** `src/worker.ts` e os arquivos dentro de `src/modules/webhooks/`, que são os arquivos **novos que esta feature cria** (estrutura completa na [seção 13](#13-integração-com-o-sistema-existente)).

---

## 1. Contexto e motivação técnica

- `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa, num único `prisma.$transaction`: validação da transição (`canTransition` em `src/modules/orders/order.status.ts`), débito ou reposição de estoque, `update` em `orders` e `insert` em `order_status_history`. É o único ponto do código onde o status de um pedido muda depois da criação.
- Não existe nenhum mecanismo de evento, fila ou chamada HTTP de saída no projeto (`package.json` não tem cliente HTTP nem broker).
- Os clientes B2B precisam saber de cada transição em **menos de 10s** ([09:02] Marcos) sem fazer polling em `GET /orders` ([09:00] Marcos).
- Restrição técnica dominante: **não fazer I/O externo dentro da transação** ([09:04] Bruno) e **não perder evento** ([09:40] Bruno). A combinação resolve com o padrão outbox ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 2. Objetivos técnicos

| ID | Objetivo | Critério verificável |
|---|---|---|
| OT-1 | Atomicidade status ⇔ evento | Rollback de `changeStatus` não deixa linha na outbox. Falha na outbox desfaz a mudança de status. |
| OT-2 | Latência de entrega | `delivered_at - created_at` p95 < 10s com o cliente saudável (até 2s de espera do polling + tempo de resposta do cliente). O timeout de 10s só se aplica a cliente degradado, que entra em retry. Um cliente lento não atrasa os demais (envio isolado por endpoint, seção 5.2). |
| OT-3 | Resiliência a cliente fora do ar | 5 retentativas em 1m/5m/30m/2h/12h; depois, dead letter. |
| OT-4 | Autenticidade e integridade | Toda requisição leva `X-Signature` HMAC-SHA256 do corpo com a secret do endpoint. |
| OT-5 | Idempotência do lado do cliente | `X-Event-Id` estável entre tentativas e replays. |
| OT-6 | Zero infraestrutura nova | Só MySQL, Prisma, Pino, Zod, Express e o Node 20 nativo (`fetch`, `crypto`). |

## 3. Escopo e exclusões

**Incluso**
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter` (Prisma + migration).
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro de `changeStatus`.
- CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ (ADMIN).
- Worker `src/worker.ts` + `src/modules/webhooks/webhook.processor.ts` (novos) com polling, assinatura, envio, retry e dead letter.
- Observabilidade com Pino e testes (Vitest + Supertest).

**Excluído** (com origem)

| Item | Motivo | Origem |
|---|---|---|
| Webhooks inbound (cliente → nós) | Só outbound | [09:02] Marcos, [09:03] Sofia |
| Notificação por e-mail ao cliente em falhas repetidas | Próxima fase | [09:37] Larissa |
| Rate limiting de saída | Observar e decidir depois | [09:39] Diego, Larissa |
| Dashboard/painel visual | Projeto separado do time de frontend | [09:40] Larissa |
| Arquivamento de linhas entregues (≈30 dias) | Fora do escopo desta feature | [09:08] Diego |
| Múltiplos workers / ordem global | Futuro; limitação conhecida | [09:13] Diego, Larissa |
| Exactly-once | Complexidade desproporcional | [09:25] Diego |
| `items` do pedido no payload | Payload enxuto; o cliente consulta `GET /orders/:id` | [09:43] Diego |
| Evento na **criação** do pedido (`OrderService.create`, status inicial `PENDING`) | A reunião só tratou `changeStatus` | [09:40] Bruno |

## 4. Modelo de dados

Acréscimos em `prisma/schema.prisma`, seguindo o padrão do arquivo: PK `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa), `createdAt`/`updatedAt` e `@@map` em snake_case.

```prisma
enum WebhookOutboxStatus {
  PENDING     // pendente (inclui aguardando retry)
  PROCESSING  // processando
  FAILED      // falhou definitivamente (foi para dead letter ou descartado)
  DELIVERED   // entregue
}

enum WebhookDeliveryResult {
  SUCCESS
  FAILURE
}

model WebhookEndpoint {
  id                      String        @id @default(uuid()) @db.Char(36)
  customerId              String        @db.Char(36)
  url                     String        @db.VarChar(2048)
  secret                  String        @db.VarChar(255)   // cifrada (AES-256-GCM), ADR-004
  previousSecret          String?       @db.VarChar(255)   // cifrada
  previousSecretExpiresAt DateTime?
  events                  Json          // OrderStatus[] — ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean       @default(true)
  createdAt               DateTime      @default(now())
  updatedAt               DateTime      @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36)   // vai em X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)           // sem FK: snapshot independe do pedido (ADR-007)
  customerId    String              @db.Char(36)
  eventType     String              @db.VarChar(64)        // "order.status_changed"
  payload       Json                                      // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(64)        // código WEBHOOK_*
  requestId     String?             @db.VarChar(64)        // correlação com a requisição HTTP de origem
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    WebhookEndpoint   @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@index([status, nextAttemptAt, createdAt])
  @@index([createdAt])
  @@index([webhookId, orderId, createdAt])   // ordem por pedido, ADR-008
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String                @id @default(uuid()) @db.Char(36)
  outboxId       String                @db.Char(36)
  webhookId      String                @db.Char(36)
  eventId        String                @db.Char(36)
  attempt        Int                                       // 1 = envio inicial, 2..6 = retentativas
  result         WebhookDeliveryResult
  httpStatus     Int?
  errorCode      String?               @db.VarChar(64)
  requestBody    Json
  responseBody   String?               @db.VarChar(2048)   // truncado
  durationMs     Int
  createdAt      DateTime              @default(now())

  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @unique @db.Char(36)
  webhookId     String    @db.Char(36)
  eventId       String    @db.Char(36)
  payload       Json
  reason        String    @db.VarChar(64)                 // código WEBHOOK_*
  lastHttpStatus Int?
  attempts      Int
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)                     // auditoria do replay (ADMIN)

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:
- `Customer` ganha a relação `webhookEndpoints WebhookEndpoint[]`. O `onDelete: Cascade` 🔧 impede que a configuração de webhook bloqueie `CustomerService.delete` (`src/modules/customers/customer.service.ts`).
- O índice composto `(status, nextAttemptAt, createdAt)` atende a query do worker. `createdAt` isolado atende ao índice pedido por Diego ([09:08]) e ao futuro arquivamento.
- **Uma linha na outbox por (evento × endpoint inscrito)** 🔧. Isso é necessário porque `X-Webhook-Id` identifica o endpoint destino ([09:44] Sofia) e porque cada endpoint tem sua secret e seu ciclo de retry.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro de `changeStatus`)

```
PATCH /api/v1/orders/:id/status
 └─ OrderController.changeStatus
     └─ OrderService.changeStatus  ── prisma.$transaction(async (tx) => { ... })
          1. findUnique(order) / valida transição (código atual, inalterado)
          2. debitStock / replenishStock            (inalterado)
          3. tx.order.update(status = to)            (inalterado)
          4. tx.orderStatusHistory.create(...)       (inalterado)
          5. await publishWebhookEvent(tx, order, from, to, { requestId })   ◄── NOVO  (5º parâmetro opcional 🔧)
               a. endpoints = tx.webhookEndpoint.findMany({ customerId: order.customerId, active: true })
               b. alvo = endpoints.filter(e => e.events.includes(to))           [09:33]–[09:34]
               c. se alvo vazio → return (nenhuma linha)                         [09:34] Bruno
               d. para cada endpoint em alvo:
                    eventId = uuidv4()
                    payload = renderPayload(eventId, order, from, to, now)      ADR-007
                    tx.webhookOutbox.create({ eventId, webhookId, orderId, customerId,
                                             eventType: 'order.status_changed', payload,
                                             status: PENDING, nextAttemptAt: now, requestId })
               e. qualquer erro → lança WebhookOutboxEnqueueError → rollback de TUDO   [09:40] Bruno
          6. refresh + return (inalterado)
```

- **Atomicidade:** `publishWebhookEvent` só usa o `tx` recebido, nunca o `prisma` global. Se a usasse, a escrita aconteceria fora da transação, quebrando o [ADR-001](adrs/ADR-001-outbox-no-mysql.md) ([09:41] Diego).
- **Payload > 64 KB não é verificado aqui** 🔧. Recusar na inserção desfaria a mudança de status por causa de um problema de notificação. A recusa acontece no worker (5.2, passo 3), o que é coerente com "a gente não envia" ([09:23] Sofia).

### 5.2 Processamento pelo worker

```
src/worker.ts (bootstrap)
 ├─ import { prisma } from './config/database.js'         [09:30] Bruno: instância própria
 │    (o singleton é criado ao importar o módulo, então é um PrismaClient novo, deste processo)
 ├─ recoverStuck(): PROCESSING → PENDING                  (crash anterior; gera reenvio = at-least-once)
 ├─ loop:
 │   processor.runOnce()
 │   await sleep(WEBHOOK_POLL_INTERVAL_MS = 2000)         [09:09] Diego
 └─ SIGINT/SIGTERM: para de buscar, aguarda envios em andamento, $disconnect, exit(0)
      🔧 (padrão de shutdown de src/server.ts; não discutido na reunião)

WebhookProcessor.runOnce():          -- não espera os envios terminarem (isolamento entre endpoints) 🔧
 1. batch = SELECT ... FROM webhook_outbox o
            WHERE status = 'PENDING' AND next_attempt_at <= NOW()
              AND webhook_id NOT IN (:busyEndpoints)       -- endpoints com envio em andamento
              AND NOT EXISTS (SELECT 1 FROM webhook_outbox o2  -- ordem por pedido (ADR-008)
                              WHERE o2.webhook_id = o.webhook_id AND o2.order_id = o.order_id
                                AND o2.created_at < o.created_at
                                AND o2.status IN ('PENDING','PROCESSING'))
            ORDER BY created_at ASC
            LIMIT WEBHOOK_BATCH_SIZE (🔧 50)               [09:08] Diego: "batch pequeno"
 2. UPDATE status = 'PROCESSING' WHERE id IN (batch)
 3. agrupa o lote por webhookId. Para cada grupo, marca o endpoint em busyEndpoints e
    dispara deliverGroup(grupo) SEM await. O grupo libera o endpoint ao terminar.

deliverGroup(grupo):                 -- dentro do grupo, SEQUENCIAL por created_at   [09:12] Diego
 para cada evento do grupo:
    a. endpoint = findUnique(webhookId)
         - inativo → status FAILED, lastError WEBHOOK_ENDPOINT_INACTIVE, sem DLQ 🔧; continue
         - secret vazia → dead letter com WEBHOOK_SECRET_REQUIRED 🔧; continue
    b. body = JSON.stringify(payload)
         - Buffer.byteLength(body) > 65_536 → dead letter WEBHOOK_PAYLOAD_TOO_LARGE (sem retry)   [09:24]
    c. headers = buildHeaders(evento, endpoint, body)          (ver 6.2)
    d. res = fetch(endpoint.url, { method: 'POST', body, headers,
                                   redirect: 'manual',
                                   signal: AbortSignal.timeout(10_000) })      [09:42] Diego
    e. grava webhook_deliveries (attempt, result, httpStatus, durationMs, responseBody truncado)
    f. 2xx → status DELIVERED, deliveredAt = now
       senão → handleFailure(evento, errorCode)                 (5.3)
         e interrompe o grupo: os eventos seguintes do mesmo endpoint voltam para PENDING
         sem consumir tentativa, e saem no próximo ciclo 🔧
```

Isolamento por endpoint e ordem por pedido: decisão registrada no [ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md). Com envio sequencial do lote inteiro, um único cliente lento (10s × N eventos) atrasaria todos os outros, o que viola o isolamento entre clientes ([09:04] Bruno) e a meta de < 10s ([09:02] Marcos). Continua sendo **um único processo worker** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). O `NOT EXISTS` garante que o evento N+1 de um pedido só sai depois que o evento N terminou (entregue ou em dead letter), mesmo durante retries.

### 5.3 Retry

```
BACKOFF_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]   // 1m 5m 30m 2h 12h  [09:17]
MAX_RETRIES         = BACKOFF_SCHEDULE_MS.length  // 5

handleFailure(evento, errorCode):
  evento.attempts += 1                     // tentativas já feitas (inicial conta como 1)
  retryIndex = evento.attempts - 1         // 0..4
  if retryIndex < MAX_RETRIES:
      status        = PENDING
      nextAttemptAt = now + BACKOFF_SCHEDULE_MS[retryIndex]
      lastError     = errorCode
      log 'webhook_retry_scheduled'
  else:
      moveToDeadLetter(evento, 'WEBHOOK_MAX_ATTEMPTS_EXCEEDED', lastHttpStatus)
```

Linha do tempo de um cliente totalmente fora do ar (t₀ = primeiro envio):

| Envio | Quando | Atraso desde t₀ |
|---|---|---|
| 1 (inicial) | t₀ | 0 |
| 2 | t₀ + 1m | 1m |
| 3 | + 5m | 6m |
| 4 | + 30m | 36m |
| 5 | + 2h | 2h36m |
| 6 | + 12h | **14h36m** → falhou → dead letter |

> **1 envio + 5 retentativas** é a decisão registrada no [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md). É a única leitura que acomoda os 5 intervalos e os "quase 15 horas" citados por Diego ([09:17]).

O que conta como falha retentável: timeout (10s), erro de rede/DNS/TLS, resposta fora de 2xx (inclusive 3xx, já que redirects não são seguidos 🔧). Falhas não retentáveis vão direto para a dead letter: `WEBHOOK_PAYLOAD_TOO_LARGE` e `WEBHOOK_SECRET_REQUIRED`.

### 5.4 Dead letter e replay

```
moveToDeadLetter(evento, reason, lastHttpStatus):    -- numa transação
  INSERT webhook_dead_letter (outboxId, webhookId, eventId, payload, reason, lastHttpStatus, attempts)
  UPDATE webhook_outbox SET status = 'FAILED', lastError = reason
  log.warn 'webhook_dead_lettered'

POST /api/v1/admin/webhooks/dead-letter/:id/replay   (authenticate + requireRole('ADMIN'))   [09:36]
  numa transação:
   1. dl = findUnique(id)                  → 404 WEBHOOK_DEAD_LETTER_NOT_FOUND
   2. dl.replayedAt != null                → 409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED 🔧
   3. endpoint inexistente/inativo         → 409 WEBHOOK_ENDPOINT_INACTIVE
   4. UPDATE webhook_outbox (dl.outboxId) SET status = PENDING, attempts = 0,
                                             nextAttemptAt = now, lastError = null
      (mesmo eventId e mesmo payload → cliente deduplica; ADR-005)
   5. UPDATE dl SET replayedAt = now, replayedById = req.user.id
  log.info 'webhook_dead_letter_replayed' { deadLetterId, eventId, replayedBy: req.user.id }   [09:36] Sofia
```

"Recoloca na outbox como pendente" ([09:18] Diego) é implementado reaproveitando a linha original da outbox 🔧. Assim o histórico de entregas fica contínuo e o `eventId` não muda.

### 5.5 Rotação de secret

```
POST /api/v1/webhooks/:id/rotate-secret
  1. endpoint = findUnique(id)  → 404 WEBHOOK_NOT_FOUND
     endpoint.active = false     → 409 WEBHOOK_ENDPOINT_INACTIVE 🔧
  2. previousSecret          = endpoint.secret             (já cifrada)
     previousSecretExpiresAt = now + 24h                         [09:21] Sofia
     secret                  = encrypt(randomBytes(32).toString('hex'))   AES-256-GCM, ADR-004
  3. retorna a nova secret (única vez em que ela aparece, além da criação)

Assinatura no worker (buildHeaders):
  sigs = ['sha256=' + hmac(secret, body)]
  if previousSecret && previousSecretExpiresAt > now:
      sigs.push('sha256=' + hmac(previousSecret, body))
  X-Signature: sigs.join(',')                                    🔧 (cliente aceita se QUALQUER uma bater)
```

Uma rotação nova durante o grace period substitui `previousSecret` pela secret vigente. A antiga anterior morre na hora 🔧.

## 6. Contratos públicos

Todos os endpoints ficam sob `/api/v1` (prefixo de `src/app.ts`), exigem `Authorization: Bearer <JWT>` (`authenticate`) e respondem erros no formato do `errorMiddleware`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

### 6.1 API de configuração e operação

#### 6.1.1 `POST /api/v1/webhooks` — cadastrar webhook ([09:31] Marcos)

Qualquer role autenticada ([09:37] Sofia). O `customerId` vai no body, não sai do JWT ([09:32] Larissa).

Request:
```http
POST /api/v1/webhooks
Authorization: Bearer <JWT>
Content-Type: application/json
```
```json
{
  "customerId": "6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`. A `secret` aparece **apenas aqui e na rotação**:
```json
{
  "id": "0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10",
  "customerId": "6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c6b1e4d...e81a",
  "createdAt": "2026-11-20T13:58:02.114Z",
  "updatedAt": "2026-11-20T13:58:02.114Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado |
| 400 `VALIDATION_ERROR` (detalhe `WEBHOOK_INVALID_URL` / `WEBHOOK_INVALID_EVENT_FILTER`) | URL não-https ou malformada; `events` vazio ou com status inexistente |
| 401 `UNAUTHORIZED` | Sem JWT ou JWT inválido |
| 404 `NOT_FOUND` | `customerId` inexistente (reusa `NotFoundError('Customer')`) |

#### 6.1.2 `GET /api/v1/customers/:customerId/webhooks?page=1&pageSize=20` — listar webhooks do customer ([09:33] Bruno)

O `customerId` vai no **path**, já que um GET não tem body ("body ou path", [09:32] Larissa).

Request (sem body):
```http
GET /api/v1/customers/6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11/webhooks?page=1&pageSize=20
Authorization: Bearer <JWT>
```

Response `200 OK` (formato `paginated()` de `src/shared/http/response.ts`, **sem** `secret`):
```json
{
  "data": [
    {
      "id": "0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10",
      "customerId": "6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotationPending": false,
      "createdAt": "2026-11-20T13:58:02.114Z",
      "updatedAt": "2026-11-20T13:58:02.114Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Status: `200`, `400 VALIDATION_ERROR` (`customerId` não-uuid), `401`, `404 NOT_FOUND` (customer inexistente).

#### 6.1.3 `PATCH /api/v1/webhooks/:id` — editar ([09:33] Bruno)

Campos aceitos (todos opcionais, pelo menos um obrigatório): `url` (https), `events` (OrderStatus[], não vazio), `active` (boolean). O `customerId` não pode ser alterado.

Request:
```http
PATCH /api/v1/webhooks/0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10
Authorization: Bearer <JWT>
Content-Type: application/json
```
```json
{ "url": "https://hooks.atlascomercial.com.br/oms/v2", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK` (mesmo formato de item de 6.1.2, **sem** `secret`):
```json
{
  "id": "0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10",
  "customerId": "6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11",
  "url": "https://hooks.atlascomercial.com.br/oms/v2",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotationPending": false,
  "createdAt": "2026-11-20T13:58:02.114Z",
  "updatedAt": "2026-11-20T15:21:47.390Z"
}
```
Status: `200`, `400 VALIDATION_ERROR` (detalhes `WEBHOOK_INVALID_URL` / `WEBHOOK_INVALID_EVENT_FILTER`), `401`, `404 WEBHOOK_NOT_FOUND`.

Mudanças em `events` e `active` só valem para eventos **futuros**. Linhas já na outbox não são refiltradas: um endpoint desativado tem seus pendentes descartados no envio (5.2, passo 3a).

#### 6.1.4 `DELETE /api/v1/webhooks/:id` — remover ([09:33] Bruno)

Request (sem body):
```http
DELETE /api/v1/webhooks/0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10
Authorization: Bearer <JWT>
```
Response `204 No Content` (sem body). Status: `204`, `401`, `404 WEBHOOK_NOT_FOUND`.
Remove também a outbox, as entregas e a dead letter do endpoint (cascade) 🔧. Para pausar sem perder histórico, use `PATCH { "active": false }`.

#### 6.1.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret ([09:21] Sofia)

Request (corpo vazio):
```http
POST /api/v1/webhooks/0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10/rotate-secret
Authorization: Bearer <JWT>
```
Response `200 OK`:
```json
{
  "id": "0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10",
  "secret": "c41d07aa9b...5f20",
  "previousSecretExpiresAt": "2026-11-21T14:10:00.000Z"
}
```
Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`, `409 WEBHOOK_ENDPOINT_INACTIVE`.

#### 6.1.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas ([09:34] Marcos)

Retorna as **últimas 100 tentativas**, da mais recente para a mais antiga. Query opcional: `result=SUCCESS|FAILURE`.

Request (sem body):
```http
GET /api/v1/webhooks/0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10/deliveries
Authorization: Bearer <JWT>
```

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "a3d9...",
      "eventId": "5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33",
      "attempt": 2,
      "result": "SUCCESS",
      "httpStatus": 200,
      "durationMs": 184,
      "requestBody": { "event_id": "5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33", "event_type": "order.status_changed", "...": "..." },
      "responseBody": "{\"ok\":true}",
      "errorCode": null,
      "createdAt": "2026-11-20T14:04:13.007Z"
    },
    {
      "id": "77c1...",
      "eventId": "5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33",
      "attempt": 1,
      "result": "FAILURE",
      "httpStatus": 503,
      "durationMs": 92,
      "requestBody": { "...": "..." },
      "responseBody": "Service Unavailable",
      "errorCode": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "createdAt": "2026-11-20T14:03:12.902Z"
    }
  ]
}
```
Status: `200`, `401`, `404 WEBHOOK_NOT_FOUND`.

#### 6.1.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay manual de DLQ ([09:18] Diego, [09:35] Diego)

`authenticate` + `requireRole('ADMIN')` ([09:36] Sofia, Larissa).

Request (corpo vazio; JWT de usuário ADMIN):
```http
POST /api/v1/admin/webhooks/dead-letter/e2a4.../replay
Authorization: Bearer <JWT de ADMIN>
```

Response `202 Accepted`. O evento volta para a fila; a entrega é assíncrona:
```json
{
  "deadLetterId": "e2a4...",
  "outboxId": "91fd...",
  "eventId": "5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33",
  "status": "PENDING",
  "replayedAt": "2026-11-21T09:12:44.530Z",
  "replayedBy": "4c7b2e10-...-admin-user-id"
}
```
| Status | Quando |
|---|---|
| 202 | Recolocado na outbox |
| 401 `UNAUTHORIZED` | Sem JWT |
| 403 `FORBIDDEN` | Role diferente de ADMIN (`requireRole` existente) |
| 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` | ID inexistente |
| 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Já reprocessado |
| 409 `WEBHOOK_ENDPOINT_INACTIVE` | Endpoint desativado; reative antes |

Não há endpoint de **listagem** da DLQ no escopo discutido. O ADMIN obtém o `id` pelo log `webhook_dead_lettered` ou por consulta ao banco.

### 6.2 Requisição de webhook enviada ao cliente (outbound)

```
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: 5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33
X-Webhook-Id: 0b8e5d0a-3a52-4b3e-8f6d-2b1f4c9e7a10
X-Timestamp: 2026-11-20T14:03:12.874Z
X-Signature: sha256=3f7a0c...d91e
```

| Header | Conteúdo | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento, estável entre tentativas e replay | [09:25] Diego |
| `X-Webhook-Id` | ID do endpoint cadastrado | [09:44] Sofia |
| `X-Timestamp` | ISO 8601 **do envio** (muda a cada tentativa), para o cliente detectar replay se quiser | [09:44] Diego |
| `X-Signature` | `sha256=<hex>` = HMAC-SHA256(secret, corpo bruto). Durante o grace period, duas assinaturas separadas por vírgula | [09:20]–[09:22] Sofia |

Corpo (snapshot gravado na inserção, [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md)):
```json
{
  "event_id": "5b0f3c2e-9a7d-4e61-8d2b-0c4f1e7a9b33",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-20T14:03:11.402Z",
  "order_id": "d4a1f6b2-7c3e-4a90-b1d5-8e2f0c6a3b77",
  "order_number": "ORD-000123",
  "customer_id": "6f1c2a4e-0b7d-4c1a-9a51-3f0e8b2d7c11",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "total_cents": 17000,
  "subtotal_cents": 18000,
  "discount_cents": 1000
}
```
- `timestamp` = momento da transição (inserção na outbox). `order_number` segue o formato `ORD-NNNNNN` de `reserveOrderNumber`.
- Os campos `*_cents` são os "campos básicos da order" ([09:43] Diego), com nomes vindos de `Order` em `prisma/schema.prisma`. **Não há `items`.**
- Resposta esperada do cliente: qualquer `2xx` em até 10s. O corpo da resposta é ignorado (só é guardado, truncado, em `webhook_deliveries`).

Verificação pelo cliente (exemplo para o portal do desenvolvedor, [09:26] Marcos):
```ts
const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = req.header('X-Signature')!.split(',').some((s) =>
  s.length === expected.length && timingSafeEqual(Buffer.from(s), Buffer.from(expected)));
// dedup: if (alreadyProcessed(req.header('X-Event-Id'))) return res.sendStatus(200);
```

## 7. Matriz de erros previstos

Todos os códigos do módulo usam o prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa). Eles aparecem em três formas, conforme o ponto do sistema em que surgem.

### 7.1 Erros de API: classes no arquivo novo `src/modules/webhooks/webhook.errors.ts` (`extends AppError`)

Serializados pelo `errorMiddleware` como `{ error: { code, message, details? } }`.

| Código | HTTP | Endpoint(s) | Causa | Classe |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | PATCH, DELETE, rotate-secret, deliveries | ID de webhook inexistente | `WebhookNotFoundError extends AppError`. Necessária porque `NotFoundError` fixa o código `NOT_FOUND`. |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | rotate-secret, replay | Endpoint com `active=false` 🔧 | `extends ConflictError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay | ID de dead letter inexistente | `extends AppError` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | replay | `replayedAt` já preenchido 🔧 | `extends ConflictError` |
| `WEBHOOK_OUTBOX_ENQUEUE_FAILED` | 500 | `PATCH /orders/:id/status` | Falha ao inserir na outbox. Rollback total da mudança de status ([09:40] Bruno) | `extends AppError`, lançada por `publishWebhookEvent` |

### 7.2 Detalhes de validação (Zod → `VALIDATION_ERROR`)

`src/middlewares/validate.middleware.ts` converte qualquer `ZodError` em `ValidationError` (código `VALIDATION_ERROR`). Como a regra de `https` fica no schema ([09:23] Sofia), o código `WEBHOOK_*` vai no `message` do detalhe:

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Validation failed",
  "details": [ { "path": "url", "message": "WEBHOOK_INVALID_URL: url must use https" } ] } }
```

| Código (em `details[].message`) | HTTP | Causa |
|---|---|---|
| `WEBHOOK_INVALID_URL` | 400 | URL malformada ou com esquema diferente de `https` ([09:23] Sofia; código citado em [09:28] Bruno) |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `events` vazio, duplicado ou com valor fora de `OrderStatus` |

### 7.3 Códigos de resultado do worker (gravados em `lastError`, `webhook_delivery.errorCode` e `webhook_dead_letter.reason`)

Não são lançados para nenhum cliente HTTP. Eles classificam o desfecho de cada tentativa.

| Código | Causa | Comportamento |
|---|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | Sem resposta em 10s ([09:42] Diego) | Retry com backoff |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Resposta fora de 2xx (inclui 3xx) | Retry com backoff; `httpStatus` registrado |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | DNS, conexão recusada, TLS inválido | Retry com backoff |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | 6º envio falhou ([09:17]) | Dead letter |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Corpo > 64 KB ([09:24] Diego, Larissa) | Dead letter imediata, sem envio |
| `WEBHOOK_SECRET_REQUIRED` | Endpoint sem secret (invariante violada). Código citado como exemplo em [09:28] Bruno, aplicado aqui 🔧 | Dead letter imediata, sem envio |
| `WEBHOOK_ENDPOINT_INACTIVE` | Endpoint desativado depois do enfileiramento 🔧 | Linha vai para `FAILED`, sem dead letter |

Erros reaproveitados sem mudança: `UNAUTHORIZED` (401) e `FORBIDDEN` (403) via `auth.middleware.ts`; `NOT_FOUND` para customer inexistente (`NotFoundError('Customer')`); `VALIDATION_ERROR` (400) via `validate.middleware.ts`.

## 8. Estratégias de resiliência

| Mecanismo | Valor | Origem |
|---|---|---|
| Desacoplamento da transação de status | Outbox transacional | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Intervalo de polling | 2 000 ms | [09:09] Diego |
| Tamanho do lote | 50 🔧 (constante `WEBHOOK_BATCH_SIZE`) | "batch pequeno", [09:08] Diego |
| Timeout HTTP | 10 000 ms (`AbortSignal.timeout`) | [09:42] Diego |
| Backoff | 1m, 5m, 30m, 2h, 12h | [09:17] Diego |
| Retentativas | 5 (+ envio inicial) | [09:17] Larissa |
| Fallback final | Dead letter + replay manual ADMIN | [09:18] Diego |
| Limite de payload | 65 536 bytes, recusa sem truncar | [09:24] Diego, Larissa |
| Recuperação de crash | No boot, `PROCESSING → PENDING` (gera reenvio, aceitável por at-least-once) | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Isolamento de processo | Worker fora da API; restart de um não afeta o outro | [09:11] Diego |
| Shutdown gracioso | Aguarda os envios em andamento antes de sair 🔧 | Padrão de `src/server.ts` (não discutido na reunião) |
| Isolamento entre clientes | Envio paralelo entre endpoints, sequencial dentro de cada endpoint 🔧 | [09:04] Bruno (cliente lento não pode travar os outros) |
| Redirects | Não seguidos (`redirect: 'manual'`) 🔧 | Coerente com "URL tem que ser https" ([09:23] Sofia) |

Não há retry automático além da dead letter, nem e-mail ao cliente ([09:37] Larissa). O fallback humano é o replay.

Todas as constantes ficam no arquivo novo `src/modules/webhooks/webhook.constants.ts` 🔧, porque nenhum valor precisa variar por ambiente. A única variável nova em `src/config/env.ts` é a chave de cifra das secrets (`WEBHOOK_SECRET_ENCRYPTION_KEY`, [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)), que por ser segredo não pode ficar no código.

## 9. Observabilidade

Sem dependências novas ([09:29] Bruno): tudo sai pelo **Pino** (`logger` de `src/shared/logger/index.ts`) em JSON estruturado. O worker usa `logger.child({ component: 'webhook-worker' })` 🔧, o que mantém o `base.service` atual (`order-management-api`) sem alterar `createLogger()`.

### 9.1 Logs (eventos nomeados em snake_case, como `server_started`/`http_request`)

| Evento | Nível | Campos |
|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus`, `requestId` |
| `webhook_worker_started` / `webhook_worker_shutdown` | info | `pollIntervalMs`, `batchSize`, `recoveredStuck` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `attempt`, `httpStatus`, `durationMs`, `lagMs` |
| `webhook_delivery_failed` | warn | idem + `errorCode` |
| `webhook_retry_scheduled` | info | `eventId`, `attempt`, `nextAttemptAt` |
| `webhook_dead_lettered` | warn | `deadLetterId`, `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, **`replayedBy`** (auditoria, [09:36] Sofia) |
| `webhook_secret_rotated` | info | `webhookId`, `previousSecretExpiresAt`, `userId` |
| `webhook_worker_cycle_failed` | error | `err` (erro inesperado no loop; o loop continua) |

**Redaction:** acrescentar `'*.secret'` e `'*.previousSecret'` a `redactPaths` em `src/shared/logger/index.ts`. Já houve cliente vazando secret em log ([09:22] Diego), e não podemos ser nós o próximo.

### 9.2 Métricas

O projeto não tem stack de métricas (`package.json`). As métricas vêm de duas fontes:

1. **Contadores e histogramas via log**, agregáveis pela ferramenta de logs: `webhook_delivery_succeeded/failed` (contagem por `errorCode` e `webhookId`), `durationMs` (latência HTTP do cliente) e `lagMs = now - createdAt` (latência ponta a ponta, que é a métrica do SLO de 10s).
2. **Gauges via SQL**, emitidos pelo worker a cada 30 ciclos (~1 min) 🔧 no log `webhook_worker_stats`:

| Métrica | Query | Alerta sugerido |
|---|---|---|
| `outbox_pending_count` | `COUNT(*) WHERE status='PENDING' AND next_attempt_at <= NOW()` | crescimento contínuo |
| `outbox_oldest_pending_age_s` | `NOW() - MIN(created_at)` dos pendentes vencidos | > 10s (SLO, [09:02]) |
| `dead_letter_unreplayed_count` | `COUNT(*) WHERE replayed_at IS NULL` | > 0 novo |
| `delivery_success_rate_1h` | `SUCCESS / total` em `webhook_deliveries` na última hora | queda brusca |

### 9.3 Tracing

Não há tracing distribuído no projeto, e a reunião não autorizou ferramenta nova. A rastreabilidade ponta a ponta usa **IDs de correlação**:

- `requestId` (gerado ou propagado por `src/middlewares/request-logger.middleware.ts` em `req.id`) é gravado na linha da outbox e repetido nos logs do worker. Isso liga o `PATCH /orders/:id/status` à entrega.
- `eventId` atravessa a fronteira até o cliente via `X-Event-Id`. Num ticket, o cliente informa o `X-Event-Id` e conseguimos achar outbox, entregas e dead letter.
- `webhookId` (`X-Webhook-Id`) identifica o cadastro.

## 10. Dependências e compatibilidade

| Item | Situação |
|---|---|
| Node ≥ 20 (`engines` em `package.json`) | `fetch` e `AbortSignal.timeout` nativos. Sem cliente HTTP novo. |
| `node:crypto` | `createHmac('sha256')` e `randomBytes(32)`. Sem dependência nova. |
| `uuid` (já em `package.json`) | `eventId` (o mesmo `v4` usado em `request-logger.middleware.ts`). |
| Prisma 5.22 / MySQL 8.0 (`docker-compose.yml`) | Migration **aditiva**: 4 tabelas, 2 enums e a relação em `Customer`. Nenhuma coluna existente muda. |
| Express, Zod, Pino, Vitest, Supertest | Reuso, sem upgrade. |
| API existente | `PATCH /orders/:id/status` mantém o contrato (request e response idênticos). Só a latência cresce um pouco (1 SELECT + N INSERTs). Os demais endpoints não mudam. |
| Deploy | Novo processo: `npm run worker` (dev) e `start:worker` (prod). Precisa da mesma `.env` (`DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV`). |
| Rollback | Desligar o worker para o envio. A outbox continua acumulando sem efeito no OMS. Reverter o código de `changeStatus` interrompe a geração de eventos. |

## 11. Critérios de aceite técnicos

1. **Atomicidade:** num teste de integração, forçar erro em `publishWebhookEvent` faz `PATCH /orders/:id/status` retornar 500, e o pedido, o `order_status_history` e o estoque ficam inalterados.
2. **Filtro na inserção:** um customer sem endpoint inscrito em `to_status` não gera linha na outbox. Com 2 endpoints inscritos, são geradas 2 linhas com `eventId`s distintos.
3. **Snapshot:** alterar o pedido depois da inserção não muda o corpo enviado.
4. **Latência:** com um servidor HTTP de teste respondendo 200, o evento chega em ≤ 2s + tempo de processamento (p95 < 10s em homologação).
5. **Assinatura:** o `X-Signature` recebido é igual a `sha256=` + HMAC-SHA256(secret, corpo bruto). Durante o grace period há duas assinaturas, e depois de 24h, só uma.
6. **Retry:** com o servidor de teste respondendo 503 ou sem responder por mais de 10s, `nextAttemptAt` segue 1m/5m/30m/2h/12h (testado com relógio controlado). Depois do 6º envio, a linha vai para `webhook_dead_letter` com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`.
7. **Payload > 64 KB** vai para a dead letter sem nenhuma chamada HTTP.
8. **Replay:** com OPERATOR → 403; com ADMIN → 202, a linha volta a `PENDING` com o mesmo `eventId`, `replayedById` é gravado e o log `webhook_dead_letter_replayed` contém `replayedBy`.
9. **Validação:** `url` com `http://` → 400 com detalhe `WEBHOOK_INVALID_URL`.
10. **Secret:** aparece só nas respostas de criação e rotação, nunca em listagem, deliveries ou logs.
11. **Processo:** matar a API não interrompe o worker, e vice-versa. `SIGTERM` no worker termina o lote e sai com código 0.
12. **Testes:** `npm test` verde, com os testes novos de webhooks em `tests/`, no mesmo padrão de `tests/orders.test.ts`.

## 12. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Falha de inserção na outbox bloqueia mudanças de status | Baixa | Alto | Por design: consistência acima de disponibilidade ([09:40]–[09:41]). INSERT simples e indexado. Teste de atomicidade (critério 1). |
| Cliente recebe duplicata e não deduplica | Média | Médio | `X-Event-Id` estável + documentação no portal ([09:26] Marcos). |
| Evento de um pedido em retry segura os eventos seguintes do mesmo pedido (até ~15h) | Média | Médio | Comportamento escolhido para manter a ordem por pedido ([ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md)). Outros pedidos não são afetados. Gauge `outbox_oldest_pending_age_s`. |
| Outbox cresce sem arquivamento | Alta (no longo prazo) | Médio | Índices em `status`/`created_at`. Arquivamento de 30 dias registrado como próxima tarefa ([09:08] Diego). Gauge de volume. |
| Worker único vira gargalo | Baixa (3 clientes) | Médio | Gauge `outbox_oldest_pending_age_s`. Particionar por `order_id` quando precisar ([09:13] Diego). |
| Cliente lento atrasa os próprios eventos (até 10s por envio) | Média | Baixo | Isolamento por endpoint (D10): só o próprio cliente sofre. Timeout de 10s. Monitorar `lagMs` por `webhookId`. Rate limiting segue como ponto em aberto ([09:39]). |
| Vazamento de secret (nosso log ou do cliente) | Média | Alto | Secret por endpoint, rotação com 24h, redaction no Pino, revisão de segurança ([09:21]–[09:22], [09:46]). |
| Perda ou vazamento da chave `WEBHOOK_SECRET_ENCRYPTION_KEY` | Baixa | Alto | Chave só no `.env` do ambiente (nunca no repositório), validada no boot por `env.ts`. Em caso de perda, rotação de todas as secrets pelos clientes ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). |
| `X-Timestamp` não entra na assinatura, o que enfraquece a detecção de replay | Baixa | Médio | A reunião decidiu assinar só o corpo ([09:22] Sofia). A detecção de replay é opcional para o cliente ([09:44] Diego). Item levado à revisão de segurança. |
| Atraso na entrega (Atlas, fim de novembro) | Média | Alto | Plano de 3 sprints da seção 15. Revisão de segurança reservada ([09:46]–[09:47]). |

## 13. Integração com o sistema existente

| Arquivo existente | Como o módulo de webhooks se integra |
|---|---|
| `src/modules/orders/order.service.ts` | **Principal alteração em lógica existente.** Em `changeStatus`, logo depois de `tx.orderStatusHistory.create(...)` e antes do `refreshed`, entra a chamada `await publishWebhookEvent(tx, order, from, to)` ([09:41] Bruno), com um 5º parâmetro opcional `{ requestId }` para correlação 🔧. O `tx` (`Prisma.TransactionClient`, já tipado como `TxClient` no arquivo) é o mesmo da transação, garantindo atomicidade. O construtor de `OrderService` **não** muda: é uma função importada do arquivo novo `src/modules/webhooks/webhook.publisher.ts`, sem repository injetado ([09:41] Diego). `create()` não é alterado. |
| `src/modules/orders/order.controller.ts` | 🔧 `changeStatus` passa `req.id` (o requestId do `request-logger`) para `OrderService.changeStatus`, que ganha um parâmetro opcional. É a única outra mudança fora do módulo novo, opcional e sem quebrar os chamadores atuais. |
| `src/modules/orders/order.status.ts` / `prisma/schema.prisma` (`enum OrderStatus`) | O filtro `events` do webhook é validado com `z.array(z.nativeEnum(OrderStatus))`, o mesmo padrão de `updateOrderStatusSchema` em `src/modules/orders/order.schemas.ts`. Assim, só status reais da máquina de estados são aceitos. |
| `prisma/schema.prisma` + `prisma/migrations/` | Novos models da seção 4 e relação `webhookEndpoints` em `Customer`. Migration via `npm run db:migrate` (`prisma migrate dev`). |
| `src/shared/errors/app-error.ts` / `src/shared/errors/http-errors.ts` | `webhook.errors.ts` declara classes que estendem `AppError` (ex.: `WebhookNotFoundError` → 404 `WEBHOOK_NOT_FOUND`) e `ConflictError` (ex.: `WebhookDeadLetterAlreadyReplayedError`), espelhando `InvalidStatusTransitionError`/`InsufficientStockError` ([09:28] Bruno). |
| `src/middlewares/error.middleware.ts` | **Sem alteração.** O ramo `err instanceof AppError` já serializa `{ error: { code, message, details } }` para os erros novos ([09:29] Bruno). |
| `src/middlewares/validate.middleware.ts` | Usado nas rotas com os schemas de `webhook.schemas.ts`. A regra `https` vira `.refine()` no Zod ([09:23] Sofia), e a resposta sai como `VALIDATION_ERROR` com o detalhe `WEBHOOK_INVALID_URL` (ver seção 7). |
| `src/middlewares/auth.middleware.ts` | `router.use(authenticate)` no router de webhooks (padrão de `customer.routes.ts`). `requireRole('ADMIN')` no router admin de replay (padrão de `user.routes.ts`) ([09:36] Larissa). |
| `src/middlewares/request-logger.middleware.ts` | Fonte do `req.id` gravado em `webhook_outbox.requestId` (seção 9.3). |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository` → `WebhookService` → `WebhookController` com o mesmo `prisma` e registra `webhooks` no objeto `Controllers`. |
| `src/routes/index.ts` | O tipo `Controllers` ganha `webhooks: WebhookController`. `buildApiRouter` monta `router.use('/webhooks', buildWebhookRouter(...))`, `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(...))` (com `Router({ mergeParams: true })`, registrado depois de `/customers`; rotas não casadas pelo router de customers seguem adiante) e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. `customer.routes.ts` não muda. |
| `src/server.ts` | Modelo para o novo `src/worker.ts`: `bootstrap()`, `logger.info(..., 'webhook_worker_started')`, handlers `SIGINT`/`SIGTERM` com `prisma.$disconnect()` ([09:11] Larissa). |
| `src/config/database.ts` | O worker importa o `prisma` exportado. Como o módulo instancia o client ao ser carregado, o processo worker tem **sua própria** instância (mesma `DATABASE_URL`) ([09:30] Bruno), sem criar um segundo client. |
| `src/config/env.ts` | O `envSchema` ganha `WEBHOOK_SECRET_ENCRYPTION_KEY: z.string().regex(/^[0-9a-f]{64}$/)` (32 bytes em hex). Se faltar, o processo não sobe, pelo mesmo mecanismo de `loadEnv()`. O worker valida o mesmo `.env`. Também entra em `.env.example`. |
| `src/shared/logger/index.ts` | Reusado (`logger.child(...)` no worker). Acrescentar `'*.secret'` e `'*.previousSecret'` em `redactPaths`. |
| `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |
| `package.json` | Scripts novos: `"worker": "tsx watch --env-file=.env src/worker.ts"` e `"start:worker": "node --env-file=.env dist/worker.js"` ([09:11] Larissa). O build atual (`tsconfig.build.json`) já compila `src/**`. |
| `tests/setup.ts` / `tests/helpers/factories.ts` | `beforeEach` passa a limpar `webhookDeadLetter`, `webhookDelivery`, `webhookOutbox` e `webhookEndpoint` antes de `customer`. Nova factory `createTestWebhook()`. |

Estrutura do módulo novo ([09:27]–[09:28] Bruno). **Nenhum destes arquivos existe hoje; todos são criados pela feature:**

```
src/worker.ts                              entry-point do processo worker
src/modules/webhooks/
  webhook.controller.ts                    CRUD, rotate-secret, deliveries, replay
  webhook.service.ts                       regras de negócio, geração de secret
  webhook.repository.ts                    acesso Prisma às 4 tabelas
  webhook.routes.ts                        buildWebhookRouter + buildCustomerWebhookRouter + buildWebhookAdminRouter
  webhook.schemas.ts                       Zod (https, events ⊂ OrderStatus)
  webhook.errors.ts                        classes WEBHOOK_* (extends AppError)
  webhook.publisher.ts                     publishWebhookEvent(tx, ...) + renderPayload
  webhook.processor.ts                     loop, envio, retry, dead letter
  webhook.signer.ts                        HMAC-SHA256 / buildHeaders
  webhook.constants.ts                     2s, 10s, backoff, 64 KB, 24h, 100
```

## 14. Decisões de implementação tomadas neste FDD

Pontos que a reunião não detalhou e que este FDD resolveu (🔧). Cada um tem fundamento numa fala ou restrição da reunião. As decisões de impacto arquitetural viraram ADR (ADR-003: contagem de tentativas; ADR-004: cifra da secret; ADR-008: ordem e isolamento).

| # | Decisão | Fundamento |
|---|---|---|
| D1 | Uma linha de outbox por (evento × endpoint) | `X-Webhook-Id` por envio ([09:44]) e secret por endpoint ([09:21]) |
| D2 | Limite de 64 KB verificado no worker, não na inserção | Não reverter a mudança de status por causa de notificação ([09:04], [09:23]) |
| D3 | Duas assinaturas em `X-Signature` durante o grace period | "Antiga fica válida por 24h em paralelo" ([09:21] Sofia) |
| D4 | Replay reaproveita a linha da outbox e o `eventId` | Dedup do cliente ([09:25]) |
| D5 | Endpoint inativo descarta pendentes sem dead letter | O cliente optou por parar. Não é falha de entrega. |
| D6 | DELETE em cascata; pausa via `active=false` | "Estado ativo" na configuração ([09:21] Bruno) |
| D7 | Redirects não seguidos | Obrigatoriedade de `https` ([09:23]) |
| D8 | Lote de 50, stats a cada ~1 min, resposta truncada em 2 KB | "Batch pequeno" ([09:08]). Valores iniciais calibráveis. |
| D9 | Constantes em código; única variável de ambiente nova é a chave de cifra | "Não vamos botar nada novo" ([09:29]); ADR-004 |
| D10 | Envio isolado por endpoint e ordem por pedido mantida durante o retry ([ADR-008](adrs/ADR-008-ordem-por-pedido-e-isolamento-por-endpoint.md)) | Cliente lento não trava os outros ([09:04]); ordem por pedido ([09:12]–[09:13]) |
| D15 | 1 envio + 5 retentativas ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)) | 5 intervalos e ~15h ([09:17]) |
| D16 | Secret cifrada com AES-256-GCM ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) | "Se vaza uma, vaza tudo" ([09:21]); vazamento real ([09:22]) |
| D11 | 5º parâmetro opcional `{ requestId }` em `publishWebhookEvent` | Correlação (seção 9.3); a assinatura de 4 parâmetros de [09:41] continua válida |
| D12 | `WEBHOOK_SECRET_REQUIRED` usado como invariante do worker; rotação e replay recusados em endpoint inativo | Códigos citados em [09:28] Bruno; estado ativo de [09:21] |
| D13 | Listagem em `/customers/:customerId/webhooks` | "body ou path" ([09:32] Larissa) |
| D14 | `logger.child` no worker; shutdown gracioso como em `src/server.ts` | Reuso do Pino ([09:29]) |

## 15. Plano de entrega

Estimativa da Larissa ([09:46]), na ordem sugerida:

| Sprint | Entregas |
|---|---|
| 1 | "Modelagem de outbox e DLQ": seção 4, migration |
| 2 | "Worker e retry": seções 5.2–5.4, assinatura HMAC (`webhook.signer.ts`), observabilidade. Testado com linhas inseridas direto na outbox pelas factories, já que a integração com pedidos só chega na sprint 3. |
| 3 | "CRUD de configuração e deliveries" (½ sprint) + "integração no order.service e testes ponta a ponta" (½ sprint) + schemas e validações + **revisão de segurança da Sofia (≥ 2 dias úteis)** ([09:46] Sofia) |

> A Larissa estimou os blocos sem ordená-los ([09:46]). A ordem acima antecipa o HMAC para a sprint 2 porque o worker não envia sem assinar.

# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Revisores | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| Status | Pronto para revisão técnica |
| Documentos relacionados | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

Este documento descreve **como construir** a feature. O porquê de cada decisão está nos ADRs; aqui só aparecem as referências.

---

## 1. Contexto e motivação técnica

A aplicação (Node.js + TypeScript, Express, Prisma sobre MySQL) não tem nenhum mecanismo de evento, fila ou notificação externa. A única forma de um cliente saber que um pedido mudou é consultar `GET /api/v1/orders` repetidamente.

Toda mudança de status passa por um único ponto: `OrderService.changeStatus` em `src/modules/orders/order.service.ts`, que roda em `prisma.$transaction` e, na mesma transação, valida a transição (`canTransition`), debita ou repõe estoque, atualiza `orders` e grava `order_status_history`. É nesse ponto que o evento de webhook nasce ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 2. Objetivos técnicos

| # | Objetivo | Medida |
| --- | --- | --- |
| OT-1 | Nenhuma mudança de status commitada sem evento correspondente, e nenhum evento sem mudança commitada | Evento gravado na mesma transação do `changeStatus` |
| OT-2 | Início da entrega em menos de 10 s após o commit | Polling de 2 s; pior caso de espera na fila = 2 s |
| OT-3 | Resistir a indisponibilidade do cliente por até ~15 h | Envio inicial + 5 tentativas: 1 min, 5 min, 30 min, 2 h, 12 h |
| OT-4 | Cliente consegue autenticar e deduplicar cada envio | `X-Signature` (HMAC-SHA256) e `X-Event-Id` |
| OT-5 | Mudança de status não fica mais lenta por causa do cliente | Nenhuma chamada HTTP dentro da transação |

## 3. Escopo e exclusões

**Incluso:** módulo novo `src/modules/webhooks/` (CRUD de configuração, histórico de entregas, rotação de secret, replay de DLQ), tabelas novas no Prisma, função `publishWebhookEvent` chamada no `changeStatus`, worker em `src/worker.ts` (novo) com script `npm run worker`.

**Excluído desta entrega:**

- Webhooks de entrada (cliente → plataforma).
- E-mail ao cliente quando o webhook falha repetidamente (próxima fase).
- Dashboard visual (projeto do time de frontend).
- Arquivamento/limpeza de linhas entregues da outbox (~30 dias).
- Rate limiting de saída por cliente (observar e decidir depois).
- Múltiplos workers e ordering global.
- Restrição do CRUD de configuração por role (hoje: qualquer usuário autenticado).

## 4. Modelo de dados

Novos modelos em `prisma/schema.prisma`, seguindo o padrão existente (`id` UUID `@db.Char(36)`, `createdAt`/`updatedAt`, `@@map` em snake_case). Os nomes `webhook_outbox` e `webhook_dead_letter` vêm da reunião; `webhook_endpoints` e `webhook_deliveries` são propostos aqui.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
}

model WebhookEndpoint {
  id                     String        @id @default(uuid()) @db.Char(36)
  customerId             String        @db.Char(36)
  url                    String        @db.VarChar(2048)
  secret                 String        @db.VarChar(128)
  previousSecret         String?       @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  statuses               Json          // OrderStatus[] que o endpoint quer receber
  active                 Boolean       @default(true)
  createdAt              DateTime      @default(now())
  updatedAt              DateTime      @updatedAt

  customer   Customer           @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36) // X-Event-Id: um UUID por evento, igual para todos os endpoints
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)
  payload       Json                // snapshot (ADR-007)
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@unique([eventId, webhookId])
  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id          String   @id @default(uuid()) @db.Char(36)
  eventId     String   @db.Char(36)   // eventId original, preservado no replay
  webhookId   String   @db.Char(36)
  orderId     String   @db.Char(36)
  payload     Json
  reason      String   @db.VarChar(500)
  attempts    Int
  failedAt    DateTime @default(now())
  replayedAt  DateTime?
  replayedById String? @db.Char(36)

  @@index([webhookId])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text
  durationMs     Int
  error          String?  @db.VarChar(500)
  createdAt      DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}
```

Notas de modelagem:

- **Uma linha de outbox por (evento × endpoint)** — proposta deste FDD: como o filtro de status e a secret são por endpoint, cada destino precisa de estado de retry próprio. O `eventId` é **um UUID por evento**, gerado na inserção e repetido em todas as linhas daquele evento, porque o `X-Event-Id` é "único por evento" ([09:25] Diego, [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). Um cliente com dois endpoints recebe o mesmo `X-Event-Id` nos dois.
- **Status da outbox.** A reunião listou "pendente, processando, falhou, entregue" ([09:08] Diego), mas depois decidiu que a falha definitiva vai para tabela separada ([09:18] Diego). Por isso a outbox não tem `FAILED`: um evento aguardando retry continua `PENDING` com `nextAttemptAt` no futuro; ao esgotar, sai da outbox e entra em `webhook_dead_letter` ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
- **Secret recuperável.** O HMAC exige a secret em claro no momento de assinar; ela não pode ser guardada só como hash. Como ela é protegida em repouso é ponto da revisão de segurança da Sofia ([09:46]) e fica como questão aberta no [RFC](RFC.md).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro do `changeStatus`)

```mermaid
sequenceDiagram
  participant C as OrderController
  participant S as OrderService.changeStatus
  participant P as publishWebhookEvent
  participant DB as MySQL (tx)
  C->>S: PATCH /orders/:id/status
  S->>DB: BEGIN; findUnique(order)
  S->>S: canTransition(from, to)
  S->>DB: debit/replenish stock
  S->>DB: update orders; insert order_status_history
  S->>P: publishWebhookEvent(tx, order, from, to)
  P->>DB: select webhook_endpoints ativos do customer cujo filtro contém "to"
  alt nenhum endpoint interessado
    P-->>S: return (não insere nada)
  else um ou mais
    P->>P: monta payload (snapshot) e valida ≤ 64 KB
    P->>DB: insert webhook_outbox (mesmo eventId, 1 linha por endpoint, status PENDING)
  end
  S->>DB: COMMIT (ou ROLLBACK de tudo se qualquer passo falhar)
```

Regras:

1. `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `tx` da transação corrente — nunca o `prisma` global ([09:41] Bruno).
2. O filtro é aplicado **aqui**: se nenhum endpoint ativo do customer quer `toStatus`, nada é inserido ([09:34] Bruno).
3. O payload é montado agora e gravado como snapshot ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)).
4. Payload serializado acima de 64 KB lança `WebhookPayloadTooLargeError`. Como qualquer falha da outbox, isso desfaz a mudança de status ([09:40] Bruno). Na prática não deve ocorrer: o payload tem tamanho fixo e não inclui itens ([09:24] Diego: "nenhum evento nosso vai chegar perto disso"; [09:43] Diego: sem items).

### 5.2 Processamento pelo worker

`src/worker.ts` (novo) cria o próprio `PrismaClient` via `createPrismaClient()` e chama o loop de `src/modules/webhooks/webhook.processor.ts` (novo):

```text
a cada 2 s:
  1. SELECT … FROM webhook_outbox
       WHERE status = 'PENDING' AND next_attempt_at <= NOW()
       ORDER BY created_at ASC
       LIMIT <lote pequeno>
  2. para cada evento, em ordem:
       a. marca PROCESSING
       b. carrega o endpoint; se inativo ou removido → move para DLQ com motivo "endpoint inativo" (proposta deste FDD)
       c. assina o corpo (payload exato gravado) e faz POST com timeout de 10 s
       d. grava uma linha em webhook_deliveries (status HTTP, corpo da resposta, duração, erro)
       e. 2xx → DELIVERED
          caso contrário → fluxo de retry (5.3)
```

- Processamento sequencial em ordem de `created_at`, com um único worker → ordem preservada por pedido ([09:12] Diego). Ordem global não é garantida.
- Ao iniciar, o worker devolve para `PENDING` eventos que ficaram em `PROCESSING` (crash no meio do envio). Isso pode gerar reenvio — coberto pelo at-least-once.
- `SIGINT`/`SIGTERM`: termina o evento corrente, para o loop e chama `$disconnect()`, no mesmo molde de `src/server.ts`.

### 5.3 Retry com backoff

| Tentativa que falhou | Próxima tentativa em |
| --- | --- |
| 1ª (envio inicial) | +1 min |
| 2ª | +5 min |
| 3ª | +30 min |
| 4ª | +2 h |
| 5ª | +12 h |
| 6ª | → DLQ |

Leitura da tabela: o envio inicial é seguido de **5 retentativas** (1m/5m/30m/2h/12h), somando quase 15 h entre a primeira falha e a última tentativa ([09:17] Diego). Em cada falha: `attempts += 1`, `lastError` preenchido, `status = PENDING`, `nextAttemptAt = now + intervalo`.

Conta como falha: erro de rede/DNS/TLS, timeout de 10 s, qualquer status fora de 2xx.

### 5.4 DLQ e replay

1. Ao esgotar as tentativas, na mesma transação: insere em `webhook_dead_letter` (`eventId`, payload, motivo da última falha, tentativas, `failedAt`) e remove a linha da outbox.
2. Um usuário `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:35] Diego).
3. O serviço, em transação: insere de volta em `webhook_outbox` com **o mesmo `eventId`** e o mesmo payload, `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`; marca `replayedAt` e `replayedById` na DLQ.
4. Loga `webhook_dlq_replayed` com o id do usuário ([09:36] Sofia).

Manter o `eventId` faz o cliente reconhecer o replay como o mesmo evento.

### 5.5 Rotação de secret

1. `POST /api/v1/webhooks/:id/rotate-secret` gera uma nova secret.
2. `previousSecret ← secret`, `previousSecretExpiresAt ← now + 24 h`, `secret ← nova`.
3. Enquanto `previousSecretExpiresAt > now`, o worker envia as duas assinaturas em `X-Signature` (nova primeiro), e o cliente aceita qualquer uma. Depois do prazo, só a nova ([09:21] Sofia). *Esta mecânica é proposta deste FDD; está como questão em aberto no RFC até a revisão de segurança.*

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <jwt>` (`authenticate`) e respondem erro no formato do `errorMiddleware`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

**Decisão de contrato sobre `customer_id`:** ele **não** vem do JWT, porque o token é do usuário operador, não do cliente ([09:32] Larissa). A reunião deixou aberto "body ou path"; este FDD escolhe o **path** (`/customers/:customerId/webhooks`) para criação e listagem, alinhado ao estilo REST das rotas existentes. Pendente de confirmação no RFC.

| # | Método e caminho | Role | Sucesso |
| --- | --- | --- | --- |
| E1 | `POST /api/v1/customers/:customerId/webhooks` | autenticado | 201 |
| E2 | `GET /api/v1/customers/:customerId/webhooks` | autenticado | 200 |
| E3 | `PATCH /api/v1/webhooks/:id` | autenticado | 200 |
| E4 | `DELETE /api/v1/webhooks/:id` | autenticado | 204 |
| E5 | `GET /api/v1/webhooks/:id/deliveries` | autenticado | 200 |
| E6 | `POST /api/v1/webhooks/:id/rotate-secret` | autenticado | 200 |
| E7 | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | `ADMIN` | 202 |

### E1 — Criar webhook

```http
POST /api/v1/customers/7b1c…/webhooks
Content-Type: application/json

{
  "url": "https://erp.atlas.example/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

`201 Created` — a secret só aparece nesta resposta e na de rotação:

```json
{
  "id": "c2f0…",
  "customerId": "7b1c…",
  "url": "https://erp.atlas.example/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_3f9a…",
  "createdAt": "2026-10-08T12:00:00.000Z"
}
```

Validação (Zod): `url` obrigatória e com protocolo `https` ([09:23] Sofia); `statuses` lista não vazia de valores de `OrderStatus`. Erros: 400 `WEBHOOK_INVALID_URL` (URL fora do padrão, ver nota da §7), 400 `VALIDATION_ERROR` (demais campos), 404 `NOT_FOUND` (customer inexistente).

### E2 — Listar webhooks do customer

`200 OK` (sem `secret`):

```json
{
  "data": [
    { "id": "c2f0…", "url": "https://erp.atlas.example/hooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"], "active": true,
      "createdAt": "2026-10-08T12:00:00.000Z" }
  ]
}
```

### E3 — Editar webhook

```json
{ "url": "https://erp.atlas.example/v2/hooks", "statuses": ["PAID", "SHIPPED"], "active": false }
```

Todos os campos opcionais, pelo menos um obrigatório. `200 OK` com o recurso (sem `secret`). Erros: 404 `WEBHOOK_NOT_FOUND`, 400 `WEBHOOK_INVALID_URL`. Tentar enviar `secret` no body retorna 400 `VALIDATION_ERROR` — secret só muda por E6.

### E4 — Remover webhook

`204 No Content`. Erro: 404 `WEBHOOK_NOT_FOUND`. Eventos pendentes desse endpoint vão para a DLQ com motivo "endpoint inativo" quando o worker os pegar (5.2-b).

### E5 — Histórico de entregas

`GET /api/v1/webhooks/:id/deliveries` — últimas 100 tentativas ([09:34] Marcos); ordenação da mais recente para a mais antiga é proposta deste FDD.

```json
{
  "data": [
    {
      "id": "d91e…",
      "eventId": "5a7d…",
      "attempt": 2,
      "success": false,
      "responseStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 412,
      "payload": { "event_type": "order.status_changed", "…": "…" },
      "createdAt": "2026-10-08T12:01:02.000Z"
    }
  ]
}
```

Erro: 404 `WEBHOOK_NOT_FOUND`.

### E6 — Rotacionar secret

Sem body. `200 OK`:

```json
{ "id": "c2f0…", "secret": "whsec_new…", "previousSecretExpiresAt": "2026-10-09T12:00:00.000Z" }
```

Erros: 404 `WEBHOOK_NOT_FOUND`; 409 `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` se já houver secret anterior dentro da carência (evita três secrets simultâneas).

### E7 — Replay de DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, sem body. `202 Accepted`:

```json
{ "eventId": "5a7d…", "status": "PENDING", "replayedBy": "a1b2…" }
```

Erros: 401 `UNAUTHORIZED`, 403 `FORBIDDEN` (não-ADMIN, via `requireRole`), 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND`, 409 `WEBHOOK_ALREADY_REPLAYED`.

### Envio para o cliente (contrato de saída)

```http
POST https://erp.atlas.example/hooks/orders
Content-Type: application/json
X-Event-Id: 5a7d…
X-Webhook-Id: c2f0…
X-Timestamp: 2026-10-08T12:00:01.234Z
X-Signature: sha256=9b2c…

{
  "event_id": "5a7d…",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-08T12:00:00.000Z",
  "order_id": "e4d3…",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "7b1c…",
  "total_cents": 125000
}
```

- Headers definidos em [09:44]–[09:45]: `X-Event-Id`, `X-Signature`, `X-Timestamp` (momento do envio, para o cliente detectar replay attack), `X-Webhook-Id`, `Content-Type`.
- `X-Signature = HMAC-SHA256(secret, corpo exato em bytes)` em hex ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Durante a carência de rotação: `sha256=<nova>,sha256=<antiga>` — *proposta deste FDD, em aberto no RFC até a revisão da Sofia* (a reunião decidiu só que a antiga vale 24 h em paralelo, [09:21] Sofia).
- Sem itens do pedido; o cliente consulta `GET /orders/:id` se precisar ([09:43] Diego).
- Resposta esperada: qualquer 2xx em até 10 s. O corpo é ignorado (só gravado em `webhook_deliveries`).

## 7. Matriz de erros

Todos estendem as classes de `src/shared/errors/http-errors.ts` e são exportados em `src/shared/errors/index.ts` ou num `webhook.errors.ts` do módulo.

| Código | HTTP | Classe base | Quando |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `AppError` (404) | `:id` de webhook inexistente ou de outro customer |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL malformada ou sem `https` |
| `WEBHOOK_SECRET_REQUIRED` | 500 | `AppError` | Worker encontra endpoint sem secret ao assinar (estado inválido; evento não é enviado sem assinatura) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload serializado > 64 KB em `publishWebhookEvent` |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Nova rotação durante carência de 24 h |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `AppError` (404) | `:id` inexistente na DLQ |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `ConflictError` | Item da DLQ com `replayedAt` preenchido |
| `VALIDATION_ERROR` | 400 | já existente | Demais falhas de schema Zod |
| `FORBIDDEN` | 403 | já existente | Replay por não-ADMIN |

**Sobre `WEBHOOK_INVALID_URL`:** o `validate()` de `src/middlewares/validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`, então uma regra `https` só no schema nunca sairia com o código que Bruno citou ([09:28]). Proposta: o schema valida formato e a regra `https` é checada no `WebhookService`, que lança `WebhookInvalidUrlError`. Isso desloca a regra que Sofia pôs "no schema Zod" ([09:23]); registrado para a revisão.

**Origem dos códigos:** `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` foram citados na reunião ([09:28] Bruno). Os demais, e os status HTTP escolhidos, são propostas deste FDD derivadas das regras decididas, seguindo o prefixo `WEBHOOK_` ([09:29] Larissa).

Observação: `NotFoundError` existente usa sempre o código `NOT_FOUND`; para ter `WEBHOOK_NOT_FOUND` a classe nova chama `AppError` direto com status 404.

Erros do lado do envio (não viram resposta HTTP, só log e `lastError`): `WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DELIVERY_HTTP_ERROR` (não-2xx), `WEBHOOK_DELIVERY_NETWORK_ERROR`, `WEBHOOK_ENDPOINT_INACTIVE`.

## 8. Estratégias de resiliência

| Mecanismo | Valor | Origem |
| --- | --- | --- |
| Timeout por chamada HTTP | 10 s | [09:42] Diego |
| Retentativas | 5, backoff 1m/5m/30m/2h/12h | [09:17] Diego |
| Fallback após esgotar | DLQ em tabela separada + replay manual ADMIN | [09:18] Diego |
| Intervalo de polling | 2 s | [09:09] Diego |
| Atomicidade evento × status | mesma transação | [09:06] Diego, [09:40] Bruno |
| Recuperação de crash | `PROCESSING` → `PENDING` no start | proposta deste FDD; o reenvio que ela pode causar é coberto pelo at-least-once ([09:24] Diego) |
| Isolamento de falha | worker em processo próprio | [09:11] Diego |
| Duplicidade | aceita; cliente deduplica por `X-Event-Id` | [09:25] Diego |

Sem rate limiting de saída nesta fase: a decisão foi observar ([09:39] Larissa). As métricas da §9 dão os dados para essa decisão futura.

## 9. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing; tem o logger Pino (`src/shared/logger/index.ts`) e o `requestLogger` com `X-Request-Id` (`src/middlewares/request-logger.middleware.ts`). A proposta usa só o que existe: **logs estruturados**, dos quais as métricas são derivadas na ferramenta de logs.

**Logs** (Pino, mesmo formato snake_case dos eventos atuais como `server_started`):

| Evento | Campos | Onde |
| --- | --- | --- |
| `webhook_event_enqueued` | `eventId`, `webhookId`, `orderId`, `toStatus` | API |
| `webhook_delivery_succeeded` | `eventId`, `webhookId`, `attempt`, `responseStatus`, `durationMs` | worker |
| `webhook_delivery_failed` | idem + `error`, `nextAttemptAt` | worker |
| `webhook_moved_to_dlq` | `eventId`, `webhookId`, `reason`, `attempts` | worker |
| `webhook_dlq_replayed` | `eventId`, `deadLetterId`, `userId` | API (auditoria, [09:36] Sofia) |
| `webhook_secret_rotated` | `webhookId`, `userId` | API |
| `worker_started` / `worker_stopped` | `pollIntervalMs` | worker |

A secret **nunca** é logada: adicionar `*.secret` e `*.previousSecret` em `redactPaths`.

**Métricas** (derivadas dos logs; nomes e limiares são propostas deste FDD, a meta de 10 s vem de [09:02] Marcos):

- `webhook_delivery_latency_ms`: `now − createdAt` do evento no primeiro sucesso. Meta: p95 < 10 s.
- `webhook_deliveries_total{result}`: sucesso/falha por endpoint.
- `webhook_outbox_pending`: contagem de `PENDING` com `nextAttemptAt <= now`, logada a cada ciclo. Crescimento contínuo indica worker parado ou lento.
- `webhook_dlq_total`: entradas na DLQ por período.
- `webhook_events_per_customer_per_minute`: insumo para a questão de rate limiting.

**Tracing:** correlação por ids. O `requestId` da chamada `PATCH /orders/:id/status` vai em `webhook_event_enqueued`; dali em diante o `eventId` liga outbox → tentativas → DLQ → replay, e chega ao cliente em `X-Event-Id`. Isso dá o rastro ponta a ponta sem adicionar dependência de tracing distribuído.

## 10. Dependências e compatibilidade

- **Sem dependência nova obrigatória.** HMAC e UUID via `node:crypto`; HTTP via `fetch` nativo com `AbortController` para o timeout (Node com `--env-file`, já usado em `package.json`, implica Node ≥ 20).
- **Migração Prisma aditiva:** só tabelas novas e uma relação nova em `Customer`. Nenhuma coluna existente muda; rollback = remover as tabelas.
- **Contrato da API atual inalterado:** `PATCH /orders/:id/status` mantém request e response. Só ganha uma escrita a mais na transação.
- **Deploy:** novo processo `npm run worker` ao lado da API, mesma `DATABASE_URL`. A API funciona sem o worker (eventos acumulam como `PENDING`).
- **Revisão de segurança:** 2 dias úteis da Sofia antes do deploy, focada em HMAC e geração de secret ([09:46] Sofia).

## 11. Integração com o sistema existente

| Arquivo | Mudança / uso |
| --- | --- |
| `src/modules/orders/order.service.ts` | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, chamar `await publishWebhookEvent(tx, order, from, to)`. Usa o mesmo `tx` do `prisma.$transaction`; qualquer erro faz rollback do status. `create` não publica evento (pedido nasce `PENDING` e não há transição). |
| `src/modules/orders/order.status.ts` | Os valores de `OrderStatus` e as transições definidas aqui são o domínio válido de `statuses` no filtro do webhook. Nenhuma mudança. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Base das classes `WEBHOOK_*` (§7), no mesmo molde de `InsufficientStockError` e `InvalidStatusTransitionError`. |
| `src/shared/errors/index.ts` | Reexporta as novas classes, se ficarem em `shared`. |
| `src/middlewares/error.middleware.ts` | Sem mudança: o ramo `err instanceof AppError` já serializa `statusCode`, `errorCode` e `details`. |
| `src/middlewares/auth.middleware.ts` | `authenticate` no router de webhooks; `requireRole('ADMIN')` no router de admin. |
| `src/middlewares/validate.middleware.ts` | `validate({ params, body })` com os schemas de `webhook.schemas.ts` (URL `https`, `statuses`). |
| `src/routes/index.ts` | `buildApiRouter` ganha `webhooks` em `Controllers` e registra três montagens: `/customers/:customerId/webhooks`, `/webhooks` e `/admin/webhooks`. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController`, como os demais módulos. |
| `src/shared/logger/index.ts` | Logger reutilizado por API e worker; incluir paths de secret em `redactPaths`. |
| `src/config/database.ts` | O worker chama `createPrismaClient()` para ter instância própria ([09:30] Bruno). |
| `src/server.ts` e `package.json` | Modelo para `src/worker.ts` (novo) (bootstrap, shutdown em `SIGINT`/`SIGTERM`) e para o script novo `worker` (`npm run worker`, [09:11] Larissa). |
| `prisma/schema.prisma` | Novos modelos da §4 e relação `webhooks WebhookEndpoint[]` em `Customer`. |
| `tests/orders.test.ts` e `tests/helpers/factories.ts` | Referência de estilo para os testes novos; factories ganham `createWebhookEndpoint`. |

Estrutura do módulo novo:

```text
src/modules/webhooks/
  webhook.controller.ts
  webhook.service.ts        # CRUD, rotação, replay
  webhook.repository.ts
  webhook.routes.ts         # buildWebhookRouter, buildCustomerWebhookRouter, buildAdminWebhookRouter
  webhook.schemas.ts        # Zod
  webhook.errors.ts         # classes WEBHOOK_*
  webhook.publisher.ts      # publishWebhookEvent(tx, order, from, to)
  webhook.processor.ts      # loop do worker, envio, retry, DLQ
  webhook.signature.ts      # HMAC-SHA256
src/worker.ts
```

## 12. Critérios de aceite técnicos

1. Mudança de status para um status filtrado por um endpoint ativo gera exatamente 1 linha `PENDING` por endpoint interessado, na mesma transação.
2. Se o insert na outbox falhar, o status do pedido, o histórico e o estoque permanecem como antes.
3. Status não filtrado por nenhum endpoint não gera linha na outbox.
4. Com cliente respondendo 2xx, o evento fica `DELIVERED` em menos de 10 s após o commit.
5. Cliente fora do ar: tentativas registradas em `webhook_deliveries` nos intervalos 1m/5m/30m/2h/12h (testável com relógio falso); após a última, linha em `webhook_dead_letter` e nenhuma na outbox.
6. Resposta após 10 s conta como falha.
7. `X-Signature` confere com `HMAC-SHA256(secret, corpo)` calculado pelo teste; `X-Event-Id` é igual em todas as tentativas e no replay.
8. Após rotação, por 24 h o envio carrega assinatura válida para a secret nova e para a antiga; depois, só a nova.
9. `POST /webhooks` com URL `http://` retorna 400 `WEBHOOK_INVALID_URL`.
10. Replay com usuário `OPERATOR` retorna 403; com `ADMIN` retorna 202 e gera log `webhook_dlq_replayed` com `userId`.
11. `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, mais recentes primeiro.
12. Nenhuma secret aparece em log.
13. Testes existentes (`npm test`) continuam passando.

## 13. Riscos e mitigação

Probabilidade e impacto são avaliação deste documento, não números discutidos na reunião.

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Falha no insert da outbox bloqueia mudanças de status | Baixa | Alto | Insert simples e indexado; teste do critério 2; alerta em erro 5xx no `PATCH /orders/:id/status` |
| Worker parado sem ninguém perceber | Média | Alto | Log `webhook_outbox_pending` a cada ciclo; alerta se crescer por N ciclos |
| Cliente lento segura o lote (10 s × N eventos) | Média | Médio | Lote pequeno; timeout de 10 s; se virar problema, entra na discussão de rate limiting / paralelismo |
| Reenvio em crash gera duplicata | Média | Baixo | At-least-once documentado; `X-Event-Id` estável |
| Secret vazada em log | Baixa | Alto | Redaction no Pino; secret só nas respostas de E1 e E6; revisão da Sofia |
| Crescimento da outbox/deliveries | Alta (longo prazo) | Médio | Índices em `status`/`created_at`; arquivamento é trabalho futuro já reconhecido |
| Ordem trocada se alguém subir 2 workers | Baixa | Médio | Documentar "um único worker" no runbook de deploy |

# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Pronto para implementação |
| **Documentos** | [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

---

## 1. Contexto e motivação técnica

O OMS não possui hoje nenhum mecanismo de notificação externa. Clientes B2B fazem polling no `GET /orders` para detectar mudanças de status — integração lenta e cara para eles. A mudança de status é o evento de negócio central: `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`) executa uma transação que valida a transição na máquina de estados (`src/modules/orders/order.status.ts`), atualiza `orders`, insere em `order_status_history` e ajusta estoque.

Este documento especifica a implementação do módulo de webhooks: captura do evento na mesma transação (padrão outbox), entrega assíncrona por worker dedicado, retry, DLQ, assinatura HMAC e API de configuração. As decisões de arquitetura estão registradas nos [ADRs](./adrs/); aqui está o "como construir".

## 2. Objetivos técnicos

1. Registrar o evento `order.status_changed` de forma **atômica** com a mudança de status (commit ⇒ evento registrado; rollback ⇒ evento descartado).
2. Entregar eventos com latência total ≤ 10s (polling de 2s + entrega HTTP).
3. Nenhuma chamada HTTP dentro da transação de pedidos; timeout de 10s por entrega.
4. Semântica at-least-once com `X-Event-Id` estável entre retries.
5. Assinatura HMAC-SHA256 verificável pelo cliente, com secret por endpoint e rotação com grace de 24h.
6. Retry com backoff 1m/5m/30m/2h/12h (5 tentativas) e DLQ persistida com replay manual.
7. Módulo estruturalmente idêntico aos existentes (`src/modules/webhooks`), reusando `AppError`, Pino, error middleware, Zod e auth middleware.

## 3. Escopo e exclusões

**Escopo:** módulo `src/modules/webhooks` (controller, service, repository, routes, schemas, processor), entry point `src/worker.ts` + script `npm run worker`, 4 tabelas novas no Prisma, alteração pontual no `changeStatus`, endpoints de CRUD/deliveries/rotação/replay.

**Exclusões** (ver [PRD §5](./PRD.md#5-escopo)): notificação por email de falhas, rate limiting de saída, dashboard visual, arquivamento da outbox (~30 dias), webhooks inbound, múltiplos workers.

## 4. Modelo de dados

Novos modelos em `prisma/schema.prisma`, seguindo as convenções existentes (UUID `@db.Char(36)`, `@@map` snake_case, índices explícitos):

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  FAILED       // falhou (aguardando retry)
  DELIVERED    // entregue
}

model WebhookEndpoint {
  id                 String    @id @default(uuid()) @db.Char(36)
  customerId         String    @db.Char(36)
  url                String    @db.VarChar(2048)      // https obrigatório (validação Zod)
  secret             String    @db.VarChar(255)       // gerada pelo sistema
  previousSecret     String?   @db.VarChar(255)       // válida durante o grace period
  previousSecretExpiresAt DateTime?                   // rotação + 24h
  statuses           Json                             // lista de OrderStatus assinados
  active             Boolean   @default(true)
  createdAt          DateTime  @default(now())
  updatedAt          DateTime  @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36) // = event_id (X-Event-Id)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)     // "order.status_changed"
  payload       Json                                    // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  @@index([status])
  @@index([createdAt])
  @@index([nextAttemptAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id              String   @id @default(uuid()) @db.Char(36)
  webhookId       String   @db.Char(36)
  eventId         String   @db.Char(36)
  attempt         Int
  success         Boolean
  httpStatus      Int?
  requestPayload  Json
  responseBody    String?  @db.Text
  responseTimeMs  Int?
  deliveredAt     DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, deliveredAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id            String   @id @default(uuid()) @db.Char(36)
  eventId       String   @db.Char(36)
  webhookId     String   @db.Char(36)
  payload       Json
  failureReason String   @db.VarChar(500)
  failedAt      DateTime @default(now())
  replayedAt    DateTime?
  replayedById  String?  @db.Char(36)   // auditoria: quem executou o replay

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas de rastreabilidade: status da outbox (pendente/processando/falhou/entregue) e índices em status/`created_at` vêm de [09:08] Diego; UUID de [09:51] Larissa; snapshot do payload de [09:52] Larissa; campos da configuração (`url + secret + customer_id + estado ativo`) de [09:21] Bruno/Sofia; DLQ com payload, motivo e timestamp de [09:18] Diego; deliveries com sucesso/falha, payload, response e tempo de resposta de [09:34] Marcos. Nomes de campos e tipos são proposta de implementação seguindo `prisma/schema.prisma`.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Dentro de `OrderService.changeStatus`, após o `tx.orderStatusHistory.create` e antes do fim da transação:

```ts
// src/modules/orders/order.service.ts (dentro do this.prisma.$transaction existente)
await publishWebhookEvent(tx, refreshed!, from, to);
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` é uma **função pura que recebe o transaction client** atual (`Prisma.TransactionClient`, o tipo `TxClient` já usado no service) — sem injetar repository no `OrderService`. Comportamento:

1. Busca os `WebhookEndpoint` ativos do `order.customerId` cujo `statuses` contenha `toStatus`.
2. **Se nenhum webhook assina o status, não insere nada** (filtragem na inserção — economiza linha na tabela).
3. Para cada webhook elegível, renderiza o **snapshot** do payload (ver §6.2) e insere uma linha em `webhook_outbox` com `status = PENDING`, `id` = UUID novo (será o `X-Event-Id`).
4. Se a inserção na outbox falhar, a exceção propaga e **a transação inteira sofre rollback** — não pode haver status alterado sem evento registrado.

### 5.2 Processamento pelo worker

Entry point `src/worker.ts` (espelha `src/server.ts`: bootstrap, logger, shutdown gracioso em SIGINT/SIGTERM), script `"worker": "tsx watch --env-file=.env src/worker.ts"` / build equivalente. Instância própria de `PrismaClient` (client é por processo), mesma `DATABASE_URL`. Loop em `src/modules/webhooks/webhook.processor.ts`:

```
a cada 2s:
  1. SELECT eventos com status IN (PENDING, FAILED) AND nextAttemptAt <= now()
     ORDER BY createdAt ASC LIMIT <batch pequeno>
  2. marca o batch como PROCESSING
  3. para cada evento:
     a. monta headers (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id)
     b. POST na url do webhook, timeout 10s
     c. registra WebhookDelivery (attempt, success, httpStatus, payload, response, responseTimeMs)
     d. 2xx  → status = DELIVERED
        outro status / timeout / erro de rede → fluxo de retry (§5.3)
```

Single-worker: a ordem de processamento é `createdAt ASC`, o que garante ordering por `order_id` enquanto houver um único worker (limitação conhecida, ver RFC §5.3).

### 5.3 Retry

Em falha de entrega (status ≠ 2xx, timeout de 10s ou erro de rede):

| Tentativa que falhou | Próximo retry após |
|---|---|
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas → **não há; vai para DLQ** |

```
attempts += 1
if attempts < 5:
  status = FAILED
  nextAttemptAt = now() + BACKOFF[attempts]   // [1m, 5m, 30m, 2h, 12h]
  lastError = <motivo>
else:
  mover para DLQ (§5.4)
```

Janela total ≈ 15 horas entre a primeira falha e a última tentativa. O `X-Event-Id` **não muda** entre tentativas (mesma linha da outbox).

### 5.4 DLQ

Na 5ª falha:

1. Insere em `webhook_dead_letter` com `payload`, `failureReason` (motivo da última falha) e `failedAt`.
2. Remove a linha da outbox (mantém a leitura da outbox principal limpa).

**Replay manual:** `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`) recoloca o evento na outbox como `PENDING` (mesmo `eventId`, `attempts = 0`), marca `replayedAt`/`replayedById` na dead letter e loga quem executou (auditoria).

### 5.5 Rotação de secret

1. `POST /webhooks/:id/rotate-secret` gera secret nova.
2. `previousSecret = secret` atual; `previousSecretExpiresAt = now() + 24h`; `secret = nova`.
3. Durante o grace period **as entregas são assinadas com a secret nova**; a antiga permanece publicada como válida para o cliente que ainda verifica com ela migrar. Após 24h, a antiga é apagada.

## 6. Contratos públicos

Todos os endpoints ficam sob o router registrado em `src/routes/index.ts` (`/webhooks` e `/admin/webhooks`), com `authenticate` aplicado ao router como nos módulos existentes. Erros seguem o envelope do `error.middleware.ts`: `{ "error": { "code", "message", "details?" } }`.

### 6.1 API de configuração

#### `POST /webhooks` — cadastrar webhook

Autenticação: JWT (qualquer role). O `customer_id` vai no **body** (o JWT é do usuário operador, não do cliente).

Request:
```json
{
  "customerId": "9f8b1c2e-1111-4222-8333-444455556666",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (única vez em que a secret aparece em claro):
```json
{
  "id": "3c1d0a8e-7777-4888-9999-000011112222",
  "customerId": "9f8b1c2e-1111-4222-8333-444455556666",
  "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
  "statuses": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d",
  "active": true,
  "createdAt": "2026-07-17T12:00:00.000Z"
}
```

Erros: `400 VALIDATION_ERROR` (Zod), `400 WEBHOOK_INVALID_URL` (URL não-https), `404 NOT_FOUND` (customer inexistente), `401 UNAUTHORIZED`.

#### `GET /webhooks?customerId=...` — listar webhooks de um customer

Response `200 OK` (paginado como os demais módulos, via `paginated` de `src/shared/http/response.ts`; secret nunca retorna):
```json
{
  "data": [
    {
      "id": "3c1d0a8e-7777-4888-9999-000011112222",
      "customerId": "9f8b1c2e-1111-4222-8333-444455556666",
      "url": "https://integracao.atlascomercial.com.br/webhooks/oms",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-17T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

#### `PATCH /webhooks/:id` — editar webhook

Request (campos opcionais):
```json
{ "url": "https://nova-url.atlascomercial.com.br/hooks", "statuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

Response `200 OK`: objeto atualizado (sem secret). Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 VALIDATION_ERROR`.

#### `DELETE /webhooks/:id` — remover webhook

Response `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

#### `POST /webhooks/:id/rotate-secret` — rotacionar secret

Response `200 OK`:
```json
{
  "id": "3c1d0a8e-7777-4888-9999-000011112222",
  "secret": "whsec_1f2e3d4c5b6a79880706152433421100",
  "previousSecretExpiresAt": "2026-07-18T12:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

#### `GET /webhooks/:id/deliveries` — histórico de entregas

Response `200 OK` (mais recentes primeiro, paginado):
```json
{
  "data": [
    {
      "id": "b1a2c3d4-aaaa-bbbb-cccc-ddddeeeeffff",
      "eventId": "e5f6a7b8-1234-4abc-9def-567890abcdef",
      "attempt": 2,
      "success": true,
      "httpStatus": 200,
      "requestPayload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseBody": "{\"ok\":true}",
      "responseTimeMs": 312,
      "deliveredAt": "2026-07-17T12:34:56.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

#### `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar evento da DLQ

Autenticação: JWT + `requireRole('ADMIN')`. Loga `userId` do executor (auditoria).

Response `202 Accepted`:
```json
{ "eventId": "e5f6a7b8-1234-4abc-9def-567890abcdef", "status": "PENDING", "replayedBy": "1a2b3c4d-...-user" }
```

Erros: `403 FORBIDDEN` (role ≠ ADMIN), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_ALREADY_REPLAYED`.

### 6.2 Contrato do webhook entregue (outbound)

`POST <url do webhook>` com timeout de 10s.

Headers:

| Header | Conteúdo |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (gerado na inserção na outbox; estável entre retries — chave de dedup) |
| `X-Signature` | HMAC-SHA256 do corpo do request, com a secret do endpoint |
| `X-Timestamp` | Timestamp do envio (permite ao cliente detectar replay attack) |
| `X-Webhook-Id` | ID do endpoint de webhook (para cliente com vários cadastros identificar qual recebeu) |

Body (payload enxuto — **sem items**; para detalhes o cliente consulta `GET /orders/:id`):
```json
{
  "event_id": "e5f6a7b8-1234-4abc-9def-567890abcdef",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-17T12:34:54.000Z",
  "order_id": "0d1e2f3a-4444-4555-8666-777788889999",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "9f8b1c2e-1111-4222-8333-444455556666",
  "total_cents": 159900
}
```

Semântica de resposta: **2xx = entregue**; qualquer outro status, timeout (10s) ou erro de rede = falha → retry. Limite de payload: **64KB** — evento acima disso não é enviado nem truncado, gera erro (`WEBHOOK_PAYLOAD_TOO_LARGE`) e vai para a DLQ.

## 7. Matriz de erros

Classes derivadas de `AppError` (`src/shared/errors/app-error.ts`), no padrão de `src/shared/errors/http-errors.ts`; o `error.middleware.ts` já as serializa sem mudança. Prefixo `WEBHOOK_` para todo o módulo ([09:28]–[09:29]); os três primeiros códigos foram citados nominalmente na reunião, os demais seguem o mesmo padrão:

| Código | HTTP | Quando ocorre | Contexto |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook `:id` inexistente em GET/PATCH/DELETE/rotate | API |
| `WEBHOOK_INVALID_URL` | 400 | URL não-`https` ou malformada no cadastro/edição | API (schema Zod) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação de assinatura sem secret disponível para o endpoint | API/worker |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de item inexistente na DLQ | API admin |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | Replay de item já reprocessado | API admin |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (worker) | Payload renderizado > 64KB; evento não é enviado, vai para DLQ | Worker |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (worker) | Entrega excedeu 10s; registrado como `lastError`/`failureReason` | Worker |
| `WEBHOOK_DELIVERY_FAILED` | — (worker) | Response ≠ 2xx ou erro de rede; registrado como `lastError`/`failureReason` | Worker |

Erros de validação genéricos continuam como `VALIDATION_ERROR` (Zod via `validate.middleware.ts`) e autorização como `UNAUTHORIZED`/`FORBIDDEN` (auth middleware), como no resto da API.

## 8. Estratégias de resiliência

- **Timeout:** 10s por chamada HTTP do worker; estouro = falha, agenda retry.
- **Retry:** backoff exponencial 1m/5m/30m/2h/12h, 5 tentativas (§5.3).
- **Fallback (falha permanente):** DLQ persistida com replay manual admin (§5.4). Notificação proativa (email) explicitamente adiada.
- **Isolamento:** nenhuma chamada HTTP na transação de pedidos; worker em processo separado — queda ou restart da API não afeta entregas, e cliente lento não afeta a API.
- **Consistência:** outbox transacional — falha na inserção do evento faz rollback da mudança de status; crash do worker entre POST e marcação de entrega resulta em reenvio (coberto pela semântica at-least-once + dedup por `X-Event-Id`).
- **Recuperação de batch órfão:** eventos presos em `PROCESSING` além de um limite de tempo (worker morreu no meio do batch) voltam a ser elegíveis — detalhe de implementação decorrente da semântica at-least-once.

## 9. Observabilidade

Baseada na infra existente: Pino estruturado (`src/shared/logger/index.ts`, com `redact` — **a secret entra na lista de campos redacted**) e `request-logger.middleware.ts` na API.

**Logs (worker e API):** eventos estruturados no padrão do projeto (`server_started`, `shutdown_initiated`): `worker_started`, `outbox_event_enqueued`, `webhook_delivery_attempt`, `webhook_delivery_succeeded` (com `responseTimeMs`), `webhook_delivery_failed` (com motivo e tentativa), `webhook_event_dead_lettered`, `webhook_dlq_replayed` (com `userId` do admin — requisito de auditoria de [09:36] Sofia). Sempre com `event_id`, `webhook_id` e `order_id` como campos.

**Métricas** (derivadas dos SLAs do PRD):
- `webhook_outbox_pending` (gauge) — backlog de pendentes; alerta se crescer continuamente.
- `webhook_delivery_latency_seconds` (histograma, do `createdAt` do evento à entrega) — SLA de 10s.
- `webhook_delivery_attempts_total` (counter, por resultado sucesso/falha).
- `webhook_dead_letter_total` (counter) — falhas permanentes.
- `webhook_delivery_response_time_ms` — tempo de resposta dos endpoints dos clientes.

**Tracing:** o `X-Event-Id` é o identificador de correlação ponta a ponta — presente no log da inserção (transação da API), em cada tentativa do worker, no header entregue ao cliente e no registro de `webhook_deliveries`; permite reconstruir a linha do tempo completa de qualquer evento. Na API, correlaciona-se com o `req.id` do `request-logger.middleware.ts`.

## 10. Dependências e compatibilidade

- **Runtime/stack:** Node ≥ 20, Express 4, Prisma 5.22 / MySQL, Zod 3, Pino 9, `uuid` — todas já no `package.json`; **nenhuma dependência nova obrigatória** (HMAC via `node:crypto`; HTTP via `fetch` nativo do Node 20).
- **Banco:** migração Prisma aditiva (4 tabelas + 1 enum); nenhuma tabela existente é alterada.
- **API existente:** contratos atuais intocados; única mudança de comportamento é a inserção na outbox dentro do `changeStatus` (falha na inserção passa a reverter a mudança de status — comportamento desejado por decisão).
- **Processos:** novo entry point `src/worker.ts` + script npm; deploy passa a ter dois processos (API e worker), mesma imagem/stack.
- **Externa:** endpoints dos clientes devem aceitar POST JSON via https e responder 2xx em ≤ 10s; documentação de integração (assinatura, dedup, semântica at-least-once) publicada no portal de desenvolvedor pelo PM.

## 11. Integração com o sistema existente

| Arquivo | Integração |
|---|---|
| `src/modules/orders/order.service.ts` | **Ponto crítico.** O método `changeStatus` (linha 126) será estendido: dentro do `this.prisma.$transaction` existente, após inserir em `orderStatusHistory`, chama `publishWebhookEvent(tx, order, from, to)` passando o `TxClient` da transação atual (tipo `Prisma.TransactionClient` já definido no arquivo). Falha na inserção da outbox propaga e reverte a transação inteira. Não se injeta repository de webhook no `OrderService` — a integração é por função que recebe o `tx`. |
| `src/modules/orders/order.status.ts` | Fonte da verdade dos status possíveis (`PENDING`→…→`DELIVERED`/`CANCELLED`). A lista `statuses` assinável por um webhook e o filtro de inserção na outbox usam o enum `OrderStatus`; nenhuma mudança neste arquivo. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | As novas classes de erro do módulo (ex.: `WebhookNotFoundError extends NotFoundError`, `WebhookInvalidUrlError extends BadRequestError`) seguem exatamente o padrão de `InsufficientStockError`/`InvalidStatusTransitionError`: herdam de `AppError` com `errorCode` simbólico prefixado `WEBHOOK_`. Exportadas via `src/shared/errors/index.ts` ou no próprio módulo. |
| `src/middlewares/error.middleware.ts` | Nenhuma alteração: já trata `AppError` (serializa `code`/`message`/`details`), `ZodError` e erros Prisma — cobre todos os erros novos do módulo. |
| `src/middlewares/auth.middleware.ts` | Reuso direto: `authenticate` aplicado ao router de webhooks (CRUD com qualquer role autenticada) e `requireRole('ADMIN')` no `POST /admin/webhooks/dead-letter/:id/replay`. |
| `src/middlewares/validate.middleware.ts` | Schemas Zod do módulo (`webhook.schemas.ts`) aplicados via `validate({ body, params, query })`, como em `order.routes.ts` — incluindo a validação de URL `https` (`z.string().url().startsWith('https://')` ou refinamento equivalente). |
| `src/shared/logger/index.ts` | Logger Pino compartilhado; o worker cria seu logger via `createLogger()`. Adicionar `*.secret` aos `redactPaths` para nunca logar secrets de webhook. |
| `src/routes/index.ts` | Registro do router novo: `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`, com o controller adicionado ao tipo `Controllers`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts`: mesmo padrão de bootstrap, logger e shutdown gracioso (SIGINT/SIGTERM com `prisma.$disconnect()`), trocando o `app.listen` pelo loop de polling. |
| `src/shared/http/response.ts` | `paginated()` reutilizado nas listagens (`GET /webhooks`, `GET /webhooks/:id/deliveries`). |
| `prisma/schema.prisma` | Novos modelos (§4) seguindo as convenções vigentes: UUID char(36), `@@map` snake_case, índices explícitos, enum para status. |

## 12. Critérios de aceite técnicos

1. Rollback verificado: falha simulada na inserção da outbox reverte a mudança de status (nenhuma linha em `orders`/`order_status_history` alterada).
2. Filtro na inserção: mudança para status não assinado por nenhum webhook do customer não gera linha na outbox.
3. Snapshot: alterar o pedido após o evento não altera o payload entregue.
4. Worker entrega evento pendente em ≤ 10s do commit (polling 2s + entrega), com os 5 headers do contrato e assinatura HMAC-SHA256 verificável com a secret do endpoint.
5. Sequência de retry: falhas consecutivas produzem `nextAttemptAt` em +1m/+5m/+30m/+2h/+12h; a 5ª falha move o evento para `webhook_dead_letter` com payload, motivo e timestamp, removendo-o da outbox.
6. `X-Event-Id` idêntico em todas as tentativas do mesmo evento.
7. Replay: endpoint admin recoloca o evento como `PENDING`, retorna 403 para role `OPERATOR` e loga o `userId` do executor.
8. Rotação: após `rotate-secret`, assinatura com a secret antiga valida por 24h e é rejeitada depois; a nova vale imediatamente.
9. URL `http://` rejeitada com `WEBHOOK_INVALID_URL` no cadastro e na edição.
10. `GET /webhooks/:id/deliveries` reflete cada tentativa com sucesso/falha, payload, response e `responseTimeMs`.
11. `kill` da API não interrompe o worker (e vice-versa); ambos fazem shutdown gracioso.
12. Secrets nunca aparecem em logs (redact do Pino) nem em respostas de listagem/edição.

## 13. Riscos e mitigação

Riscos de produto/negócio no [PRD §10](./PRD.md#10-riscos-e-mitigação). Riscos específicos de implementação:

| Risco | Mitigação |
|---|---|
| Crash do worker entre o POST e a marcação `DELIVERED` gera reentrega | Comportamento aceito pela semântica at-least-once; dedup pelo `X-Event-Id` no cliente; timeout de recuperação de eventos presos em `PROCESSING` |
| Inserção na outbox alonga a transação de `changeStatus` | Operação é um INSERT simples na mesma conexão (sem I/O de rede); medir no teste ponta a ponta |
| Secret em claro no banco e trafegando na criação/rotação | Resposta expõe a secret apenas na criação/rotação; redact no Pino; revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy cobrindo HMAC e geração de secret |
| Backlog na outbox se todos os clientes ficarem lentos simultaneamente | Batch pequeno + índices (status, `nextAttemptAt`); métrica `webhook_outbox_pending` com alerta; retry espaça tentativas automaticamente |
| Query do worker concorrendo com a API no mesmo MySQL | Batch pequeno em índice seletivo a cada 2s (custo baixo); pool separado por processo dimensionado na config |

# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

O Order Management System (OMS) não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Quando o status de um pedido muda, a alteração fica confinada ao banco de dados. Clientes B2B precisam fazer polling do endpoint `GET /orders` periodicamente para detectar mudanças — uma abordagem ineficiente e que gera carga desnecessária na API.

Este documento especifica **como construir** o Sistema de Webhooks de Notificação de Pedidos. Ele detalha fluxos, contratos, erros, estratégias de resiliência e a integração com o código existente, em nível de implementação. As decisões arquiteturais que fundamentam este design estão nos ADRs ([ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-006](adrs/ADR-006-reuso-padroes-projeto.md)) e a visão geral da proposta está no [RFC](RFC.md).

## Objetivos Técnicos

1. **Desacoplar** a notificação de webhooks da transação de mudança de status, sem perder a garantia de que todo evento será registrado.
2. **Entregar** notificações HTTP para endpoints externos com latência máxima de 10 segundos (p95).
3. **Garantir** at-least-once delivery com idempotência detectável pelo cliente.
4. **Prover** resiliência com retry, backoff e DLQ para eventos que falharem permanentemente.
5. **Reutilizar** ao máximo os padrões, bibliotecas e middlewares já existentes no projeto.
6. **Permitir** que clientes configurem, monitorem e gerenciem seus webhooks via API.

## Escopo e Exclusões

### Incluso

- CRUD de configuração de webhooks por cliente (criação, listagem, edição, remoção)
- Geração de secret criptográfica no cadastro, com rotação suportada
- Filtro de eventos por status do pedido (cliente escolhe quais status quer receber)
- Inserção de eventos na outbox dentro da transação de `changeStatus`
- Worker em processo separado com polling de 2 segundos
- Entrega HTTP POST com assinatura HMAC-SHA256, timeout de 10s
- Retry com backoff exponencial (5 tentativas, ~15h de janela total)
- Dead Letter Queue em tabela separada
- Endpoint de replay manual de DLQ (restrito a ADMIN)
- Histórico de entregas por webhook (últimos 100)
- Headers de idempotência e segurança: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`

### Fora de Escopo (esta fase)

- Email ou notificação proativa de falha para o cliente
- Dashboard visual para gestão de webhooks
- Rate limiting de envio por cliente (observar primeiro)
- Múltiplos workers em paralelo
- Arquivamento automático de eventos entregues com mais de 30 dias
- Revogação imediata de secret (sem grace period)

## Fluxos Detalhados

### Fluxo 1: Criação de evento na Outbox

**Trigger**: método `OrderService.changeStatus()` em `src/modules/orders/order.service.ts`.

**Pré-condição**: a transição de status foi validada (`canTransition(from, to)`), o estoque foi ajustado (`debitStock` ou `replenishStock`).

**Dentro da transação Prisma** (após `tx.order.update` e `tx.orderStatusHistory.create`):

```
1. Buscar webhooks ativos do customer com filtro de status compatível
   SELECT * FROM webhook_config
   WHERE customer_id = :customerId
     AND active = true
     AND (event_types IS NULL OR :toStatus MEMBER OF event_types)

2. Para cada webhook compatível, inserir na outbox:
   INSERT INTO webhook_outbox (id, webhook_id, event_id, event_type, payload, status, created_at)
   VALUES (:uuid, :webhookId, :eventId, 'order.status_changed', :payload, 'PENDING', NOW())

3. Se nenhum webhook compatível, não inserir nada (sem overhead)
```

**Snapshot do payload**: o payload JSON é renderizado no momento da inserção. Se o pedido mudar de status depois, o evento já registrado não é alterado — reflete o estado do pedido no instante da transição.

**Rollback**: se a inserção na outbox falhar, a transação inteira sofre rollback (incluindo `update order` e `insert history`). Consistência garantida.

**Payload do evento** (JSON):

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-29T14:30:00.000Z",
  "data": {
    "order_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "order_number": "ORD-000042",
    "customer_id": "c3d4e5f6-a7b8-9012-cdef-2345678901ab",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "total_cents": 15990
  }
}
```

**Nota**: o payload não inclui `items` para manter o tamanho enxuto. Se o cliente precisar dos detalhes do pedido, ele usa o `order_id` para chamar `GET /orders/:id` ([09:43-44] Diego: "Não manda items pra não inflar").

### Fluxo 2: Processamento pelo Worker

**Entry-point**: `src/worker.ts` (processo Node.js independente, iniciado via `npm run worker`).

**Loop principal**:

```
while (true) {
  const batch = await fetchPendingEvents(limit=20)

  for (const event of batch) {
    await processEvent(event)
  }

  sleep(2000)
}
```

**Detalhamento de `processEvent(event)`**:

```
1. Marcar evento como PROCESSING:
   UPDATE webhook_outbox SET status = 'PROCESSING' WHERE id = :id

2. Buscar configuração do webhook (url, secret):
   SELECT url, secret FROM webhook_config WHERE id = :webhookId AND active = true

3. Se webhook inativo ou não encontrado: marcar como FAILED, pular.

4. Montar requisição HTTP:
   - URL: webhook.url
   - Method: POST
   - Headers:
     Content-Type: application/json
     X-Event-Id: <event.event_id>
     X-Signature: sha256=<HMAC-SHA256(payload, webhook.secret)>
     X-Timestamp: <ISO-8601 now>
     X-Webhook-Id: <webhook_id>
   - Body: event.payload (raw JSON)
   - Timeout: 10_000 ms

5. Enviar requisição.

6. Avaliar resposta:
   - 2xx → marcar DELIVERED (sucesso)
   - 4xx (exceto 429) → marcar FAILED, não retentar
   - 5xx, timeout, network error → marcar FAILED, agendar retry

7. Registrar delivery (tabela webhook_delivery):
   - event_id, webhook_id, request_payload, response_status, response_body (truncado),
     duration_ms, created_at
```

**Batch size**: 20 eventos por poll. Suficiente para volume esperado sem sobrecarregar o banco. Ajustável por variável de ambiente.

**Índices da tabela outbox**:
- `(status, created_at)` para o fetch do worker
- `(webhook_id, created_at)` para queries de histórico

### Fluxo 3: Retry com Backoff Exponencial

Após falha no envio:

```
1. Incrementar attempt_count na outbox
2. Calcular next_retry_at = NOW() + backoff_seconds(attempt_count)
3. Próximo poll: worker só processa eventos com status='PENDING' AND
   (next_retry_at IS NULL OR next_retry_at <= NOW())
```

**Tabela de backoff**:

| Tentativa | Intervalo | next_retry_at |
|-----------|-----------|---------------|
| 1 (envio inicial) | — | — |
| 2 (1º retry) | 1 min | +1 min |
| 3 (2º retry) | 5 min | +6 min |
| 4 (3º retry) | 30 min | +36 min |
| 5 (4º retry) | 2 h | +2h 36min |
| 6 (5º retry) | 12 h | +14h 36min |

Após a 6ª tentativa falhar, o evento é movido para DLQ.

### Fluxo 4: Dead Letter Queue

**Movimento para DLQ** (transação):

```
1. INSERT INTO webhook_dead_letter (event_id, webhook_id, payload, last_response_status,
     last_response_body, failure_reason, attempt_count)
2. DELETE FROM webhook_outbox WHERE id = :id
```

**Reprocessamento** (`POST /admin/webhooks/dead-letter/:id/replay`):

```
1. Validar role ADMIN (requireRole middleware)
2. Buscar registro na webhook_dead_letter
3. Inserir novo registro em webhook_outbox com status PENDING,
   mesmo event_id (para idempotência do lado do cliente),
   attempt_count = 0, next_retry_at = NULL
4. Remover da webhook_dead_letter (ou marcar como replayed)
5. Logar: quem fez replay, qual evento, timestamp
```

## Contratos Públicos

### 1. Criar Webhook — `POST /webhooks`

**Request:**
```json
{
  "customer_id": "c3d4e5f6-a7b8-9012-cdef-2345678901ab",
  "url": "https://client-api.example.com/webhooks/orders",
  "event_types": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "description": "Webhook de produção Atlas"
}
```

**Validação (Zod):**
- `url`: string, deve começar com `https://` (rejeitar `http://`)
- `event_types`: array de `OrderStatus`, opcional (vazio ou omitido = todos os status)
- `customer_id`: UUID do cliente existente
- `description`: string opcional, máx 255 chars

**Response `201 Created`:**
```json
{
  "id": "w01a02b0-3c4d-5e6f-7890-abcdef123456",
  "customer_id": "c3d4e5f6-a7b8-9012-cdef-2345678901ab",
  "url": "https://client-api.example.com/webhooks/orders",
  "secret": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "event_types": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "description": "Webhook de produção Atlas",
  "created_at": "2026-07-29T14:30:00.000Z",
  "updated_at": "2026-07-29T14:30:00.000Z"
}
```

**Importante**: `secret` é retornada **apenas na criação**. Endpoints de leitura (GET, PATCH) nunca re-expõem o valor da secret.

**Status codes**: `201` Created, `400` Validation Error, `404` Customer not found, `409` URL já cadastrada para o mesmo customer.

### 2. Listar Webhooks — `GET /webhooks?customer_id=<id>`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "w01a02b0-3c4d-5e6f-7890-abcdef123456",
      "customer_id": "c3d4e5f6-a7b8-9012-cdef-2345678901ab",
      "url": "https://client-api.example.com/webhooks/orders",
      "event_types": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
      "active": true,
      "description": "Webhook de produção Atlas",
      "created_at": "2026-07-29T14:30:00.000Z",
      "updated_at": "2026-07-29T14:30:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

**Status codes**: `200` OK, `400` customer_id obrigatório.

### 3. Atualizar Webhook — `PATCH /webhooks/:id`

**Request:**
```json
{
  "url": "https://client-api.example.com/webhooks/orders-v2",
  "event_types": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

Todos os campos são opcionais. Enviar apenas o que deseja alterar.

**Response `200 OK`:**
```json
{
  "id": "w01a02b0-3c4d-5e6f-7890-abcdef123456",
  "url": "https://client-api.example.com/webhooks/orders-v2",
  "event_types": ["SHIPPED", "DELIVERED"],
  "active": true,
  "updated_at": "2026-07-29T15:00:00.000Z"
}
```

**Status codes**: `200` OK, `400` Validation Error (URL http://), `404` Not Found.

### 4. Remover Webhook — `DELETE /webhooks/:id`

**Response `204 No Content`**

Webhooks removidos são soft-deleted (`active = false`). Eventos já na outbox para este webhook continuam sendo processados normalmente.

**Status codes**: `204` No Content, `404` Not Found.

### 5. Histórico de Entregas — `GET /webhooks/:id/deliveries`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "d01e02f0-3a4b-5c6d-7890-abcdef123456",
      "event_id": "550e8400-e29b-41d4-a716-446655440000",
      "webhook_id": "w01a02b0-3c4d-5e6f-7890-abcdef123456",
      "event_type": "order.status_changed",
      "response_status": 200,
      "duration_ms": 187,
      "created_at": "2026-07-29T14:30:01.500Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 100,
    "total": 1,
    "totalPages": 1
  }
}
```

Payloads de request e response são omitidos na listagem para reduzir tamanho da resposta. Detalhes completos disponíveis via `GET /webhooks/:id/deliveries/:deliveryId`.

**Status codes**: `200` OK, `404` Webhook not found.

### 6. Reprocessar DLQ — `POST /admin/webhooks/dead-letter/:id/replay`

**Auth**: `authenticate` + `requireRole(['ADMIN'])`.

**Response `200 OK`:**
```json
{
  "message": "Event re-enqueued for delivery",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "webhook_id": "w01a02b0-3c4d-5e6f-7890-abcdef123456"
}
```

**Status codes**: `200` OK, `401` Unauthorized, `403` Forbidden, `404` Dead letter event not found.

### 7. Rotacionar Secret — `POST /webhooks/:id/rotate-secret`

**Response `200 OK`:**
```json
{
  "message": "Secret rotated. Old secret remains valid for 24 hours.",
  "secret": "f6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c1b0a9f8e7d6c5b4a3f2e1d0c9b8a7",
  "previous_secret_expires_at": "2026-07-30T15:00:00.000Z"
}
```

A `secret` retornada é a nova. A anterior permanece válida por 24h.

## Matriz de Erros

Todos os erros seguem o padrão `AppError` e usam o prefixo `WEBHOOK_`. O error middleware centralizado (`src/middlewares/error.middleware.ts`) os serializa automaticamente no formato `{ error: { code, message, details? } }`.

| Código | HTTP | Gatilho | Classe |
|--------|------|---------|--------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook não encontrado pelo ID | `WebhookNotFoundError` extends `NotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | URL não começa com `https://` | `WebhookInvalidUrlError` extends `ValidationError` |
| `WEBHOOK_URL_ALREADY_EXISTS` | 409 | URL já cadastrada para o mesmo customer | `WebhookUrlAlreadyExistsError` extends `ConflictError` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Tentativa de criar webhook sem secret (não deveria ocorrer — secret é gerada) | `WebhookSecretRequiredError` extends `ValidationError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento excede 64KB | `WebhookPayloadTooLargeError` extends `UnprocessableEntityError` |
| `WEBHOOK_DELIVERY_FAILED` | — | (interno) Falha no envio HTTP que dispara retry | `WebhookDeliveryError` extends `AppError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Evento na DLQ não encontrado | `WebhookDeadLetterNotFoundError` extends `NotFoundError` |
| `WEBHOOK_INACTIVE` | 422 | Tentativa de operação em webhook inativo | `WebhookInactiveError` extends `UnprocessableEntityError` |

**Formato de resposta de erro (padrão do projeto):**
```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must start with https://",
    "details": {
      "url": "http://client-api.example.com/webhooks"
    }
  }
}
```

## Estratégias de Resiliência

### Timeouts

- **HTTP call do worker**: 10 segundos. Cliente que não responde em 10s é tratado como falha e vai para retry ([09:42] Diego).
- **Conexão TCP**: 5 segundos (conexão que não estabelece em 5s é tratada como timeout de rede).
- **Transação de banco**: sem timeout explícito — a transação do `changeStatus` é curta (operações de update/insert locais). A inserção na outbox adiciona overhead desprezível.

### Retry com Backoff

Ver [Fluxo 3](#fluxo-3-retry-com-backoff-exponencial). Backoff exponencial com 5 tentativas, janela total de ~15h.

**Regras de retry:**
- **Retentável**: timeout (ETIMEDOUT, ECONNABORTED), 5xx, ECONNREFUSED, ENOTFOUND, 429 (Rate Limit)
- **Não retentável**: 4xx (exceto 429) — erros do cliente (URL errada, payload inválido, auth falhou). Estes vão direto para FAILED sem retry.

### DLQ

Ver [Fluxo 4](#fluxo-4-dead-letter-queue). Eventos que esgotarem as 5 tentativas são movidos para `webhook_dead_letter`, visíveis para inspeção e reprocessamento manual.

### Fallback

Não há fallback automático (ex: enviar por email em vez de webhook). Se o webhook falhar permanentemente, o evento vai para DLQ e o time de operação decide como agir.

### Graceful Shutdown

O worker (`src/worker.ts`) implementa graceful shutdown similar ao `src/server.ts`:
- Captura SIGINT/SIGTERM
- Termina o processamento do batch atual (não inicia novas requisições)
- Fecha a conexão Prisma (`prisma.$disconnect()`)
- Loga o shutdown

## Observabilidade

### Métricas

Métricas expostas via endpoint `/metrics` ou coletadas por agente externo. Nomes seguem convenção `webhook_*`:

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_outbox_size` | Gauge | Número de eventos com status PENDING |
| `webhook_delivery_total` | Counter | Total de tentativas de entrega (labels: `status=success|failed`) |
| `webhook_delivery_duration_ms` | Histogram | Duração da chamada HTTP (buckets: 50, 100, 250, 500, 1000, 2500, 5000, 10000) |
| `webhook_dlq_size` | Gauge | Número de eventos na dead letter queue |
| `webhook_retry_count` | Counter | Número de retries disparados |
| `webhook_worker_heartbeat` | Gauge | Timestamp do último poll bem-sucedido (para detectar worker parado) |

### Logs

Logger Pino (`src/shared/logger/index.ts`) reutilizado. Níveis e eventos:

| Nível | Evento | Campos |
|-------|--------|--------|
| `info` | Evento inserido na outbox | `event_id`, `webhook_id`, `order_id`, `event_type` |
| `info` | Entrega bem-sucedida | `event_id`, `webhook_id`, `response_status`, `duration_ms` |
| `warn` | Retry agendado | `event_id`, `webhook_id`, `attempt_count`, `next_retry_at`, `failure_reason` |
| `error` | Evento movido para DLQ | `event_id`, `webhook_id`, `attempt_count`, `last_response_status` |
| `info` | Replay de DLQ | `event_id`, `webhook_id`, `replayed_by` (userId do admin) |
| `warn` | Timeout HTTP | `event_id`, `webhook_id`, `url`, `timeout_ms` |
| `error` | Erro inesperado no worker | `err`, `stack` |

**Redaction**: secrets e tokens são automaticamente redactados pelo Pino (configuração existente em `src/shared/logger/index.ts:4-11`).

### Tracing

O middleware `request-logger.middleware.ts` já gera correlation IDs para requisições HTTP da API. Para o worker, cada batch de processamento gera um `batch_id` (UUID) repassado a todas as operações daquele ciclo, permitindo correlacionar logs de um mesmo poll.

## Dependências e Compatibilidade

### Dependências novas

Nenhuma. A feature não introduz novas bibliotecas. HMAC-SHA256 usa o módulo nativo `crypto` do Node.js. HTTP calls usam `fetch` nativo (Node 18+) ou `http`/`https` nativos.

### Schema do banco — novas tabelas

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookConfig {
  id          String     @id @default(uuid()) @db.Char(36)
  customerId  String     @db.Char(36)
  url         String     @db.VarChar(2048)
  secret      String     @db.VarChar(255)
  eventTypes  Json?      // ["PAID", "SHIPPED", ...] ou null = todos
  active      Boolean    @default(true)
  description String?    @db.VarChar(255)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  customer    Customer   @relation(fields: [customerId], references: [id])
  deliveries  WebhookDelivery[]

  @@index([customerId, active])
  @@index([url, customerId])
  @@map("webhook_configs")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  webhookId     String              @db.Char(36)
  eventId       String              @db.Char(36)  // X-Event-Id UUID
  eventType     String              @db.VarChar(100)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attemptCount  Int                 @default(0)
  nextRetryAt   DateTime?
  createdAt     DateTime            @default(now())

  webhook       WebhookConfig       @relation(fields: [webhookId], references: [id])

  @@index([status, createdAt])
  @@index([webhookId, createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id              String   @id @default(uuid()) @db.Char(36)
  webhookId       String   @db.Char(36)
  eventId         String   @db.Char(36)
  requestPayload  Json
  responseStatus  Int?
  responseBody    String?  @db.Text
  durationMs      Int
  createdAt       DateTime @default(now())

  webhook         WebhookConfig @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id                  String   @id @default(uuid()) @db.Char(36)
  eventId             String   @db.Char(36)
  webhookId           String   @db.Char(36)
  payload             Json
  lastResponseStatus  Int?
  lastResponseBody    String?  @db.Text
  failureReason       String?  @db.VarChar(1000)
  attemptCount        Int
  createdAt           DateTime @default(now())

  @@index([webhookId])
  @@map("webhook_dead_letters")
}
```

### Rotação de secret — schema

Para suportar grace period de 24h, o modelo `WebhookConfig` precisa armazenar a secret anterior e sua data de expiração. Alternativa: campo `previousSecret` + `previousSecretExpiresAt`. Esta modelagem cabe ao time de implementação decidir entre:

1. **Campos adicionais em `WebhookConfig`**: `previousSecret`, `previousSecretExpiresAt`
2. **Tabela separada `WebhookSecretHistory`**: mais limpa, permite auditoria de rotações passadas

Recomendação: opção 1 para a v1 (simples). Opção 2 se auditoria de rotações for necessária.

## Critérios de Aceite Técnicos

1. **Atomicidade**: a inserção na `webhook_outbox` e a mudança de status do pedido são atômicas — ou ambas commitam ou ambas sofrem rollback.
2. **Latência**: 95% dos eventos são entregues em menos de 10 segundos da mudança de status, medido como `webhook_delivery.created_at - webhook_outbox.created_at`.
3. **Dedup**: o mesmo `event_id` nunca gera duas inserções na outbox para o mesmo webhook.
4. **Retry**: um evento com `status=FAILED` e `attempt_count < 5` tem `next_retry_at` preenchido. O worker processa eventos cujo `next_retry_at <= NOW()`.
5. **DLQ**: evento com `attempt_count >= 6` e último envio com falha é movido para `webhook_dead_letter` e removido da `webhook_outbox`.
6. **HMAC**: toda requisição HTTP do worker inclui header `X-Signature` com HMAC-SHA256 válido sobre o body.
7. **TLS**: cadastro de webhook com URL `http://` retorna 400 `WEBHOOK_INVALID_URL`.
8. **Admin replay**: endpoint `POST /admin/webhooks/dead-letter/:id/replay` é acessível apenas para role `ADMIN`.
9. **Secret não vaza**: endpoints `GET /webhooks` e `PATCH /webhooks/:id` nunca retornam o campo `secret`.
10. **Payload snapshot**: o `payload` na outbox reflete o estado do pedido no momento da transição, e não muda se o pedido for alterado posteriormente.

## Integração com o Sistema Existente

Esta seção descreve exatamente como o novo módulo de webhooks se integrará ao código base existente, arquivo por arquivo.

### 1. `src/modules/orders/order.service.ts`

**O que é**: o método `changeStatus()` (linhas 126-179) executa a transação de mudança de status. Hoje ele atualiza `orders`, insere `order_status_history` e ajusta `stock_quantity`.

**Como integrar**: dentro da transação `prisma.$transaction(async (tx) => { ... })`, após `tx.orderStatusHistory.create(...)` e antes do `tx.order.findUnique(...)` final, adicionar:

```typescript
// Publicar eventos de webhook
await publishWebhookEvent(tx, {
  orderId: order.id,
  orderNumber: order.orderNumber,
  customerId: order.customerId,
  fromStatus: from,
  toStatus: to,
  totalCents: order.totalCents,
});
```

A função `publishWebhookEvent` é importada de `src/modules/webhooks/webhook.service.ts` e recebe o `tx` (TransactionClient) — mesma assinatura das funções privadas `debitStock(tx, items)` e `replenishStock(tx, items)` que já existem no service.

**Impacto**: adição de ~5 linhas. A assinatura pública de `changeStatus()` não muda. Nenhum teste existente quebra, desde que o mock do Prisma permita a nova tabela (ou que a função seja mockada nos testes).

### 2. `prisma/schema.prisma`

**O que é**: define o schema do banco. Hoje contém `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`.

**Como integrar**: adicionar os modelos `WebhookConfig`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`, enum `WebhookOutboxStatus`. Adicionar relação `Customer.webhooks WebhookConfig[]` ao modelo `Customer`.

**Impacto**: apenas novas tabelas — nenhuma tabela existente é alterada. Rodar `npx prisma migrate dev` para gerar migration.

### 3. `src/shared/errors/http-errors.ts`

**O que é**: define as classes de erro do projeto (`NotFoundError`, `ConflictError`, `ValidationError`, `UnprocessableEntityError`, etc.) e erros específicos como `InvalidStatusTransitionError` e `InsufficientStockError`.

**Como integrar**: adicionar novas classes estendendo as existentes, seguindo o mesmo padrão:

```typescript
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook'); this.errorCode = 'WEBHOOK_NOT_FOUND'; }
}

export class WebhookInvalidUrlError extends ValidationError {
  constructor(url: string) {
    super('Webhook URL must start with https://', [{ path: 'url', message: `Invalid URL: ${url}` }]);
    this.errorCode = 'WEBHOOK_INVALID_URL';
  }
}

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(size: number, maxSize: number) {
    super(`Webhook payload size ${size} exceeds limit of ${maxSize} bytes`, 'WEBHOOK_PAYLOAD_TOO_LARGE');
  }
}
```

**Impacto**: novas classes no mesmo arquivo, mesmo padrão. O `error.middleware.ts` as trata automaticamente (pois estendem `AppError`).

### 4. `src/middlewares/auth.middleware.ts`

**O que é**: middlewares `authenticate` (JWT) e `requireRole(roles)`.

**Como integrar**: usado sem alterações nas rotas de webhook:

```typescript
// src/modules/webhooks/webhook.routes.ts
router.post('/', authenticate, validate(createWebhookSchema), controller.create);
router.get('/', authenticate, validate(listWebhooksSchema), controller.list);
router.patch('/:id', authenticate, validate(updateWebhookSchema), controller.update);
router.delete('/:id', authenticate, controller.delete);
router.get('/:id/deliveries', authenticate, controller.listDeliveries);

// Admin only:
router.post('/admin/webhooks/dead-letter/:id/replay', authenticate, requireRole('ADMIN'), controller.replayDlq);
```

**Impacto**: zero alterações no middleware. Reuso puro.

### 5. `src/middlewares/error.middleware.ts`

**O que é**: middleware centralizado que captura `AppError`, `ZodError` e `Prisma` errors.

**Como integrar**: **zero alterações**. Como os novos erros (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`, o bloco `if (err instanceof AppError)` (linha 15) os captura automaticamente e serializa no formato `{ error: { code, message, details? } }`.

### 6. `src/middlewares/validate.middleware.ts`

**O que é**: middleware factory que valida `body`, `query` e `params` contra schemas Zod.

**Como integrar**: usado nas rotas de webhook com schemas definidos em `src/modules/webhooks/webhook.schemas.ts`:

```typescript
export const createWebhookSchema = z.object({
  body: z.object({
    customer_id: z.string().uuid(),
    url: z.string().url().refine(u => u.startsWith('https://'), 'URL must start with https://'),
    event_types: z.array(z.nativeEnum(OrderStatus)).optional(),
    description: z.string().max(255).optional(),
  }),
});
```

**Impacto**: zero alterações no middleware. Apenas schemas novos no módulo.

### 7. `src/shared/logger/index.ts`

**O que é**: factory do Pino com redaction de campos sensíveis.

**Como integrar**: importado e usado no worker e no módulo de webhooks. A configuração de redaction já cobre `*.token`, `*.secret`, `*.password` — a secret do webhook é automaticamente redactada.

### 8. `src/server.ts`

**O que é**: bootstrap da API HTTP.

**Como integrar**: **sem alterações**. A API continua iniciando normalmente. O worker tem sua própria entry-point `src/worker.ts`, que segue o mesmo padrão de bootstrap:

```typescript
// src/worker.ts
import { prisma } from './config/database.js';
import { logger } from './shared/logger/index.js';
import { WebhookProcessor } from './modules/webhooks/webhook.processor.js';

async function bootstrap(): Promise<void> {
  const processor = new WebhookProcessor(prisma);
  // ... loop de polling com graceful shutdown
}

bootstrap().catch((err) => {
  logger.fatal({ err }, 'worker_bootstrap_failed');
  process.exit(1);
});
```

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Conflito de conexão Prisma entre API e worker (ambos acessam mesma tabela outbox) | Baixa | Médio | PrismaClient separado por processo. MySQL gerencia concorrência via MVCC. |
| Worker processar mesmo evento duas vezes (race condition entre poll e update de status) | Baixa | Médio | `UPDATE ... WHERE id = :id AND status = 'PENDING'` com verificação de linhas afetadas. Sempre usar status como condição no update. |
| Payload do evento exceder 64KB | Muito Baixa | Baixo | Payload enxuto (sem items, só campos essenciais). Validação de tamanho na inserção. |
| Cliente cadastrar URL maliciosa (SSRF) | Baixa | Alto | Validação de URL (precisa ser https). Timeout de 10s. Worker isolado em rede (recomendação de infra). |
| Tabela outbox crescer sem controle | Média | Alto | Índices otimizados. Arquivamento de eventos `DELIVERED` > 30 dias (fora do escopo da v1, mas estruturalmente previsto). |

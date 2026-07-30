# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

Três clientes B2B estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente um mecanismo de notificação em tempo real sobre mudanças de status de pedidos. Hoje, esses clientes fazem polling periódico do endpoint `GET /orders` para detectar alterações, o que gera latência imprevisível, carga desnecessária na API e insatisfação crescente.

A Atlas Comercial indicou que, se a funcionalidade não for entregue até o fim do trimestre, considerará migrar para um concorrente — o que representa risco de churn para um cliente estratégico.

A feature "Sistema de Webhooks de Notificação de Pedidos" permitirá que clientes B2B cadastrem endpoints HTTP para receber notificações automáticas sempre que o status de um pedido seu mudar, eliminando a necessidade de polling.

## Problema e Motivação

### Problema atual

| Dimensão | Descrição |
|----------|-----------|
| **Latência** | Clientes só detectam mudanças de status no próximo ciclo de polling, que pode levar minutos |
| **Carga** | A maioria das chamadas a `GET /orders` retorna "sem mudanças", consumindo recursos da API e do cliente |
| **Experiência** | Integração é reativa e manual, em contraste com concorrentes que oferecem webhooks nativos |
| **Risco comercial** | Atlas Comercial ameaçou churn se a funcionalidade não for entregue até o fim do trimestre |

### Proposta de valor

Clientes B2B poderão ser notificados automaticamente sobre mudanças de status de pedidos, com latência inferior a 10 segundos, payload assinado criptograficamente para verificação de autenticidade, e visibilidade completa do histórico de entregas.

## Público-alvo e Cenários de Uso

### Público-alvo primário

**Desenvolvedores e engenheiros de integração** dos clientes B2B que consomem a API do OMS. Eles precisam:

- Configurar endpoints de webhook para receber notificações
- Verificar a autenticidade das notificações recebidas (assinatura HMAC)
- Monitorar o histórico de entregas e investigar falhas
- Rotacionar secrets de autenticação

### Público-alvo secundário

**Operadores e administradores internos** (role `ADMIN`) que:

- Reprocessam eventos que foram para a dead letter queue
- Investigam falhas de entrega em nome dos clientes

### Cenários de uso

1. **Cadastro de webhook**: um cliente B2B acessa o portal de desenvolvedor, cadastra a URL `https://api.cliente.com/webhooks/orders`, seleciona os status `SHIPPED` e `DELIVERED`, e recebe uma secret para configurar a verificação de assinatura no lado dele.

2. **Recebimento de notificação**: um pedido do cliente muda de `PAID` para `PROCESSING`. Em até 10 segundos, o cliente recebe um HTTP POST no endpoint cadastrado com os dados do pedido e headers de segurança.

3. **Investigação de falha**: o cliente percebe que não recebeu notificações nas últimas horas. Ele consulta `GET /webhooks/:id/deliveries`, vê que as últimas entregas falharam com erro `ECONNREFUSED`, corrige o endpoint e as próximas notificações chegam normalmente.

4. **Reprocessamento de DLQ**: um operador ADMIN acessa o endpoint de replay para reenfileirar um evento que foi para a dead letter queue após o cliente corrigir o endpoint.

## Objetivos e Métricas de Sucesso

### Objetivo 1: Entrega de notificações com baixa latência

- **Métrica**: latência entre mudança de status e entrega do webhook (p95)
- **Meta**: ≤ 10 segundos
- **Medição**: `webhook_delivery.created_at - webhook_outbox.created_at` para eventos com `status=DELIVERED`

### Objetivo 2: Alta taxa de entrega bem-sucedida

- **Métrica**: percentual de eventos entregues com sucesso na primeira tentativa
- **Meta**: ≥ 95%
- **Medição**: `webhook_delivery_total{status="success"} / webhook_delivery_total` (excluindo retries)

### Objetivo 3: Zero perda de eventos

- **Métrica**: eventos de mudança de status que não geraram registro na outbox
- **Meta**: 0 (zero)
- **Medição**: garantia por design (inserção atômica na transação); verificável por auditoria amostral

### Objetivo 4: Adoção pelos clientes-alvo

- **Métrica**: número de clientes B2B com pelo menos um webhook ativo
- **Meta**: Atlas Comercial, MaxDistribuição e Nova Cargo ativos em até 30 dias pós-launch

## Escopo

### Incluso

| ID | Requisito | Descrição Resumida |
|----|-----------|-------------------|
| FR-01 | Cadastro de webhook | Cliente cadastra URL, eventos de interesse; sistema gera e retorna secret |
| FR-02 | Listagem de webhooks | Cliente lista seus webhooks cadastrados com paginação |
| FR-03 | Edição de webhook | Cliente altera URL, eventos de interesse e status ativo/inativo |
| FR-04 | Remoção de webhook | Cliente remove (soft-delete) um webhook |
| FR-05 | Filtro por status | Cliente escolhe quais status de pedido disparam notificação |
| FR-06 | Notificação automática | Sistema envia HTTP POST com payload JSON a cada mudança de status |
| FR-07 | Assinatura HMAC-SHA256 | Cada notificação inclui header X-Signature para verificação de autenticidade |
| FR-08 | Idempotência por evento | Cada notificação inclui X-Event-Id UUID único; cliente deduplica |
| FR-09 | Histórico de entregas | Cliente consulta últimas 100 entregas por webhook (status, duração, resposta) |
| FR-10 | Rotação de secret | Cliente solicita nova secret; anterior permanece válida por 24h (grace period) |
| FR-11 | Reprocessamento de DLQ | Admin reprocessa evento da dead letter queue via endpoint restrito |
| FR-12 | TLS obrigatório | Cadastro de URL http:// é rejeitado com erro de validação |

### Fora de Escopo

| Item | Motivo | Origem |
|------|--------|--------|
| **Notificação de falha por email** | Complexidade adicional de integração com serviço de email. Será reavaliado após medição do volume real de DLQ em produção. | [09:37] Larissa: "Não. Email tá fora de escopo dessa fase." |
| **Dashboard visual para clientes** | Projeto separado do time de frontend. Clientes gerenciam webhooks exclusivamente via API. | [09:39-40] Marcos e Larissa |
| **Rate limiting de envio por cliente** | Observar comportamento em produção antes de implementar. Sem dados, qualquer limite seria arbitrário. | [09:38-39] Diego e Larissa |
| **Múltiplos workers em paralelo** | Single worker atende o volume esperado. Ordenação implícita por order_id seria perdida com paralelismo. | [09:12-13] Diego |
| **Arquivamento automático de eventos antigos** | Eventos DELIVERED com +30 dias serão arquivados, mas o mecanismo fica para fase seguinte. | [09:08] Diego |

## Requisitos Funcionais

### RF-01: Cadastro de endpoint de webhook

`POST /webhooks` — Cliente autenticado cadastra URL, lista de status de interesse e descrição opcional. Sistema gera secret criptográfica (32 bytes hex), retorna na resposta (única vez). URL precisa usar HTTPS.

**Validações**: URL `https://`, customer_id válido, sem duplicata de URL por customer.

### RF-02: Listagem de webhooks

`GET /webhooks?customer_id=<id>` — Lista webhooks do cliente com paginação. Não retorna o campo `secret`.

### RF-03: Edição de webhook

`PATCH /webhooks/:id` — Permite alterar URL, event_types, active, description. Não altera a secret (uso de endpoint específico de rotação).

### RF-04: Remoção de webhook

`DELETE /webhooks/:id` — Soft-delete (`active = false`). Eventos já na outbox continuam sendo processados normalmente.

### RF-05: Filtro de eventos por status

No cadastro/edição, o campo `event_types` aceita lista de `OrderStatus`. Eventos só são inseridos na outbox se o `to_status` estiver na lista (ou se a lista for nula/vazia = todos os status).

### RF-06: Notificação em mudança de status

Toda transição de status de pedido gera inserção na tabela `webhook_outbox` com payload JSON snapshot. Worker entrega via HTTP POST para a URL cadastrada.

### RF-07: Assinatura criptográfica

Cada requisição de webhook inclui header `X-Signature: sha256=<hex>`, onde `<hex>` é o HMAC-SHA256 do corpo da requisição usando a secret do webhook.

### RF-08: Idempotência

Cada requisição inclui header `X-Event-Id` com UUID único do evento. Cliente é responsável por deduplicar usando este identificador. Semântica de entrega: at-least-once.

### RF-09: Histórico de entregas

`GET /webhooks/:id/deliveries` — Retorna últimas 100 entregas com status HTTP, duração e timestamp. Payload completo disponível em endpoint de detalhe.

### RF-10: Rotação de secret

`POST /webhooks/:id/rotate-secret` — Gera nova secret. A anterior permanece válida por 24h. Resposta informa data de expiração da secret antiga.

### RF-11: Reprocessamento de DLQ

`POST /admin/webhooks/dead-letter/:id/replay` — Restrito a ADMIN. Reinsere evento na outbox com status PENDING. Registra em log quem executou o replay.

### RF-12: Validação de TLS

Cadastro com URL `http://` retorna erro `400 WEBHOOK_INVALID_URL`. Apenas `https://` é aceito.

## Requisitos Não Funcionais

| ID | Requisito | Descrição |
|----|-----------|-----------|
| RNF-01 | Latência | p95 de entrega ≤ 10 segundos da mudança de status ([09:02] Marcos) |
| RNF-02 | Atomicidade | Inserção na outbox é atômica com a transação de mudança de status ([09:06] Diego) |
| RNF-03 | Resiliência | 5 tentativas com backoff exponencial; falha permanente vai para DLQ ([09:17] Diego) |
| RNF-04 | Timeout HTTP | 10 segundos por chamada de webhook ([09:42] Diego) |
| RNF-05 | Limite de payload | 64KB máximo; payloads maiores geram erro ([09:24] Diego e Larissa) |
| RNF-06 | Segurança | HMAC-SHA256, TLS obrigatório, secret por endpoint, rotação com grace period ([09:20-23] Sofia) |
| RNF-07 | Isolamento | Worker executa como processo separado da API ([09:11] Diego) |
| RNF-08 | Observabilidade | Métricas de entrega, tamanho de outbox, heartbeat do worker; logs estruturados com Pino |
| RNF-09 | Consistência | Reuso dos padrões do projeto: AppError, Zod, Pino, estrutura de módulos ([09:30] Larissa) |
| RNF-10 | Disponibilidade | Worker com health check e restart automático (Docker, systemd ou PM2) |

## Decisões e Trade-offs Principais

| Decisão | Trade-off | ADR |
|---------|-----------|-----|
| Outbox em MySQL | Simplicidade e zero infra adicional vs. latência de polling (2s) e tabela crescente | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| 5 retries com backoff | Alta probabilidade de entrega (~15h de janela) vs. latência máxima de ~15h para eventos que falham | [ADR-002](adrs/ADR-002-retry-backoff-dlq.md) |
| HMAC-SHA256 por endpoint | Segurança forte e isolada por cliente vs. responsabilidade de verificação no cliente | [ADR-003](adrs/ADR-003-hmac-sha256-autenticacao.md) |
| At-least-once com X-Event-Id | Simplicidade de implementação vs. dedup é responsabilidade do cliente | [ADR-004](adrs/ADR-004-at-least-once-idempotencia.md) |
| Worker separado com polling | Isolamento de falha e deploy vs. single worker é ponto único de falha na v1 | [ADR-005](adrs/ADR-005-worker-polling-separado.md) |
| Reuso dos padrões existentes | Consistência e menor superfície de bugs vs. menor flexibilidade para escolhas diferentes | [ADR-006](adrs/ADR-006-reuso-padroes-projeto.md) |

## Dependências

| Dependência | Tipo | Estado |
|-------------|------|--------|
| MySQL (mesma instância) | Infraestrutura | Existente |
| Prisma ORM | Biblioteca | Existente |
| Pino (logger) | Biblioteca | Existente |
| Middleware de autenticação JWT | Código | Existente — `src/middlewares/auth.middleware.ts` |
| Middleware de erros centralizado | Código | Existente — `src/middlewares/error.middleware.ts` |
| Node.js `crypto` (HMAC) | Runtime | Nativo (Node 18+) |
| Node.js `fetch` (HTTP client) | Runtime | Nativo (Node 18+) |

A feature **não introduz novas dependências** de infraestrutura ou bibliotecas externas.

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| **Worker cair silenciosamente** — webhooks param de ser entregues sem alerta | Média | Alto | Health check + restart automático. Métrica `webhook_worker_heartbeat` com alerta se parar por >30s. |
| **Tabela outbox crescer e degradar performance do banco** — eventos acumulados não processados ou não arquivados | Média | Alto | Índices em `(status, created_at)`. Métrica `webhook_outbox_size` com alerta. Arquivamento programado (pós-v1). |
| **Cliente B2B não implementar verificação de assinatura** — perda da garantia de autenticidade para aquele cliente | Alta | Médio | Documentação destacada no portal de desenvolvedor ([09:26] Marcos). Header `X-Signature` sempre enviado. |
| **Vazamento de secret de cliente** — atacante pode forjar webhooks | Baixa | Alto | Rotação com grace period de 24h. Secret nunca é re-exposta. Logs redactam secrets. Endpoint de revogação imediata pode ser adicionado no futuro. |
| **Cliente cadastrar endpoint lento ou não confiável** — worker gasta recursos com timeouts | Média | Médio | Timeout de 10s por chamada. Backoff exponencial espaça retries. Se persistir, evento vai para DLQ. |

## Critérios de Aceitação

### CA-01: Cadastro de webhook
**Dado** que sou um cliente autenticado, **quando** cadastro uma URL HTTPS e seleciono os status `SHIPPED` e `DELIVERED`, **então** recebo resposta 201 com ID do webhook e secret gerada.

### CA-02: Notificação entregue com latência ≤ 10s
**Dado** que um webhook está ativo para o status `PROCESSING`, **quando** um pedido do cliente muda para `PROCESSING`, **então** o endpoint cadastrado recebe um HTTP POST em até 10 segundos (p95).

### CA-03: Verificação de assinatura
**Dado** que recebi uma notificação no meu endpoint, **quando** verifico o header `X-Signature` com minha secret, **então** o HMAC-SHA256 do body confere com a assinatura.

### CA-04: Deduplicação por event_id
**Dado** que já recebi o evento com `X-Event-Id: abc-123`, **quando** recebo a mesma requisição novamente, **então** identifico a duplicata pelo `X-Event-Id` e descarto.

### CA-05: Retry e DLQ
**Dado** que meu endpoint está offline, **quando** um evento é disparado, **então** o sistema retenta 5 vezes com backoff e, após esgotar, move o evento para a dead letter queue.

### CA-06: Reprocessamento de DLQ
**Dado** que sou um ADMIN autenticado, **quando** chamo `POST /admin/webhooks/dead-letter/:id/replay`, **então** o evento volta para a fila de entrega com status PENDING.

### CA-07: TLS obrigatório
**Dado** que tento cadastrar uma URL `http://`, **quando** submeto a requisição, **então** recebo erro 400 com código `WEBHOOK_INVALID_URL`.

### CA-08: Secret não re-exposta
**Dado** que tenho webhooks cadastrados, **quando** consulto `GET /webhooks` ou `PATCH /webhooks/:id`, **então** o campo `secret` nunca aparece na resposta.

### CA-09: Rotação de secret
**Dado** que solicito rotação de secret, **quando** recebo a nova secret, **então** a anterior ainda funciona por 24 horas e a resposta informa a data de expiração.

### CA-10: Fora de escopo respeitado
**Dado** que um webhook falhou permanentemente, **então** o sistema NÃO envia email de notificação (fora de escopo). O cliente consulta o status via `GET /webhooks/:id/deliveries`.

## Estratégia de Testes e Validação

### Testes unitários

- **`webhook.service.test.ts`**: CRUD de configuração, validações de schema (URL, event_types), rotação de secret, regras de filtro de eventos
- **`webhook.processor.test.ts`**: assinatura HMAC, montagem de headers, avaliação de resposta HTTP (2xx, 4xx, 5xx, timeout), lógica de retry e DLQ
- **`order.service.test.ts`**: verificar que `publishWebhookEvent` é chamada dentro da transação de `changeStatus` com os parâmetros corretos

### Testes de integração

- **Fluxo ponta a ponta**: criar webhook → mudar status de pedido → verificar inserção na outbox → mockar worker → verificar chamada HTTP com headers corretos
- **Retry e DLQ**: simular endpoint que falha 5 vezes → verificar movimento para DLQ → replay → verificar reinserção na outbox
- **Atomicidade**: simular falha na inserção da outbox → verificar que `order.status` não foi alterado (rollback)

### Testes de segurança

- **HMAC**: verificar que assinatura confere e que adulteração do payload quebra a assinatura
- **TLS**: verificar que URL `http://` é rejeitada no schema Zod
- **Autorização**: verificar que endpoint de replay exige role ADMIN
- **Secret não vaza**: verificar que `GET /webhooks` não inclui campo `secret`

### Testes de performance

- Medir latência p95 de entrega com volume simulado de 100 eventos
- Medir impacto no `changeStatus` (a inserção na outbox não deve adicionar mais que 5ms à transação)

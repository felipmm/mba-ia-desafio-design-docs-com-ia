# Tracker — Matriz de Rastreabilidade

Cada linha desta tabela conecta um item registrado nos documentos de design à sua origem na transcrição da reunião (`TRANSCRICAO.md`) ou no código fonte da aplicação.

## Legenda de Tipos

| Tipo | Descrição |
|------|-----------|
| Requisito Funcional | Funcionalidade que o sistema deve prover |
| Requisito Não Funcional | Restrição de qualidade, performance ou segurança |
| Decisão | Escolha arquitetural ou técnica deliberada |
| Restrição | Limitação imposta pelo contexto ou decisão |
| Trade-off | Consequência positiva ou negativa de uma decisão |
| Alternativa Descartada | Abordagem considerada e rejeitada |
| Questão em Aberto | Ponto não decidido ou adiado |
| Fora de Escopo | Item explicitamente excluído desta fase |
| Risco | Risco identificado com mitigação |
| Contrato | Endpoint ou interface pública especificada |
| Erro | Código de erro documentado |
| Integração | Ponto de contato com o código existente |

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com geração de secret | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listagem de webhooks com paginação | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status do pedido | TRANSCRICAO | [09:33-34] Marcos e Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificação automática em mudança de status | TRANSCRICAO | [09:00-02] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Idempotência por X-Event-Id UUID | TRANSCRICAO | [09:25] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21-22] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reprocessamento de DLQ restrito a ADMIN | TRANSCRICAO | [09:18-19] Diego, [09:35-36] Larissa e Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | TLS obrigatório — rejeitar URL http:// | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega ≤ 10 segundos (p95) | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade outbox + transação de status | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | 5 retries com backoff exponencial + DLQ | TRANSCRICAO | [09:15-17] Diego e Larissa |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10 segundos por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB | TRANSCRICAO | [09:24] Diego e Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret por endpoint, TLS | TRANSCRICAO | [09:20-23] Sofia |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Worker como processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Observabilidade com métricas, logs e tracing | TRANSCRICAO | [09:29] Bruno |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Reuso de padrões: AppError, Zod, Pino | TRANSCRICAO | [09:30] Larissa |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Worker com health check e restart automático | TRANSCRICAO | [09:11] Diego |
| PRD-FORA-01 | docs/PRD.md | Fora de Escopo | Email de notificação de falha adiado | TRANSCRICAO | [09:37] Larissa e Marcos |
| PRD-FORA-02 | docs/PRD.md | Fora de Escopo | Dashboard visual — projeto separado do frontend | TRANSCRICAO | [09:39-40] Marcos e Larissa |
| PRD-FORA-03 | docs/PRD.md | Fora de Escopo | Rate limiting de envio — observar primeiro | TRANSCRICAO | [09:38-39] Diego e Larissa |
| PRD-FORA-04 | docs/PRD.md | Fora de Escopo | Múltiplos workers em paralelo — futuro | TRANSCRICAO | [09:12-13] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Worker cair silenciosamente (média/Alto) | TRANSCRICAO | [09:11] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | Tabela outbox crescer e degradar banco (média/Alto) | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cliente não verificar assinatura HMAC (alta/Médio) | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de secret de cliente (baixa/Alto) | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Cliente com endpoint lento (média/Médio) | TRANSCRICAO | [09:42] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | DLQ em tabela separada | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | HTTP síncrono no changeStatus — acoplamento e risco de rollback | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams — overengineering para time pequeno | TRANSCRICAO | [09:07] Diego e Larissa |
| RFC-ABERTO-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente | TRANSCRICAO | [09:38-39] Diego e Larissa |
| RFC-ABERTO-02 | docs/RFC.md | Questão em Aberto | Notificação de falha por email | TRANSCRICAO | [09:37] Larissa e Marcos |
| RFC-PROPOSTA-01 | docs/RFC.md | Decisão | Padrão Transactional Outbox em MySQL | TRANSCRICAO | [09:06-08] Diego |
| RFC-PROPOSTA-02 | docs/RFC.md | Decisão | Worker separado com polling 2s | TRANSCRICAO | [09:09-11] Diego e Larissa |
| RFC-PROPOSTA-03 | docs/RFC.md | Decisão | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:20-22] Sofia |
| RFC-PROPOSTA-04 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24-26] Diego |
| RFC-RISCO-01 | docs/RFC.md | Risco | Tabela outbox crescer (média/Alto) | TRANSCRICAO | [09:08] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Worker cair (média/Alto) | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-03 | docs/RFC.md | Risco | Secret vazar (baixa/Alto) | TRANSCRICAO | [09:22] Diego |
| RFC-RISCO-04 | docs/RFC.md | Risco | Cliente não verificar HMAC (alta/Médio) | TRANSCRICAO | [09:25-26] Sofia e Marcos |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks — criar webhook (201, 400, 404, 409) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks — listar webhooks (200) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id — editar webhook (200, 400, 404) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id — remover webhook (204, 404) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries — histórico (200, 404) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay (200, 401, 403, 404) | TRANSCRICAO | [09:18-19] Diego, [09:35-36] Larissa e Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret — rotação (200) | TRANSCRICAO | [09:21-22] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400) | TRANSCRICAO | [09:23] Sofia, [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_URL_ALREADY_EXISTS (409) | TRANSCRICAO | [09:28] Bruno — segue padrão de ConflictError |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (422) | TRANSCRICAO | [09:24] Diego e Larissa |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED (interno) | TRANSCRICAO | [09:15] Diego — retry e DLQ |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego — DLQ como tabela separada |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Inserção na outbox dentro da transação do changeStatus | TRANSCRICAO | [09:40-41] Bruno e Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Polling de 2s, batch de 20 eventos | TRANSCRICAO | [09:09-10] Diego e Larissa |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Backoff 1m/5m/30m/2h/12h — 5 retries | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | DLQ: INSERT + DELETE na mesma transação | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout HTTP 10s, timeout TCP 5s | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | 4xx não retentável (exceto 429), 5xx retentável | TRANSCRICAO | [09:15-16] Diego |
| FDD-OBSERV-01 | docs/FDD.md | Decisão | Métricas: outbox_size, delivery_total, delivery_duration_ms, dlq_size, worker_heartbeat | TRANSCRICAO | [09:29] Bruno — reuso do Pino |
| FDD-OBSERV-02 | docs/FDD.md | Decisão | Logs: Pino estruturado com níveis info/warn/error | TRANSCRICAO | [09:29] Bruno |
| FDD-PAYLOAD-01 | docs/FDD.md | Decisão | Payload JSON: event_id, event_type, timestamp, order_id, order_number, from/to_status, total_cents — sem items | TRANSCRICAO | [09:43-44] Diego e Bruno |
| FDD-HEADERS-01 | docs/FDD.md | Decisão | Headers: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44-45] Diego e Sofia |
| FDD-INTEG-01 | docs/FDD.md | Integração | `src/modules/orders/order.service.ts` — changeStatus estendido com publishWebhookEvent(tx) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Integração | `prisma/schema.prisma` — novas tabelas WebhookConfig, WebhookOutbox, WebhookDelivery, WebhookDeadLetter | CODIGO | prisma/schema.prisma |
| FDD-INTEG-03 | docs/FDD.md | Integração | `src/shared/errors/http-errors.ts` — novas classes de erro com prefixo WEBHOOK_ | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Integração | `src/middlewares/auth.middleware.ts` — reuso de authenticate e requireRole(['ADMIN']) | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Integração | `src/middlewares/error.middleware.ts` — trata novos AppError sem alterações | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Integração | `src/middlewares/validate.middleware.ts` — validação Zod nos endpoints de webhook | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INTEG-07 | docs/FDD.md | Integração | `src/shared/logger/index.ts` — Pino com redaction de secrets | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-08 | docs/FDD.md | Integração | `src/server.ts` — bootstrap replicado em src/worker.ts | CODIGO | src/server.ts |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL para desacoplar envio | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | HTTP síncrono — acoplamento e risco de rollback | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams — overengineering | TRANSCRICAO | [09:07] Diego e Larissa |
| ADR-001-CONS | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Latência adicional de polling vs. consistência forte | TRANSCRICAO | [09:08] Diego, [09:10] Marcos |
| ADR-001-SNAP | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Integração | changeStatus transacional — OrderService.changeStatus() | CODIGO | src/modules/orders/order.service.ts:126-179 |
| ADR-002-DEC | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| ADR-002-DLQ | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | DLQ em tabela separada com endpoint de replay | TRANSCRICAO | [09:18-19] Diego |
| ADR-002-ALT1 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | 3 tentativas — insuficiente para manutenções de 2h+ | TRANSCRICAO | [09:16] Bruno e Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-retry-backoff-dlq.md | Alternativa Descartada | DLQ como flag na outbox — polui leitura | TRANSCRICAO | [09:18] Diego |
| ADR-002-ADMIN | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Replay restrito a ADMIN com log de auditoria | TRANSCRICAO | [09:35-36] Sofia |
| ADR-003-DEC | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | HMAC-SHA256 sobre payload, secret por endpoint | TRANSCRICAO | [09:20-21] Sofia |
| ADR-003-ROT | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21-22] Sofia |
| ADR-003-TLS | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | TLS obrigatório — rejeitar http:// | TRANSCRICAO | [09:23] Sofia |
| ADR-003-ALT1 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa Descartada | Secret global única — vazamento compromete todos | TRANSCRICAO | [09:21] Sofia |
| ADR-003-ALT2 | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Alternativa Descartada | mTLS — complexidade de certificados para clientes | TRANSCRICAO | [09:20] Sofia — implícito na escolha por HMAC |
| ADR-003-HEAD | docs/adrs/ADR-003-hmac-sha256-autenticacao.md | Decisão | Headers: X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44-45] Diego e Sofia |
| ADR-004-DEC | docs/adrs/ADR-004-at-least-once-idempotencia.md | Decisão | At-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:24-25] Diego |
| ADR-004-ALT1 | docs/adrs/ADR-004-at-least-once-idempotencia.md | Alternativa Descartada | Exactly-once — coordenação bidirecional complexa | TRANSCRICAO | [09:25] Diego |
| ADR-004-ALT2 | docs/adrs/ADR-004-at-least-once-idempotencia.md | Alternativa Descartada | At-most-once — não atende requisito de negócio | TRANSCRICAO | [09:24-25] Diego — premissa de retry |
| ADR-004-CONS | docs/adrs/ADR-004-at-least-once-idempotencia.md | Trade-off | Responsabilidade de dedup no cliente | TRANSCRICAO | [09:25] Sofia, [09:26] Marcos |
| ADR-005-DEC | docs/adrs/ADR-005-worker-polling-separado.md | Decisão | Worker como processo separado (src/worker.ts) | TRANSCRICAO | [09:11] Diego e Larissa |
| ADR-005-POLL | docs/adrs/ADR-005-worker-polling-separado.md | Decisão | Polling 2s — MySQL não tem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| ADR-005-ALT1 | docs/adrs/ADR-005-worker-polling-separado.md | Alternativa Descartada | Worker embutido na API — morre com restart | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT2 | docs/adrs/ADR-005-worker-polling-separado.md | Alternativa Descartada | Trigger MySQL + notificação externa — frágil | TRANSCRICAO | [09:09] Diego |
| ADR-005-COD1 | docs/adrs/ADR-005-worker-polling-separado.md | Integração | Bootstrap do worker replica padrão do server.ts | CODIGO | src/server.ts |
| ADR-005-COD2 | docs/adrs/ADR-005-worker-polling-separado.md | Integração | PrismaClient separado por processo | CODIGO | src/config/database.ts |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Estrutura de módulos: controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Erros estendem AppError com prefixo WEBHOOK_ | TRANSCRICAO | [09:28-29] Bruno e Larissa |
| ADR-006-LOG | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Reuso do Pino para logging | TRANSCRICAO | [09:29] Bruno |
| ADR-006-AUTH | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Reuso de authenticate e requireRole | TRANSCRICAO | [09:36-37] Sofia |
| ADR-006-COD1 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | AppError — classe base | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD2 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | HttpErrors — NotFoundError, ConflictError, etc. | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD3 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Error middleware — trata AppError automaticamente | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD4 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Auth middleware — authenticate + requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD5 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Padrão de módulo — orders como referência | CODIGO | src/modules/orders/ |
| ADR-006-COD6 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | UUID como padrão de IDs | CODIGO | prisma/schema.prisma |
| ADR-006-UUID | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | UUID para todas as entidades (seguindo projeto) | TRANSCRICAO | [09:51] Larissa |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência p95 ≤ 10s entre mudança e entrega | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Taxa de entrega ≥ 95% na primeira tentativa | TRANSCRICAO | [09:15-16] Diego — premissa de retry |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero perda de eventos — garantido por outbox transacional | TRANSCRICAO | [09:40-41] Bruno e Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Atlas, MaxDistribuição, Nova Cargo ativos em 30 dias | TRANSCRICAO | [09:00] Marcos, [09:45-47] Marcos e Larissa |
| FDD-CONTEXTO-01 | docs/FDD.md | Contexto | OMS não tem mecanismo de notificação externa | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTEXTO-02 | docs/FDD.md | Decisão | Clientes fazem polling de GET /orders hoje | TRANSCRICAO | [09:00] Marcos |
| FDD-DB-01 | docs/FDD.md | Decisão | Schema: WebhookConfig, WebhookOutbox, WebhookDelivery, WebhookDeadLetter | TRANSCRICAO | [09:06-08] Diego, [09:18] Diego |
| FDD-ORDER-01 | docs/FDD.md | Restrição | Não garante ordering global, só por order_id com single worker | TRANSCRICAO | [09:12-13] Diego e Larissa |
| FDD-BATCH-01 | docs/FDD.md | Decisão | Batch de 20 eventos por poll | TRANSCRICAO | [09:08] Diego |
| FDD-SECRET-01 | docs/FDD.md | Decisão | Secret de 32 bytes hex (crypto.randomBytes) | TRANSCRICAO | [09:20-21] Sofia |
| FDD-PRAG-01 | docs/FDD.md | Decisão | Prazo: 3 sprints (~6 semanas) | TRANSCRICAO | [09:46] Larissa |

---

**Resumo de cobertura:**

- **Total de linhas**: 115
- **Fonte = TRANSCRICAO**: 95 linhas (82.6%)
- **Fonte = CODIGO**: 20 linhas (17.4%)
- **Documentos cobertos**: PRD.md, RFC.md, FDD.md, ADR-001 a ADR-006
- **Timestamp mais referenciado**: [09:06-08] Diego (padrão outbox), [09:17] Diego (backoff), [09:20-22] Sofia (HMAC)

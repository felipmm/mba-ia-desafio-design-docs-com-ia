# Architectural Decision Records

Este diretório contém os ADRs (Architectural Decision Records) do Sistema de Webhooks de Notificação de Pedidos.

## Índice

| ADR | Título | Decisão |
|-----|--------|---------|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Publicação de eventos via Transactional Outbox em MySQL |
| [ADR-002](ADR-002-retry-backoff-dlq.md) | Retry com Backoff e DLQ | 5 tentativas com backoff exponencial + Dead Letter Queue |
| [ADR-003](ADR-003-hmac-sha256-autenticacao.md) | Autenticação HMAC-SHA256 | Assinatura criptográfica com secret por endpoint |
| [ADR-004](ADR-004-at-least-once-idempotencia.md) | At-Least-Once com X-Event-Id | Garantia de entrega com dedup pelo cliente |
| [ADR-005](ADR-005-worker-polling-separado.md) | Worker em Processo Separado | Polling de 2s em processo Node.js independente |
| [ADR-006](ADR-006-reuso-padroes-projeto.md) | Reuso dos Padrões Existentes | AppError, Pino, Zod, estrutura de módulos, códigos WEBHOOK_ |

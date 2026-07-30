# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|-------|-------|
| **Título** | Sistema de Webhooks de Notificação de Mudança de Status de Pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Draft — aberto para revisão |
| **Data** | 2026-07-29 |
| **Revisores** | Diego (Plataforma), Bruno (Pedidos), Sofia (Segurança), Marcos (Produto) |

## Resumo Executivo (TL;DR)

Três clientes B2B estratégicos (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados automaticamente quando o status de seus pedidos mudar, em vez de fazer polling manual da API. A Atlas indicou que pode migrar para o concorrente se a funcionalidade não for entregue até o fim do trimestre.

**Propomos** um sistema de webhooks outbound baseado no padrão Transactional Outbox em MySQL: eventos são persistidos atomicamente junto com a mudança de status do pedido e entregues por um worker independente com retry e dead letter queue. Autenticação via HMAC-SHA256 com secret por cliente. Latência máxima de 2 segundos (bem abaixo do SLA de 10s exigido). Prazo estimado de 3 sprints.

**Pedimos revisão** especialmente nos pontos de segurança (HMAC, rotação de secrets) e nos itens deixados em aberto (rate limiting de saída, notificação de falha por email).

## Contexto e Problema

### Situação atual

Hoje, clientes B2B que precisam acompanhar mudanças de status de pedidos fazem polling periódico do endpoint `GET /orders` com filtros. Essa abordagem tem três problemas:

1. **Latência imprevisível**: o cliente só descobre a mudança no próximo ciclo de polling, que pode ser de minutos.
2. **Carga desnecessária**: a maioria das chamadas retorna "sem mudanças", desperdiçando recursos de ambos os lados.
3. **Risco de churn**: a Atlas Comercial formalizou que, sem notificações em tempo real, considerará migrar para o concorrente.

### Requisito de negócio

Clientes B2B devem receber notificações HTTP (webhooks) sempre que um pedido seu mudar de status, com latência inferior a 10 segundos. Eles precisam conseguir:

- Cadastrar endpoints de webhook (URL, eventos de interesse)
- Receber notificações com dados do pedido e assinatura criptográfica para verificação de autenticidade
- Consultar histórico de entregas
- Reprocessar eventos que falharam (via time de operação, sob demanda)

### Restrições

- Time pequeno (2-3 engenheiros), sem capacidade para operar infraestrutura adicional complexa
- Banco de dados MySQL existente, sem message broker
- Aplicação Node.js + TypeScript com Prisma ORM
- A mudança de status de pedidos já é transacional (atualiza orders, history, estoque) — o mecanismo de notificação não pode comprometer essa transação

## Proposta Técnica

### Visão geral

A solução segue o padrão **Transactional Outbox**:

```
Mudança de status (API)         Worker (processo separado)        Cliente B2B
┌─────────────────────┐        ┌──────────────────────────┐       ┌──────────┐
│ OrderService        │        │ webhook.processor.ts     │       │ Endpoint │
│ changeStatus()      │        │                          │       │ externo  │
│                     │        │ poll a cada 2s            │       │          │
│ ┌─────────────────┐ │        │ ┌──────────────────────┐ │       │          │
│ │ Transação Prisma│ │        │ │ Busca PENDING na     │ │       │          │
│ │                 │ │        │ │ webhook_outbox       │ │       │          │
│ │ update order    │ │        │ └──────────────────────┘ │       │          │
│ │ insert history  │ │        │           │              │       │          │
│ │ adjust stock    │ │        │           ▼              │       │          │
│ │ insert outbox ──┼─┼──DB───┼─► assina HMAC-SHA256    │       │          │
│ └─────────────────┘ │        │           │              │       │          │
│                     │        │           ▼              │       │          │
│ commit ou rollback  │        │  HTTP POST com headers:  │─────►│          │
└─────────────────────┘        │  X-Event-Id,             │      │          │
                               │  X-Signature,            │      │          │
                               │  X-Timestamp             │      │          │
                               │           │              │       │          │
                               │           ▼              │       │          │
                               │  2xx → marca DELIVERED   │       │          │
                               │  falha → agenda retry    │       │          │
                               │  esgotado → move DLQ     │       │          │
                               └──────────────────────────┘       └──────────┘
```

### Componentes principais

1. **Tabela `webhook_outbox`** no MySQL: inserida dentro da mesma transação do `OrderService.changeStatus()`. Contém o payload do evento já renderizado (snapshot), status (`PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`), `event_id` UUID e `created_at`.

2. **Worker (`src/worker.ts`)**: processo Node.js independente da API, mesmo banco, PrismaClient próprio. Polling a cada 2s, busca eventos `PENDING` em batch, assina com HMAC-SHA256, dispara HTTP POST com timeout de 10s.

3. **Política de retry**: 5 tentativas com backoff exponencial (1m → 5m → 30m → 2h → 12h). Após esgotar, evento vai para `webhook_dead_letter` (DLQ), onde fica visível para inspeção e reprocessamento manual.

4. **Segurança**: HMAC-SHA256 sobre o payload, secret única por endpoint cadastrado, rotação com grace period de 24h, TLS obrigatório (URLs http são rejeitadas no cadastro).

5. **API de configuração**: CRUD de webhooks em `src/modules/webhooks/`, seguindo os mesmos padrões dos módulos existentes (controller, service, repository, schemas Zod, error handling com AppError, logging com Pino).

### Decisões arquiteturais

As decisões abaixo estão documentadas em detalhe nos ADRs correspondentes:

| Decisão | ADR |
|---------|-----|
| Padrão Outbox no MySQL | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Retry com backoff exponencial e DLQ | [ADR-002](adrs/ADR-002-retry-backoff-dlq.md) |
| Autenticação HMAC-SHA256 com secret por endpoint | [ADR-003](adrs/ADR-003-hmac-sha256-autenticacao.md) |
| Garantia at-least-once com X-Event-Id | [ADR-004](adrs/ADR-004-at-least-once-idempotencia.md) |
| Worker em processo separado com polling | [ADR-005](adrs/ADR-005-worker-polling-separado.md) |
| Reuso dos padrões existentes do projeto | [ADR-006](adrs/ADR-006-reuso-padroes-projeto.md) |

## Alternativas Consideradas

### Alternativa 1: Chamada HTTP síncrona no `OrderService.changeStatus()`

**Descrição**: disparar a requisição HTTP para o webhook do cliente imediatamente, dentro da mesma transação (ou logo após o commit), sem outbox ou worker.

**Trade-off que motivou o descarte**: acoplamento forte entre mudança de estado do pedido e entrega de notificação. Se o cliente estiver offline, a transação de mudança de status falha — ou precisa de rollback, o que é inaceitável. Além disso, adiciona latência de rede externa (possivelmente centenas de ms ou timeout de 10s) dentro do caminho crítico da API, impactando todos os outros clientes. ([09:04] Bruno, [09:06] Diego).

### Alternativa 2: Redis Streams como fila de mensagens

**Descrição**: publicar eventos em uma stream Redis no momento da mudança de status, com um consumer group fazendo o dispatch para os endpoints dos clientes.

**Trade-off que motivou o descarte**: exigiria subir, configurar e operar um Redis Cluster (ou ao menos uma instância) adicional. Para um time de 2-3 engenheiros que já opera MySQL em produção, a sobrecarga operacional não se justifica para o volume esperado de eventos. Redis seria a escolha certa se já existisse na stack — mas introduzi-lo só para webhooks é overengineering. ([09:07] Diego e Larissa).

## Questões em Aberto

### 1. Rate limiting de envio por cliente

**Contexto**: um cliente com muitos pedidos pode receber dezenas de webhooks em segundos. Hoje não há mecanismo de rate limiting no worker — todos os eventos pendentes para um mesmo endpoint são disparados em sequência, potencialmente sobrecarregando o servidor do cliente.

**Pergunta**: devemos implementar rate limiting (ex: máximo de 10 requisições simultâneas por endpoint) já na primeira versão, ou observar o comportamento em produção primeiro?

**Encaminhamento**: observar e decidir com dados. Registrar como ponto em aberto para revisão pós-launch. ([09:38-39] Diego e Larissa).

### 2. Notificação de falha para o cliente (email de alerta)

**Contexto**: se um webhook falhar repetidamente e for para a DLQ, o cliente não fica sabendo automaticamente — ele precisa consultar proativamente o endpoint de deliveries ou o time de operação precisa detectar e agir.

**Pergunta**: devemos implementar notificação proativa (email, alerta no portal) quando um webhook atinge a DLQ?

**Encaminhamento**: fora de escopo da fase atual. Reavaliar após medição do volume real de DLQ em produção. O endpoint `GET /webhooks/:id/deliveries` permite que o cliente monitore ativamente. ([09:37] Larissa e Marcos).

## Impacto e Riscos

### Impacto na aplicação existente

| Componente | Impacto |
|-----------|---------|
| `OrderService.changeStatus()` | **Alteração pontual**: nova chamada a `publishWebhookEvent(tx, ...)` dentro da transação existente. Assinatura do método não muda. |
| `prisma/schema.prisma` | **Novas tabelas**: `webhook_config`, `webhook_outbox`, `webhook_dead_letter`. Tabelas existentes não são alteradas. |
| `src/server.ts` | **Sem alteração**: a API continua iniciando normalmente. |
| Middleware de erros | **Sem alteração**: novos erros estendem `AppError` e são tratados automaticamente. |
| Middleware de auth | **Sem alteração**: `authenticate` e `requireRole` são reutilizados. |
| Pool de conexão do Prisma | **Sem alteração**: worker usa instância própria de `PrismaClient`. |

### Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Tabela outbox crescer descontroladamente (eventos não processados ou não arquivados) | Média | Alto — degradação de performance do banco | Índices em `status` e `created_at`. Arquivamento de eventos `DELIVERED` com mais de 30 dias. Métrica de outbox size. |
| Worker cair e webhooks pararem de ser entregues | Média | Alto — clientes não recebem notificações | Health check + restart automático (Docker, systemd). Métrica de heartbeat do worker. Single worker é ponto único de falha — aceitamos na v1. |
| Secret de cliente vazar e ser usada para forjar webhooks | Baixa | Alto — cliente recebe notificações falsas | Rotação de secret com grace period. Auditoria de acessos. Secret não é re-exposta em endpoints de leitura. |
| Cliente não implementar verificação de assinatura HMAC | Alta | Médio — perda de garantia de autenticidade para aquele cliente | Documentação destacada no portal de desenvolvedor. Header `X-Signature` sempre enviado, mesmo que o cliente ignore. |

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Política de Retry com Backoff e DLQ](adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](adrs/ADR-003-hmac-sha256-autenticacao.md)
- [ADR-004: Garantia At-Least-Once com X-Event-Id](adrs/ADR-004-at-least-once-idempotencia.md)
- [ADR-005: Worker em Processo Separado com Polling](adrs/ADR-005-worker-polling-separado.md)
- [ADR-006: Reuso dos Padrões Existentes do Projeto](adrs/ADR-006-reuso-padroes-projeto.md)

---

**Status:** Aguardando revisão de Diego, Bruno, Sofia e Marcos.

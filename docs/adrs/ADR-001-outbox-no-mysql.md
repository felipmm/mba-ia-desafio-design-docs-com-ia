# ADR-001: Padrão Outbox no MySQL para Publicação de Eventos de Webhook

**Data:** 2026-07-29

**Participantes:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)

## Status

Aceito

## Contexto

O sistema precisa notificar clientes B2B externos sobre mudanças de status de pedidos via webhooks HTTP. A mudança de status ocorre no método `changeStatus` do `OrderService` (`src/modules/orders/order.service.ts`), que hoje executa dentro de uma transação Prisma: atualiza a tabela `orders`, insere em `order_status_history`, e ajusta `stock_quantity` dos produtos.

A questão central é: como garantir que o evento de notificação seja publicado de forma confiável, sem acoplar a chamada HTTP externa à transação de banco de dados da mudança de status?

Duas forças conflitantes estão em jogo:

1. **Consistência**: se a transação de mudança de status commitar, o evento de notificação precisa ser disparado. Se a transação der rollback, o evento não pode ser publicado.
2. **Resiliência**: se o cliente de webhook estiver offline ou lento, isso não pode travar a mudança de status de outros pedidos — o HTTP call não pode estar na mesma transação.

## Decisão

**Usaremos o padrão Transactional Outbox no MySQL existente.**

Quando o status do pedido muda, dentro da mesma transação SQL que atualiza `orders`, `order_status_history` e `stock_quantity`, uma nova linha é inserida em uma tabela `webhook_outbox` com o payload do evento já renderizado (snapshot). Um worker separado, executado como processo independente, fará polling dessa tabela e disparará as chamadas HTTP.

### Mecanismo

1. O `OrderService.changeStatus()` chamará uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` (Prisma TransactionClient) da transação ativa.
2. Essa função insere na tabela `webhook_outbox` dentro da mesma transação, com status `PENDING`.
3. Se a transação principal commitar, o evento fica persistido. Se der rollback, o evento desaparece junto — garantia de consistência sem transação distribuída.
4. O worker lê eventos `PENDING` em batch, dispara HTTP, e atualiza o status para `PROCESSING` → `DELIVERED` ou `FAILED`.

### Justificativa

- **MySQL não tem NOTIFY/LISTEN** nativo como o PostgreSQL ([09:09] Diego). Alternativas seriam gambiarras (trigger que escreve em arquivo ou bate em endpoint) — frágeis e fora de padrão.
- **Single-source de verdade**: a transação do banco é a fonte da verdade tanto para o estado do pedido quanto para o evento. Não há janela de inconsistência.
- **Zero infra adicional**: não requer Redis, Kafka, RabbitMQ ou qualquer middleware de mensageria.
- **Time pequeno**: subir e operar um Redis Cluster para isso seria overengineering ([09:07] Diego).

## Alternativas Consideradas

### Alternativa 1: Chamada HTTP síncrona dentro da transação do `changeStatus`

Descartada porque:
- Acrescenta latência de rede externa dentro de uma transação de banco aberta, travando outras operações ([09:04] Bruno).
- Se o cliente estiver offline, a transação falha e o status do pedido sofre rollback — inaceitável ([09:04] Bruno: "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá").

### Alternativa 2: Redis Streams como fila de mensagens

Descartada porque:
- Exigiria subir e manter infraestrutura adicional (Redis Cluster) para um time pequeno ([09:07] Diego).
- O volume de eventos esperado não justifica a complexidade operacional adicional.
- Introduz um segundo sistema que precisa estar em sync com o banco de dados.

## Consequências

### Positivas

- **Consistência forte**: commit da transação = evento persistido; rollback = evento descartado. Sem mensagens órfãs ou perdidas.
- **Desacoplamento**: a API de pedidos não sabe nada sobre HTTP, timeouts, ou retries de webhook.
- **Simplicidade operacional**: zero dependências novas de infraestrutura. Só MySQL e Node.js.
- **Auditabilidade**: todos os eventos ficam registrados em tabela, com timestamp e status, facilitando debug e reconciliação.
- **Snapshot imutável**: o payload é renderizado no momento da inserção, refletindo o estado do pedido naquele instante — mesmo que o pedido mude depois ([09:52] Larissa).

### Negativas

- **Latência adicional**: o pior caso é o intervalo de polling do worker (2 segundos). Para o requisito de "abaixo de 10 segundos" ([09:02] Marcos), é aceitável.
- **Tabela cresce**: a tabela `webhook_outbox` acumula linhas processadas. Mitigação: arquivamento de linhas `DELIVERED` após 30 dias (fora do escopo desta feature) ([09:08] Diego).
- **Polling não é reativo**: ao contrário de um message broker com push, o worker precisa ficar consultando o banco periodicamente, consumindo recursos mesmo quando não há eventos.

## Referências

- Transcrição: [09:06] Diego — introdução do padrão outbox
- Transcrição: [09:07] Larissa e Diego — descarte de Redis Streams
- Transcrição: [09:08] Diego — detalhamento de índices e arquivamento
- Transcrição: [09:52] Larissa — decisão de snapshot na inserção
- Código: `src/modules/orders/order.service.ts:126-179` — método `changeStatus` que será estendido
- Código: `prisma/schema.prisma` — schema que receberá a nova tabela `webhook_outbox`

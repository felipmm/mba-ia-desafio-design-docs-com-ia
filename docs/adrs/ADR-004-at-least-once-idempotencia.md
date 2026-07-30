# ADR-004: Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id

**Data:** 2026-07-29

**Participantes:** Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança), Marcos (Product Manager), Larissa (Tech Lead)

## Status

Aceito

## Contexto

O worker de webhooks fará chamadas HTTP para endpoints externos. Essas chamadas podem falhar por timeout de rede (o servidor do cliente processou a requisição mas a resposta nunca chegou ao worker), ou o worker pode ser reiniciado entre o envio bem-sucedido e a atualização do status no banco.

Nesses cenários, o worker reenviará o mesmo evento na tentativa seguinte. O cliente **vai** receber eventos duplicados em condições de falha de rede ou reinicialização do worker.

Precisamos decidir qual semântica de entrega garantimos e como o cliente lida com duplicatas.

## Decisão

**Garantia at-least-once. O sistema pode entregar o mesmo evento mais de uma vez. Cada evento carrega um identificador único (`X-Event-Id`) gerado na inserção da outbox, e o cliente é responsável por deduplicar usando esse identificador.**

### Detalhamento

1. **Geração do Event ID**: um UUID v4 é gerado no momento da inserção do evento na tabela `webhook_outbox`. É imutável — mesmo que o evento seja reprocessado (retry), o `event_id` permanece o mesmo.

2. **Header `X-Event-Id`**: o worker envia o UUID no header HTTP `X-Event-Id` de cada requisição. O cliente usa esse valor como chave de idempotência.

3. **Contrato com o cliente**: documentado explicitamente no portal de desenvolvedor que a semântica é at-least-once e que o cliente deve deduplicar por `X-Event-Id` ([09:26] Marcos: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes").

4. **Não garantimos ordering global**: enquanto houver um único worker, eventos do mesmo `order_id` serão processados em ordem de `created_at` da outbox. Mas isso não é uma garantia contratual — se no futuro escalarmos para múltiplos workers, a ordering por `order_id` pode ser perdida ([09:12-13] Diego).

5. **Limitação documentada**: "Não é garantia de ordering global, só por order_id e enquanto for single-worker" ([09:13] Larissa).

### Justificativa

- **Exactly-once exigiria coordenação dos dois lados** (two-phase commit ou protocolo de idempotência bidirecional), o que é significativamente mais complexo e frágil ([09:25] Diego).
- **Padrão de mercado**: Stripe, GitHub, Shopify e praticamente todos os provedores de webhook usam at-least-once com event ID para dedup ([09:25] Diego).
- **Resolve 99% dos casos**: com event ID, o cliente pode implementar dedup simples (manter um set de IDs processados com TTL razoável).

## Alternativas Consideradas

### Alternativa 1: Exactly-once com protocolo de confirmação bidirecional

Descartada porque:
- Exigiria que o cliente respondesse com um acknowledgment e que o worker aguardasse esse ack antes de marcar o evento como entregue.
- Introduz latência adicional e novo ponto de falha (e se o ack se perder?).
- Muito mais complexo de implementar e depurar para um ganho marginal — at-least-once com dedup resolve o mesmo problema na prática ([09:25] Diego).

### Alternativa 2: At-most-once (sem retry, sem garantia de entrega)

Descartada porque:
- Não atende ao requisito de negócio — os clientes B2B precisam ser notificados sobre mudanças de status de pedidos.
- Perder notificações significaria que o cliente precisaria continuar fazendo polling do `GET /orders`, exatamente o problema que a feature pretende eliminar.
- Não foi discutida seriamente na reunião — a premissa desde o início era que o sistema faria retry.

## Consequências

### Positivas

- **Simplicidade de implementação**: o worker não precisa de lógica de confirmação bidirecional, apenas enviar HTTP e verificar resposta 2xx.
- **Resiliência**: retry + event ID significa que eventos não são perdidos e duplicatas são detectáveis.
- **Padrão familiar**: qualquer desenvolvedor que já integrou com Stripe ou GitHub entende o modelo.
- **Baixo acoplamento**: a responsabilidade de dedup é do cliente, não do nosso sistema.

### Negativas

- **Carga de implementação no cliente**: o cliente precisa implementar dedup. Isso está documentado como requisito explícito de integração ([09:25] Sofia: "Isso joga responsabilidade pro cliente"). Para clientes menos maduros, pode ser uma barreira.
- **Sem ordering global**: em cenários de múltiplos workers no futuro, eventos do mesmo pedido podem chegar fora de ordem. Ex: `SHIPPED` antes de `PROCESSING`. O cliente precisa estar preparado para processar eventos fora de ordem ou usar o timestamp do evento para ordenar do lado dele.
- **Duplicatas são esperadas**: em condições normais de operação (sem falhas), duplicatas não ocorrem. Mas em cenários de borda (timeout de rede, restart do worker), o cliente receberá duplicatas e precisa tratá-las.

## Referências

- Transcrição: [09:24] Diego — proposta de at-least-once com event_id
- Transcrição: [09:25] Diego — X-Event-Id com UUID gerado na outbox
- Transcrição: [09:25] Diego — referência a Stripe e GitHub como padrão de mercado
- Transcrição: [09:25] Sofia — observação sobre responsabilidade do cliente
- Transcrição: [09:26] Marcos — compromisso de documentar no portal de desenvolvedor
- Transcrição: [09:12-13] Diego e Larissa — limitação de ordering

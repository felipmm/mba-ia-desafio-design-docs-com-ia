# ADR-002: Política de Retry com Backoff Exponencial e Dead Letter Queue

**Data:** 2026-07-29

**Participantes:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos), Marcos (Product Manager)

## Status

Aceito

## Contexto

O worker de webhooks fará chamadas HTTP para endpoints externos que podem estar indisponíveis por diversos motivos: manutenção programada do cliente, falhas de rede, picos de carga, ou erros de aplicação do lado do cliente.

Uma política de retry é necessária para maximizar a taxa de entrega sem:
- Congestionar o sistema com tentativas infinitas para endpoints que nunca responderão
- Perder eventos por desistir cedo demais de clientes em manutenção legítima
- Deixar a tabela `webhook_outbox` poluída com eventos que nunca serão entregues

Além disso, eventos que esgotarem as tentativas precisam de um destino visível e reprocessável — não podem simplesmente desaparecer.

## Decisão

**Adotaremos 5 tentativas com backoff exponencial, e eventos que esgotarem as tentativas serão movidos para uma Dead Letter Queue (DLQ) em tabela separada.**

### Política de retry

| Tentativa | Intervalo após falha | Tempo acumulado |
|-----------|---------------------|-----------------|
| 1 | — (envio inicial) | 0 |
| 2 | 1 minuto | 1 min |
| 3 | 5 minutos | 6 min |
| 4 | 30 minutos | 36 min |
| 5 | 2 horas | 2h 36min |
| 6 (última) | 12 horas | ~14h 36min |

Total: aproximadamente 15 horas entre a primeira tentativa e o esgotamento.

Após a 6ª falha (5ª retentativa), o evento é movido para a tabela `webhook_dead_letter` e removido da `webhook_outbox`.

### DLQ

- **Tabela separada** (`webhook_dead_letter`), não apenas uma flag na outbox principal ([09:18] Diego: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento").
- Colunas: `id`, `event_id`, `webhook_id`, `payload`, `last_response_status`, `last_response_body`, `failure_reason`, `attempt_count`, `created_at`.
- **Reprocessamento manual** via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role `ADMIN` ([09:18-19] Diego, [09:36] Sofia). O replay reinsere o evento na `webhook_outbox` com status `PENDING` e reseta o contador de tentativas.
- O endpoint de replay registra em log quem fez a operação (auditoria) ([09:36] Sofia).

### Justificativa do número de tentativas

- **3 tentativas foram descartadas** por serem insuficientes: cobrem janela de ~30 minutos, e clientes podem ter manutenções planejadas de 2+ horas ([09:16] Diego).
- **5 tentativas cobrem ~15 horas**, o que sobrevive a uma janela de manutenção típica e a indisponibilidades prolongadas.
- **Retry indefinido foi descartado**: eventos ficariam pendentes para sempre se o endpoint do cliente sumisse ([09:15] Diego).

## Alternativas Consideradas

### Alternativa 1: 3 tentativas com backoff mais curto

Proposta pelo Bruno ([09:16] Bruno: "3 não é melhor? Mais agressivo.").

Descartada porque:
- Cobriria apenas ~30 minutos de indisponibilidade.
- Clientes com manutenção planejada de 2+ horas perderiam notificações ([09:16] Diego).
- A economia de recursos é marginal comparada ao custo de perder eventos para clientes B2B.

### Alternativa 2: DLQ como flag na própria tabela outbox

Descartada porque:
- Poluiria a leitura da outbox principal com eventos mortos ([09:18] Diego).
- A tabela outbox é otimizada para leitura de `PENDING` por índice; misturar milhares de eventos `DEAD` degrada a performance do worker.
- Tabela separada facilita queries de auditoria, dashboards de falha e reprocessamento em lote no futuro.

## Consequências

### Positivas

- **Alta probabilidade de entrega**: 15 horas de janela cobrem a maioria dos cenários de indisponibilidade de clientes B2B.
- **DLQ visível e operável**: falhas permanentes não são perdidas; equipe de operação consegue inspecionar e reprocessar.
- **Outbox principal enxuta**: worker lê apenas eventos vivos (`PENDING`), sem scans sobre eventos mortos.
- **Auditoria**: o replay de DLQ é registrado (quem, quando, qual evento), atendendo a requisitos de segurança e compliance.

### Negativas

- **Latência máxima de ~15h**: um evento que entre na outbox segundos antes de uma indisponibilidade de 14h do cliente só será movido para DLQ ~15h depois. É inerente à política de backoff e aceitável dado que o cliente está offline.
- **Complexidade adicional**: duas tabelas (outbox + dead_letter), endpoint de replay, lógica de backoff no worker — comparado a "tentar 1x e desistir".
- **Reprocessamento manual**: não há replay automático da DLQ. Se muitos eventos caírem na DLQ (ex: bug no payload que causa erro 400), a equipe precisa intervir manualmente ou criar um script ad-hoc.

## Referências

- Transcrição: [09:15] Diego — proposta de backoff exponencial com DLQ
- Transcrição: [09:16] Bruno e Diego — debate 3 vs 5 tentativas
- Transcrição: [09:17] Diego — progressão 1m/5m/30m/2h/12h
- Transcrição: [09:18] Diego — DLQ em tabela separada
- Transcrição: [09:35-36] Larissa e Sofia — endpoint de replay restrito a ADMIN

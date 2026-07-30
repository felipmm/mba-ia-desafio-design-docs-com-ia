# ADR-005: Worker em Processo Separado com Polling de 2 Segundos

**Data:** 2026-07-29

**Participantes:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos)

## Status

Aceito

## Contexto

O padrão outbox (ADR-001) estabelece que os eventos de webhook serão persistidos em uma tabela `webhook_outbox` e consumidos por um worker. Agora precisamos decidir:

1. **Onde o worker executa**: como parte do processo da API ou como processo independente?
2. **Como o worker descobre novos eventos**: push (notificação reativa) ou pull (polling)?
3. **Com que frequência o worker consulta a tabela**: qual o intervalo de polling?

## Decisão

**O worker executará como processo Node.js separado da API (`src/worker.ts`), fará polling da tabela `webhook_outbox` a cada 2 segundos, e usará a mesma stack (Prisma, mesmo banco, Pino) mas com sua própria instância de `PrismaClient`.**

### Detalhamento

1. **Processo separado**: o worker será uma entry-point independente (`src/worker.ts`), iniciada com um script npm separado (`npm run worker`). Isso garante que:
   - Reinicializações da API (deploy, crash, restart) não matem o worker.
   - O worker possa ser escalado, monitorado e gerenciado independentemente ([09:11] Diego: "o worker tem que rodar como processo separado, não dentro da mesma instância da API").

2. **PrismaClient separado**: como é outro processo Node, o worker instancia seu próprio `PrismaClient` (mesmo `DATABASE_URL`, mas instância nova). Isso é importante porque `PrismaClient` não é seguro para compartilhamento entre processos ([09:30] Bruno: "Separado. PrismaClient é por processo").

3. **Polling a cada 2 segundos**: o worker consulta eventos com `status = PENDING`, ordenados por `created_at ASC`, em batch pequeno. Intervalo de 2 segundos entre polls.

4. **Latência**: pior caso de latência = 2 segundos (evento inserido logo após o último poll). Isso atende ao requisito de "abaixo de 10 segundos" estabelecido pelo PM ([09:10] Marcos: "2 segundos serve, perfeito").

5. **Single worker por enquanto**: um único worker garante ordering implícita por `order_id` (eventos do mesmo pedido são processados em ordem de inserção). Se no futuro escalarmos para múltiplos workers, precisaremos de particionamento ou lock pessimista ([09:12-13] Diego).

6. **Mesma stack**: o worker reutiliza Pino (logger), Prisma (ORM), e os tipos compartilhados do projeto. Não introduz novas dependências.

### Justificativa do Polling

- **MySQL não tem mecanismo nativo de notificação** como o PostgreSQL (`NOTIFY/LISTEN`) ([09:09] Diego).
- **Triggers do MySQL** executam SQL, não notificam processos externos. Improvisar uma notificação (escrever em arquivo, bater em endpoint HTTP) seria frágil e fora de padrão ([09:09] Diego).
- **Polling é simples e confiável**: o worker consulta o banco, processa o que encontrar, dorme 2s, repete. Sem moving parts além do MySQL e do Node.js.

## Alternativas Consideradas

### Alternativa 1: Worker embutido no processo da API (ex: `setInterval` no `server.ts`)

Descartada porque:
- Se a API reiniciar (deploy, crash), o worker morre junto e eventos ficam parados até o próximo restart ([09:11] Diego).
- O loop de polling competiria por CPU e event loop com as requisições HTTP da API.
- Dificulta monitoramento independente (métricas de worker misturadas com métricas de API).

### Alternativa 2: Trigger MySQL + notificação externa

Descartada porque:
- MySQL não tem `NOTIFY/LISTEN`. Trigger pode executar SQL, mas não notificar um processo externo diretamente ([09:09] Diego).
- Alternativas como trigger que escreve em arquivo ou chama UDF para bater em endpoint são frágeis, não portáteis e difíceis de manter.

### Alternativa 3: Polling com intervalo menor (ex: 500ms)

Descartada porque:
- Reduziria a latência do pior caso de 2s para 0.5s, mas sem ganho perceptível de negócio (o requisito é <10s).
- Aumentaria a carga no banco (3x mais queries).
- O ganho marginal não justifica o custo operacional.

## Consequências

### Positivas

- **Isolamento de falha**: crash do worker não afeta a API; deploy da API não interrompe entregas de webhook.
- **Escalabilidade**: worker pode ser escalado horizontalmente no futuro (múltiplas instâncias, com lock ou particionamento).
- **Monitoramento independente**: métricas, logs e health checks do worker são separados da API.
- **Simplicidade**: polling é um padrão trivial de implementar, depurar e testar.
- **Latência dentro do SLA**: 2 segundos está bem abaixo do requisito de 10 segundos.

### Negativas

- **Latência mínima de 2 segundos**: eventos não são entregues instantaneamente. Para o caso de uso (notificação de mudança de status de pedido), é aceitável.
- **Recursos ociosos**: mesmo sem eventos, o worker fica acordando a cada 2s para consultar o banco. Mitigação: em fase futura, pode-se implementar backoff adaptativo (se não há eventos por N ciclos, aumentar intervalo).
- **Ponto único de falha**: com single worker, se ele cair, nenhum webhook é entregue até que seja reiniciado. Mitigação: health check + restart automático (Docker, systemd, PM2).
- **Ordenação frágil**: a garantia de ordenação por `order_id` depende de single worker. Se escalar, precisa de mecanismo adicional (lock, particionamento).

## Referências

- Transcrição: [09:09] Diego — polling como mecanismo de leitura
- Transcrição: [09:09] Diego — MySQL não tem NOTIFY/LISTEN
- Transcrição: [09:10] Larissa — decisão de intervalo de 2s
- Transcrição: [09:11] Diego — worker como processo separado
- Transcrição: [09:11] Larissa — entry-point `src/worker.ts` e script `npm run worker`
- Transcrição: [09:30] Bruno — PrismaClient separado por processo
- Transcrição: [09:12-13] Diego — limitação de ordering com múltiplos workers
- Código: `src/server.ts` — padrão de bootstrap que será replicado em `src/worker.ts`
- Código: `src/config/database.ts` — factory do PrismaClient reutilizada pelo worker

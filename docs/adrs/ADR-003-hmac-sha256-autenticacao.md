# ADR-003: Autenticação de Webhooks com HMAC-SHA256 e Secret por Endpoint

**Data:** 2026-07-29

**Participantes:** Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Larissa (Tech Lead)

## Status

Aceito

## Contexto

O sistema enviará requisições HTTP POST para endpoints externos controlados pelos clientes B2B, contendo dados de pedidos (status, valores, customer_id). Esses endpoints estão fora da infraestrutura da empresa — qualquer intermediário de rede entre a origem e o destino pode, em tese, interceptar ou modificar o payload.

O cliente precisa ter certeza de que:
1. A requisição realmente se originou do nosso sistema (autenticidade).
2. O payload não foi alterado em trânsito (integridade).

Ao mesmo tempo, diferentes clientes não podem compartilhar o mesmo segredo — se um cliente vazar sua secret, isso não pode comprometer as notificações dos outros clientes ([09:21] Sofia).

## Decisão

**Usaremos HMAC-SHA256 para assinar o payload de cada requisição de webhook, com uma secret criptográfica única por endpoint cadastrado, e suporte a rotação de secret com grace period de 24 horas.**

### Detalhamento

1. **Algoritmo**: HMAC-SHA256. Padrão de mercado (Stripe, GitHub, Shopify) com suporte nativo em todas as linguagens e bibliotecas relevantes ([09:20] Sofia).

2. **Secret por endpoint**: cada registro na tabela `webhook_config` armazena sua própria `secret`. Não existe secret global da plataforma. Se um cliente vazar a secret dele, os demais não são afetados ([09:21] Sofia).

3. **Header de assinatura**: a requisição HTTP inclui o header `X-Signature` com o valor do HMAC calculado sobre o corpo do request (raw JSON body). O formato é `sha256=<hex-encoded-hmac>`.

4. **Rotação de secret**: o cliente pode solicitar nova secret via API. A secret antiga permanece válida por 24 horas (grace period) em paralelo com a nova, para que o cliente tenha tempo de atualizar seus sistemas. Após 24h, a antiga é invalidada ([09:21-22] Sofia).

5. **Geração de secret**: a secret é gerada pelo servidor no momento da criação do webhook, usando gerador criptograficamente seguro (`crypto.randomBytes(32).toString('hex')`). O valor é retornado ao cliente uma única vez na resposta da criação — não é armazenado em logs nem re-exposto em endpoints de leitura.

6. **Validação de TLS**: URLs de webhook são validadas no cadastro — precisam usar `https://`. URLs `http://` são rejeitadas com erro de validação no schema Zod ([09:23] Sofia).

### Headers da requisição de webhook

```
X-Event-Id: <uuid>
X-Signature: sha256=<hex-encoded-hmac>
X-Timestamp: <ISO-8601>
X-Webhook-Id: <webhook_config_id>
Content-Type: application/json
```

## Alternativas Consideradas

### Alternativa 1: Secret global única para toda a plataforma

Descartada porque:
- Se um cliente vazar a secret (ex: erro de configuração, log de aplicação), todos os clientes são comprometidos ([09:21] Sofia: "Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo").
- Impossibilita rotação independente por cliente.
- Referência: incidente anterior com cliente que vazou secret em log de aplicação ([09:22] Diego).

### Alternativa 2: mTLS (autenticação mútua por certificado)

Descartada porque:
- Exigiria que cada cliente B2B mantivesse e renovasse certificados TLS do lado deles.
- Aumenta significativamente a barreira de adoção para clientes menos maduros tecnicamente.
- HMAC-SHA256 resolve os mesmos problemas (autenticidade + integridade) com complexidade operacional muito menor para ambos os lados.
- Não foi discutida na reunião, mas é a alternativa padrão de mercado que foi implicitamente preterida em favor de HMAC.

## Consequências

### Positivas

- **Isolamento de segurança**: vazamento de uma secret não compromete outros clientes.
- **Padrão de mercado**: qualquer time de integração do cliente sabe implementar verificação de HMAC-SHA256.
- **Rotação operacional**: cliente consegue trocar secret sem janela de indisponibilidade, graças ao grace period de 24h.
- **TLS obrigatório**: validação no cadastro garante que nenhum webhook trafegue em texto plano, independentemente de configuração do cliente.
- **Secret não trafega**: a assinatura é enviada, não a secret. Um interceptador não consegue derivar a secret a partir do header `X-Signature`.

### Negativas

- **Responsabilidade do cliente**: o cliente precisa implementar a verificação de assinatura do lado dele. Se ele não verificar, a segurança é unilateral (só garante que o payload saiu íntegro da nossa origem, mas se o destino aceitar qualquer coisa, não adianta).
- **Grace period é estado adicional**: a tabela de secrets precisa suportar "secret atual + secret anterior dentro do prazo", adicionando complexidade ao modelo de dados.
- **Sem revogação imediata**: se uma secret for comprovadamente comprometida, o grace period de 24h significa que o atacante ainda pode forjar assinaturas nesse intervalo. Mitigação: endpoint adicional de revogação imediata pode ser adicionado em fase futura.

## Referências

- Transcrição: [09:20] Sofia — proposta de HMAC-SHA256
- Transcrição: [09:21] Sofia — secret única por endpoint
- Transcrição: [09:21-22] Sofia — rotação com grace period de 24h
- Transcrição: [09:23] Sofia — TLS obrigatório com validação no schema
- Transcrição: [09:22] Diego — incidente anterior de vazamento de secret
- Transcrição: [09:44-45] Diego e Sofia — headers X-Signature, X-Webhook-Id, X-Timestamp

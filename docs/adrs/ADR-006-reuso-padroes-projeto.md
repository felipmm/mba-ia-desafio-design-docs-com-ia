# ADR-006: Reuso dos Padrões Existentes do Projeto no Módulo de Webhooks

**Data:** 2026-07-29

**Participantes:** Bruno (Engenheiro Pleno, Pedidos), Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança)

## Status

Aceito

## Contexto

A aplicação existente (Order Management System) estabeleceu padrões consistentes ao longo dos módulos `auth`, `users`, `customers`, `products` e `orders`. Esses padrões cobrem estrutura de módulos, tratamento de erros, logging, validação, autenticação e autorização.

Ao construir o novo módulo de webhooks, temos duas opções: seguir os mesmos padrões ou introduzir novos. A segunda opção criaria inconsistência na codebase, aumentaria a carga cognitiva para o time e potencialmente introduziria bugs por divergência de convenções.

A decisão aqui não é sobre *o que* usar (as ferramentas já estão no projeto), mas sobre o compromisso explícito de *seguir* os padrões existentes em vez de introduzir variações.

## Decisão

**O módulo de webhooks seguirá estritamente os padrões já estabelecidos no projeto: estrutura de módulos, hierarquia de erros (AppError), logging (Pino), validação (Zod), autenticação/autorização (JWT + requireRole), e nomenclatura de códigos de erro com prefixo `WEBHOOK_`.**

### Padrões reutilizados

#### 1. Estrutura de módulos

Cada domínio do projeto segue o padrão `src/modules/<domain>/` com: `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`. O módulo `webhooks` seguirá a mesma estrutura ([09:27] Bruno):

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.routes.ts
├── webhook.schemas.ts
├── webhook.processor.ts    (lógica de envio HTTP, retry e DLQ)
└── webhook.errors.ts       (erros específicos com prefixo WEBHOOK_)
```

#### 2. Hierarquia de erros

O projeto define `AppError` como classe base (`src/shared/errors/app-error.ts`), com subclasses semânticas: `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `ValidationError`, etc. (`src/shared/errors/http-errors.ts`).

Os erros do módulo de webhooks estenderão essas mesmas classes base, com códigos prefixados `WEBHOOK_` ([09:28-29] Bruno e Larissa):

- `WebhookNotFoundError` extends `NotFoundError` — código `WEBHOOK_NOT_FOUND`
- `WebhookInvalidUrlError` extends `ValidationError` — código `WEBHOOK_INVALID_URL`
- `WebhookSecretRequiredError` extends `ValidationError` — código `WEBHOOK_SECRET_REQUIRED`
- `WebhookPayloadTooLargeError` extends `UnprocessableEntityError` — código `WEBHOOK_PAYLOAD_TOO_LARGE`
- `WebhookDeliveryError` extends `AppError` — código `WEBHOOK_DELIVERY_FAILED`

#### 3. Error middleware

O middleware centralizado (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros do Prisma. Como nossos novos erros estendem `AppError`, eles são automaticamente serializados no formato padrão `{ error: { code, message, details? } }` — sem nenhuma alteração no middleware ([09:29] Bruno).

#### 4. Logging com Pino

O projeto usa Pino (`src/shared/logger/index.ts`) com structured logging, correlation IDs e redaction de campos sensíveis. O worker e o módulo de webhooks usarão o mesmo logger, sem introduzir nova biblioteca de logging ([09:29] Bruno: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo").

#### 5. Validação com Zod

Todos os módulos usam schemas Zod para validação de input, integrados via `validate` middleware (`src/middlewares/validate.middleware.ts`). Os endpoints de webhook seguirão o mesmo padrão: schemas definidos em `webhook.schemas.ts`, aplicados via `validate({ body: ..., query: ..., params: ... })`.

#### 6. Autenticação e autorização

Os middlewares `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`) serão reutilizados sem alterações:
- CRUD de configuração de webhooks: `authenticate` obrigatório, qualquer role autenticada ([09:37] Sofia: "Por enquanto sim. Mais pra frente a gente pode endurecer").
- Endpoint de replay de DLQ: `authenticate` + `requireRole(['ADMIN'])` ([09:36] Sofia: "Tem que ser ADMIN sim. Mexer em fila de entrega de notificação não é coisa de operador").

#### 7. IDs UUID

O projeto usa UUIDs (string, `@db.Char(36)`) para todas as entidades. As novas tabelas (`webhook_config`, `webhook_outbox`, `webhook_dead_letter`) seguirão o mesmo padrão ([09:51] Larissa: "UUID, segue o padrão do resto do projeto").

## Alternativas Consideradas

### Alternativa 1: Estrutura de erros própria para o módulo (sem AppError)

Descartada porque:
- Criaria dois estilos de erro na mesma aplicação, confundindo o time e dificultando a manutenção.
- O error middleware não serializaria esses erros corretamente sem alterações adicionais.
- Perderia o benefício de ter todas as respostas de erro seguindo o mesmo formato JSON.

### Alternativa 2: Framework de logging diferente para o worker

Descartada porque:
- Dificultaria a correlação de logs entre API e worker (IDs de correlação diferentes, formatos diferentes).
- Aumentaria a superfície de dependências sem ganho funcional.
- Pino já atende todos os requisitos (structured logging, redaction, níveis, pretty-print em dev).

## Consequências

### Positivas

- **Consistência**: qualquer desenvolvedor do time que conhece o módulo de `orders` consegue navegar e contribuir no módulo de `webhooks` sem surpresas.
- **Reuso real**: o error middleware, o validate middleware, o auth middleware e o logger funcionam para o novo módulo sem alterações.
- **Onboarding rápido**: novos desenvolvedores aprendem um padrão e aplicam em todos os módulos.
- **Menor superfície de bugs**: padrões já testados em produção (orders, products, customers) são reaplicados em vez de reinventados.
- **Segurança consistente**: autenticação e autorização seguem exatamente o mesmo modelo, com `requireRole` reutilizado para o endpoint admin.

### Negativas

- **Menor flexibilidade**: se um padrão existente não for ideal para webhooks (ex: paginação no repository, que webhooks podem não precisar), ainda assim seguimos o padrão para manter consistência. Ajustes pontuais são possíveis, mas o desvio precisa ser justificado.
- **Prefixo WEBHOOK_ é convenção local**: não há mecanismo que force o prefixo — depende de disciplina do time e revisão de código.

## Referências

- Transcrição: [09:27] Bruno — proposta de estrutura `src/modules/webhooks`
- Transcrição: [09:28] Bruno — padrão de erros com prefixo `WEBHOOK_`
- Transcrição: [09:29] Bruno — reuso de Pino, error middleware e padrões existentes
- Transcrição: [09:30] Larissa — decisão formal de reuso máximo
- Transcrição: [09:29] Bruno — "middleware de erro centralizado já trata AppError, Zod e Prisma"
- Transcrição: [09:36] Sofia — `requireRole` para endpoint admin
- Transcrição: [09:37] Sofia — CRUD autenticado normal
- Transcrição: [09:51] Larissa — UUID como padrão
- Código: `src/shared/errors/app-error.ts` — classe base de erros
- Código: `src/shared/errors/http-errors.ts` — errors específicos (NotFoundError, ConflictError, etc.)
- Código: `src/middlewares/error.middleware.ts` — middleware centralizado de erros
- Código: `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`
- Código: `src/middlewares/validate.middleware.ts` — middleware de validação Zod
- Código: `src/shared/logger/index.ts` — factory do Pino
- Código: `src/modules/orders/` — padrão de estrutura de módulo replicado

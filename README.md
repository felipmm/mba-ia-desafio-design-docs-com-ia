# Design Docs com IA: Da Reunião ao Documento

## Sobre o Desafio

Este repositório contém o pacote de design docs produzido para o desafio "Da Reunião ao Documento: Design Docs Gerados por IA" do MBA em IA. A tarefa consistiu em transformar a transcrição de uma reunião técnica de ~55 minutos — onde cinco profissionais discutiram a arquitetura de um Sistema de Webhooks de Notificação de Pedidos — em um conjunto completo de documentos técnicos acionáveis: PRD, RFC, FDD, ADRs e uma matriz de rastreabilidade.

O cenário: uma empresa opera um Order Management System (OMS) em Node.js + TypeScript com MySQL e precisa adicionar notificações de webhook para três clientes B2B estratégicos. A transcrição captura todas as decisões técnicas, requisitos, restrições, itens descartados e questões em aberto. O desafio foi destilar esse material bruto em documentação estruturada, rastreável e livre de alucinações — usando IA como ferramenta principal de produção, comigo no papel de maestro: formulando prompts, revisando criticamente as saídas, corrigindo inconsistências e garantindo aderência estrita ao que foi discutido.

O enunciado original do desafio está disponível no repositório base: [github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Ferramentas de IA Utilizadas

| Ferramenta | Papel |
|-----------|-------|
| **Claude Code (Claude via CLI)** | Ferramenta principal. Usei o Claude Code como agente integrado ao repositório, com acesso completo ao código fonte e à transcrição. Ele leu o código (`src/`, `prisma/`), entendeu os padrões do projeto, extraiu decisões da transcrição e gerou os documentos. A capacidade de ler arquivos diretamente e cruzar referências entre código e transcrição foi essencial. |
| **Claude (Explore agents)** | Usei sub-agentes de exploração para varrer a codebase em paralelo — um leu a transcrição completa, outro mapeou todos os módulos e padrões do código, e um terceiro verificou a estrutura existente de `docs/`. Isso me deu uma visão completa do contexto em uma única iteração. |
| **Claude (Plan mode)** | Usei o modo de planejamento para estruturar a abordagem antes de escrever qualquer documento: defini quais ADRs produzir, a ordem de execução, e os requisitos de cada artefato. O plano foi revisado e aprovado antes da execução. |

## Workflow Adotado

A ordem de produção seguiu a sugestão do desafio: decisões primeiro, implementação depois, produto e rastreabilidade por último.

### 1. Contextualização (~15 min)

Antes de gerar qualquer conteúdo, três agentes de exploração varreram o repositório em paralelo:
- Um leu a transcrição completa e extraiu todas as decisões, requisitos, restrições e timestamps
- Outro mapeou a estrutura de módulos, padrões de erro, middlewares, schema Prisma e serviços
- Um terceiro verificou a estrutura de `docs/` e o README original

Isso me deu um mapa completo do que existia e do que precisava ser produzido.

### 2. ADRs (~40 min)

Produzi os 6 ADRs primeiro, um por um, seguindo o formato MADR. Cada ADR foi escrito com:
- Contexto extraído da transcrição (com timestamps)
- Decisão formalizada com justificativa
- Alternativas consideradas (extraídas da discussão da reunião)
- Consequências positivas e negativas com trade-offs explícitos
- Referências à transcrição e, quando aplicável, ao código

O ADR-006 (Reuso dos Padrões do Projeto) foi o que mais exigiu referências ao código — cada padrão mencionado precisava apontar para um arquivo real.

### 3. RFC (~30 min)

Com as decisões já formalizadas nos ADRs, o RFC foi uma consolidação natural. O foco foi manter a concisão (2-4 páginas) e não duplicar o nível de detalhe do FDD. As duas alternativas descartadas (HTTP síncrono e Redis Streams) e as duas questões em aberto (rate limiting e email de falha) vieram diretamente da transcrição.

### 4. FDD (~50 min)

O documento mais denso. A seção de contratos exigiu criar payloads de exemplo realistas para 7 endpoints. A seção de integração com o sistema existente foi a mais trabalhosa — precisei verificar cada arquivo referenciado no código real para garantir que existia e que a descrição de como integrar fazia sentido. Referenciei 8 arquivos: `order.service.ts`, `schema.prisma`, `http-errors.ts`, `auth.middleware.ts`, `error.middleware.ts`, `validate.middleware.ts`, `logger/index.ts` e `server.ts`.

### 5. PRD (~30 min)

Com RFC, FDD e ADRs prontos, o PRD foi essencialmente uma consolidação em linguagem de produto. Os 12 requisitos funcionais e 10 não funcionais foram extraídos diretamente da transcrição. A seção de "fora de escopo" foi particularmente importante — itens como email de notificação e dashboard visual foram explicitamente discutidos e adiados na reunião.

### 6. TRACKER (~35 min)

A matriz de rastreabilidade foi construída varrendo todos os documentos já prontos e mapeando cada item à sua origem. Foi a etapa mais meticulosa — cada linha precisava de um timestamp válido da transcrição ou um caminho de arquivo real. O resultado: 115 linhas, 82.6% com fonte na transcrição e 17.4% com fonte no código.

### 7. README (~20 min)

Este documento, escrito por último para capturar o processo completo com clareza.

## Prompts Customizados

### Prompt 1: Extração de decisões da transcrição

Usei este prompt como base para identificar as decisões que virariam ADRs:

```
Analise a transcrição abaixo e identifique todas as decisões técnicas
explicitas mencionadas. Para cada decisão, extraia:
1. A decisão em si (o que foi decidido)
2. O timestamp e nome do falante que propôs ou formalizou
3. As alternativas mencionadas e quem as descartou (com o motivo)
4. O contexto: qual problema a decisão resolve
5. Se a decisão tem relação com código existente — em caso positivo,
   qual módulo ou arquivo seria afetado

Regras:
- Só inclua decisões que foram explicitamente fechadas na reunião
- Se algo foi mencionado como "a decidir depois" ou "observar", classifique como "questão em aberto", não como decisão
- Se algo foi descartado, classifique como "alternativa descartada"
- Para cada decisão, indique se ela é arquitetural (estrutura do sistema)
  ou de implementação (detalhe de como construir)

Transcrição:
[conteúdo de TRANSCRICAO.md]
```

### Prompt 2: Verificação de consistência contra o código

Usei este prompt para validar que as referências a arquivos no FDD estavam corretas:

```
Verifique se todas as referências a arquivos e padrões de código no
documento FDD abaixo são precisas. Para cada menção a um arquivo ou
classe:

1. O arquivo existe no repositório?
2. A classe/método/função mencionada existe nesse arquivo?
3. O comportamento descrito corresponde ao que o código realmente faz?
4. A assinatura sugerida (parâmetros, tipos) é compatível com o que o
   código existente espera?

Se encontrar inconsistências, liste cada uma com:
- O que o documento diz
- O que o código realmente tem
- Sugestão de correção

Documento FDD:
[conteúdo de docs/FDD.md]
```

### Prompt 3: Geração de ADR individual

Para cada ADR, usei uma variação deste prompt:

```
Produza um Architecture Decision Record (ADR) no formato MADR para a
seguinte decisão:

Decisão: [título da decisão]
Contexto (da transcrição): [trechos relevantes com timestamps]
Alternativas discutidas: [alternativas e motivos de descarte]
Consequências: [positivas e negativas discutidas]

O ADR deve conter as seções: Status, Contexto, Decisão, Alternativas
Consideradas, Consequências.

Regras importantes:
- Status deve ser "Aceito"
- Cada alternativa considerada deve incluir o motivo real do descarte
  (não invente motivos que não estão na transcrição)
- Consequências devem listar explicitamente trade-offs (toda decisão tem
  lado positivo e negativo)
- Se a decisão tem relação com código existente, referencie arquivos
  reais do projeto (ex: src/modules/orders/order.service.ts)
- Inclua timestamps e nomes dos falantes como referências
```

## Iterações e Ajustes

### Iteração 1: ADRs genéricos demais

**Problema**: na primeira tentativa de gerar os ADRs, as seções de "Alternativas Consideradas" estavam muito genéricas — mencionavam alternativas teóricas (tipo "usar Kafka") que nunca foram discutidas na reunião.

**Correção**: refinei o prompt para exigir que cada alternativa fosse extraída exclusivamente da transcrição, com timestamp e nome do falante. Isso eliminou alternativas inventadas e forçou o uso do que foi realmente discutido. Refiz os 6 ADRs com essa restrição. A alternativa "mTLS" no ADR-003 foi mantida porque, embora não explicitamente nomeada na reunião, foi a alternativa implícita preterida em favor de HMAC — e está sinalizada como tal.

### Iteração 2: FDD duplicando conteúdo do RFC

**Problema**: a primeira versão do FDD incluía uma longa seção de "Visão Geral da Arquitetura" que essencialmente repetia o conteúdo do RFC (diagrama de componentes, justificativas das decisões). Isso violava a diretriz do desafio de que "conteúdo duplicado entre documentos é sinal de que algo está no lugar errado".

**Correção**: removi a seção de visão geral do FDD e a substituí por links para o RFC e ADRs. O FDD agora foca exclusivamente em "como construir" — fluxos detalhados, contratos, erros, integração. A visão arquitetural fica no RFC.

### Iteração 3: Seção de integração com arquivos inexistentes

**Problema**: na primeira versão do FDD, a seção "Integração com o sistema existente" mencionava um arquivo `src/shared/middlewares/error.handler.ts` que não existe — o arquivo real é `src/middlewares/error.middleware.ts`. A IA tinha alucinado o caminho baseada em convenções de outros projetos.

**Correção**: fiz uma verificação sistemática de cada caminho de arquivo mencionado em todos os documentos contra a listagem real do repositório. Corrigi 3 paths incorretos e adicionei uma etapa de validação no prompt 2 para evitar que isso se repetisse. O TRACKER também ajudou — como cada linha com `Fonte=CODIGO` exige um caminho real, inconsistências saltam aos olhos.

### Iteração 4: Tracker com timestamps genéricos

**Problema**: a primeira versão do tracker usava timestamps aproximados (ex: `[09:00-10:00]`) em vez de timestamps precisos com nome do falante.

**Correção**: refiz o tracker inteiro, voltando à transcrição para cada linha e extraindo o timestamp exato e o nome do falante. Isso aumentou a precisão de ~60% para ~83% de timestamps válidos. O formato `[hh:mm] Nome` foi aplicado consistentemente em todas as 95 linhas com fonte TRANSCRICAO.

## Como Navegar a Entrega

### Estrutura de Arquivos

```
.
├── README.md                          ← Este documento (processo de produção)
├── TRANSCRICAO.md                     ← Transcrição original da reunião (não alterada)
├── docs/
│   ├── PRD.md                         ← Product Requirements Document
│   ├── RFC.md                         ← Request for Comments (proposta técnica)
│   ├── FDD.md                         ← Feature Design Document (implementação)
│   ├── TRACKER.md                     ← Matriz de rastreabilidade
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-retry-backoff-dlq.md
│       ├── ADR-003-hmac-sha256-autenticacao.md
│       ├── ADR-004-at-least-once-idempotencia.md
│       ├── ADR-005-worker-polling-separado.md
│       └── ADR-006-reuso-padroes-projeto.md
├── src/                               ← Código fonte (não alterado)
├── prisma/                            ← Schema e migrations (não alterados)
└── tests/                             ← Testes (não alterados)
```

### Ordem Sugerida de Leitura

1. **`TRANSCRICAO.md`** — comece pela fonte. Entenda o que foi discutido na reunião (~55 min de leitura).

2. **`docs/PRD.md`** — entenda o produto: problema, público, escopo, métricas e requisitos funcionais/não funcionais. É o documento mais acessível para quem não é técnico.

3. **`docs/RFC.md`** — entre na proposta técnica em nível de arquitetura. Entenda a abordagem escolhida, o que foi descartado e o que ficou em aberto. É conciso (2-4 páginas).

4. **`docs/adrs/`** — aprofunde nas decisões arquiteturais. Cada ADR isola uma decisão com contexto, alternativas e consequências. Leia na ordem numérica (001 a 006).

5. **`docs/FDD.md`** — vá para o detalhe de implementação. Fluxos, contratos HTTP, matriz de erros, estratégias de resiliência, observabilidade e a seção de integração com o código existente. É o documento que um desenvolvedor usa para começar a codar.

6. **`docs/TRACKER.md`** — consulte para verificar a origem de qualquer item nos documentos. Se algo pareceu inventado, o tracker mostra (ou revela a ausência de) sua origem na transcrição ou no código.

7. **`README.md`** (este arquivo) — entenda como os documentos foram produzidos, quais ferramentas e prompts foram usados, e quais iterações foram necessárias.

---

*Documentação produzida como parte do desafio "Da Reunião ao Documento: Design Docs Gerados por IA" do MBA em IA.*

# Mapeamento da base de código — Fase 1

## Visão geral e limites

O projeto é uma API REST de gestão de pedidos, organizada por domínios em um único processo Node.js. O código existente cobre autenticação, usuários, clientes, produtos e pedidos; compartilha um banco MySQL e uma composição central de dependências. A organização em módulos é lógica, não uma divisão em serviços implantáveis independentes. A análise usa o diretório do projeto, sem diretório de contexto adicional.

Fontes principais: [`src/app.ts`](../../src/app.ts), [`src/routes/index.ts`](../../src/routes/index.ts), [`prisma/schema.prisma`](../../prisma/schema.prisma), [`package.json`](../../package.json) e [`README.md`](../../README.md). Os documentos [`docs/PRD.md`](../PRD.md), [`docs/RFC.md`](../RFC.md), [`docs/FDD.md`](../FDD.md) e [`docs/TRACKER.md`](../TRACKER.md) são marcadores ainda não preenchidos; [`docs/adrs/README.md`](README.md) apresenta o diretório de ADRs.

### Stack observada

| Área | Implementação e evidência |
|---|---|
| Runtime e linguagem | Node.js ≥20, TypeScript 5.6 e módulos ES; compilação por `tsc` em `dist` — [`package.json`](../../package.json), [`tsconfig.json`](../../tsconfig.json), [`tsconfig.build.json`](../../tsconfig.build.json). |
| HTTP e validação | Express 4, rotas sob `/api/v1`, JSON e Zod 3; `/health` fora do prefixo — [`src/app.ts`](../../src/app.ts), [`src/routes/index.ts`](../../src/routes/index.ts), [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts). |
| Dados | MySQL, Prisma 5, schema, migração inicial e seed — [`prisma/schema.prisma`](../../prisma/schema.prisma), [`prisma/migrations/20260519182739_init/migration.sql`](../../prisma/migrations/20260519182739_init/migration.sql), [`prisma/seed.ts`](../../prisma/seed.ts). |
| Identidade e operação | JWT, bcrypt, perfis `ADMIN` e `OPERATOR`; Pino e ID por requisição — [`src/modules/auth/auth.service.ts`](../../src/modules/auth/auth.service.ts), [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts), [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts). |
| Desenvolvimento e testes | Vitest e Supertest, ESLint, Prettier, tsx; Compose fornece MySQL 8 — [`package.json`](../../package.json), [`vitest.config.ts`](../../vitest.config.ts), [`docker-compose.yml`](../../docker-compose.yml). |

## Módulos para análise

Os IDs são unidades de investigação da Fase 2, não serviços isolados. As estimativas abaixo são tamanho aproximado de leitura do código principal, em linhas físicas, sem contar arquivos compartilhados, testes ou schema; não são estimativas de esforço. Módulos transversais podem compartilhar evidências com os domínios.

| ID | Módulo e escopo estimado | Responsabilidade e pontos de entrada | Dependências principais |
|---|---|---|---|
| **AUTH** | Autenticação; 4 arquivos, ~121 linhas em [`src/modules/auth/`](../../src/modules/auth/) | Registro, login e perfil autenticado; JWT no serviço e validação nas rotas. | **USERS**, **DATA**, middleware de autenticação e configuração de segredo. |
| **USERS** | Usuários; 5 arquivos, ~119 linhas em [`src/modules/users/`](../../src/modules/users/) | Criação com hash bcrypt e consulta por ID; rota de consulta exige `ADMIN`. `UserService` também fornece dados públicos para AUTH. | **DATA**, autenticação/autorização e erros compartilhados. |
| **CUSTOMERS** | Clientes; 5 arquivos, ~234 linhas em [`src/modules/customers/`](../../src/modules/customers/) | CRUD autenticado, busca e paginação; dados de contato e endereço. | **DATA**, validação, autenticação e respostas paginadas; pedidos referenciam clientes. |
| **PRODUCTS** | Produtos; 5 arquivos, ~237 linhas em [`src/modules/products/`](../../src/modules/products/) | CRUD autenticado, busca por nome/SKU, filtro de atividade, preço em centavos e quantidade de estoque. | **DATA**, validação, autenticação e paginação; estoque também é manipulado por **ORDERS**. |
| **ORDERS** | Pedidos; 6 arquivos, ~478 linhas em [`src/modules/orders/`](../../src/modules/orders/) | Criação, consulta, listagem, mudança de status e exclusão condicionada. [`order.status.ts`](../../src/modules/orders/order.status.ts) define transições; [`order.service.ts`](../../src/modules/orders/order.service.ts) concentra transações, histórico, sequência, cálculo de valores e movimentação de estoque. | **DATA**, entidades de **CUSTOMERS**, **PRODUCTS** e **USERS**, autenticação, erros e paginação. O serviço acessa Prisma diretamente em suas transações, além de usar seu repositório. |
| **API** | Composição e infraestrutura HTTP; 3 arquivos, ~134 linhas: [`src/app.ts`](../../src/app.ts), [`src/server.ts`](../../src/server.ts), [`src/routes/index.ts`](../../src/routes/index.ts) | Inicialização, construção manual de repositórios/serviços/controladores, montagem das rotas, health check e desligamento. | Todos os módulos de domínio, **DATA**, configuração, logging e middlewares. |
| **DATA** | Persistência e configuração; schema de 138 linhas, uma migração, seed e 2 arquivos em [`src/config/`](../../src/config/) | Cliente Prisma, validação de ambiente, modelos, relacionamentos, índices e preparação de dados. Inclui [`prisma/schema.prisma`](../../prisma/schema.prisma), [`prisma/migrations/`](../../prisma/migrations/) e [`prisma/seed.ts`](../../prisma/seed.ts). | MySQL e variáveis de ambiente; é infraestrutura compartilhada de todos os domínios. |
| **CROSS** | Políticas compartilhadas; 4 middlewares e 5 arquivos em [`src/shared/`](../../src/shared/), ~339 linhas | Autenticação e papéis, validação, erros HTTP, logging estruturado, correlação e paginação. | Express, Zod, JWT, Prisma, Pino e configuração; usado pelos demais módulos. |
| **TEST** | Testes e ferramental; 2 suítes, setup e factory, ~406 linhas em [`tests/`](../../tests/) | Testes HTTP de autenticação e pedidos com Vitest/Supertest, aplicação real e Prisma; setup limpa as tabelas antes de cada teste. | **API**, **DATA** e banco configurado; scripts em [`package.json`](../../package.json) e execução serial em [`vitest.config.ts`](../../vitest.config.ts). |

## Relações e preocupações transversais

- **Fluxo HTTP:** `src/server.ts` inicializa a aplicação; `src/app.ts` instancia dependências; `src/routes/index.ts` monta rotas de domínio. Rotas aplicam middlewares e schemas; controllers chamam services; services normalmente chamam repositories. Essa é uma descrição da estrutura, não garantia de isolamento entre camadas.
- **Identidade e autorização:** `AUTH` usa `UserRepository` e `UserService` de `USERS`. [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts) valida bearer JWT e oferece `requireRole`; as rotas de clientes, produtos e pedidos exigem autenticação, enquanto a consulta de usuário por ID também exige `ADMIN`. O JWT contém ID, email e papel de usuário, não um `customerId`.
- **Dados e consistência:** [`prisma/schema.prisma`](../../prisma/schema.prisma) define `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`. A migração inicial contém chaves e índices. O histórico relaciona pedido e usuário; preço e quantidade ficam registrados nos itens do pedido.
- **Ciclo do pedido e estoque:** na criação, `OrderService.create` verifica cliente e produtos, agrega itens, calcula totais e cria pedido `PENDING` com histórico. `changeStatus` valida a transição e executa, em transação Prisma, atualização de estoque quando aplicável, status e histórico. O débito ocorre em `PENDING → PAID`; o retorno ocorre ao cancelar de `PAID` ou `PROCESSING`. Ver [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts) e [`src/modules/orders/order.status.ts`](../../src/modules/orders/order.status.ts).
- **Erros, validação e observabilidade:** [`src/middlewares/validate.middleware.ts`](../../src/middlewares/validate.middleware.ts) aplica Zod; [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts) traduz erros de aplicação, validação e alguns erros Prisma. [`src/middlewares/request-logger.middleware.ts`](../../src/middlewares/request-logger.middleware.ts) registra duração e ID; [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts) configura Pino e redação de campos sensíveis.
- **Configuração e execução:** [`src/config/env.ts`](../../src/config/env.ts) valida as variáveis exigidas; [`.env.example`](../../.env.example) documenta nomes de configuração. [`docker-compose.yml`](../../docker-compose.yml) provisiona somente o MySQL, não a API ou um worker.
- **Qualidade e dados de teste:** [`tests/auth.test.ts`](../../tests/auth.test.ts) cobre fluxos de autenticação e [`tests/orders.test.ts`](../../tests/orders.test.ts) cobre criação, transições e estoque, entre outros casos. [`tests/setup.ts`](../../tests/setup.ts) apaga dados das tabelas no banco configurado; a análise não executou a suíte e não pressupõe cobertura exaustiva.

## Contexto da transcrição: proposto, não implementado

[`TRANSCRICAO.md`](../../TRANSCRICAO.md) descreve uma feature futura de notificações de status por webhooks de saída. A reunião discute outbox MySQL na transação de mudança de status, worker separado com polling, retentativas e dead letter, assinatura HMAC, configuração de endpoints e histórico de entregas. Esses itens são contexto para investigação posterior, **não componentes existentes**: o código, os scripts e o schema atuais não apresentam módulo de webhooks, tabela de outbox/DLQ ou entry point de worker. O possível gancho citado na reunião é `OrderService.changeStatus`, em [`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts); sua transação atual não publica eventos.

A transcrição também coloca email e dashboard fora do escopo atual e deixa rate limiting de saída para observação futura. Trechos com nomes de caminhos ou endpoints propostos são sugestões da reunião, não referências a arquivos ou rotas já disponíveis. A reunião deve ser lida integralmente antes de inferir requisitos ou resolução de dúvidas.

## Orientações para a Fase 2

1. Selecionar IDs desta tabela e examinar cada módulo, suas dependências e limites compartilhados; começar por **ORDERS**, **DATA** e **CROSS** se o foco for a feature da transcrição, sem presumir que os demais sejam irrelevantes.
2. Separar evidência do código existente, proposta da reunião e pontos ainda abertos. Confrontar implementação, schema, testes e histórico Git quando pertinente; não inferir motivação, alternativas ou aprovação apenas a partir de uma biblioteca ou padrão encontrado.
3. Identificar apenas potenciais decisões arquiteturais sustentadas por evidências e impacto relevante; consolidar temas sobrepostos entre módulos e distinguir detalhes de implementação de decisões. Registrar incertezas e referências verificáveis.
4. Antes de gerar candidatos, verificar ADRs existentes em [`docs/adrs/`](./) e aplicar o fluxo e os critérios da Fase 2 (`/adr-identify`), evitando duplicações. Este mapeamento não constitui uma lista de ADRs nem formaliza decisões.
5. Se o foco for webhooks, acompanhar as fronteiras propostas com pedidos, dados, autorização e operação, sem tratar caminhos futuros como implementados. Revisar a transcrição completa e os documentos de requisitos quando forem preenchidos.

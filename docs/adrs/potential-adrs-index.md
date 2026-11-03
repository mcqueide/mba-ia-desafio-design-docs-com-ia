# Índice de ADRs potenciais

Os registros abaixo são candidatos arquivados após a geração de rascunhos de ADRs. A pontuação é uma avaliação inicial e as decisões de webhooks registradas na reunião ainda não estão implementadas. Os sete rascunhos foram revisados com esclarecimentos do usuário e numerados; mantêm o status proposto.

## Progresso da análise

### Módulos analisados

- **ORDERS** — Pedidos: 1 candidato de alta prioridade, 0 de média.
- **DATA** — Persistência e configuração: 2 candidatos de alta prioridade, 0 de média.
- **CROSS** — Políticas compartilhadas: 2 candidatos de alta prioridade, 0 de média.
- **AUTH** — Autenticação: nenhum candidato novo; coberto por CROSS.
- **USERS** — Usuários: nenhum candidato novo; sobreposição com CROSS e DATA.
- **CUSTOMERS** — Clientes: evidências de arquitetura HTTP incorporadas ao candidato transversal de API.
- **PRODUCTS** — Produtos: nenhum candidato novo; sobreposição com ORDERS, DATA e CROSS.
- **API** — Composição e infraestrutura HTTP: 2 candidatos de alta prioridade, 0 de média.
- **TEST** — Testes e ferramental: nenhum candidato novo.

### Módulos pendentes

Nenhum módulo pendente no mapeamento atual.

## Alta prioridade

| Módulo | Candidato | Categoria | Pontuação |
|---|---|---|---|
| ORDERS | [Webhooks assíncronos de status de pedidos](potential-adrs/done/ORDERS/webhooks-assincronos-de-status-de-pedidos.md) | Integração/API, proposta | 115/150 |
| DATA | [MySQL como banco relacional compartilhado](potential-adrs/done/DATA/mysql-como-banco-relacional-compartilhado.md) | Infraestrutura existente | 130/150 |
| DATA | [Prisma como camada de acesso a dados](potential-adrs/done/DATA/prisma-como-camada-de-acesso-a-dados.md) | ORM existente | 130/150 |
| CROSS | [Autenticação JWT e autorização por papéis nas rotas](potential-adrs/done/CROSS/autenticacao-jwt-e-autorizacao-por-papeis.md) | Segurança existente | 125/150 |
| CROSS | [Autenticação das notificações de webhook por HMAC](potential-adrs/done/CROSS/assinatura-hmac-de-webhooks.md) | Segurança proposta | 105/150 |
| API | [Express como framework HTTP](potential-adrs/done/API/express-como-framework-http.md) | Framework existente | 130/150 |
| API | [API REST HTTP versionada](potential-adrs/done/API/api-rest-versionada.md) | Arquitetura de API existente | 135/150 |

## Média prioridade

Nenhum candidato identificado nos módulos analisados.

## Sobreposições e limites

- O candidato de ORDERS reúne outbox, worker, entrega e assinatura HMAC como partes da integração. O candidato de HMAC também aborda assinatura e rotação: avaliar seus limites antes de gerar ADRs formais.
- MySQL, Prisma e JWT documentam escolhas existentes; o histórico disponível não estabelece suas motivações e alternativas originais. A motivação do Prisma foi esclarecida posteriormente pelo usuário; para MySQL e para a escolha original de JWT, permanecem desconhecidas. Sua reutilização na feature de webhooks é contexto de uma proposta futura.
- A análise de CUSTOMERS reconheceu o mesmo estilo HTTP documentado no candidato transversal de API; suas evidências foram incorporadas ali, sem criar um candidato por recurso. AUTH e USERS complementam o candidato de JWT; PRODUCTS e TEST também fornecem contexto para candidatos existentes.
- O registro atual não exige autenticação e seu schema aceita o papel ADMIN. Os esclarecimentos posteriores para a feature exigem restringir esse comportamento e aplicar papéis por vínculo com cliente; tais mudanças ainda não foram implementadas.
- A ausência de candidatos separados para retry, DLQ, outbox ou worker não significa que essas decisões não foram discutidas; elas aparecem no candidato de integração. O enquadramento final deve considerar as exigências do projeto e evitar duplicação.

## Resumo

- Alta prioridade: **7**
- Média prioridade: **0**
- Total: **7**
- Módulos analisados: **9 de 9**

## Rascunhos de ADR gerados

| Módulo | ADR | Situação |
|---|---|---|
| ORDERS | [Webhooks assíncronos](generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | Entradas preenchidas; proposto |
| CROSS | [Assinatura HMAC](generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md) | Entradas preenchidas; proposto |
| CROSS | [JWT e papéis](generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | Entradas preenchidas; proposto |
| DATA | [MySQL compartilhado](generated/DATA/ADR-005-mysql-compartilhado.md) | Revisado com lacunas históricas e operacionais; proposto |
| DATA | [Prisma](generated/DATA/ADR-006-prisma-acesso-a-dados.md) | Entradas preenchidas; proposto |
| API | [API versionada](generated/API/ADR-001-api-rest-versionada.md) | Revisado com lacunas de alternativas e política; proposto |
| API | [Express](generated/API/ADR-002-express-framework-http.md) | Revisado com lacunas de alternativas e manutenção; proposto |

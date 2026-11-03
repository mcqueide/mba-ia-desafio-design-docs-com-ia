# Relatório de relações entre ADRs numerados

## Escopo

Foram analisados sete ADRs nos módulos API, CROSS, DATA e ORDERS. Após a renumeração, os destinos dos links já apontavam para os arquivos atuais; os rótulos ainda usavam `ADR-XXX` e foram atualizados para os números correspondentes. O agente especializado estava indisponível e a análise foi executada diretamente.

## Relações preservadas

| ADR | ADR relacionado | Justificativa |
|---|---|---|
| [ADR-001: API versionada](../generated/API/ADR-001-api-rest-versionada.md) | [ADR-002: Express](../generated/API/ADR-002-express-framework-http.md) | Contrato HTTP e framework da API. |
| [ADR-001: API versionada](../generated/API/ADR-001-api-rest-versionada.md) | [ADR-004: JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | Contrato e autenticação da API. |
| [ADR-003: HMAC](../generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md) | [ADR-004: JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | Segurança das notificações de saída e autorização de sua gestão. |
| [ADR-003: HMAC](../generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md) | [ADR-007: Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | Assinatura e entrega das notificações. |
| [ADR-004: JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | [ADR-007: Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | Autorização do replay e gestão dos webhooks. |
| [ADR-005: MySQL](../generated/DATA/ADR-005-mysql-compartilhado.md) | [ADR-006: Prisma](../generated/DATA/ADR-006-prisma-acesso-a-dados.md) | Banco e camada de acesso a dados. |
| [ADR-005: MySQL](../generated/DATA/ADR-005-mysql-compartilhado.md) | [ADR-007: Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | Reutilização do banco na outbox proposta. |

Todas as relações são do tipo **Related to** e estão presentes nos dois sentidos. Não foram identificadas novas dependências, substituições ou emendas: os números não comprovam a ordem histórica e as datas continuam desconhecidas.

## Resumo

- ADRs analisados: 7.
- Relações preservadas: 7 pares, 14 links.
- Alterações de status ou conteúdo dos ADRs: nenhuma.

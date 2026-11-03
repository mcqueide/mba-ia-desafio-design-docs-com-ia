# Relatório de relações entre ADRs

## Escopo

- Sete ADRs em `docs/adrs/generated/`, nos módulos ORDERS, CROSS, DATA e API.
- Todos têm status proposto, data desconhecida e identificador provisório `XXX`; os caminhos e títulos distinguem cada documento.
- O histórico Git disponível não registra os ADRs gerados e não demonstra precedência, substituição ou alteração parcial entre eles.
- O agente especializado não estava disponível; a análise foi executada diretamente conforme suas regras.

## Relações registradas

As sete relações abaixo são **Related to**, com links nos dois sentidos:

| ADR | ADR relacionado | Evidência da relação |
|---|---|---|
| [Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | [HMAC](../generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md) | Integração de saída e autenticação das notificações da mesma feature. |
| [Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | [JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | O replay da integração exige autorização administrativa. |
| [Webhooks assíncronos](../generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) | [MySQL](../generated/DATA/ADR-005-mysql-compartilhado.md) | A outbox proposta reutiliza o banco existente. |
| [HMAC](../generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md) | [JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | Autenticação das notificações de saída e autorização para gestão das configurações e secrets são fronteiras complementares. |
| [JWT e papéis](../generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) | [API versionada](../generated/API/ADR-001-api-rest-versionada.md) | As decisões tratam de autenticação e contrato HTTP da mesma API. |
| [API versionada](../generated/API/ADR-001-api-rest-versionada.md) | [Express](../generated/API/ADR-002-express-framework-http.md) | Contrato HTTP e framework que implementa a interface são aspectos distintos da API. |
| [MySQL](../generated/DATA/ADR-005-mysql-compartilhado.md) | [Prisma](../generated/DATA/ADR-006-prisma-acesso-a-dados.md) | Banco compartilhado e camada de acesso compõem a persistência existente. |

## Limites e alertas

- Nenhuma relação de dependência foi inferida: as datas das decisões são desconhecidas e os critérios temporais não podem ser comprovados. A associação da outbox ao MySQL foi registrada como relação, não como precedência histórica.
- Nenhuma substituição ou emenda foi identificada; não há evidência de evolução entre os ADRs.
- A autenticação HMAC e a integração de webhooks têm assuntos sobrepostos, mas tratam de aspectos diferentes. A numeração posterior exigirá atualizar rótulos e caminhos dos links.

## Resumo

- ADRs examinados e atualizados: 7.
- Relações: 7 pares, 14 links nos cabeçalhos.
- Supersedes, Depends on e Amends: 0.

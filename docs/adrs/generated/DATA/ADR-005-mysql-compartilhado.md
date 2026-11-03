# ADR-005: MySQL como banco relacional compartilhado

**Status:** Proposto
**Data:** Desconhecida
**Related to:**
- [ADR-007: Webhooks assíncronos para mudanças de status](../ORDERS/ADR-007-webhooks-assincronos-de-status.md)
- [ADR-006: Prisma como camada de acesso a dados](ADR-006-prisma-acesso-a-dados.md)

## Contexto e problema

O sistema existente usa MySQL para os dados dos domínios da API. O repositório contém schema, migração e configuração de desenvolvimento, mas o histórico disponível não explica a escolha original nem a topologia de produção.

A reunião escolheu aproveitar o banco existente para a outbox proposta, em vez de adicionar Redis para essa finalidade.

## Fatores de decisão

- Compartilhar uma camada persistente entre os domínios atuais.
- Preservar operações transacionais necessárias ao fluxo de pedidos.
- Evitar infraestrutura adicional para a outbox proposta.
- Considerar o custo de manutenção e migração de dados.

## Opções consideradas

1. MySQL como banco relacional compartilhado.

Não há registro das alternativas consideradas para a escolha original do banco.

## Resultado da decisão

O uso de MySQL está evidenciado na implementação atual. Para a feature de webhooks, a reunião decidiu reutilizá-lo para a outbox; esse motivo não comprova por que o banco foi escolhido originalmente.

A motivação original não é conhecida, e não há informação disponível sobre a implantação e a operação do banco nos diferentes ambientes.

## Prós e contras das opções

### MySQL compartilhado

- **Pró:** os domínios atuais já usam o mesmo banco e suas transações.
- **Pró:** a outbox proposta aproveita infraestrutura existente.
- **Contra:** a migração de banco envolve schema, dados e integrações.
- **Contra:** a outbox adicionaria carga e operação à infraestrutura compartilhada.

### Alternativas

Não há evidência suficiente para descrever os trade-offs das alternativas originais.

## Consequências

O schema e a migração dependem de características MySQL. A infraestrutura descrita no Compose é evidência do ambiente configurado no repositório, não de uma topologia de produção; responsáveis por manutenção, backup e recuperação não foram identificados.

A outbox continua sendo proposta: sua capacidade, retenção e comportamento de concorrência não são garantias da implementação existente.

## Referências

- [Schema Prisma](../../../../prisma/schema.prisma)
- [Migração inicial](../../../../prisma/migrations/20260519182739_init/migration.sql)
- [Docker Compose](../../../../docker-compose.yml)
- [Transcrição: outbox e Redis](../../../../TRANSCRICAO.md#L30-L58)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: motivação original desconhecida, alternativas sem registro e operação por ambiente sem informação disponível.

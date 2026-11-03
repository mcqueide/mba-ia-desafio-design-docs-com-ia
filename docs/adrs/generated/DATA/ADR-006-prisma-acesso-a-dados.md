# ADR-006: Prisma como camada de acesso a dados

**Status:** Proposto
**Data:** Desconhecida
**Related to:** [ADR-005: MySQL como banco relacional compartilhado](ADR-005-mysql-compartilhado.md)

## Contexto e problema

A aplicação usa Prisma Client, schema e migrações para acessar o banco. A composição fornece o cliente a repositórios e serviços; operações transacionais de pedidos também acessam o cliente diretamente.

A reunião propôs reutilizar a mesma stack de dados no worker de webhooks, com instância separada por processo. Esse worker ainda não existe.

## Fatores de decisão

- Compartilhar tipos e acesso ao banco entre os domínios.
- Suportar operações transacionais envolvendo múltiplas alterações.
- Manter schema e migrações versionados.
- Reutilizar os padrões de persistência existentes na feature proposta.

## Opções consideradas

1. Prisma como camada de acesso e migração.
2. TypeORM.

## Resultado da decisão

O uso de Prisma é evidenciado pelo código existente, e a reunião escolheu reutilizá-lo para o processamento proposto. Conforme esclarecido posteriormente, a escolha priorizou produtividade, segurança de tipos com menos trabalho manual e clareza de um schema declarativo concentrado em um arquivo. Também se buscou reduzir a criação de consultas SQL desorganizadas, sem que a ferramenta por si só garanta esse resultado.

Serviços podem acessar diretamente o cliente transacional Prisma quando necessário.

## Prós e contras das opções

### Prisma

- **Pró:** mantém cliente, schema e migrações na mesma stack.
- **Pró:** permite agrupar alterações relacionadas em transações.
- **Contra:** a API do ORM está presente também fora dos repositórios.
- **Contra:** uma substituição exigiria revisão de acessos, transações e migrações.

### Alternativas

O TypeORM foi avaliado. A preferência pelo Prisma foi fundamentada nos objetivos de produtividade, tipagem e clareza do schema; não há registro adicional de uma análise comparativa detalhada do TypeORM.

## Consequências

A decisão influencia os limites entre serviços e persistência. A integração da outbox proposta dependerá da transação existente, mas o código atual não registra eventos.

A reunião prevê um cliente por processo para o worker; isso não demonstra comportamento operacional antes da implementação.

## Referências

- [Configuração do banco](../../../../src/config/database.ts)
- [Schema Prisma](../../../../prisma/schema.prisma)
- [Serviço de pedidos](../../../../src/modules/orders/order.service.ts#L126-L178)
- [Transcrição: processo e transação](../../../../TRANSCRICAO.md#L238-L244)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: motivação, TypeORM e acesso transacional direto.

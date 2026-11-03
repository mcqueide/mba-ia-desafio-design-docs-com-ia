# Potential ADR: Prisma como camada de acesso a dados

**Module**: DATA  
**Category**: Technology — ORM e acesso a dados  
**Priority**: Must Document (Score: 130/150)  
**Date Identified**: 2026-11-02  
**Estado da evidência**: implementação existente; reuso pelo worker de webhooks é decisão da reunião, ainda não implementada.

## What Was Identified

O projeto usa Prisma Client e Prisma CLI 5.22.0, com schema e migração versionados. A configuração cria um `PrismaClient`, e a composição da API o passa aos repositórios dos domínios e ao serviço de pedidos. O acesso não é inteiramente encapsulado por repositórios: `OrderService.changeStatus` usa diretamente uma transação Prisma para atualizar estoque quando aplicável, status e histórico.

Schema, cliente, scripts e migração já estavam no commit inicial `7ef4317` em 2026-06-24. A mensagem “init repository” não informa a motivação da escolha nem as alternativas consideradas; não há evolução anterior observável no Git disponível.

Na reunião, decidiu-se que um worker separado compartilharia o banco e a stack, mas usaria uma nova instância de `PrismaClient` no outro processo. Para outbox, propôs-se receber o cliente transacional existente na integração com pedidos. Isso descreve intenção futura: não há worker ou outbox implementados no código analisado.

## Why This Might Deserve an ADR

- **Impacto**: cliente e tipos Prisma atravessam a composição e os repositórios de vários domínios, bem como transações em pedidos.
- **Trade-offs**: acesso transacional direto permite que várias operações usem o mesmo cliente de transação, mas expõe a API Prisma ao serviço; uma substituição exigiria revisar esses pontos e as migrações.
- **Conhecimento da equipe**: é importante conhecer os limites entre repositórios, cliente e operações transacionais antes de estender persistência.
- **Pontuação estimada**: Step 0, ORM **75**; escopo/impacto **20** (camada central de múltiplos módulos); custo de mudança **15** (estimativa de 2–8 semanas para adaptar acessos, transações e testes, não medição); conhecimento **20** (persistência permeia a maioria das funcionalidades). Total **130/150**. Estimativa sujeita à validação da equipe.

## Evidence Found in Codebase

- [`package.json:10-23`](../../../../../package.json#L10-L23), [`25-27`](../../../../../package.json#L25-L27) e [`48-50`](../../../../../package.json#L48-L50): scripts de migração e seed, dependências Prisma.
- [`prisma/schema.prisma:1-9`](../../../../../prisma/schema.prisma#L1-L9) e [`prisma/migrations/20260519182739_init/migration.sql`](../../../../../prisma/migrations/20260519182739_init/migration.sql): geração do cliente, datasource e migração versionada.
- [`src/config/database.ts:1-10`](../../../../../src/config/database.ts#L1-L10): criação e exportação do cliente.
- [`src/app.ts:23-43`](../../../../../src/app.ts#L23-L43): composição do cliente com repositórios e `OrderService`.
- [`src/modules/orders/order.service.ts:126-178`](../../../../../src/modules/orders/order.service.ts#L126-L178): transação existente de mudança de status; não insere eventos de webhook.
- [`TRANSCRICAO.md:70-76`](../../../../../TRANSCRICAO.md#L70-L76), [`176-180`](../../../../../TRANSCRICAO.md#L176-L180) e [`238-244`](../../../../../TRANSCRICAO.md#L238-L244): proposta de worker separado, cliente por processo e integração com a transação.

```ts
return this.prisma.$transaction(async (tx) => {
  // A implementação atual usa tx para as operações da mudança de status.
```

O trecho de código acima abre a transação em `src/modules/orders/order.service.ts:131`; o comentário é explicativo, não foi extraído do código. O histórico dos arquivos principais contém apenas o commit inicial; ele não comprova estabilidade operacional.

## Questions to Address in ADR (if created)

- Qual foi a motivação original para Prisma e quais alternativas foram consideradas?
- Qual fronteira de acesso direto ao Prisma se pretende manter nos serviços?
- Como as futuras inserções de outbox participarão da transação e como isso será testado?

## Related Potential ADRs

- [MySQL como banco relacional compartilhado](mysql-como-banco-relacional-compartilhado.md): relacionado, porém trata o serviço e o armazenamento, não a escolha do ORM.

## Additional Notes

Não foram encontrados ADRs formais em `docs/adrs/` ou `docs/adrs/generated/` na análise. O uso de transações em pedidos e a outbox de webhooks podem se sobrepor a análises de ORDERS ou do coordenador; não se presume garantia de entrega, ordenação ou atomicidade da outbox antes da sua implementação.

# Potential ADR: MySQL como banco relacional compartilhado

**Module**: DATA  
**Category**: Technology — serviço de infraestrutura  
**Priority**: Must Document (Score: 130/150)  
**Date Identified**: 2026-11-02  
**Estado da evidência**: implementação existente; extensão para webhooks decidida na reunião, ainda não implementada.

## What Was Identified

O datasource Prisma usa MySQL e `DATABASE_URL`; o schema reúne os modelos usados pelos domínios da API. A migração inicial cria tabelas, índices e chaves estrangeiras. O Compose disponibiliza MySQL 8 com volume nomeado; isso demonstra a configuração disponível no repositório, não a topologia de produção.

Esses arquivos aparecem no commit inicial `7ef4317` de 2026-06-24, cuja mensagem é apenas “init repository”. O histórico disponível não explica por que MySQL foi escolhido nem mostra avaliação de outros bancos. A migração tem uma data no nome, mas sua presença no Git só é comprovada a partir desse commit.

Para a feature futura, a reunião decidiu reutilizar o MySQL existente para uma outbox transacional em vez de adicionar Redis Streams/Cluster: os participantes apontaram o custo de infraestrutura adicional para uma equipe pequena. Essa justificativa refere-se à **outbox proposta**, não à escolha original do banco. A decisão de inserir evento na transação de status e a proposta de uma DLQ em tabela separada não estão implementadas no schema atual.

## Why This Might Deserve an ADR

- **Impacto**: infraestrutura central para usuários, clientes, produtos e pedidos; trocar o banco envolve schema, migração e integrações.
- **Trade-offs**: há tipos e SQL específicos de MySQL na migração. A outbox proposta reutilizaria essa infraestrutura, mas introduziria nela carga e operação de entregas; os efeitos efetivos precisam de avaliação.
- **Conhecimento da equipe**: registrar os limites de configuração, evolução e operação evita confundir Compose com a implantação real.
- **Pontuação estimada**: Step 0, infraestrutura **75**; escopo/impacto **20** (infraestrutura central de múltiplos módulos); custo de mudança **15** (estimativa de 2–8 semanas, por migração e validação de dados, não medição); conhecimento **20** (persistência afeta a maioria das funcionalidades). Total **130/150**. Os custos devem ser validados com a equipe.

## Evidence Found in Codebase

- [`prisma/schema.prisma:1-9`](../../../../../prisma/schema.prisma#L1-L9): datasource MySQL e URLs de banco.
- [`prisma/schema.prisma:25-138`](../../../../../prisma/schema.prisma#L25-L138): modelos dos domínios em um schema.
- [`prisma/migrations/20260519182739_init/migration.sql:1-125`](../../../../../prisma/migrations/20260519182739_init/migration.sql#L1-L125): tabelas, índices e relações na migração inicial.
- [`docker-compose.yml:1-28`](../../../../../docker-compose.yml#L1-L28): MySQL 8, volume e health check para o ambiente configurado.
- [`src/config/env.ts:3-9`](../../../../../src/config/env.ts#L3-L9): validação de `DATABASE_URL`.
- [`TRANSCRICAO.md:30-58`](../../../../../TRANSCRICAO.md#L30-L58), [`108-114`](../../../../../TRANSCRICAO.md#L108-L114), [`194-200`](../../../../../TRANSCRICAO.md#L194-L200) e [`238-244`](../../../../../TRANSCRICAO.md#L238-L244): decisão de outbox, alternativa Redis, DLQ e integração transacional propostas. A reunião também prevê filtrar eventos antes da inserção, portanto a emissão não se aplica automaticamente a toda mudança de status.

```prisma
datasource db {
  provider          = "mysql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

O trecho é de `prisma/schema.prisma:5-9`. O histórico de cada arquivo principal acima contém apenas o commit inicial; isso não comprova estabilidade operacional ou implantação.

## Questions to Address in ADR (if created)

- Por que MySQL foi originalmente escolhido e quais alternativas foram avaliadas?
- Como o banco é implantado, mantido e recuperado em cada ambiente?
- A outbox de webhooks deve ser tratada em decisão própria de integração? Como validar atomicidade, capacidade, concorrência e retenção quando for implementada?

## Related Potential ADRs

- [Prisma como camada de acesso a dados](prisma-como-camada-de-acesso-a-dados.md): relacionado, mas cobre cliente, schema e transações, não a escolha do serviço de banco.

## Additional Notes

Não foram encontrados ADRs formais em `docs/adrs/` ou `docs/adrs/generated/` na análise. A outbox pode se sobrepor a candidatos de ORDERS ou ao trabalho do coordenador sobre webhooks; a reunião também discute worker, retentativas e entrega. Esses aspectos não são garantias da implementação existente. Aplicando a regra literal da Fase 2, a outbox isolada não recebe base Step 0: escopo **20**, custo estimado **15**, conhecimento **15** = **50/150**, abaixo do limiar 75. É registrada aqui como contexto, sem criar um candidato separado ou elevar artificialmente a nota.

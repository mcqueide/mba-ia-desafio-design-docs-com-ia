# Potencial ADR: Webhooks assíncronos de status de pedidos

**Módulo:** ORDERS  
**Categoria:** arquitetura de integração/API  
**Prioridade:** Must Document (115/150)  
**Identificado em:** 2026-11-02  
**Estado da evidência:** decisão registrada na reunião para uma feature futura; não implementada.

## O que foi identificado

A reunião escolheu uma integração de saída por webhook para mudanças de status de pedidos: gravar um evento na outbox MySQL na mesma transação de `changeStatus`, e despachá-lo por um worker separado. O filtro por status e cliente ocorreria na inserção; se nenhum webhook quiser o status, não haveria evento. A reunião rejeitou a chamada HTTP síncrona no serviço por associar a disponibilidade e a latência do cliente à mudança de status e considerou Redis Streams como alternativa com mais infraestrutura ([transcrição, linhas 18–58](../../../../../TRANSCRICAO.md#L18-L58), [194–200](../../../../../TRANSCRICAO.md#L194-L200), [238–244](../../../../../TRANSCRICAO.md#L238-L244)).

A decisão inclui o contrato externo e a operação da entrega: payload JSON em snapshot, identidade de evento para deduplicação no cliente, HMAC-SHA256 com secret por endpoint e rotação, worker em polling, tentativas limitadas, dead letter e replay administrativo. Esses aspectos são consolidados aqui por sustentarem a mesma integração, não por estarem implementados ([transcrição, linhas 60–86](../../../../../TRANSCRICAO.md#L60-L86), [102–158](../../../../../TRANSCRICAO.md#L102-L158), [254–266](../../../../../TRANSCRICAO.md#L254-L266), [302–314](../../../../../TRANSCRICAO.md#L302-L314)).

Hoje `changeStatus` já usa uma transação Prisma para estoque, status e histórico, mas não insere eventos nem faz chamadas a webhooks. O schema atual não define outbox, dead letter ou configurações de webhook. A transcrição afirma que o design seria escrito antes de começar a codar; a data de calendário da reunião não é informada ([serviço, linhas 126–178](../../../../../src/modules/orders/order.service.ts#L126-L178), [schema](../../../../../prisma/schema.prisma), [transcrição, linha 292](../../../../../TRANSCRICAO.md#L292)).

## Por que pode merecer um ADR

- **Impacto:** altera o estilo de integração com clientes e cruza pedidos, persistência, processamento separado e contratos externos.
- **Trade-offs relatados:** transação com outbox evita vincular a operação ao HTTP do cliente, mas introduz processamento assíncrono, possibilidade de duplicatas e responsabilidade de deduplicação no consumidor. A retenção, a recuperação e os limites de ordenação precisam ser especificados.
- **Conhecimento da equipe:** implementação e consumidores precisam distinguir registro atômico, tentativas de entrega, falha permanente e replay; registrar a assinatura e sua rotação também afeta integrações externas.
- **Pontuação provisória:** base **75** pelo Step 0, categoria arquitetura de API: webhooks HTTP de saída são um novo estilo de integração externa, não uma escolha genérica de MySQL ou Prisma; **15** por escopo (ORDERS, DATA e módulo de webhooks proposto, além dos clientes); **10** por custo de mudança estimado conservadoramente em 1–2 semanas; **15** por conhecimento necessário nas áreas de pedidos, dados e integração. O custo é hipótese de classificação, não dado histórico: a reunião estimou três sprints para a feature inteira, não o custo de substituí-la ([linhas 268–276](../../../../../TRANSCRICAO.md#L268-L276)). Total: **115/150**.
- **Filtro:** decisão estrutural, relevante para engenharia e pretendida como contrato duradouro; valores isolados de polling, timeout, limite de payload e backoff são detalhes desta estratégia, não candidatos separados.

## Evidências e limites

- [`order.service.ts`, linhas 126–167](../../../../../src/modules/orders/order.service.ts#L126-L167): transação existente, validação de transição, movimentação de estoque e gravação do status e histórico. Excerto:

  ```ts
  return this.prisma.$transaction(async (tx) => {
    // ... validação e movimentação de estoque ...
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({ /* ... */ });
  ```

- [`tests/orders.test.ts`, linhas 59–86 e 109–131](../../../../../tests/orders.test.ts#L59-L131): testes existentes verificam status, histórico, estoque e rejeição por falta de estoque; não comprovam entrega de webhooks. Não foram executados nesta análise documental.
- [`prisma/schema.prisma`, linhas 72–138](../../../../../prisma/schema.prisma#L72-L138): modelos de pedido, itens, histórico e sequência existentes; nenhuma tabela de outbox ou DLQ está definida no schema completo.
- [`TRANSCRICAO.md`, linhas 146–158 e 280–288](../../../../../TRANSCRICAO.md#L146-L158): a reunião chama o objetivo de entrega de *at-least-once* e prevê `X-Event-Id` para deduplicação; tentativas limitadas e DLQ não comprovam entrega eventual ao destinatário em todos os cenários.
- [`TRANSCRICAO.md`, linhas 78–86](../../../../../TRANSCRICAO.md#L78-L86): a reunião espera ordem por pedido com um worker, mas não especifica como retries, desempates e replay preservarão essa ordem. Não é uma garantia demonstrada.
- [`TRANSCRICAO.md`, linhas 18–22, 60–68](../../../../../TRANSCRICAO.md#L18-L68): há expectativa de latência abaixo de 10 segundos e polling a cada 2 segundos; isso, por si só, não garante latência de entrega.
- [`TRANSCRICAO.md`, linhas 184–200 e 202–216](../../../../../TRANSCRICAO.md#L184-L216): `customer_id` não vem do JWT atual; a reunião prevê CRUD autenticado e replay com `ADMIN` e auditoria. As regras de autorização por cliente precisam ser explicitadas, sem pressupor isolamento existente.

O commit `7ef4317` (`init repository`, 2026-06-24) introduziu `order.service.ts`, o schema, os testes e a transcrição no histórico disponível. Para cada um desses arquivos, o histórico consultado contém apenas esse commit; ele não documenta a motivação da transação nem a data ou implementação da decisão de webhooks. A duração real da decisão não pode ser inferida.

## Questões para o ADR formal

1. Como definir atomicidade quando não há assinatura interessada, quando existem múltiplos endpoints e quando ocorre falha na gravação? Como validar rollback sem afirmar entrega HTTP atômica?
2. Cinco tentativas incluem a primeira chamada ou são cinco retentativas? Como reconciliar essa contagem com os cinco intervalos listados? Qual a semântica de replay e o tratamento de falha permanente?
3. Como implementar e testar ordenação por pedido diante de retries, empates de timestamp, reinícios e replay? Há alguma garantia efetiva de latência ou apenas uma meta?
4. Como definir persistência e rotação segura das secrets, autenticação e autorização de configurações por cliente, assinatura verificável e auditoria do replay?
5. Como versionar o payload snapshot, preservar o identificador em retentativas/replay e documentar a deduplicação do consumidor?
6. Quem opera worker e DLQ, e como monitorar atrasos? A reunião deixa rate limiting de saída para observação futura e exclui email e dashboard nesta fase ([linhas 218–234](../../../../../TRANSCRICAO.md#L218-L234)).

## Contexto de ADRs e sobreposições

Na leitura de `docs/adrs/` não havia ADR formal nem diretório `generated/`; havia apenas README e mapeamento, portanto não há duplicata formal identificada. Esta análise não cria candidatos genéricos para MySQL, Prisma, REST ou autenticação: podem pertencer aos módulos DATA, API e CROSS. A integração proposta se sobrepõe a esses módulos em persistência, segurança e operação; consolidar com os candidatos dos respectivos responsáveis antes da Fase 3. A transação atual de pedidos, as regras de status e o estoque são evidências e contexto, não ADRs separados de workflow.

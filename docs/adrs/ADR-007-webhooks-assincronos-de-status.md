# ADR-007: Webhooks assíncronos para mudanças de status

**Status:** Proposto
**Data:** Desconhecida
**Related to:**
- [ADR-003: Autenticação de webhooks por HMAC](ADR-003-assinatura-hmac-de-webhooks.md)
- [ADR-004: Autenticação JWT e autorização por papéis](ADR-004-autenticacao-jwt-e-papeis.md)
- [ADR-005: MySQL como banco relacional compartilhado](ADR-005-mysql-compartilhado.md)

## Contexto e problema

Clientes B2B precisam receber notificações sobre mudanças de status de pedidos sem consultar a API continuamente. A reunião rejeitou o envio HTTP síncrono durante a alteração de status: a disponibilidade e a latência do destinatário não devem impedir a operação do pedido.

A aplicação já mantém status, histórico e movimentação de estoque em transação. Não há outbox ou processamento de webhooks implementados.

## Fatores de decisão

- Preservar a consistência entre a mudança do pedido e o registro de eventos elegíveis.
- Evitar que chamadas externas atrasem ou revertam a operação principal.
- Reutilizar o MySQL existente e evitar infraestrutura adicional.
- Permitir recuperação de falhas de entrega e deduplicação pelo destinatário.
- Manter uma expectativa de baixa latência sem presumir garantia sob falhas ou backlog.

## Opções consideradas

1. Outbox transacional no MySQL e processamento assíncrono separado.
2. Chamada HTTP síncrona durante a mudança de status.
3. Uso de Redis Streams como infraestrutura adicional.

## Resultado da decisão

A reunião escolheu a primeira opção: registrar eventos elegíveis na mesma transação da mudança e entregá-los por um worker separado. O conteúdo é um snapshot do evento, com identificador para deduplicação; a entrega é concebida como *at-least-once*, com retentativas limitadas, dead letter e replay administrativo.

Após a chamada inicial, serão feitas até cinco retentativas, com intervalos sucessivos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas: no máximo seis chamadas. Esse esclarecimento posterior substitui a interpretação de cinco tentativas totais registrada na reunião.

Enquanto um evento estiver em retentativa, eventos posteriores do mesmo pedido aguardam. Após o sucesso ou a transferência para a DLQ, os posteriores podem seguir. O replay manual pelo endpoint administrativo recoloca o evento na outbox como pendente, preservando o mesmo `X-Event-Id`.

## Prós e contras das opções

### Outbox transacional e worker

- **Pró:** registra a mudança e o evento de forma atômica.
- **Pró:** desacopla a operação principal da disponibilidade do destinatário.
- **Contra:** exige operação de processamento, retentativas e tratamento de falhas.
- **Contra:** pode produzir entregas duplicadas, exigindo deduplicação no consumidor.

### Chamada HTTP síncrona

- **Pró:** evita o processamento assíncrono separado.
- **Contra:** vincula a alteração do pedido à latência e à disponibilidade do cliente.

### Redis Streams

- **Pró:** oferece uma infraestrutura de mensageria separada.
- **Contra:** acrescenta infraestrutura e operação para a equipe.

## Consequências

O envio não é parte atômica da transação do pedido: a atomicidade cobre o registro do evento, não o recebimento externo. As retentativas podem terminar em falha permanente e não demonstram entrega eventual em todos os cenários.

Um único worker foi a premissa discutida para ordenação por pedido. Empates de timestamp serão desfeitos pelo identificador; isso produz uma ordem determinística, mas não garante a ordem real entre eventos com o mesmo timestamp. Após reinício, eventos não concluídos serão retomados, admitindo entrega duplicada com o mesmo identificador.

Como eventos posteriores podem seguir após a transferência para a DLQ, um replay pode chegar fora da ordem original. A preservação do identificador permite a deduplicação pelo destinatário, inclusive se ele já tiver recebido o evento antes de uma falha de confirmação.

## Referências

- [Transcrição: outbox, alternativas e worker](../../TRANSCRICAO.md#L30-L86)
- [Transcrição: retentativas, DLQ e deduplicação](../../TRANSCRICAO.md#L88-L158)
- [Transcrição: integração e snapshot](../../TRANSCRICAO.md#L238-L244)
- [Serviço de pedidos](../../src/modules/orders/order.service.ts#L126-L178)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: contagem de tentativas, ordenação, recuperação e replay.

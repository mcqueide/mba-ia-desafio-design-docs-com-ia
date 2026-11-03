# RFC-001: Webhooks para notificações de status de pedidos

| Metadado | Valor |
|---|---|
| **Autor** | GitHub Copilot CLI, com base na reunião e nos esclarecimentos posteriores do usuário |
| **Revisores** | Larissa (Tech Lead), Marcos (Produto), Bruno (Pedidos), Diego (Plataforma) e Sofia (Segurança) |
| **Data de publicação** | Novembro de 2026 |
| **Status** | Proposed |
| **Obsoletes / Updates** | Não se aplica: nenhuma RFC anterior identificada. |

## Resumo executivo

Propõe-se notificar clientes B2B sobre mudanças de status de pedidos por webhooks de saída. Eventos elegíveis serão registrados em uma outbox MySQL na mesma transação da mudança de status e entregues por um worker separado. O fluxo prevê assinatura HMAC, retentativas, dead letter e replay administrativo, sem vincular a operação de pedidos à disponibilidade do cliente.

Esta RFC está submetida para revisão: as decisões da reunião e os esclarecimentos posteriores estão refletidos nos ADRs relacionados, também em status **Proposto**. Detalhes de contratos e implementação pertencem ao FDD.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam pedidos periodicamente para descobrir alterações, tornando suas integrações lentas e custosas. Para esses clientes, notificações em menos de dez segundos são percebidas como tempo real ([reunião: 09:00–09:02](../TRANSCRICAO.md#L18-L28)). Essa é uma meta de experiência, não uma garantia de entrega durante indisponibilidade ou acúmulo de eventos.

A API atual usa Node.js, Express, Prisma e MySQL. A mudança de status já atualiza pedido, histórico e, quando aplicável, estoque em uma transação ([serviço de pedidos](../src/modules/orders/order.service.ts)). Não há mecanismo de webhook, outbox ou worker implementado. Uma chamada externa nessa transação faria a operação depender da latência e da disponibilidade do destinatário ([reunião: 09:03–09:06](../TRANSCRICAO.md#L30-L44)).

## Proposta técnica

### Publicação e entrega

- **Registro atômico:** ao alterar um status, selecionar os webhooks ativos do cliente interessados naquele status e registrar os eventos elegíveis com um snapshot do pedido na mesma transação. Se não houver interessados, não há evento; se a gravação falhar, a transação é revertida. Isso garante o registro junto à mudança, não o recebimento HTTP pelo cliente ([reunião: 09:33–09:34 e 09:40–09:41](../TRANSCRICAO.md#L194-L244)).
- **Processamento independente:** um único worker, em processo separado da API, consulta a outbox a cada dois segundos e envia os eventos. Ele compartilha o banco e a stack Prisma, com cliente próprio no processo. O polling contribui para a meta de latência, mas não a assegura sob carga, timeout ou falhas ([reunião: 09:09–09:13](../TRANSCRICAO.md#L60-L86)).
- **Falhas e identidade:** a entrega é *at-least-once*: duplicatas são possíveis e o destinatário deve deduplicar pelo `X-Event-Id`. Após a chamada inicial, serão até cinco retentativas, com espera de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas; esgotadas, o evento vai para uma DLQ persistente. O replay manual preserva o identificador. Essa contagem de **seis chamadas no máximo** é esclarecimento posterior que substitui a formulação ambígua de cinco tentativas da reunião ([ADR-007](adrs/generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md)). A DLQ limita as tentativas automáticas: não há garantia de recebimento eventual sem recuperação.
- **Ordenação:** enquanto um evento é retentado, eventos posteriores do mesmo pedido aguardam; após sucesso ou envio à DLQ, podem prosseguir. O replay pode chegar fora da ordem original. Empates de timestamp serão desfeitos por identificador, sem garantir a ordem real entre eles; após reinício, eventos não concluídos serão retomados, com possibilidade de duplicatas ([ADR-007](adrs/generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md)).

### Configuração e segurança

O cliente poderá configurar endpoints HTTPS por cliente, escolher os status de interesse, consultar histórico de entregas e rotacionar sua secret. As notificações terão payload enxuto, sem itens do pedido, limitado a 64 KB; chamadas de envio terão timeout de dez segundos ([reunião: 09:23–09:24](../TRANSCRICAO.md#L138-L144), [09:31–09:34](../TRANSCRICAO.md#L184-L202) e [09:42–09:44](../TRANSCRICAO.md#L248-L258)).

Cada endpoint terá secret própria em um secret manager e assinatura HMAC-SHA256. A rotação manterá a chave anterior válida por 24 horas. O ADR esclarece posteriormente a identificação da chave e a assinatura opcional do timestamp; esta limita a idade da mensagem, mas não elimina a necessidade de deduplicação dentro da janela de tolerância ([ADR-003](adrs/generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md)).

O CRUD, o histórico e a consulta de secrets exigirão autenticação e vínculo com o cliente; o replay exigirá papel ADMIN no cliente e registro de auditoria. Papéis por vínculo e provisionamento inicial foram definidos posteriormente e **não existem no modelo atual**: o JWT hoje carrega papel global e não associa usuário a cliente. A mudança também restringirá o registro público de ADMIN e o cadastro de clientes ([ADR-004](adrs/generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md); [middleware atual](../src/middlewares/auth.middleware.ts)).

O módulo seguirá os padrões existentes de módulos, Zod, `AppError`, códigos `WEBHOOK_*` e Pino. O worker precisará tratar suas próprias falhas de processo; o middleware HTTP não se aplica automaticamente a ele ([reunião: 09:27–09:30](../TRANSCRICAO.md#L164-L182)).

**Fora desta fase:** notificações por e-mail, dashboard visual e arquivamento de entregas antigas. Rate limiting de saída fica para observação e decisão posterior ([reunião: 09:08](../TRANSCRICAO.md#L56-L58) e [09:37–09:40](../TRANSCRICAO.md#L218-L234)).

## Alternativas consideradas

| Alternativa | Trade-off e motivo do descarte |
|---|---|
| Enviar HTTP sincronamente na mudança de status | Evitaria um worker, mas um cliente lento ou indisponível atrasaria a operação ou exigiria decidir sobre rollback. A reunião rejeitou acoplar o pedido à chamada externa ([09:03–09:06](../TRANSCRICAO.md#L30-L44)). |
| Redis Streams em vez de outbox no MySQL | Introduziria infraestrutura de mensageria adicional; a equipe considerou sua operação excessiva para o contexto atual e preferiu reutilizar o MySQL ([09:07](../TRANSCRICAO.md#L48-L52)). |
| Trigger de banco para avisar o worker | Foi sugerida para maior reatividade, mas a trigger não notifica diretamente um processo externo; exigiria um mecanismo adicional. Preferiu-se polling ([09:09](../TRANSCRICAO.md#L60-L64)). |

## Questões em aberto

1. **Limitação de taxa:** após observar o volume de envios, será necessário limitar chamadas por destinatário? A reunião adiou essa decisão ([09:38–09:39](../TRANSCRICAO.md#L224-L230)).
2. **Retenção e arquivamento:** como e quando arquivar ou remover entregas e eventos antigos? A reunião mencionou cerca de 30 dias, mas deixou o arquivamento fora do escopo; não há política fechada ([09:08](../TRANSCRICAO.md#L54-L56)).
3. **Escala e ordenação:** se um único worker não for suficiente, qual estratégia permitirá paralelismo sem perder o comportamento por pedido? Particionamento e locking foram mencionados para avaliação futura ([09:12–09:13](../TRANSCRICAO.md#L78-L86)).
4. **Operação e segurança:** como serão monitorados backlog e DLQ e protegidos destinos HTTPS contra acesso indevido a redes internas ou redirecionamentos? A revisão de segurança é necessária antes do deploy; a reunião não fechou esses mecanismos ([09:46](../TRANSCRICAO.md#L272-L276)).

## Impacto e riscos

| Impacto ou risco | Consideração para revisão |
|---|---|
| **Pedidos e banco** | Inserir na transação protege a consistência dos eventos elegíveis, mas uma falha da outbox também impede a mudança de status e aumenta a carga no MySQL. |
| **Entrega e operação** | Um único worker simplifica a operação inicial, mas indisponibilidade, backlog e bloqueio por retentativas podem atrasar eventos; a DLQ e seu replay exigem acompanhamento. |
| **Consumidores** | Duplicatas e replay fora de ordem exigem deduplicação e tratamento adequado pelo cliente; o timeout e a DLQ impedem prometer entrega bem-sucedida ou latência inferior a dez segundos em todas as situações. |
| **Segurança e autorização** | Segredos por endpoint, rotação e auditoria reduzem exposição, mas exigem gestão segura; vínculos e papéis por cliente demandam mudanças no modelo de identidade atual. Sofia solicitou revisão de segurança antes do deploy. |

## Decisões relacionadas

- [ADR-007 — Webhooks assíncronos para mudanças de status](adrs/generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md): outbox, worker, entrega e recuperação.
- [ADR-003 — Autenticação de webhooks por HMAC](adrs/generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md): assinatura, secrets e rotação.
- [ADR-004 — Autenticação JWT e autorização por papéis](adrs/generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md): autenticação, vínculo com cliente e replay.
- [ADR-005 — MySQL como banco relacional compartilhado](adrs/generated/DATA/ADR-005-mysql-compartilhado.md) e [ADR-006 — Prisma como camada de acesso a dados](adrs/generated/DATA/ADR-006-prisma-acesso-a-dados.md): infraestrutura existente a ser reutilizada.

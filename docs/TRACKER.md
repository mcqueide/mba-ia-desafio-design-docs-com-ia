# Tracker de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B precisam de notificações de status sem consultas periódicas. | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Objetivo | Entregas abaixo de 10 segundos são percebidas como tempo real. | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | A feature é somente de webhooks de saída. | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Polling recorrente torna a integração dos clientes lenta e cara. | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Risco de negócio | A Atlas Comercial pode migrar para concorrente sem a feature. | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-001 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint HTTPS, filtros e secret gerada pela plataforma. | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-002 | docs/PRD.md | Requisito Funcional | Listar os webhooks configurados para um cliente. | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-003 | docs/PRD.md | Requisito Funcional | Editar ou remover uma configuração de webhook. | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-004 | docs/PRD.md | Requisito Funcional | Filtrar endpoints pelos status de pedido de interesse durante a inserção na outbox. | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-005 | docs/PRD.md | Requisito Funcional | Persistir evento e entregas elegíveis na transação da mudança de status. | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-006 | docs/PRD.md | Requisito Funcional | Worker separado entrega eventos em polling e aplica retentativas e DLQ. | TRANSCRICAO | [09:06] Diego |
| PRD-FR-007 | docs/PRD.md | Requisito Funcional | Consultar até 100 entregas recentes de um webhook. | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-008 | docs/PRD.md | Requisito Funcional | Rotacionar secret por endpoint, preservando a anterior por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-009 | docs/PRD.md | Requisito Funcional | ADMIN pode solicitar replay manual de entrada da DLQ com auditoria. | TRANSCRICAO | [09:18] Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Worker consulta entregas pendentes a cada 2 segundos. | TRANSCRICAO | [09:09] Diego |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Chamada HTTP possui timeout de 10 segundos. | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Corpo do evento é limitado a 64 KB e não deve ser truncado. | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Eventos usam HMAC-SHA256, secret exclusiva por endpoint e HTTPS. | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Entrega é at-least-once e o consumidor deduplica por X-Event-Id. | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Logs reutilizam Pino e não expõem secrets ou payloads. | TRANSCRICAO | [09:29] Bruno |
| PRD-ARC-01 | docs/PRD.md | Decisão | Usar outbox transacional no MySQL em vez de chamada HTTP síncrona. | TRANSCRICAO | [09:06] Diego |
| PRD-ARC-02 | docs/PRD.md | Trade-off | HTTP síncrono vincularia mudança do pedido à latência e disponibilidade do cliente. | TRANSCRICAO | [09:04] Bruno |
| PRD-ARC-03 | docs/PRD.md | Trade-off | Redis Streams adicionaria infraestrutura excessiva para o contexto atual. | TRANSCRICAO | [09:07] Diego |
| PRD-ARC-04 | docs/PRD.md | Decisão | Worker roda em processo separado e usa instância própria do PrismaClient. | TRANSCRICAO | [09:11] Larissa |
| PRD-ARC-05 | docs/PRD.md | Decisão | O payload é um snapshot criado ao registrar o evento, sem itens do pedido. | TRANSCRICAO | [09:43] Diego |
| PRD-ARC-06 | docs/PRD.md | Compatibilidade | API e worker reutilizam Node.js, Prisma e MySQL existentes. | CODIGO | package.json |
| PRD-SCP-01 | docs/PRD.md | Fora de Escopo | Não publicar evento para a criação inicial do pedido em PENDING. | CODIGO | src/modules/orders/order.service.ts |
| PRD-SCP-02 | docs/PRD.md | Fora de Escopo | Notificação por e-mail fica para fase posterior. | TRANSCRICAO | [09:37] Larissa |
| PRD-SCP-03 | docs/PRD.md | Fora de Escopo | Dashboard visual fica fora desta fase. | TRANSCRICAO | [09:39] Larissa |
| PRD-SCP-04 | docs/PRD.md | Restrição | Rate limiting de saída será observado e decidido posteriormente. | TRANSCRICAO | [09:38] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | A autorização por cliente exige vínculo que não existe no JWT ou schema atual. | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Banco precisa de migração para configurações, outbox, entregas, DLQ e auditoria. | CODIGO | prisma/schema.prisma |
| PRD-RISK-01 | docs/PRD.md | Risco | Um worker único pode acumular backlog e atrasar eventos durante retentativas. | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Duplicatas e replay fora de ordem exigem deduplicação no consumidor. | TRANSCRICAO | [09:24] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Segurança deve revisar geração de secret e HMAC antes do deploy. | TRANSCRICAO | [09:46] Sofia |
| PRD-ACC-01 | docs/PRD.md | Critério de Aceitação | Resposta HTTP 2xx conclui a entrega. | TRANSCRICAO | [09:31] Diego |
| PRD-ACC-02 | docs/PRD.md | Critério de Aceitação | Falhas usam esperas de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas antes da DLQ. | TRANSCRICAO | [09:17] Diego |
| PRD-ACC-03 | docs/PRD.md | Critério de Aceitação | Cada envio inclui X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id. | TRANSCRICAO | [09:44] Diego |
| RFC-001-01 | docs/RFC.md | Contexto | API atual usa Node.js, Express, Prisma e MySQL. | CODIGO | package.json |
| RFC-001-02 | docs/RFC.md | Restrição | A mudança de status atualiza pedido, histórico e estoque na mesma transação. | CODIGO | src/modules/orders/order.service.ts |
| RFC-001-03 | docs/RFC.md | Decisão | Criar eventos somente para endpoints ativos interessados no novo status. | TRANSCRICAO | [09:33] Bruno |
| RFC-001-04 | docs/RFC.md | Decisão | Sem destinatários elegíveis, não registrar evento. | TRANSCRICAO | [09:34] Bruno |
| RFC-001-05 | docs/RFC.md | Decisão | A ordenação é por pedido enquanto houver worker único. | TRANSCRICAO | [09:12] Diego |
| RFC-001-06 | docs/RFC.md | Restrição | Não há garantia de ordenação global em cenário com múltiplos workers. | TRANSCRICAO | [09:13] Diego |
| RFC-001-07 | docs/RFC.md | Decisão | Falha de entrega é armazenada em tabela de DLQ separada. | TRANSCRICAO | [09:18] Diego |
| RFC-001-08 | docs/RFC.md | Requisito de Segurança | Rotação mantém chave antiga válida por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| RFC-001-09 | docs/RFC.md | Restrição | Replay administrativo exige ADMIN e auditoria. | TRANSCRICAO | [09:35] Diego |
| RFC-001-10 | docs/RFC.md | Questão em Aberto | Retenção, escala do worker e rate limiting não foram definidos nesta fase. | TRANSCRICAO | [09:08] Diego |
| FDD-OBJ-01 | docs/FDD.md | Objetivo Técnico | Registro do evento elegível precisa ser atômico com a mudança de status. | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo Técnico | O mesmo event_id deve identificar retentativas e replays. | TRANSCRICAO | [09:24] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | O filtro por status é avaliado no momento da inserção da outbox. | TRANSCRICAO | [09:33] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Eventos pendentes são processados por polling em lote pequeno. | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo Alternativo | Endpoint removido ou desativado não recebe chamada HTTP. | TRANSCRICAO | [09:31] Marcos |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo Alternativo | Falha não 2xx, timeout ou rede dispara retentativa e depois DLQ. | TRANSCRICAO | [09:15] Diego |
| FDD-CTR-01 | docs/FDD.md | Contrato | Cadastro recebe URL e status de interesse; a plataforma gera a secret. | TRANSCRICAO | [09:31] Marcos |
| FDD-CTR-02 | docs/FDD.md | Contrato | Edição e remoção de configurações de webhook são operações previstas. | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-03 | docs/FDD.md | Contrato | Histórico expõe sucesso, falha, payload, resposta e tempo de resposta. | TRANSCRICAO | [09:34] Marcos |
| FDD-CTR-04 | docs/FDD.md | Contrato | Replay é exposto por endpoint administrativo. | TRANSCRICAO | [09:18] Diego |
| FDD-CTR-05 | docs/FDD.md | Contrato | Payload contém identificador, status anterior e novo, pedido, cliente e total, sem itens. | TRANSCRICAO | [09:43] Diego |
| FDD-CTR-06 | docs/FDD.md | Contrato | Webhook usa POST JSON, headers de identificação e assinatura HMAC. | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição | Códigos específicos de webhooks usam o prefixo `WEBHOOK_*`, incluindo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED`. | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Decisão | Todos os códigos de erro do módulo de webhooks devem usar o prefixo `WEBHOOK_`. | TRANSCRICAO | [09:29] Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | O projeto usa Pino como logger compartilhado. | TRANSCRICAO | [09:29] Bruno |
| FDD-COMP-01 | docs/FDD.md | Compatibilidade | Mudança de status já usa Prisma em transação e requer evolução sem quebrar a operação atual. | CODIGO | src/modules/orders/order.service.ts |
| ADR-001 | docs/adrs/ADR-001-api-rest-versionada.md | Decisão | API existente expõe recursos HTTP JSON com versão no caminho. | CODIGO | src/routes/index.ts |
| ADR-002 | docs/adrs/ADR-002-express-framework-http.md | Decisão | Express é o framework HTTP usado pela aplicação. | CODIGO | package.json |
| ADR-003 | docs/adrs/ADR-003-assinatura-hmac-de-webhooks.md | Decisão | HMAC-SHA256 e uma secret independente por endpoint autenticam notificações. | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/ADR-004-autenticacao-jwt-e-papeis.md | Restrição | JWT atual contém papel global ADMIN ou OPERATOR e não vínculo com cliente. | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-005 | docs/adrs/ADR-005-mysql-compartilhado.md | Decisão | MySQL existente é reutilizado para a outbox em vez de Redis Streams. | TRANSCRICAO | [09:07] Diego |
| ADR-006 | docs/adrs/ADR-006-prisma-acesso-a-dados.md | Decisão | Prisma é a camada de acesso a dados e deve ser reutilizado pelo worker. | TRANSCRICAO | [09:11] Bruno |
| ADR-007 | docs/adrs/ADR-007-webhooks-assincronos-de-status.md | Decisão | Eventos são registrados em outbox e entregues por worker assíncrono. | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Envio HTTP síncrono foi descartado porque vincula a alteração do pedido à disponibilidade do cliente. | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams foi descartado por adicionar infraestrutura excessiva. | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco foi descartada porque não notifica diretamente processo externo. | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída permanece para decisão posterior. | TRANSCRICAO | [09:38] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Retenção e arquivamento de entregas antigas permanecem sem política definida. | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | A estratégia de escala sem perder ordenação por pedido permanece futura. | TRANSCRICAO | [09:12] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Monitoramento de backlog, DLQ e proteção de destinos requer revisão de segurança. | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Persistir outbox na transação aumenta a carga no MySQL e pode bloquear a mudança de status. | TRANSCRICAO | [09:40] Bruno |
| RFC-RISK-02 | docs/RFC.md | Risco | Worker único, indisponibilidade e retentativas podem atrasar entregas. | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Destinatários devem deduplicar eventos recebidos mais de uma vez. | TRANSCRICAO | [09:24] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Gestão de secrets e revisão de segurança são necessárias antes do deploy. | TRANSCRICAO | [09:46] Sofia |
| FDD-CTR-07 | docs/FDD.md | Contrato | Rotação gera nova secret e mantém a anterior válida por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| FDD-CTR-10 | docs/FDD.md | Contrato | Notificação contém snapshot enxuto sem itens do pedido. | TRANSCRICAO | [09:43] Diego |
| FDD-CTR-11 | docs/FDD.md | Contrato | Notificação usa os headers X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id. | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-03 | docs/FDD.md | Erro | URL ou filtro de status inválido usa código específico do módulo de webhooks. | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | Endpoint inexistente usa código específico do módulo de webhooks. | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-05 | docs/FDD.md | Erro | Falha de persistência do evento reverte a transação do pedido. | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | Payload acima de 64 KB não é enviado. | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-07 | docs/FDD.md | Erro | Endpoint desativado ou removido gera falha permanente sem envio HTTP. | TRANSCRICAO | [09:33] Bruno |
| FDD-ERR-08 | docs/FDD.md | Erro | Resposta não 2xx, timeout ou erro de rede aplicam retentativas e DLQ. | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | Interrupção do worker preserva a identidade do evento para recuperação. | TRANSCRICAO | [09:11] Larissa |
| FDD-ERR-11 | docs/FDD.md | Erro | Replay já ativo não deve criar um segundo ciclo. | TRANSCRICAO | [09:18] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Histórico com payload e resposta pode crescer e expor dados sensíveis. | TRANSCRICAO | [09:34] Marcos |
| FDD-RISK-02 | docs/FDD.md | Risco | Destinos HTTPS e redirecionamentos exigem revisão de segurança. | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-03 | docs/FDD.md | Risco | Backlog, duplicatas e ordenação exigem recuperação e deduplicação. | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Falhas na gestão de secret podem impedir assinatura de notificações. | TRANSCRICAO | [09:21] Sofia |
| ADR-005-ALT-01 | docs/adrs/ADR-005-mysql-compartilhado.md | Alternativa | Redis Streams foi considerado para a outbox e descartado por custo operacional. | TRANSCRICAO | [09:07] Diego |
| ADR-007-ALT-01 | docs/adrs/ADR-007-webhooks-assincronos-de-status.md | Alternativa | Chamada HTTP síncrona foi rejeitada porque atrasaria a mudança de status. | TRANSCRICAO | [09:04] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-webhooks-assincronos-de-status.md | Alternativa | Redis Streams foi rejeitado por adicionar infraestrutura. | TRANSCRICAO | [09:07] Diego |
| ADR-007-ALT-03 | docs/adrs/ADR-007-webhooks-assincronos-de-status.md | Alternativa | Trigger de banco não resolve a notificação de processo externo. | TRANSCRICAO | [09:09] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Notificações incluem X-Timestamp para o destinatário detectar replays. | TRANSCRICAO | [09:44] Diego |
| PRD-ARC-07 | docs/PRD.md | Decisão | HMAC-SHA256 com secret exclusiva autentica o payload e pode usar timestamp. | TRANSCRICAO | [09:20] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | A publicação proposta se integra à transação de mudança de status existente. | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | O schema Prisma existente deve receber os modelos da feature. | CODIGO | prisma/schema.prisma |
| FDD-INT-03 | docs/FDD.md | Integração | As rotas da feature serão montadas no agregador de rotas existente. | CODIGO | src/routes/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | As dependências da feature serão registradas na composição da aplicação. | CODIGO | src/app.ts |
| FDD-INT-05 | docs/FDD.md | Integração | A autenticação existente precisa evoluir para autorização por vínculo de cliente. | CODIGO | src/middlewares/auth.middleware.ts |

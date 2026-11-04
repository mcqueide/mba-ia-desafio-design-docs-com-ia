# Tracker de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B precisam de notificações de status sem consultas periódicas. | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Objetivo | Entregas abaixo de 10 segundos são percebidas como tempo real. | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | A feature é somente de webhooks de saída. | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Polling recorrente torna a integração dos clientes lenta e cara. | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Risco de negócio | A Atlas Comercial pode migrar para concorrente sem a feature. | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-001 | docs/PRD.md | Requisito Funcional | CRUD autenticado de endpoints HTTPS e filtros por status. | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-002 | docs/PRD.md | Requisito Funcional | Persistir evento e entregas elegíveis na transação da mudança de status. | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-003 | docs/PRD.md | Requisito Funcional | Worker separado entrega eventos em polling e aplica retentativas e DLQ. | TRANSCRICAO | [09:06] Diego |
| PRD-FR-004 | docs/PRD.md | Requisito Funcional | Consultar até 100 entregas recentes de um webhook. | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-005 | docs/PRD.md | Requisito Funcional | Rotacionar secret por endpoint, preservando a anterior por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-006 | docs/PRD.md | Requisito Funcional | ADMIN pode solicitar replay manual de entrada da DLQ com auditoria. | TRANSCRICAO | [09:18] Diego |
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
| FDD-OBS-01 | docs/FDD.md | Observabilidade | O projeto usa Pino como logger compartilhado. | TRANSCRICAO | [09:29] Bruno |
| FDD-COMP-01 | docs/FDD.md | Compatibilidade | Mudança de status já usa Prisma em transação e requer evolução sem quebrar a operação atual. | CODIGO | src/modules/orders/order.service.ts |
| ADR-001 | docs/adrs/generated/API/ADR-001-api-rest-versionada.md | Decisão | API existente expõe recursos HTTP JSON com versão no caminho. | CODIGO | src/routes/index.ts |
| ADR-002 | docs/adrs/generated/API/ADR-002-express-framework-http.md | Decisão | Express é o framework HTTP usado pela aplicação. | CODIGO | package.json |
| ADR-003 | docs/adrs/generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md | Decisão | HMAC-SHA256 e uma secret independente por endpoint autenticam notificações. | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md | Restrição | JWT atual contém papel global ADMIN ou OPERATOR e não vínculo com cliente. | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-005 | docs/adrs/generated/DATA/ADR-005-mysql-compartilhado.md | Decisão | MySQL existente é reutilizado para a outbox em vez de Redis Streams. | TRANSCRICAO | [09:07] Diego |
| ADR-006 | docs/adrs/generated/DATA/ADR-006-prisma-acesso-a-dados.md | Decisão | Prisma é a camada de acesso a dados e deve ser reutilizado pelo worker. | TRANSCRICAO | [09:11] Bruno |
| ADR-007 | docs/adrs/generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md | Decisão | Eventos são registrados em outbox e entregues por worker assíncrono. | TRANSCRICAO | [09:06] Diego |

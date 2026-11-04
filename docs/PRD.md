### PRD: Sistema de Pedidos Webhooks de mudanças de status de pedidos

Versão: 1.0
Data: 2026-11-04
Responsável: Larissa, Tech Lead

---

### Resumo

A feature disponibiliza webhooks de saída para que clientes B2B sejam notificados sobre mudanças de status de seus pedidos sem consultar periodicamente a API. Eventos serão registrados de forma atômica com a mudança de status e entregues de modo assíncrono, com assinatura HMAC, retentativas, DLQ e replay administrativo.

---

### Contexto e problema

Público-alvo
- Usuários autenticados vinculados à Atlas Comercial, MaxDistribuição e Nova Cargo.
- Sistemas consumidores de pedidos desses clientes B2B.

Cenários de uso chave
- Um cliente configura um endpoint HTTPS e os status de pedido que deseja receber.
- O sistema entrega ao endpoint um evento quando um pedido do cliente muda para um status selecionado.
- Um administrador consulta falhas de entrega e solicita replay de um evento na DLQ.

Onde essa feature será implantada
- Sistema de Pedidos existente, composto por API Node.js com Express, Prisma e MySQL, acrescido de um worker Node.js executado em processo separado.

Problemas priorizados
- Clientes consultam repetidamente pedidos para detectar mudanças, o que torna a integração lenta e cara. Prioridade: alta.
- A ausência de notificação em tempo próximo do real ameaça a retenção da Atlas Comercial, que sinalizou possível migração para um concorrente. Prioridade: alta.

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Entregar notificações de mudança de status em condições favoráveis     | Latência entre o registro do evento e a entrega HTTP bem-sucedida | Menor que 10 segundos     |
| Habilitar os clientes solicitantes a integrar webhooks                 | Quantidade de clientes solicitantes habilitados                 | 3 clientes                |

---

### Escopo

Incluso
- CRUD autenticado de endpoints HTTPS por cliente, com filtros por status, histórico de entregas e gerenciamento de secrets.
- Registro transacional de eventos, worker único, retentativas, DLQ persistente, recuperação após reinício e replay administrativo auditado.
- Assinatura HMAC-SHA256, secret exclusiva por endpoint, rotação com janela de 24 horas e deduplicação do destinatário por `X-Event-Id`.
- Autorização por vínculo entre usuário e cliente e papel ADMIN para replay.

Fora de escopo
- Evento para a criação inicial do pedido no status `PENDING`.
- Notificações por e-mail, dashboard visual, limpeza ou arquivamento automático de entregas.
- Rate limiting de saída, múltiplos workers, circuit breaker, garantia de entrega eventual, exatamente uma vez ou ordenação global.
- Assinatura que inclua timestamp e validação de idade da mensagem.

---

### Requisitos funcionais

#### FR-001 Configurar endpoints de webhook
Usuários vinculados ao cliente devem criar, listar, editar e remover endpoints HTTPS ativos e selecionar os status de pedido que desejam receber.

**Fluxo principal**
- Usuário autenticado e vinculado ao cliente cadastra URL HTTPS e uma lista não vazia de status.
- A plataforma gera uma secret exclusiva, persiste a configuração e a retorna somente na criação.
- Usuário lista, atualiza URL, filtros ou estado ativo, ou remove o endpoint.

**Fluxos alternativos e exceções**
- Alterar filtros não cancela entregas já elegíveis; a URL atualizada é usada para entregas pendentes.
- Endpoint desativado ou removido antes do envio gera falha permanente sem chamada HTTP.

**Erros previstos**
- URL não HTTPS ou lista de status vazia retorna erro de validação.
- Usuário sem autenticação ou sem vínculo com o cliente recebe erro de acesso.
- Endpoint inexistente retorna erro de recurso não encontrado.

**Prioridade:** alta

---

#### FR-002 Publicar evento na mudança de status
O sistema deve registrar um snapshot do pedido e uma entrega por endpoint elegível na mesma transação da mudança de status.

**Fluxo principal**
- O serviço valida a transição, atualiza pedido, histórico e estoque quando aplicável.
- Seleciona endpoints ativos do cliente cujo filtro contém o novo status, exceto a criação inicial em `PENDING`.
- Gera um `event_id`, persiste o snapshot sem itens e as entregas independentes por endpoint, e confirma a transação.

**Fluxos alternativos e exceções**
- Sem endpoints elegíveis, a mudança de status é confirmada sem evento.
- Snapshot acima de 65.536 bytes é persistido com falha permanente, sem envio e sem truncamento.

**Erros previstos**
- Falha ao persistir evento, entrega ou falha por tamanho reverte status, histórico e estoque.
- Transição de status inválida permanece rejeitada pelas regras atuais do pedido.

**Prioridade:** alta

---

#### FR-003 Entregar eventos de forma assíncrona
Um worker separado deve enviar eventos pendentes aos endpoints públicos configurados, preservar o `event_id` e aplicar retentativas e DLQ.

**Fluxo principal**
- A cada 2 segundos, o worker busca até 100 entregas elegíveis, processadas sequencialmente.
- Para cada entrega, reserva uma tentativa, assina os bytes do payload com HMAC-SHA256 e envia uma requisição HTTP POST com timeout de 10 segundos.
- Qualquer resposta 2xx conclui a entrega; falhas aplicam esperas de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.

**Fluxos alternativos e exceções**
- Após seis reservas por ciclo, o worker cria entrada persistente na DLQ com payload, motivo e horário.
- Após queda do processo, o worker recupera entregas interrompidas sem alterar o `event_id`; uma reserva já consumida pode resultar em duplicata.
- Enquanto uma entrega de um evento estiver em retentativa, eventos posteriores do mesmo pedido aguardam; replay pode chegar fora da ordem original.

**Erros previstos**
- Resposta não 2xx, timeout ou erro de rede registram a tentativa e agendam nova entrega.
- Destino não público ou redirecionamento que não seja HTTPS para destino público é bloqueado e tratado como falha de entrega.
- Payload acima do limite não é enviado.

**Prioridade:** alta

---

#### FR-004 Consultar histórico de entregas
Usuários vinculados ao cliente devem consultar o histórico das últimas 100 entregas de um endpoint.

**Fluxo principal**
- Usuário autenticado solicita o histórico de um endpoint ao qual está vinculado.
- A API retorna até 100 registros recentes com resultado, payload, resposta quando disponível e duração.

**Fluxos alternativos e exceções**
- Falhas permanentes sem envio HTTP aparecem no histórico sem resposta ou duração.

**Erros previstos**
- Usuário sem vínculo recebe erro de acesso.
- Endpoint inexistente retorna erro de recurso não encontrado.

**Prioridade:** media

---

#### FR-005 Rotacionar e consultar secrets
Usuários vinculados devem rotacionar a secret de um endpoint e consultar identificadores de chave sem expor secrets já reveladas.

**Fluxo principal**
- Usuário solicita rotação e a plataforma cria uma nova secret e um novo `kid`.
- A nova chave e seu metadado são persistidos no banco e no secret manager antes de confirmar sucesso.
- A chave anterior permanece utilizável até a confirmação da migração ou por 24 horas; a secret nova pode ser revelada uma única vez.

**Fluxos alternativos e exceções**
- Se a resposta que revelou a secret for perdida, uma nova rotação cria outro `kid`.
- Após expiração, o `kid` continua consultável, mas a secret não pode ser revelada.

**Erros previstos**
- Falha parcial entre banco e secret manager exige compensação ou reconciliação; se não for possível, a chave é bloqueada.
- Secret já revelada ou expirada não é revelada novamente.

**Prioridade:** alta

---

#### FR-006 Reprocessar eventos da DLQ
Administradores vinculados ao cliente devem solicitar replay de uma entrada da DLQ, preservando sua identidade e registrando auditoria.

**Fluxo principal**
- ADMIN vinculado solicita replay de uma entrada da DLQ.
- O sistema registra usuário, webhook e horário da ação e cria um novo ciclo de até seis reservas com o mesmo `event_id`.
- O worker processa o replay como entrega pendente.

**Fluxos alternativos e exceções**
- Apenas um replay pode permanecer ativo por entrada da DLQ.
- Nova exaustão cria nova entrada na DLQ e preserva o histórico anterior.

**Erros previstos**
- Usuário sem papel ADMIN ou sem vínculo recebe erro de acesso.
- Solicitação para replay já ativo não cria ciclo duplicado.

**Prioridade:** media

---

### Requisitos não funcionais

Performance
- O worker consulta entregas a cada 2 segundos, em lotes de até 100 registros sequenciais.
- Em condições favoráveis, a latência de entrega deve ser menor que 10 segundos; não há garantia sob backlog, timeout ou falhas.
- Cada chamada HTTP deve expirar após 10 segundos e o corpo de envio não pode exceder 65.536 bytes.

Disponibilidade
- A feature deve ter disponibilidade mensal de 99,9% em produção.

Segurança e autorização
- Autenticação é obrigatória para gerenciamento e histórico; autorização exige vínculo com o cliente do recurso.
- Replay exige papel ADMIN vinculado ao cliente e auditoria persistida.
- Cada endpoint usa secret exclusiva em secret manager e HMAC-SHA256 sobre os bytes do payload.
- Aceitar apenas destinos públicos HTTPS e redirecionamentos HTTPS para destinos públicos.

Observabilidade
- Logs Pino estruturados devem registrar `event_id`, `webhook_id`, pedido, cliente, reserva, resultado, status HTTP e duração.
- Logs não podem conter secrets, tokens, payloads ou respostas externas.
- Métricas, tracing, dashboard e alertas automáticos não fazem parte desta fase.

Confiabilidade e integridade de dados
- Mudança de status, histórico, estoque aplicável e persistência de eventos elegíveis devem ocorrer na mesma transação.
- A entrega é at-least-once; destinatários devem deduplicar pelo `X-Event-Id`.
- O worker deve persistir reservas e retomar entregas após reinício sem alterar o `event_id`.

Compatibilidade e portabilidade
- APIs administrativas usam REST JSON versionado em `/api/v1` e JWT já existente.
- API e worker executam em processos Node.js 20 separados, usam Prisma e compartilham a mesma `DATABASE_URL` MySQL.

Compliance
- Histórico de entrega e auditoria de replay devem permanecer acessíveis somente a usuários vinculados ao cliente.
- Payloads e respostas persistidos exigem controle de acesso por poderem conter dados sensíveis.

Acessibilidade no frontend consumidor
- Não há frontend consumidor incluído nesta fase; a feature expõe somente APIs REST para integração.

---

### Arquitetura e abordagem

Abordagem
- Outbox transacional no MySQL: o serviço de pedidos registra eventos e entregas na mesma transação da mudança de status, enquanto um worker único em processo separado faz o envio assíncrono.

Componentes
- Módulo `src/modules/webhooks` com controller, service, repository, routes e schemas Zod.
- Serviço de pedidos integrado por `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- Worker `src/worker.ts` com `PrismaClient` próprio, polling e processador de entregas.
- MySQL como fonte de verdade para configurações, outbox, entregas, tentativas, DLQ, chaves e auditoria.
- Secret manager, com provedor ainda a ser definido pela plataforma.

Integrações
- Endpoints HTTPS públicos configurados pelos clientes recebem notificações HTTP POST assinadas.
- Secret manager armazena secrets HMAC por endpoint.
- Middleware de autenticação evolui para validar vínculo e papel por cliente.

### Decisões e trade-offs

#### Decisão: Usar outbox transacional no MySQL
- **Justificativa:** garante que o evento elegível seja registrado junto com a mudança de status, sem introduzir nova infraestrutura de mensageria.
- **Trade-off:** aumenta a carga e o modelo de dados do MySQL; falha na persistência impede a mudança de status.

#### Decisão: Executar worker único com polling de 2 segundos
- **Justificativa:** desacopla a disponibilidade do destinatário da API e atende à meta de experiência em condições favoráveis.
- **Trade-off:** backlog e retentativas podem atrasar entregas; não há escala horizontal nesta fase.

#### Decisão: Entrega at-least-once com DLQ e replay manual
- **Justificativa:** oferece recuperação de falhas sem a complexidade de coordenação necessária para exatamente uma vez.
- **Trade-off:** destinatários devem deduplicar por `X-Event-Id`, e replay pode ser recebido fora da ordem.

#### Decisão: Assinar somente o payload com HMAC-SHA256
- **Justificativa:** permite validar autenticidade e integridade com uma integração simples para os clientes.
- **Trade-off:** não limita a idade da mensagem; a deduplicação por `event_id` continua obrigatória.

---

### Dependências

#### técnica: Secret manager
A área de Plataforma precisa definir e disponibilizar o provedor, SDK, políticas de acesso e operação do secret manager para persistência das secrets HMAC.

#### técnica: Migração de identidade e autorização
O time responsável pela API precisa criar vínculos usuário-cliente e papéis por cliente, migrar usuários existentes e restringir o registro público de ADMIN antes da ativação.

#### organizacional: Revisão de segurança
Sofia, ou a equipe de Segurança, precisa revisar HMAC, geração e rotação de secrets e as proteções para destinos públicos antes do deploy.

---

### Riscos e mitigação

#### Crescimento de banco e exposição de dados no histórico
- **Probabilidade:** alta
- **Impacto:** payloads e respostas completas podem consumir armazenamento e conter dados sensíveis.
- **Mitigação:**
  - Restringir consultas ao cliente associado.
  - Não registrar corpos, secrets ou tokens nos logs.
  - Acompanhar a capacidade do banco por meios operacionais existentes.
- **Plano de contingência:** realizar intervenção operacional de capacidade e tratamento de dados; não presumir limpeza automática.

#### Backlog, duplicatas e entrega fora de ordem
- **Probabilidade:** media
- **Impacto:** worker único, processamento sequencial e bloqueio por retry podem atrasar eventos e produzir duplicatas ou ordem diferente no replay.
- **Mitigação:**
  - Persistir reservas, histórico e DLQ.
  - Documentar deduplicação por `event_id` para os destinatários.
  - Auditar replays administrativos.
- **Plano de contingência:** corrigir a causa da falha e executar replay manual autorizado; avaliar estratégia multiworker em fase futura.

#### Inconsistência de chave entre banco e secret manager
- **Probabilidade:** media
- **Impacto:** falha parcial pode impedir assinatura ou deixar uma chave inconsistente.
- **Mitigação:**
  - Confirmar sucesso somente após persistência em ambos os sistemas.
  - Compensar ou reconciliar falhas.
  - Bloquear uso da chave afetada quando a compensação falhar.
- **Plano de contingência:** reconciliar manualmente e, se necessário, gerar novo `kid` e secret.

#### Destino público mal configurado ou redirecionado
- **Probabilidade:** media
- **Impacto:** envio a destino indevido pode expor dados de pedidos.
- **Mitigação:**
  - Validar HTTPS e destino público no cadastro e antes do envio.
  - Permitir somente redirecionamentos HTTPS para destinos públicos.
  - Submeter a implementação à revisão de segurança.
- **Plano de contingência:** desativar o endpoint afetado, investigar o envio e corrigir a configuração.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Uma mudança válida de status com endpoints elegíveis persiste o mesmo `event_id` e entregas independentes; sem interessados não há evento e a criação em `PENDING` não publica.
- Falha ao persistir evento, entrega ou falha por tamanho reverte status, estoque e histórico; payload de 65.537 bytes não é enviado.
- O worker consulta pendências a cada 2 segundos, processa até 100 entregas por lote e aplica timeout HTTP de 10 segundos.
- Resposta 2xx conclui a entrega; falhas usam as cinco esperas definidas e no máximo seis reservas por ciclo, com DLQ persistente ao esgotar.
- Cada envio contém `X-Event-Id`, `X-Webhook-Id`, `X-Key-Id` e assinatura HMAC-SHA256 verificável sobre o payload.
- Endpoints e redirecionamentos para destinos não públicos ou não HTTPS são recusados ou bloqueados.
- CRUD, histórico e secrets negam acesso sem vínculo; replay exige ADMIN vinculado e gera registro de auditoria.
- A rotação mantém a chave anterior por até 24 horas e nenhuma secret é revelada mais de uma vez.
- Logs operacionais incluem identificadores e resultado sem expor secrets, tokens, payloads ou respostas externas.
- Testes unitários e de integração passam, e o QA manual executa roteiro aprovado para os fluxos principal, falhas, retentativas, DLQ e replay.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para regras de elegibilidade, assinatura, retentativas, rotação de chave e autorização.
- Testes de integração para mudança de status, persistência transacional da outbox, worker, envio HTTP, DLQ e replay.
- Testes de segurança para autorização por vínculo e papel, proteção de destinos públicos e não exposição de secrets nos logs.

Estratégia de validação
- Executar testes unitários e de integração automatizados e QA manual guiado por roteiro cobrindo cadastro, entrega bem-sucedida, falhas, recuperação após reinício, rotação, histórico e replay.

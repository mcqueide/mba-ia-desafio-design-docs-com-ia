### FDD: Webhooks de mudanças de status de pedidos

Versão: 1.0, proposta para revisão
Data: 2026-11-03
Responsável: Larissa, Tech Lead

---

### 1. Contexto e motivação técnica

Clientes B2B consultam pedidos repetidamente para descobrir mudanças de status. A feature propõe notificações HTTP assíncronas com snapshot do pedido, desacopladas da disponibilidade do destinatário. A [RFC-001](RFC.md) e os ADRs [003](adrs/generated/CROSS/ADR-003-assinatura-hmac-de-webhooks.md), [004](adrs/generated/CROSS/ADR-004-autenticacao-jwt-e-papeis.md) e [007](adrs/generated/ORDERS/ADR-007-webhooks-assincronos-de-status.md) permanecem propostos e devem ser revisados junto com este documento.

Os atores são usuários autenticados vinculados a um cliente, administradores desse cliente, o worker e os sistemas destinatários. A API e o worker compartilham MySQL, mas executam em processos separados. A atomicidade cobre a mudança de status e o registro de eventos elegíveis, não a recepção HTTP.

**Restrições e hipóteses:** apenas um worker ativo; entrega sujeita a falhas permanentes e limite de tentativas; relógios sincronizados quando a proteção de timestamp for usada. Nomes de campos, estados internos, códigos HTTP e exemplos não especificados nas fontes são propostas para revisão, não interfaces já existentes. O provedor e a versão do secret manager ainda não foram escolhidos.

O limite de seis chamadas descrito na RFC-001 e no ADR-007 é refinado neste FDD, conforme esclarecimento posterior: são seis **reservas** por ciclo, que podem resultar em menos de seis chamadas efetivas. A RFC e o ADR precisam ser reconciliados durante a revisão.

---

### 2. Objetivos técnicos

- Registrar, na transação da mudança de status, um evento e uma entrega por endpoint elegível, ou reverter toda a transação se a persistência necessária falhar.
- Enviar assinaturas HMAC-SHA256 verificáveis e preservar o mesmo `event_id` em todas as entregas, retentativas e replays da mudança.
- Limitar cada ciclo a seis reservas de tentativa, incluindo a inicial, com cinco esperas de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- Buscar latência de entrega inferior a 10 segundos em condições favoráveis, sem estabelecer SLO ou garantia para backlog, falhas ou retries.
- Preservar isolamento por cliente e restringir replay a ADMIN do cliente correspondente.

---

### 3. Escopo e exclusões

**Incluído**
- Mudanças posteriores à criação do pedido, configuração de endpoints HTTPS, filtros por status, histórico, secrets, rotação e replay administrativo.
- Outbox transacional, worker único, retentativas, DLQ persistente, recuperação após reinício e auditoria de replay.
- Vínculos usuário-cliente e autorização por papel necessários ao acesso à feature, inclusive regras de provisionamento descritas no ADR-004.

**Excluído**
- Evento para o `PENDING` inicial, notificação por e-mail, dashboard visual, arquivamento ou limpeza automática.
- Rate limiting de saída, múltiplos workers, garantia de ordenação global e circuit breaker.
- Garantia de entrega eventual, exatamente uma vez ou latência inferior a 10 segundos em todas as condições.

---

### 4. Fluxos detalhados e diagramas
**Fluxo principal**
- A mudança de status valida a transição e atualiza pedido, estoque quando necessário e histórico na transação existente. A criação inicial em `PENDING` não publica evento.
- Na mesma transação, selecionar endpoints ativos do cliente cujo filtro contém o novo status. Sem destinatários elegíveis, não registrar evento. O filtro é avaliado somente neste momento.
- Gerar um UUID `event_id` por mudança e serializar em UTF-8 um snapshot JSON sem itens. Persistir o evento e uma entrega independente por endpoint elegível, sem chamada externa na transação. A falha de qualquer persistência necessária reverte status, histórico e estoque.
- O worker consulta entregas elegíveis a cada 2 segundos, em lotes de até 100, sequencialmente. Para um mesmo pedido, processa eventos por timestamp e identificador como desempate. Empates não garantem a ordem real.
- Resolver a configuração atual do endpoint e a chave utilizável, serializar os bytes persistidos sem alterá-los e verificar o limite de 65.536 bytes. Reservar e persistir a tentativa antes da chamada HTTP. Assinar e enviar o corpo com timeout de 10 segundos.
- Qualquer resposta HTTP 2xx conclui a entrega; guardar no histórico a tentativa, payload, resposta completa e tempo de resposta. A identidade do evento é compartilhada, mas estado e retentativas são independentes por endpoint.
- Enquanto qualquer entrega de um evento estiver em retentativa, eventos posteriores do mesmo pedido aguardam em todos os endpoints. Quando todas as entregas que bloqueiam chegarem a sucesso, DLQ ou falha permanente, o processamento posterior pode prosseguir.

**Fluxos alternativos e exceções**
- Se o snapshot exceder 65.536 bytes, persistir integralmente o evento e o registro de falha permanente na transação, sem envio e sem truncamento. A mudança é confirmada apenas se esses registros forem persistidos.
- Se um endpoint estiver desativado ou excluído quando a entrega for processada, registrar falha permanente sem resposta HTTP e sem DLQ; liberar o bloqueio correspondente. Alterações posteriores no filtro não cancelam entregas já elegíveis. Alterações de URL usam a configuração atual.
- Em status não 2xx, erro de rede ou timeout, registrar a tentativa e agendar a próxima pela sequência de esperas. Ao esgotar seis reservas, persistir entrada de DLQ separada com payload, motivo e horário.
- Uma reserva conta mesmo se o processo cair antes de enviar. Ao iniciar, o worker retorna entregas interrompidas a pendente com o mesmo `event_id`, respeitando as reservas já consumidas; caso não reste reserva, encaminha à DLQ. Uma chamada recebida sem confirmação persistida pode ser repetida.
- O replay exige ADMIN do cliente e auditoria com usuário, webhook e horário. Admite no máximo um replay ativo por entrada da DLQ, preserva `event_id` e inicia novo ciclo de até seis reservas. Uma nova exaustão gera outra entrada na DLQ, preservando histórico. O replay pode chegar depois de eventos posteriores.
- No cadastro, gerar secret própria e armazená-la no secret manager; retornar a secret uma vez na resposta de criação, consumindo sua revelação. Na rotação, gerar novo `kid` e secret; a consulta autenticada a revela uma única vez. Se a resposta com a secret se perder, gerar outra secret com novo `kid`.
- Durante a rotação, usar a chave anterior até a confirmação de migração ou o fim da janela de 24 horas, o que ocorrer primeiro; depois usar a nova. A chave anterior expira após 24 horas. O `kid` continua consultável; consultar secret expirada retorna `SIGNATURE_KEY_EXPIRED`.
- Cadastro e rotação só retornam sucesso após persistência no banco e no secret manager. Em falha parcial, retornar erro e executar compensação ou reconciliação. Se a compensação falhar, impedir o uso da chave afetada e exigir intervenção manual.

**Integração com o sistema existente**
- Em [`src/modules/orders/order.service.ts`](../src/modules/orders/order.service.ts), chamar a função proposta `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação de `changeStatus`; não chamar HTTP nesse ponto.
- Evoluir [`prisma/schema.prisma`](../prisma/schema.prisma) com vínculo por cliente e registros de configuração, evento, entregas, tentativas, DLQ, chaves e auditoria. Como proposta, indexar elegibilidade por estado e próximo horário, e ordenação por pedido, timestamp e ID; garantir unicidade do replay ativo por entrada de DLQ de forma transacional. O esquema físico exato requer revisão.
- Criar `src/modules/webhooks/` conforme o padrão de controller, service, repository, routes e schemas; montar rotas em [`src/routes/index.ts`](../src/routes/index.ts) e dependências em [`src/app.ts`](../src/app.ts). Criar `src/worker.ts` com PrismaClient próprio para a mesma `DATABASE_URL`.
- Evoluir [`src/middlewares/auth.middleware.ts`](../src/middlewares/auth.middleware.ts) para consultar vínculo e papel por cliente: o papel global atual não comprova associação. Somente ADMIN cadastra clientes e gerencia vínculos dos clientes associados; o solicitante vira primeiro ADMIN do novo cliente. Somente ADMIN cria ADMIN e não se remove o último ADMIN. O primeiro cliente, usuário e vínculo ADMIN são provisionados por operador autorizado fora do registro público, que não poderá criar ADMIN.

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

Todos os contratos e exemplos desta seção são **propostas** quando não explicitamente definidos na reunião ou nos ADRs. A API usa prefixo `/api/v1` e `Authorization: Bearer <JWT>`; requisições com corpo JSON usam `Content-Type: application/json`. IDs nos exemplos são ilustrativos. A autenticação usa JWT existente e autorização consulta o vínculo com o cliente do recurso, não um `customer_id` presumido no JWT. CRUD, histórico e chaves admitem qualquer usuário associado; replay exige ADMIN associado. Sem vínculo, retornar `403`; sem autenticação válida, `401`. As rotas preservam o envelope `{ "error": { "code": "...", "message": "...", "details": ... } }`, e os códigos compartilhados `UNAUTHORIZED`, `FORBIDDEN` e `VALIDATION_ERROR`. Códigos específicos do módulo usam `WEBHOOK_*`, com a exceção `SIGNATURE_KEY_EXPIRED` prevista no ADR-003.

Não foram definidos limites de taxa, tamanho dos requests da API, timeout dos endpoints administrativos ou valores de amostragem. O teto de 65.536 bytes aplica-se ao corpo de **envio** do evento, não às respostas administrativas. O versionamento das rotas segue `/api/v1`; mudanças incompatíveis exigem revisão de versão e migração de consumidores.

**Cadastro de webhook**
- Tipo: endpoint. Assinatura/Rota: `POST /api/v1/customers/:customerId/webhooks`. Método: POST.
- Campos propostos: `url` HTTPS, `statuses` não vazio com valores de `OrderStatus`. O webhook nasce ativo; `secret` é gerada pela plataforma.
- Semântica de status/headers: `201` criado com `kid` e secret revelada uma vez; `400` validação; `401` ou `403` conforme autenticação e vínculo.

**Exemplo de requisição**
```json
{"url":"https://cliente.example/webhooks","statuses":["SHIPPED","DELIVERED"]}
```

**Exemplo de resposta**
```json
{"id":"wh-1","customerId":"customer-1","url":"https://cliente.example/webhooks","statuses":["SHIPPED","DELIVERED"],"active":true,"kid":"key-1","secret":"secret-exemplo"}
```

**Listagem por cliente**
- Tipo: endpoint. Assinatura/Rota: `GET /api/v1/customers/:customerId/webhooks`. Método: GET.
- Semântica de status/headers: `200` lista configurações sem secrets; `401` ou `403` conforme autenticação e vínculo. Paginação não foi definida.

**Exemplo de requisição**
```http
GET /api/v1/customers/customer-1/webhooks
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"webhooks":[{"id":"wh-1","url":"https://cliente.example/webhooks","statuses":["SHIPPED"],"active":true}]}
```

**Edição de webhook**
- Tipo: endpoint. Assinatura/Rota: `PATCH /api/v1/webhooks/:id`. Método: PATCH.
- Campos propostos são parciais: `url`, `statuses`, `active`. URL deve permanecer HTTPS e uma lista informada não pode ser vazia.
- Semântica de status/headers: `200` configuração atualizada sem secret; `400` validação; `404` recurso inexistente; autenticação e vínculo são obrigatórios.

**Exemplo de requisição**
```json
{"url":"https://cliente.example/novo","statuses":["DELIVERED"],"active":true}
```

**Exemplo de resposta**
```json
{"id":"wh-1","url":"https://cliente.example/novo","statuses":["DELIVERED"],"active":true}
```

**Remoção de webhook**
- Tipo: endpoint. Assinatura/Rota: `DELETE /api/v1/webhooks/:id`. Método: DELETE.
- Semântica de status/headers: `204` removido, sem corpo; `404` inexistente. Preservar os registros históricos necessários e impedir futuras entregas para o cadastro removido.

**Exemplo de requisição**
```http
DELETE /api/v1/webhooks/wh-1
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```http
HTTP/1.1 204 No Content
```

**Histórico de entregas**
- Tipo: endpoint. Assinatura/Rota: `GET /api/v1/webhooks/:id/deliveries`. Método: GET.
- Semântica de status/headers: `200` com no máximo 100 registros mais recentes, sem paginação. Uma entrada por tentativa HTTP e registros adicionais de falha permanente sem envio; resposta HTTP e tempo podem estar ausentes nesses casos. A resposta externa completa é armazenada sem limite adicional e pode conter dados sensíveis.

**Exemplo de requisição**
```http
GET /api/v1/webhooks/wh-1/deliveries
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"deliveries":[{"eventId":"event-1","outcome":"success","payload":{"event_id":"event-1"},"responseStatus":200,"responseBody":"ok","durationMs":120}]}
```

**Rotação de secret**
- Tipo: endpoint. Assinatura/Rota: `POST /api/v1/webhooks/:id/rotate-secret`. Método: POST.
- Semântica de status/headers: `200` após persistência bem-sucedida, com novo `kid` e prazo da chave anterior, sem revelar secret. Falhas de persistência não retornam sucesso.

**Exemplo de requisição**
```http
POST /api/v1/webhooks/wh-1/rotate-secret
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"kid":"key-2","previousKeyExpiresAt":"2026-11-04T12:00:00Z"}
```

**Consulta de identificadores de chave**
- Tipo: endpoint. Assinatura/Rota: `GET /api/v1/webhooks/:id/keys`. Método: GET.
- Semântica de status/headers: `200` com identificadores e metadados, nunca secrets; identificadores permanecem consultáveis após a revelação.

**Exemplo de requisição**
```http
GET /api/v1/webhooks/wh-1/keys
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"keys":[{"kid":"key-2","state":"pending_confirmation"}]}
```

**Revelação única de secret**
- Tipo: endpoint. Assinatura/Rota: `GET /api/v1/webhooks/:id/keys/:kid/secret`. Método: GET.
- Semântica de status/headers: `200` revela uma única vez a secret ainda não revelada; `403` com `SIGNATURE_KEY_EXPIRED` se expirada. O tratamento específico para secret já revelada permanece a definir; nunca revelá-la novamente.

**Exemplo de requisição**
```http
GET /api/v1/webhooks/wh-1/keys/key-2/secret
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"kid":"key-2","secret":"nova-secret-exemplo"}
```

**Confirmação de migração de chave**
- Tipo: endpoint. Assinatura/Rota: `POST /api/v1/webhooks/:id/keys/:kid/confirm`. Método: POST.
- Semântica de status/headers: `200` confirma migração e passa a usar a nova chave nos envios; a chave antiga mantém sua janela de validade até expirar.

**Exemplo de requisição**
```http
POST /api/v1/webhooks/wh-1/keys/key-2/confirm
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"kid":"key-2","confirmed":true}
```

**Replay da DLQ**
- Tipo: endpoint. Assinatura/Rota: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`. Método: POST.
- Semântica de status/headers: `202` para replay aceito; requer ADMIN do cliente e auditoria. Repetir a solicitação com replay ativo não cria segundo ciclo; o status e o corpo dessa repetição ainda devem ser definidos.

**Exemplo de requisição**
```http
POST /api/v1/admin/webhooks/dead-letter/dlq-1/replay
Authorization: Bearer <JWT>
```

**Exemplo de resposta**
```json
{"deadLetterId":"dlq-1","eventId":"event-1","status":"pending"}
```

**Notificação ao destinatário**
- Tipo: endpoint. Assinatura/Rota: URL HTTPS configurada pelo cliente. Método: POST.
- Corpo: `event_id` identifica a mudança, `event_type` é `order.status_changed`, `timestamp` é ISO 8601, `order_id`, `order_number`, `customer_id`, `from_status`, `to_status` e `total_cents` refletem o snapshot. Não incluir itens.
- Headers: `Content-Type: application/json`; `X-Event-Id` contém o UUID estável; `X-Webhook-Id` identifica o cadastro; `X-Key-Id` identifica a chave; `X-Signature` contém HMAC-SHA256 em hexadecimal; `X-Timestamp` contém segundos Unix do envio.
- Sem proteção por timestamp, assinar os bytes exatos do corpo. Com proteção habilitada, assinar bytes UTF-8 de `X-Timestamp`, seguidos de `.` e dos bytes exatos do corpo; o destinatário deve exigir o timestamp e validar tolerância de 30 segundos. Mesmo nessa janela, deduplicar por `event_id`. O modo e sua configuração padrão permanecem a definir.
- Semântica de status/headers: qualquer `2xx` é sucesso; outros status, timeout e erro de rede acionam retry. A chamada usa timeout de 10 segundos e corpo de até 65.536 bytes. Seguir redirecionamentos HTTPS; bloquear redirecionamento para HTTP.

**Exemplo de requisição**
```http
POST /webhooks HTTP/1.1
Content-Type: application/json
X-Event-Id: 11111111-1111-4111-8111-111111111111
X-Webhook-Id: wh-1
X-Key-Id: key-1
X-Timestamp: 1793707200
X-Signature: <hmac-sha256-hex>

{"event_id":"11111111-1111-4111-8111-111111111111","event_type":"order.status_changed","timestamp":"2026-11-03T12:00:00Z","order_id":"order-1","order_number":"1001","from_status":"PROCESSING","to_status":"SHIPPED","customer_id":"customer-1","total_cents":12000}
```

**Exemplo de resposta**
```http
HTTP/1.1 204 No Content
```

---

### 6. Erros, exceções e fallback

| Condição | Tratamento |
| --- | --- |
| Validação, autenticação ou vínculo inválido | Rejeitar requisição com envelope de erro existente; não alterar recursos. |
| Falha ao persistir evento ou falha por tamanho na transação do pedido | Reverter toda a transação. |
| Payload acima de 65.536 bytes | Persistir snapshot e falha permanente; não enviar, não truncar. |
| Endpoint desativado ou excluído | Registrar falha permanente sem HTTP e sem DLQ; liberar bloqueio. |
| Não 2xx, timeout ou erro de rede | Registrar tentativa e aplicar esperas; ao esgotar reservas, enviar à DLQ. |
| Worker interrompido | Recuperar estado pendente, manter identidade e reservas; não supor que o destinatário deixou de receber. |
| Falha parcial entre banco e secret manager | Retornar erro, compensar ou reconciliar; se não for possível compensar, bloquear a chave e exigir intervenção manual. |
| Replay já ativo | Não criar ciclo adicional; resposta específica pendente de definição. |

- Estratégias de resiliência: timeout HTTP de 10 segundos, até cinco retries com espera de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas; sem circuit breaker. O timeout do secret manager e suas políticas de retry permanecem a definir.
- Política de fallback: DLQ após esgotar reservas e replay manual auditado; não há envio alternativo nem e-mail. Erros de persistência não são convertidos em sucesso.
- Invariantes: no máximo seis reservas por ciclo; uma reserva pode não resultar em chamada; `event_id` permanece estável; não enviar snapshot acima do limite; não revelar uma secret mais de uma vez; não usar chave com persistência inconsistente.

---

### 7. Observabilidade

**Métricas**

- Nesta fase, não há implementação de métricas. Permanecem pendentes instrumentação e metas para backlog, idade do evento, latência, tentativas, falhas, DLQ e replay.

**Logs**

- Usar Pino estruturado, com evento operacional, timestamp, `event_id`, `webhook_id`, `order_id`, identificador do cliente, número da reserva, resultado, status HTTP quando disponível e duração. Replay inclui usuário responsável e ID da DLQ.
- Não registrar secret, token, corpo do evento ou resposta externa nos logs; expandir a redação de [`src/shared/logger/index.ts`](../src/shared/logger/index.ts) para os novos campos sensíveis. O histórico persistido contém payload e resposta completa, cujo acesso exige vínculo; isso não implica que a resposta externa esteja livre de dados sensíveis.

**Tracing**

- Não será implementado nesta fase. Spans de publicação, reserva, envio e replay, propagação e amostragem ficam pendentes de definição.

**Dashboards e alertas**

- Não há dashboard nem alertas implementados nesta fase. Monitoramento operacional de backlog, DLQ e falhas por logs requer definição posterior; não presumir alertas automáticos.

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| Node.js | 20 | Versão exigida em [`package.json`](../package.json); API e worker em processos separados. |
| Express | Não definida | Projeto usa 4.21.1; rotas seguem prefixo e middlewares existentes. |
| Prisma | Não definida | Projeto usa 5.22.0; worker usa cliente próprio e o mesmo banco. |
| MySQL | Não definida | Compose usa MySQL 8; outbox, histórico e DLQ exigem migração. |
| Zod e Pino | Não definida | Projeto usa Zod 3.23.8 e Pino 9.5.0; reutilizar validação e logs existentes. |
| Secret manager | Não definida | Provedor, SDK, timeout e política de acesso ainda não definidos; necessário para persistir secrets. |

**Garantias de compatibilidade**

- Os endpoints atuais de pedido mantêm seus contratos, mas a mudança de status passa a falhar se a persistência transacional dos registros elegíveis falhar.
- Autorização por vínculo, restrição de registro público e cadastro de clientes alteram comportamentos existentes; planejar migração de usuários, vínculos e consumidores antes da ativação. O JWT atual e o `requireRole` global, isoladamente, não cumprem a regra por cliente.
- Destinatários devem validar a assinatura conforme o modo acordado e tolerar duplicatas e replay fora de ordem. A resposta HTTP externa completa é persistida; planejar capacidade e acesso apropriado.

---

### 9. Critérios de aceite técnicos

- Uma mudança válida com múltiplos endpoints elegíveis registra o mesmo `event_id` e entregas independentes; sem interessados não há evento; criação em `PENDING` não publica.
- Falha de persistência da outbox ou da falha por tamanho reverte status, estoque e histórico; snapshot de 65.536 bytes pode ser enviado e um de 65.537 bytes é persistido com falha permanente, sem HTTP.
- O worker consulta a cada 2 segundos, processa no máximo 100 entregas por lote sequencialmente e usa timeout de 10 segundos. O objetivo inferior a 10 segundos não é critério de SLO.
- Toda resposta 2xx conclui entrega; falhas aplicam as cinco esperas e no máximo seis reservas. Queda após reserva consome tentativa; reinício recupera sem trocar `event_id` e encaminha à DLQ se o ciclo estiver esgotado.
- Retry de um endpoint bloqueia eventos posteriores do mesmo pedido para todos os endpoints; sucesso, DLQ ou falha permanente liberam o bloqueio correspondente. Empates de timestamp não prometem ordem real.
- Desativação ou exclusão produz falha permanente sem DLQ nem resposta HTTP; edição de filtro não cancela entregas já elegíveis e edição de URL usa a configuração atual.
- Replay exige ADMIN vinculado, gera auditoria, impede mais de um replay ativo por entrada e preserva identidade e histórico; nova exaustão cria nova entrada DLQ.
- CRUD, histórico e secrets negam acesso sem vínculo. Histórico retorna no máximo 100 registros recentes, incluindo falhas sem envio, e armazena a resposta externa completa.
- HMAC é verificável sobre os bytes definidos em ambos os modos; a rotação usa a chave anterior até confirmação ou 24 horas. Cadastro e cada `kid` revelam a secret uma única vez; chave inconsistente não é usada.
- Pino registra eventos operacionais e auditoria sem secrets, tokens, payloads ou respostas externas; métricas, tracing e alertas automáticos não são declarados como implementados.

---

### 10. Riscos e mitigação

### Crescimento de banco e exposição no histórico

- **Probabilidade:** alta.
- **Impacto:** payloads e respostas completas sem limite adicional ou retenção automática podem consumir armazenamento e conter dados sensíveis.
- **Mitigação:**
    - Restringir consultas ao cliente associado e não registrar corpos ou secrets nos logs.
    - Acompanhar capacidade do banco por meios operacionais existentes; definir retenção posteriormente.
- **Plano de contingência:** intervenção operacional para capacidade e tratamento de dados, sem presumir limpeza automática.

### Destinos internos e redirecionamentos

- **Probabilidade:** média, hipótese para revisão.
- **Impacto:** SSRF ou vazamento de dados, pois qualquer URL HTTPS, inclusive interna, e redirecionamentos HTTPS são permitidos.
- **Mitigação:**
    - Exigir HTTPS e bloquear redirecionamento para HTTP.
    - Submeter a implementação à revisão de segurança antes do deploy. Não há bloqueio de rede interna nesta fase.
- **Plano de contingência:** desativar o endpoint afetado e investigar; restrições adicionais de destino exigem decisão posterior.

### Backlog, duplicatas e ordenação

- **Probabilidade:** média, hipótese para revisão.
- **Impacto:** worker único, processamento sequencial e bloqueio por retry podem atrasar eventos; recuperação e replay podem gerar duplicatas ou chegada fora de ordem.
- **Mitigação:**
    - Persistir reservas, histórico e DLQ; documentar deduplicação por `event_id` para destinatários.
    - Acompanhar o processamento e auditar replay administrativo.
- **Plano de contingência:** corrigir a causa da falha e executar replay manual autorizado; estratégia multiworker fica para revisão futura.

### Inconsistência de chave entre sistemas

- **Probabilidade:** média, hipótese para revisão.
- **Impacto:** falha parcial entre banco e secret manager pode impedir assinatura ou deixar uma chave inconsistente.
- **Mitigação:**
    - Só confirmar sucesso após ambos persistirem; compensar ou reconciliar falhas.
    - Bloquear uso da chave afetada quando a compensação falhar, sem registrar secrets.
- **Plano de contingência:** intervenção manual para reconciliar e, se necessário, gerar novo `kid` e secret.

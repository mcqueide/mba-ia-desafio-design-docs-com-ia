# Potencial ADR: API REST HTTP versionada

**Módulo:** API  
**Categoria:** arquitetura e protocolo de API — Step 0, categoria 4  
**Prioridade:** Must document (135/150)  
**Identificado em:** 2026-11-02  
**Estado da evidência:** implementação existente; extensões de webhook são propostas para o futuro.

## O que foi identificado

O projeto se descreve como API REST e expõe rotas HTTP de recursos sob `/api/v1`, com corpos JSON, métodos como GET, POST, PATCH e DELETE e um health check separado em `/health`. Rotas de autenticação também incluem ações como login; a designação REST descreve o estilo adotado, sem provar conformidade integral com todas as restrições REST.

O commit `7ef4317` (`init repository`, 2026-06-24) introduziu `package.json`, `src/app.ts` e `src/routes/index.ts`. O histórico disponível de cada arquivo contém apenas esse commit, sem justificativa para o protocolo, o versionamento ou eventuais alternativas.

## Por que pode merecer um ADR

- **Impacto:** o contrato HTTP atende os cinco domínios e consumidores externos; alterar paths, métodos ou representações pode exigir adaptações nesses consumidores.
- **Trade-off observável:** o prefixo incorpora a versão ao caminho e delimita o contrato exposto, mas nenhuma política de compatibilidade ou migração entre versões foi encontrada nos arquivos analisados.
- **Pontuação provisória:** base **75** pela arquitetura de API; escopo **25** por abranger todos os domínios e sua integração com clientes; custo **15**, estimativa de 2–8 semanas para migrar contratos e consumidores, não medição; conhecimento **20**, necessário na maioria das funcionalidades expostas. Total **135/150**.
- **3 E's:** é estrutural e a motivação importa a implementadores e consumidores; a estabilidade é pressuposta para o contrato, não demonstrada por histórico de várias versões.

## Evidências

- [`package.json:2–4`](../../../../../package.json#L2-L4): descrição do produto como REST API.
- [`src/app.ts:55–73`](../../../../../src/app.ts#L55-L73): parser JSON, health check fora do prefixo, montagem de `/api/v1` e tratamento de rotas inexistentes.
- [`src/routes/index.ts:21–30`](../../../../../src/routes/index.ts#L21-L30): recursos de autenticação, usuários, clientes, produtos e pedidos.
- [`src/modules/orders/order.routes.ts:12–24`](../../../../../src/modules/orders/order.routes.ts#L12-L24): métodos de consulta, criação, alteração de status e exclusão.
- [`src/modules/customers/customer.routes.ts:12–24`](../../../../../src/modules/customers/customer.routes.ts#L12-L24) e [`customer.controller.ts`](../../../../../src/modules/customers/customer.controller.ts): exemplo do contrato de clientes, incluindo autenticação, validação e respostas HTTP; a listagem usa paginação.
- [`tests/orders.test.ts:66–74`](../../../../../tests/orders.test.ts#L66-L74): testes existentes exercitam criação e alteração de status por HTTP; não foram executados nesta análise.

```ts
// src/modules/orders/order.routes.ts
router.get('/:id', validate({ params: orderIdParamSchema }), controller.getById);
router.post('/', validate({ body: createOrderSchema }), controller.create);
router.patch('/:id/status', validate({ params: orderIdParamSchema, body: updateOrderStatusSchema }), controller.changeStatus);
```

## Questões para um ADR formal

- Por que REST sobre HTTP e versionamento no caminho foram escolhidos? Quais alternativas foram realmente consideradas?
- Qual política governará compatibilidade, descontinuação e coexistência de versões?
- Como documentar os contratos de entrada e saída sem afirmar que o webhook proposto já está implementado?

## Sobreposições e limites

[Express como framework HTTP](express-como-framework-http.md) é a escolha de implementação, distinta do contrato. A reunião propõe CRUD autenticado e replay administrativo para webhooks ([transcrição:184–216](../../../../../TRANSCRICAO.md#L184-L216)); essas rotas não existem atualmente. A integração de saída, seu contrato de entrega e operação já estão cobertos pelo [candidato de ORDERS](../ORDERS/webhooks-assincronos-de-status-de-pedidos.md), sem duplicá-los aqui. A autenticação existente e proposta pertence ao [candidato de CROSS](../CROSS/autenticacao-jwt-e-autorizacao-por-papeis.md). Não foram localizados ADRs formais em `docs/adrs/`.

A análise de CUSTOMERS identificou a mesma arquitetura HTTP para o recurso de clientes. Suas evidências foram incorporadas a este candidato transversal, sem propor um ADR por recurso; as regras de autorização por cliente permanecem em aberto.

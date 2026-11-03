# Potencial ADR: Autenticação JWT e autorização por papéis nas rotas

**Módulo:** CROSS  
**Categoria:** Segurança — autenticação  
**Prioridade:** Must document (125/150)  
**Data de identificação:** 2026-11-02

## O que foi identificado

**Implementado:** o login verifica a senha e emite JWT Bearer contendo identificador do usuário, email e papel. O middleware compartilhado verifica o token com `JWT_SECRET`, popula `req.user` e disponibiliza `requireRole` para restrição por papel. Pedidos, produtos e clientes exigem autenticação; a consulta de usuário por ID também exige `ADMIN`. Não é evidência de autorização por cliente: o token não contém `customerId` e o middleware não faz essa associação.

**Decidido na reunião, ainda não implementado:** reutilizar a autenticação para o CRUD de configuração dos webhooks, com qualquer papel autenticado por enquanto; exigir `ADMIN` via `requireRole` e registrar o autor do replay da dead letter. Após uma proposta inicial de obter o cliente do JWT, a reunião corrigiu que o `customer_id` viria do corpo ou da rota. A transcrição não resolve como autorizar acesso aos dados de um cliente específico.

## Por que merece um ADR

- **Impacto:** é uma fronteira de identidade e autorização compartilhada por múltiplos domínios e clientes da API; o replay proposto acrescenta uma operação administrativa.
- **Trade-offs e questões:** a validação do papel é reutilizável, mas não substitui uma política de acesso por cliente. A transcrição não apresenta alternativas para a autenticação existente nem define o controle por cliente.
- **Pontuação estimada:** base **70** (Step 0: autenticação essencial à API); escopo **20** (infraestrutura comum a AUTH, USERS, CUSTOMERS, PRODUCTS e ORDERS); custo de mudança **15** (estimativa de 2–8 semanas para adaptar emissores, verificadores e consumidores, não medição); conhecimento **20** (relevante para a maioria das funcionalidades protegidas). Total **125**. A estabilidade futura é uma hipótese; o padrão aparece no commit inicial e a reunião prevê reutilizá-lo.

## Evidências

- [`src/modules/auth/auth.service.ts:31–49`](../../../../../src/modules/auth/auth.service.ts#L31-L49): login e emissão do JWT.
- [`src/middlewares/auth.middleware.ts:6–10,19–60`](../../../../../src/middlewares/auth.middleware.ts#L6-L60): claims, verificação e checagem de papéis.
- [`src/modules/orders/order.routes.ts:12–24`](../../../../../src/modules/orders/order.routes.ts#L12-L24), [`src/modules/users/user.routes.ts:12–17`](../../../../../src/modules/users/user.routes.ts#L12-L17), [`src/modules/customers/customer.routes.ts:12–14`](../../../../../src/modules/customers/customer.routes.ts#L12-L14) e [`src/modules/products/product.routes.ts:12–14`](../../../../../src/modules/products/product.routes.ts#L12-L14): aplicação nas rotas.
- [`TRANSCRICAO.md:184–190,204–216`](../../../../../TRANSCRICAO.md#L184-L216): origem de `customer_id`, papéis e auditoria planejados.
- [`tests/auth.test.ts:41–52,65–83`](../../../../../tests/auth.test.ts#L41-L83): testes de login e acesso autenticado; não demonstram isolamento por cliente nem webhooks.

```ts
// src/middlewares/auth.middleware.ts:40–42
const payload = jwt.verify(token, env.JWT_SECRET) as JwtPayload;
req.user = { id: payload.sub, email: payload.email, role: payload.role };
```

O commit `7ef4317` (`init repository`, 2026-06-24) introduziu o middleware, o serviço de autenticação e a transcrição. O histórico desses arquivos consultado não revela motivação ou evolução posterior; a data do commit não identifica a data da reunião.

## Perguntas para um ADR formal

- Qual era a motivação da estratégia JWT e quais alternativas foram avaliadas?
- Como relacionar usuário e cliente e autorizar CRUD, histórico e rotação por cliente?
- Como registrar a autoria do replay e como serão aplicados os controles às novas rotas?

## Sobreposições

Há sobreposição com **AUTH** (emissão e ciclo do token) e **USERS** (papéis). Consolidar em uma decisão de autenticação/autorização, preservando a aplicação transversal em CROSS, em vez de duplicar. A autenticação dos destinatários de webhooks é uma fronteira distinta: [assinatura HMAC](assinatura-hmac-de-webhooks.md).

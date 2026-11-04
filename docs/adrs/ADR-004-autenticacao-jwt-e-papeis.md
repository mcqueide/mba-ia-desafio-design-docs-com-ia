# ADR-004: Autenticação JWT e autorização por papéis

**Status:** Proposto
**Data:** Desconhecida
**Related to:**
- [ADR-007: Webhooks assíncronos para mudanças de status](ADR-007-webhooks-assincronos-de-status.md)
- [ADR-003: Autenticação de webhooks por HMAC](ADR-003-assinatura-hmac-de-webhooks.md)
- [ADR-001: API HTTP orientada a recursos e versionada](ADR-001-api-rest-versionada.md)

## Contexto e problema

A API existente autentica usuários por token JWT e disponibiliza restrição por papéis globais. Os domínios protegidos compartilham esse mecanismo; a identidade no token não inclui vínculo com cliente e o schema atual não registra essa associação.

A reunião propôs reutilizar a autenticação no CRUD de webhooks e restringir o replay administrativo ao papel ADMIN. A feature ainda não está implementada.

## Fatores de decisão

- Aplicar autenticação consistente nas rotas protegidas.
- Permitir restrições por papel nas operações administrativas.
- Reutilizar os mecanismos já presentes na aplicação.
- Não confundir autenticação de usuário com autorização de acesso a um cliente.

## Opções consideradas

1. Reutilizar JWT para autenticação e autorização por papéis.

Não foram avaliadas outras alternativas para a feature: o JWT foi escolhido por já ser utilizado pela API. A motivação da escolha original da API permanece desconhecida.

## Resultado da decisão

O código existente usa JWT e papéis; a feature reutilizará esse padrão. CRUD, histórico e consulta de secrets poderão ser acessados por qualquer usuário autenticado, enquanto o replay exige ADMIN.

O acesso será restrito aos clientes associados ao usuário. O papel será definido por vínculo: uma pessoa poderá ser ADMIN de um cliente e OPERATOR de outro. O replay exige papel ADMIN no cliente correspondente.

Somente um ADMIN pode cadastrar novos clientes e gerenciar usuários e vínculos dos clientes aos quais está associado. No cadastro de um cliente, o ADMIN solicitante será associado como seu primeiro ADMIN. Somente ADMIN poderá criar outros usuários ADMIN; não será permitida a remoção ou desassociação do último ADMIN de um cliente.

O primeiro cliente, usuário e vínculo ADMIN serão provisionados por um operador autorizado, fora do registro público. Essa regra complementa a decisão de restringir o registro público: ele não poderá criar usuários ADMIN.

## Prós e contras das opções

### JWT e papéis

- **Pró:** o mecanismo já é compartilhado por várias rotas.
- **Pró:** papéis permitem limitar operações administrativas.
- **Contra:** o papel global no token atual não representa os papéis por vínculo; a autorização deverá considerar a associação com o cliente.
- **Contra:** mudanças de identidade e autorização afetam consumidores da API.

### Alternativas

Nenhuma alternativa foi avaliada para a reutilização na feature; não há evidência das alternativas originais da API.

## Consequências

A autorização por cliente não pode ser inferida do JWT atual e depende de vínculos e papéis por cliente ainda não implementados. O replay deve registrar a identidade do usuário, o ID do webhook e o horário.

O registro público existente aceita ADMIN e o cadastro atual de clientes exige apenas autenticação. Ambos os comportamentos precisarão ser restringidos para atender às regras confirmadas; elas ainda não estão implementadas. O provisionamento inicial exige um canal controlado fora do registro público.

## Referências

- [Middleware de autenticação](../../src/middlewares/auth.middleware.ts)
- [Serviço de autenticação](../../src/modules/auth/auth.service.ts)
- [Rotas de autenticação](../../src/modules/auth/auth.routes.ts)
- [Transcrição: autorização e replay](../../TRANSCRICAO.md#L184-L216)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: reutilização de JWT, papéis por vínculo, provisionamento inicial e auditoria.

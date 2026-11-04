# ADR-001: API HTTP orientada a recursos e versionada

**Status:** Proposto
**Data:** Desconhecida
**Related to:**
- [ADR-002: Express como framework HTTP](ADR-002-express-framework-http.md)
- [ADR-004: Autenticação JWT e autorização por papéis](ADR-004-autenticacao-jwt-e-papeis.md)

## Contexto e problema

A aplicação existente se descreve como API REST e expõe recursos HTTP com JSON e versão no caminho. Há também operações de autenticação; a descrição do estilo não comprova conformidade com todas as restrições REST.

O histórico disponível não registra as alternativas consideradas. Conforme esclarecido posteriormente, o estilo orientado a recursos foi escolhido para seguir práticas REST, e a versão no caminho para permitir evolução do modelo sem exigir que todos os clientes migrem simultaneamente.

## Fatores de decisão

- Oferecer um contrato HTTP comum aos domínios.
- Permitir evolução do contrato identificando uma versão.
- Manter consistência entre integrações e clientes da API.
- Evitar presumir uma política de compatibilidade não documentada.

## Opções consideradas

1. API HTTP orientada a recursos com versão no caminho.

Não há registro dos protocolos ou estratégias de versionamento alternativos avaliados.

## Resultado da decisão

O código demonstra o estilo HTTP e a versão no caminho. A intenção é permitir a evolução do modelo sem quebrar integrações de clientes que permanecem em versões anteriores. A presença de uma versão no caminho, por si só, não garante compatibilidade.

## Prós e contras das opções

### API HTTP versionada

- **Pró:** os domínios compartilham uma interface e convenções HTTP.
- **Pró:** a versão no caminho identifica o contrato exposto.
- **Contra:** mudanças de rotas e representações podem exigir adaptações nos consumidores.
- **Contra:** não há política de migração entre versões identificada no código analisado.

### Alternativas

Não há registro das opções efetivamente avaliadas e de seus trade-offs.

## Consequências

O estilo organiza a integração dos consumidores com os domínios atuais. Ainda não há política de compatibilidade, coexistência ou descontinuação de versões. As rotas de configuração e replay de webhooks descritas na reunião são propostas e não existem na API atual.

A autenticação das chamadas de entrada e a entrega de notificações de saída são preocupações distintas deste contrato geral.

## Referências

- [Composição da aplicação](../../src/app.ts)
- [Rotas de domínio](../../src/routes/index.ts)
- [Rotas de clientes](../../src/modules/customers/customer.routes.ts)
- [Transcrição: CRUD e replay propostos](../../TRANSCRICAO.md#L184-L216)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: motivações, ausência de registro das alternativas e ausência de política de versões.

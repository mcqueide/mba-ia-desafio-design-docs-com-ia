# ADR-002: Express como framework HTTP

**Status:** Proposto
**Data:** Desconhecida
**Related to:** [ADR-001: API HTTP orientada a recursos e versionada](ADR-001-api-rest-versionada.md)

## Contexto e problema

O processo HTTP existente usa Express para compor a aplicação, os routers e os middlewares que atendem os domínios. As dependências são montadas explicitamente na aplicação.

O histórico disponível mostra a presença do framework desde o primeiro commit, mas não registra a data real ou alternativas da decisão. Conforme esclarecido posteriormente, a equipe escolheu Express por ser amplamente usado e reconhecido no mercado.

## Fatores de decisão

- Disponibilizar uma infraestrutura HTTP comum aos domínios.
- Compor rotas e middlewares compartilhados.
- Manter a integração com os mecanismos atuais de validação, autenticação e erros.
- Preservar o contrato público independentemente da biblioteca de implementação.

## Opções consideradas

1. Express como framework HTTP.

Não há registro dos frameworks ou abordagens originalmente avaliados.

## Resultado da decisão

Express é a implementação observada no código; a escolha levou em conta sua ampla adoção. Este registro retrospectivo não presume aprovação formal. Não há decisão registrada sobre compromissos específicos de manutenção para composição, middlewares ou testes.

## Prós e contras das opções

### Express

- **Pró:** fornece uma estrutura comum para routers e middlewares.
- **Pró:** integra os domínios à mesma aplicação HTTP.
- **Contra:** rotas e middlewares dependem das interfaces do framework.
- **Contra:** uma substituição exigiria adaptar a superfície HTTP existente.

### Alternativas

Não há registro suficiente para comparar as alternativas originais.

## Consequências

A escolha afeta a composição da API e seus componentes HTTP. Ela é distinta do estilo do contrato público e não implica que o worker de webhooks proposto exista ou execute no mesmo processo.

## Referências

- [Dependências e scripts](../../../../package.json)
- [Composição da aplicação](../../../../src/app.ts)
- [Inicialização HTTP](../../../../src/server.ts)
- [Rotas de domínio](../../../../src/routes/index.ts)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: motivação, ausência de registro das alternativas e ausência de decisão sobre manutenção.

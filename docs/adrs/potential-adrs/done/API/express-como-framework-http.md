# Potencial ADR: Express como framework HTTP

**Módulo:** API  
**Categoria:** framework principal — Step 0, categoria 2  
**Prioridade:** Must document (130/150)  
**Identificado em:** 2026-11-02  
**Estado da evidência:** implementação existente.

## O que foi identificado

O processo HTTP constrói uma aplicação Express 4.21.1, instala middleware de JSON, logging e erros e monta os routers dos cinco domínios. `server.ts` inicializa essa aplicação e abre a porta configurada. A composição manual de repositórios, serviços e controllers ocorre em `buildControllers`; não há evidência de um contêiner de injeção de dependências.

Os arquivos `src/app.ts`, `src/server.ts`, `src/routes/index.ts` e `package.json` foram introduzidos no commit `7ef4317` (`init repository`, 2026-06-24); o histórico disponível de cada arquivo contém somente esse commit. Isso demonstra presença desde o primeiro commit, não a data real da decisão, sua motivação ou estabilidade futura.

## Por que pode merecer um ADR

- **Impacto:** o framework estrutura o ponto de entrada, os routers e os middlewares que atendem todos os domínios.
- **Trade-off observável:** trocar o framework exigiria adaptar a composição HTTP e as interfaces Express usadas por rotas e middlewares; a extensão dessa migração depende da alternativa escolhida.
- **Pontuação provisória:** base **75** pelo framework principal; escopo **20** por infraestrutura comum aos cinco domínios; custo **15**, estimativa de 2–8 semanas para adaptar a superfície HTTP, não medição; conhecimento **20**, relevante para a maioria das funcionalidades HTTP. Total **130/150**.
- **3 E's:** é estrutural e sua justificativa interessa a quem mantém a API; a permanência futura é uma hipótese razoável dada a integração transversal, não uma garantia comprovada pelo histórico.

## Evidências

- [`package.json:7–13,25–28`](../../../../../package.json#L7-L28): Node.js >=20, scripts de execução e Express 4.21.1.
- [`src/app.ts:26–75`](../../../../../src/app.ts#L26-L75): composição manual, middlewares, health check, router e tratamento centralizado de erros.
- [`src/server.ts:6–21`](../../../../../src/server.ts#L6-L21): criação da aplicação, escuta e tratamento de sinais de encerramento.
- [`src/routes/index.ts:21–30`](../../../../../src/routes/index.ts#L21-L30): router compartilhado dos cinco domínios.

```ts
// src/app.ts
const app = express();
app.use(express.json({ limit: '1mb' }));
app.use(requestLogger);
app.use('/api/v1', buildApiRouter(controllers));
app.use(errorMiddleware);
```

## Questões para um ADR formal

- Por que Express foi escolhido e quais alternativas foram realmente avaliadas? O histórico consultado não informa.
- Quais compromissos de composição, middlewares e testes precisam ser mantidos numa eventual migração?
- Como o ciclo de vida HTTP se relacionará com o worker separado proposto na reunião, sem presumir que ele já exista?

## Sobreposições e limites

O candidato [API REST versionada](api-rest-versionada.md) trata do contrato externo, não da biblioteca de implementação. A autenticação e autorização estão no [candidato transversal de JWT](../CROSS/autenticacao-jwt-e-autorizacao-por-papeis.md); Prisma está no [candidato de DATA](../DATA/prisma-como-camada-de-acesso-a-dados.md). O worker e os webhooks propostos estão consolidados no [candidato de ORDERS](../ORDERS/webhooks-assincronos-de-status-de-pedidos.md). A composição manual isolada não recebeu candidato próprio: sem base Step 0, escopo **20** + custo hipotético **10** + conhecimento **15** = **45/150**, abaixo do limiar. Não foram localizados ADRs formais em `docs/adrs/`.

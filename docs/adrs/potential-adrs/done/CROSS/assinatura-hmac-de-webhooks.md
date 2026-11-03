# Potencial ADR: Autenticação das notificações de webhook por HMAC

**Módulo:** CROSS  
**Categoria:** Segurança — autenticação de integração  
**Prioridade:** Must document (105/150)  
**Data de identificação:** 2026-11-02

## O que foi identificado

**Decidido na reunião, não implementado:** assinar o corpo das notificações de saída com HMAC-SHA256, enviando a assinatura em `X-Signature`, e usar uma secret própria para cada endpoint. A secret deve poder ser rotacionada, mantendo a anterior válida por 24 horas em paralelo. A intenção expressa é permitir que o destinatário verifique origem e integridade, além de limitar o impacto do vazamento de uma secret.

A reunião também prevê URL HTTPS, criação de secret pelo sistema, `X-Timestamp` para possível detecção de replay pelo cliente e revisão de segurança antes do deploy. Não especifica serialização canônica, formato exato da assinatura, proteção das secrets armazenadas, nem como a sobreposição de chaves será aplicada à assinatura ou à verificação. A presença de timestamp não é, por si só, uma garantia de proteção contra replay.

**Código atual:** o mecanismo existente de autenticação é JWT para requisições recebidas; não há módulo de webhook ou implementação de assinatura HMAC. O logger Pino redige alguns campos específicos, mas a lista atual não evidencia redação de secrets de webhook. Não presumir que o middleware HTTP de erro se aplica automaticamente ao worker separado.

## Por que merece um ADR

- **Impacto:** define um contrato de confiança entre a aplicação e cada consumidor externo da feature B2B; mudar algoritmo ou chaves exige coordenação com integrações.
- **Trade-offs:** secrets por endpoint e rotação exigem gestão de chaves e migração pelos destinatários; a reunião não apresenta uma alternativa concreta à assinatura HMAC.
- **Pontuação estimada:** base **70** (Step 0, autenticação essencial à integração externa de webhooks, infraestrutura específica do domínio da feature; não é categoria universal); escopo **10** (módulo proposto e consumidores externos); custo de mudança **10** (estimativa de 1–2 semanas para adaptar contrato e integrações, não medição); conhecimento **15** (crítico para operação e integração de webhooks). Total **105**. Estabilidade é uma hipótese baseada no contrato proposto, não um histórico de execução.

## Evidências

- [`TRANSCRICAO.md:118–138`](../../../../../TRANSCRICAO.md#L118-L138): objetivo, HMAC-SHA256, secret por endpoint, rotação e HTTPS.
- [`TRANSCRICAO.md:184–190`](../../../../../TRANSCRICAO.md#L184-L190): geração e devolução da secret na criação e contexto de autenticação da API.
- [`TRANSCRICAO.md:260–266,272–288`](../../../../../TRANSCRICAO.md#L260-L288): headers, revisão de segurança e confirmação das decisões na reunião.
- [`src/middlewares/auth.middleware.ts:27–46`](../../../../../src/middlewares/auth.middleware.ts#L27-L46): verificação JWT existente, distinta da autenticação de mensagens de saída.
- [`src/shared/logger/index.ts:4–19`](../../../../../src/shared/logger/index.ts#L4-L19) e [`src/middlewares/error.middleware.ts:14–23`](../../../../../src/middlewares/error.middleware.ts#L14-L23): escopo observado das facilidades compartilhadas.

```ts
// src/shared/logger/index.ts:4–10 — campos atualmente configurados para redação
const redactPaths = [
  'req.headers.authorization',
  'req.headers.cookie',
  '*.password',
  '*.passwordHash',
  '*.token',
  '*.accessToken',
];
```

A transcrição e o logger foram introduzidos no commit `7ef4317` (`init repository`, 2026-06-24); o histórico consultado não mostra implementação posterior de webhooks nem data da reunião. A decisão é um registro de discussão, não uma garantia operacional.

## Perguntas para um ADR formal

- Qual representação exata do corpo e da assinatura será verificável pelos clientes?
- Como proteger, exibir e rotacionar secrets e como assinar durante a janela de sobreposição?
- Como validar destinos HTTPS e lidar com redirecionamentos e endereços internos?
- Qual política de timestamp e replay será adotada e verificada pelos destinatários?

## Sobreposições

Relaciona-se à [autenticação JWT e autorização por papéis](autenticacao-jwt-e-autorizacao-por-papeis.md), mas protege outra fronteira: mensagens de saída, não chamadas à API. O candidato de [webhooks assíncronos de ORDERS](../ORDERS/webhooks-assincronos-de-status-de-pedidos.md) também consolida HMAC e rotação no contrato de entrega: há sobreposição explícita e o coordenador deve decidir se esta fronteira de segurança justifica ADR próprio ou se deve ser incorporada ao candidato de ORDERS. Não criar ADRs separados por header, HTTPS ou duração da janela.

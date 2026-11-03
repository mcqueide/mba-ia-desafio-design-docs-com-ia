# ADR-003: Autenticação de webhooks por HMAC

**Status:** Proposto
**Data:** Desconhecida
**Related to:**
- [ADR-007: Webhooks assíncronos para mudanças de status](../ORDERS/ADR-007-webhooks-assincronos-de-status.md)
- [ADR-004: Autenticação JWT e autorização por papéis](ADR-004-autenticacao-jwt-e-papeis.md)

## Contexto e problema

As notificações propostas enviam dados de pedidos para sistemas externos. Os destinatários precisam verificar a origem e a integridade das mensagens, sem compartilhar uma única secret entre todos os endpoints.

O código atual autentica chamadas recebidas por JWT; não há assinatura de webhooks implementada.

## Fatores de decisão

- Permitir verificação da integridade e autenticidade da mensagem.
- Limitar o impacto do vazamento de uma secret.
- Permitir rotação sem exigir migração instantânea dos destinatários.
- Exigir transporte HTTPS para as notificações.

## Opções consideradas

1. HMAC-SHA256 com secret independente por endpoint.

Nenhuma alternativa foi discutida. A escolha considerou o uso difundido de HMAC-SHA256 e a disponibilidade de bibliotecas nos clientes.

## Resultado da decisão

A assinatura HMAC-SHA256 é enviada em hexadecimal no header `X-Signature`, e o identificador da chave usada é enviado em `X-Key-Id`. Cada endpoint possui uma secret própria, armazenada em um secret manager.

Sem proteção por timestamp, a assinatura cobre os bytes exatos do corpo. Quando essa proteção estiver habilitada, a mensagem assinada será a concatenação dos bytes UTF-8 de `X-Timestamp` em segundos Unix, um ponto (`.`) e os bytes exatos do corpo. O timestamp será obrigatório nesse caso e terá tolerância de 30 segundos.

Na rotação, a chave antiga permanece válida por 24 horas. O emissor continua a usá-la até o cliente confirmar a migração ou a janela expirar; depois usa a nova chave. O destinatário seleciona a secret pelo identificador recebido. A secret pode ser revelada uma única vez por identificador em consulta autenticada, enquanto o identificador permanece consultável. Se a secret for perdida, deve ser gerada outra com novo identificador. A consulta de chave expirada retorna HTTP 403 com `SIGNATURE_KEY_EXPIRED`.

## Prós e contras das opções

### HMAC-SHA256 com secret por endpoint

- **Pró:** permite ao destinatário verificar origem e integridade.
- **Pró:** limita o alcance do comprometimento de uma secret.
- **Contra:** exige secret manager, gestão de chaves e rotação de secrets.
- **Contra:** o destinatário precisa implementar a verificação e acompanhar a rotação.

### Alternativas

Nenhuma alternativa foi discutida na reunião.

## Consequências

O contrato de autenticação depende da concordância sobre os bytes assinados, o identificador de chave e a sincronização dos relógios. A secret nova fica disponível sem reiniciar o container; se o cliente não migrar até o fim da janela, o emissor passa a usar a nova chave mesmo assim.

O timestamp assinado limita a idade aceita das mensagens, mas não impede a repetição de uma mensagem válida dentro da janela de 30 segundos. A deduplicação do evento continua necessária.

## Referências

- [Transcrição: assinatura, rotação e HTTPS](../../../../TRANSCRICAO.md#L118-L138)
- [Transcrição: headers e revisão de segurança](../../../../TRANSCRICAO.md#L260-L288)
- [Logger compartilhado](../../../../src/shared/logger/index.ts#L4-L19)
- Esclarecimentos fornecidos pelo usuário em 02-11-2026: contrato de assinatura, rotação, secret manager e proteção opcional por timestamp.

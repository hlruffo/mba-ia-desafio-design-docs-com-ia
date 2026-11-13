# ADR-004 — HMAC-SHA256 com secret por endpoint

## Status
Aceita

## Contexto
Os eventos carregam dados de pedido e precisam ser autenticáveis pelo recebedor. Uma secret global ampliaria o impacto de um vazamento.

## Decisão
Assinar o corpo JSON com HMAC-SHA256 e enviar a assinatura em `X-Signature`. Cada configuração de webhook possui uma secret própria, gerada pela plataforma. A rotação mantém a secret antiga válida por 24 horas; a URL deve ser HTTPS e o payload máximo é 64 KiB.

## Alternativas Consideradas
- **Secret global por plataforma**: rejeitada porque o comprometimento de um cliente afetaria todos os endpoints.
- **HTTP sem assinatura**: rejeitada porque o cliente não conseguiria validar origem e integridade do corpo.

## Consequências
### Positivas
- Isolamento de credenciais por endpoint e verificação independente pelo cliente.
- Rotação com período de transição reduz indisponibilidade durante a troca.

### Negativas e trade-offs
- Cada cliente precisa implementar verificação HMAC e rotação.
- O serviço deve proteger secrets em persistência e nunca incluí-las em logs.

# ADR-005 — At-least-once com `X-Event-Id`

## Status
Aceita

## Contexto
Uma falha após o recebimento pelo cliente e antes da confirmação pode provocar nova tentativa. Exactly-once exigiria coordenação distribuída entre a plataforma e todos os clientes.

## Decisão
Garantir at-least-once. Gerar UUID único quando o evento entra na outbox e enviá-lo no header `X-Event-Id` e no corpo. O cliente deve deduplicar recebimentos repetidos pelo identificador.

## Alternativas Consideradas
- **Exactly-once**: rejeitada pelo custo e complexidade de coordenação entre sistemas independentes.
- **Sem identificador de evento**: rejeitada porque o consumidor não teria uma chave estável para deduplicação.

## Consequências
### Positivas
- Reduz risco de perda em falhas intermediárias.
- Consumidores podem implementar idempotência simples e observável.

### Negativas e trade-offs
- Duplicatas são comportamento esperado e precisam ser documentadas para os clientes.
- A responsabilidade de deduplicar é compartilhada com o consumidor.

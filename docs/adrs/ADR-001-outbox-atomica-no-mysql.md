# ADR-001 — Outbox atômica no MySQL

## Status
Aceita

## Contexto
A mudança de status já atualiza `orders`, `order_status_history` e estoque dentro de uma transação Prisma. Uma chamada HTTP síncrona poderia bloquear a transação ou provocar rollback por indisponibilidade do cliente.

## Decisão
Registrar um snapshot do evento em uma tabela `webhook_outbox` na mesma transação MySQL da mudança de status. O evento só será criado quando houver webhook interessado no status de destino; o payload será materializado no momento da inserção.

## Alternativas Consideradas
- **HTTP síncrono no `OrderService`**: rejeitado porque cliente lento bloquearia outros pedidos e a indisponibilidade não deve desfazer uma mudança já válida.
- **Redis Streams**: rejeitado porque exigiria infraestrutura adicional; o MySQL existente atende ao volume e ao requisito de consistência transacional.

## Consequências
### Positivas
- Commit do pedido e registro do evento têm a mesma fronteira atômica.
- Não há evento para uma transação que sofreu rollback.
- A solução reutiliza o banco já operado.

### Negativas e trade-offs
- O worker consulta uma tabela, gerando polling e necessidade de índices/limpeza futura.
- A entrega é assíncrona, portanto o cliente não recebe confirmação dentro da requisição de mudança de status.

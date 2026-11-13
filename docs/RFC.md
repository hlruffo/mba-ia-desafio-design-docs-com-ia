# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autor | Equipe de Engenharia |
| Status | Proposto para revisão |
| Data | Após a reunião técnica de quinta-feira |
| Revisores | Larissa, Marcos, Bruno, Diego e Sofia |

## TL;DR
Propomos um mecanismo outbound para notificar clientes B2B sobre mudanças de status de pedidos. O evento será gravado como snapshot em uma outbox MySQL dentro da mesma transação do pedido. Um worker separado fará polling de 2 segundos, entregará por HTTP com HMAC-SHA256 e aplicará retry limitado com DLQ. A semântica será at-least-once, identificada por `X-Event-Id`.

## Contexto e problema
Atlas Comercial, MaxDistribuição e Nova Cargo hoje consultam `GET /orders` repetidamente. Isso aumenta custo e latência da integração. Para eles, uma entrega abaixo de 10 segundos caracteriza tempo real. O webhook é somente outbound e deve ser desacoplado da transação de pedidos.

## Proposta técnica
Adicionar um domínio de webhooks à API existente. Configurações por customer terão URL HTTPS, secret própria e filtros de status. Quando `changeStatus` confirmar uma transição, a mesma transação registrará snapshots apenas para endpoints interessados. Um processo separado consumirá a outbox, assinará o corpo, enviará a chamada e registrará sucesso/falha. Cinco falhas levam à DLQ e o replay será administrativo.

A proposta preserva os padrões atuais do projeto e separa o nível arquitetural do detalhamento operacional que está no [FDD](./FDD.md). As decisões fechadas estão nos [ADRs](./adrs/ADR-001-outbox-atomica-no-mysql.md), [ADR-002](./adrs/ADR-002-worker-separado-polling.md), [ADR-003](./adrs/ADR-003-retry-backoff-dlq.md), [ADR-004](./adrs/ADR-004-hmac-por-endpoint.md), [ADR-005](./adrs/ADR-005-at-least-once-event-id.md) e [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md).

## Alternativas consideradas
1. **HTTP síncrono no fluxo de pedidos** — descartado: cliente lento bloquearia a transação e uma indisponibilidade não pode causar rollback do status.
2. **Redis Streams** — descartado: adicionaria infraestrutura para um time pequeno, enquanto o MySQL já oferece a fronteira transacional necessária.
3. **Trigger para acordar o worker** — descartado: o MySQL não notifica naturalmente um processo Node externo; polling de 2 segundos atende à meta.

## Questões em aberto
- Rate limiting de saída para clientes com muitos pedidos: observar o comportamento e decidir depois, não faz parte desta fase.
- Arquivamento da outbox após 30 dias: foi mencionado, mas ficou fora da feature.
- O desenho para escalar além de um worker (particionamento por `order_id` ou lock pessimista) é futuro.

## Impacto e riscos
A mudança acrescenta tabelas, um módulo HTTP, um processo de worker e integração transacional no domínio de pedidos. Os principais riscos são crescimento de outbox/DLQ, exposição indevida de secret e duplicatas no consumidor. Índices e operação de DLQ mitigam o primeiro; HMAC, HTTPS, rotação e redaction mitigam o segundo; `X-Event-Id` e documentação explícita mitigam o terceiro.

## Critérios de revisão
- Confirmar que todo evento é snapshot atômico e filtrado na inserção.
- Confirmar que secrets não aparecem em logs ou payloads de entrega.
- Confirmar que replay exige `ADMIN` e deixa auditoria.
- Confirmar que os contratos e códigos de erro do FDD não contradizem a API existente.

# FDD — Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica
O OMS possui transições de pedido transacionais, mas não possui eventos ou notificações externas. O módulo deve preservar a transação existente e deslocar I/O para um processo separado.

## Objetivos técnicos
- Persistir snapshot e `X-Event-Id` atomicamente com `changeStatus`.
- Entregar JSON assinado para endpoints HTTPS.
- Implementar polling, timeout, retry, DLQ, deliveries e replay.
- Reutilizar padrões de autenticação, validação, erros e logs.

## Escopo e exclusões
Inclui configuração, seleção de status, outbox, worker, entrega e DLQ. Exclui email, dashboard, rate limiting e arquivamento automático de 30 dias.

## Modelo lógico proposto
`webhook_config`: UUID, `customer_id`, URL, secret atual, secret anterior e expiração da anterior, status ativos, `active`, timestamps. `webhook_outbox`: UUID/event_id, webhook_id, customer_id, snapshot JSON, status, attempts, next_attempt_at, created_at, last_error. `webhook_delivery`: tentativa, resposta, status HTTP, duração e timestamps. `webhook_dead_letter`: payload, motivo, número de tentativas, timestamps e referência do webhook.

## Fluxos detalhados
### Criação do evento na outbox
1. `OrderService.changeStatus` inicia a transação já existente.
2. Valida transição, estoque, atualiza `orders` e insere `order_status_history`.
3. Consulta configurações ativas do customer cujo filtro contém `toStatus`.
4. Para cada configuração, gera UUID e snapshot contendo `event_type`, timestamp ISO 8601, `order_id`, `order_number`, `customer_id`, `from_status`, `to_status` e `total_cents`; não inclui `items`.
5. Insere outbox usando o mesmo `tx`; erro de inserção desfaz toda a transação.

### Processamento pelo worker
A cada 2 segundos, a entry-point de worker busca o lote pequeno de pendentes mais antigos, faz claim seguro, e envia cada snapshot pelo processador do módulo de webhooks. O worker usa PrismaClient próprio e o mesmo `DATABASE_URL`, mas não compartilha o processo da API. Uma confirmação HTTP de sucesso marca a entrega e registra status/duração.

### Retry
Timeout de 10 segundos ou resposta não bem-sucedida gera tentativa com backoff de 1m, 5m, 30m, 2h e 12h. O evento permanece elegível em `next_attempt_at`; logs incluem identificadores, não secrets. A tentativa deve ser segura para repetição.

### DLQ e replay
Depois da quinta falha, persistir payload, motivo e timestamp em `webhook_dead_letter`. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige JWT com role `ADMIN`, registra o usuário executor e recria uma entrada pendente. O histórico original permanece consultável.

## Contratos públicos
Prefixo sugerido: `/api/v1`.

### POST `/webhooks`
Autenticado. Body:
```json
{"customer_id":"cus-uuid","url":"https://client.example/hooks/orders","events":["SHIPPED","DELIVERED"]}
```
`201` (a secret aparece na criação):
```json
{"id":"wh-uuid","customer_id":"cus-uuid","url":"https://client.example/hooks/orders","events":["SHIPPED","DELIVERED"],"secret":"generated-once","active":true}
```
`400` para URL não HTTPS ou payload inválido; `401` sem JWT.

### GET `/webhooks?customer_id=cus-uuid`
Autenticado. `200`:
```json
{"items":[{"id":"wh-uuid","url":"https://client.example/hooks/orders","events":["SHIPPED"],"active":true}],"total":1}
```
A resposta não inclui secret. `400` query inválida; `401` sem JWT.

### PATCH `/webhooks/:id`
Autenticado. Body:
```json
{"url":"https://client.example/hooks/v2","events":["DELIVERED"],"active":true}
```
`200` retorna a configuração sem secret; `404` usa `WEBHOOK_NOT_FOUND`; `400` validação.

### DELETE `/webhooks/:id`
Autenticado. `204` sem corpo; `404` `WEBHOOK_NOT_FOUND`; `401` sem JWT. Remoção não apaga deliveries históricos.

### GET `/webhooks/:id/deliveries?limit=100`
Autenticado. `200` retorna no máximo 100:
```json
{"items":[{"event_id":"evt-uuid","status":"DELIVERED","http_status":200,"response_time_ms":142,"attempts":1,"created_at":"2026-01-01T09:00:00Z"}],"limit":100}
```
`400` limite inválido; `404` `WEBHOOK_NOT_FOUND`.

### POST `/admin/webhooks/dead-letter/:id/replay`
Autenticado com `ADMIN`. `202`:
```json
{"dead_letter_id":"dlq-uuid","event_id":"evt-uuid","status":"PENDING"}
```
`401` sem JWT, `403` sem role, `404` `WEBHOOK_DLQ_NOT_FOUND`, `409` se já estiver em replay.

### Rotação de secret
`POST /webhooks/:id/rotate-secret`, autenticado, retorna a nova secret uma vez e mantém a anterior por 24h. `200` em sucesso; `404` `WEBHOOK_NOT_FOUND`.

### Headers de entrega
`Content-Type: application/json`, `X-Event-Id: <uuid>`, `X-Signature: sha256=<hex>`, `X-Timestamp: <ISO-8601>`, `X-Webhook-Id: <uuid>`. HMAC é calculado sobre os bytes do corpo enviado.

## Matriz de erros
| Código | HTTP | Condição |
|---|---:|---|
| `WEBHOOK_NOT_FOUND` | 404 | configuração, delivery ou DLQ inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL não HTTPS ou malformada |
| `WEBHOOK_SECRET_REQUIRED` | 400 | tentativa de configurar secret inválida |
| `WEBHOOK_INVALID_EVENTS` | 400 | status inexistente ou lista inválida |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | snapshot excede 64 KiB |
| `WEBHOOK_DELIVERY_TIMEOUT` | 504 interno/worker | receptor não respondeu em 10 s |
| `WEBHOOK_DELIVERY_FAILED` | 502 interno/worker | resposta/rede falhou |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | item de DLQ inexistente |
| `WEBHOOK_REPLAY_FORBIDDEN` | 403 | usuário não é `ADMIN` |
| `WEBHOOK_REPLAY_CONFLICT` | 409 | item não pode ser reprocessado |

## Estratégias de resiliência
- Timeout de 10 segundos por chamada.
- Backoff fixo crescente conforme ADR-003; máximo de cinco tentativas.
- Fallback operacional é DLQ persistida e replay manual, não chamada síncrona.
- Claim/estado devem impedir dois processamentos simultâneos no single-worker.
- Secrets são redacted e nunca fazem parte do payload de delivery.

## Observabilidade
- **Métricas**: `webhook_delivery_success_total`, `webhook_delivery_failure_total`, `webhook_delivery_latency_ms`, `webhook_outbox_pending`, `webhook_dlq_size`, `webhook_retry_total`.
- **Logs Pino**: `event_id`, `webhook_id`, `customer_id`, tentativa, status HTTP e duração; nunca URL com secret embutida, secret ou corpo sensível.
- **Tracing**: propagar um correlation/request id da transição para o evento e registrar spans de seleção, claim, chamada HTTP e persistência do resultado. Se não houver tracer no projeto, manter os mesmos campos nos logs até sua adoção.

## Integração com o sistema existente
- `src/modules/orders/order.service.ts`: estender `changeStatus` para chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` após a mudança e history, ainda dentro da transação; snapshot usa o pedido daquele momento.
- `src/modules/orders/order.routes.ts`: manter autenticação e validação no padrão atual; a feature não altera o endpoint de status além do efeito transacional.
- `src/routes/index.ts`: registrar `buildWebhookRouter` sob `/webhooks` e a rota administrativa, ampliando o tipo `Controllers`.
- `src/app.ts`: construir repository/service/controller do domínio webhooks e injetar Prisma; manter o middleware de erro e o limite JSON existentes.
- `src/middlewares/auth.middleware.ts`: reutilizar `authenticate` e `requireRole('ADMIN')` no replay.
- `src/shared/errors/app-error.ts` e `src/shared/errors/index.ts`: criar classes do módulo baseadas em `AppError`, com códigos `WEBHOOK_*`, para o `errorMiddleware` existente serializar sem mudança estrutural.
- `src/middlewares/error.middleware.ts`: nenhum novo formato; erros de domínio continuam `{ error: { code, message, details } }`.
- `src/shared/logger/index.ts`: usar Pino e sua política de redaction, adicionando somente campos operacionais seguros.
- `prisma/schema.prisma`: acrescentar modelos e índices da outbox/configuração/delivery/DLQ sem modificar os modelos atuais além das relações necessárias.
- `src/server.ts`: não iniciar o worker na entry-point HTTP; criar uma entry-point de processo separada, próxima ao padrão de bootstrap existente, com PrismaClient e ciclo de shutdown próprios.

## Dependências e compatibilidade
Node >=20, TypeScript, Express, Prisma/MySQL, Zod, JWT, Pino e UUID já estão no projeto. A implementação deve ser compatível com os enums de `OrderStatus`, o formato de `AppError` e os endpoints atuais. Alterações em `src/`, `prisma/`, testes ou configuração pertencem à implementação futura e estão proibidas nesta entrega documental.

## Critérios de aceite técnicos
- Teste transacional prova que rollback não deixa outbox.
- Teste prova filtragem por status e snapshot sem `items`.
- Assinatura calculada sobre o corpo entregue verifica com a secret correta e falha com secret incorreta.
- Worker respeita timeout, cinco intervalos, DLQ e replay ADMIN.
- CRUD, deliveries e rotação respeitam autenticação e respostas documentadas.
- Métricas/logs contêm IDs e não secrets; build/testes do baseline permanecem inalterados.

## Riscos e mitigação
Crescimento da outbox é mitigado por índices e batches; duplicatas por `X-Event-Id`; vazamento por redaction/rotação; escala futura por registrar single-worker como limitação e deixar particionamento para fase posterior.

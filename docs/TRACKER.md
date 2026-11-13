# Tracker de Rastreabilidade

Cada linha aponta para uma afirmação nos documentos e sua origem primária. A maior parte das decisões e requisitos tem origem explícita na transcrição; os caminhos de código foram conferidos no repositório base.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito funcional | CRUD de configuração | TRANSCRICAO | [09:31] Marcos; [09:33] Bruno |
| PRD-FR-02 | docs/PRD.md | Requisito funcional | Filtro por status na outbox | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito funcional | Histórico dos últimos 100 deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito funcional | Replay administrativo da DLQ | TRANSCRICAO | [09:18] Diego; [09:35] Diego |
| PRD-FR-05 | docs/PRD.md | Requisito funcional | Replay somente ADMIN e auditado | TRANSCRICAO | [09:35] Sofia; [09:36] Larissa |
| PRD-FR-06 | docs/PRD.md | Requisito funcional | Secret gerada na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito funcional | Rotação com grace period | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito funcional | Evento at-least-once com ID | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| PRD-FR-09 | docs/PRD.md | Restrição | URL deve ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-10 | docs/PRD.md | Restrição | Payload máximo de 64 KiB | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-NFR-01 | docs/PRD.md | NFR | Latência abaixo de 10 segundos | TRANSCRICAO | [09:01] Marcos; [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | NFR | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | NFR | Retry 5x com backoff definido | TRANSCRICAO | [09:15] Diego; [09:17] Diego |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Email de alerta | TRANSCRICAO | [09:37] Marcos; [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:39] Marcos; [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limiting observado depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-DEC-01 | docs/RFC.md | Decisão | Outbox no MySQL | TRANSCRICAO | [09:06] Diego; [09:07] Larissa |
| RFC-DEC-02 | docs/RFC.md | Decisão | Worker separado e polling | TRANSCRICAO | [09:09] Diego; [09:11] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | HTTP síncrono descartado | TRANSCRICAO | [09:03] Larissa; [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado | TRANSCRICAO | [09:07] Diego |
| RFC-Q-01 | docs/RFC.md | Questão aberta | Rate limiting | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão aberta | Arquivamento após 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-Q-03 | docs/RFC.md | Questão futura | Escala multi-worker | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Snapshot na transação de status | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Polling em lote de 2 segundos | TRANSCRICAO | [09:08] Diego; [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | DLQ separada e replay | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Payload JSON enxuto sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Headers de entrega | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | HMAC-SHA256 por endpoint | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| FDD-ERR-01 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ | TRANSCRICAO | [09:28] Bruno; [09:29] Larissa |
| FDD-RES-01 | docs/FDD.md | Resiliência | Cinco retries e backoff | TRANSCRICAO | [09:15] Diego; [09:17] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Pino e auditoria do replay | TRANSCRICAO | [09:29] Bruno; [09:35] Sofia |
| ADR-001 | docs/adrs/ADR-001-outbox-atomica-no-mysql.md | Decisão | Outbox atômica no MySQL | TRANSCRICAO | [09:06] Diego; [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-polling.md | Decisão | Worker separado e polling | TRANSCRICAO | [09:09] Diego; [09:11] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-dlq.md | Decisão | Retry e DLQ | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-por-endpoint.md | Decisão | HMAC e secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-event-id.md | Decisão | At-least-once e X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso dos padrões | TRANSCRICAO | [09:30] Larissa |
| CODE-01 | docs/FDD.md | Integração | `changeStatus` é transacional | CODIGO | src/modules/orders/order.service.ts |
| CODE-02 | docs/FDD.md | Integração | Rotas de pedidos usam authenticate/Zod | CODIGO | src/modules/orders/order.routes.ts |
| CODE-03 | docs/FDD.md | Integração | Registro de módulos no roteador | CODIGO | src/routes/index.ts |
| CODE-04 | docs/FDD.md | Integração | Construção de controllers e middleware | CODIGO | src/app.ts |
| CODE-05 | docs/FDD.md | Integração | `requireRole` disponível para ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| CODE-06 | docs/FDD.md | Integração | Formato de AppError | CODIGO | src/shared/errors/app-error.ts |
| CODE-07 | docs/FDD.md | Integração | Serialização centralizada de erros | CODIGO | src/middlewares/error.middleware.ts |
| CODE-08 | docs/FDD.md | Integração | Logger Pino com redaction | CODIGO | src/shared/logger/index.ts |
| CODE-09 | docs/FDD.md | Integração | MySQL e modelos UUID Prisma | CODIGO | prisma/schema.prisma |

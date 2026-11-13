# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature
Clientes B2B precisam saber quando um pedido muda de status sem consultar repetidamente a API. A feature adiciona webhooks outbound configuráveis por customer para entregar snapshots de mudanças de status.

## Problema e motivação
Hoje Atlas Comercial, MaxDistribuição e Nova Cargo fazem polling de `GET /orders`, o que torna a integração lenta e cara. Um atraso inferior a 10 segundos é considerado tempo real pelos clientes.

## Público-alvo e cenários de uso
- **Clientes B2B**: cadastram um endpoint HTTPS e escolhem status de interesse.
- **Operadores autenticados**: administram configurações e consultam entregas.
- **Administradores**: reprocessam itens da DLQ.

Cenários: cadastrar/editar/remover endpoint; listar configurações; receber `order.status_changed`; consultar até os últimos 100 deliveries; replay administrativo de falha.

## Objetivos e métricas de sucesso
- Disponibilizar notificações outbound para os clientes solicitantes.
- Entregar eventos em até **10 segundos** em condições normais; o polling de 2 segundos é a meta operacional da primeira versão.
- Não perder um evento cuja transação de mudança de status tenha commitado.
- Medir taxa de entrega, latência e volume de DLQ antes de decidir rate limiting.

## Escopo
### Incluído
1. CRUD autenticado de configurações por customer.
2. Filtro por lista de status.
3. Outbox MySQL atômica com mudança de status e payload snapshot.
4. Worker separado com polling de 2 segundos.
5. HMAC-SHA256, secret por endpoint e rotação com validade antiga de 24h.
6. Retry em 5 tentativas e DLQ persistida.
7. Histórico de deliveries e replay de DLQ por `ADMIN`.
8. Entrega at-least-once com `X-Event-Id`.

### Fora de escopo
- **Email de alerta após três falhas**: explicitamente deixado para uma próxima fase.
- **Dashboard visual para clientes**: separado para o time de frontend.
- Rate limiting de saída: apenas observar nesta fase.
- Arquivamento de eventos entregues após 30 dias: operação futura.

## Requisitos funcionais
- **FR-01** Cadastrar webhook com URL, customer e status monitorados; secret é gerada e devolvida apenas na criação.
- **FR-02** Editar configuração existente.
- **FR-03** Remover configuração.
- **FR-04** Listar webhooks de um customer.
- **FR-05** Aceitar somente URL HTTPS.
- **FR-06** Filtrar o evento na inserção da outbox quando nenhum endpoint ouvir o status.
- **FR-07** Gravar o evento na mesma transação da mudança de status.
- **FR-08** Entregar JSON com dados básicos do pedido e `event_id`.
- **FR-09** Enviar `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type`.
- **FR-10** Permitir consulta dos últimos 100 deliveries por webhook.
- **FR-11** Reprocessar DLQ por endpoint administrativo.
- **FR-12** Exigir role `ADMIN` no replay e registrar o administrador.
- **FR-13** Permitir rotação da secret com grace period de 24h.
- **FR-14** Retentar falhas conforme a política e mover a falha permanente para DLQ.

## Requisitos não funcionais
- HMAC-SHA256 sobre o corpo exato enviado.
- Timeout HTTP de 10 segundos.
- Limite de payload de 64 KiB; excedente falha, não é truncado.
- Garantia at-least-once; consumidores devem deduplicar por UUID.
- Worker separado da API, com PrismaClient próprio.
- Ordenação por `order_id` enquanto houver single-worker; não há ordering global.
- Secrets não devem ser expostas em logs.

## Decisões e trade-offs principais
Outbox privilegia consistência sobre a simplicidade de uma chamada síncrona; polling privilegia infraestrutura pequena sobre latência zero; at-least-once evita perda, aceitando duplicatas; cinco retries equilibram manutenção transitória e retenção indefinida.

## Dependências
MySQL/Prisma existentes, autenticação JWT, `requireRole`, schemas Zod, `AppError`, middleware de erros e logger Pino. A entrega depende de revisão de segurança de Sofia antes do deploy e de três sprints estimadas.

## Riscos e mitigação
| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente indisponível acumular eventos | Alta | Alto | timeout de 10s, backoff limitado, DLQ e replay |
| Secret vazar em logs ou integração | Média | Alto | redaction, HTTPS, secret por endpoint e rotação |
| Duplicata no consumidor | Média | Médio | `X-Event-Id`, semântica explícita at-least-once |
| Outbox crescer sem arquivamento | Média | Médio | índices, batches pequenos, observabilidade e decisão posterior de retenção |

## Critérios de aceitação
- Uma transição commitada cria snapshots apenas para webhooks interessados.
- Rollback da transação não deixa evento.
- Worker entrega com headers e assinatura verificável.
- Timeout/falha segue os cinco intervalos e termina na DLQ.
- CRUD, deliveries e replay retornam contratos documentados; replay sem `ADMIN` é recusado.
- Uma configuração HTTPS inválida é recusada e nenhum código de aplicação existente fora da feature é alterado.

## Estratégia de testes e validação
Testar unidade para filtro, snapshot, assinatura, backoff e códigos; integração para transação/outbox, Prisma e DLQ; contrato HTTP para CRUD, deliveries e replay; cenário ponta a ponta com endpoint receptor, timeout, duplicata e replay. Revisar manualmente rastreabilidade e consistência com `TRANSCRICAO.md` e código.

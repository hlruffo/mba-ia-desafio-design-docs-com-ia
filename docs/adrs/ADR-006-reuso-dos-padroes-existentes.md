# ADR-006 — Reutilizar padrões do projeto

## Status
Aceita

## Contexto
A aplicação possui módulos por domínio, schemas Zod, `AppError`, middleware centralizado, autenticação JWT, `requireRole` e logger Pino. Criar mecanismos paralelos aumentaria a superfície de manutenção.

## Decisão
Implementar webhooks como `src/modules/webhooks`, registrar suas rotas no roteador existente, reutilizar `AppError`/códigos `WEBHOOK_*`, validação Zod, `authenticate`, `requireRole`, middleware de erros e Pino. O worker terá um PrismaClient separado por ser outro processo.

## Alternativas Consideradas
- **Criar um framework ou middleware de erros próprio**: rejeitado por duplicar comportamento já centralizado.
- **Colocar rotas e lógica em `orders`**: rejeitado porque configuração, entrega e DLQ formam um domínio próprio.

## Consequências
### Positivas
- Menor divergência de contratos e operação com o restante da API.
- Erros e logs seguem convenções já conhecidas.

### Negativas e trade-offs
- O módulo precisará respeitar as limitações e contratos atuais do projeto.
- A integração ainda exigirá extensão de `OrderService` e do schema Prisma, sem alterar o comportamento existente fora da feature.

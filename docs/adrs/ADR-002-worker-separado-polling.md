# ADR-002 — Worker separado com polling de dois segundos

## Status
Aceita

## Contexto
O envio precisa ocorrer fora do processo HTTP. O MySQL não oferece um listener nativo que notifique um processo Node externo, e a meta de produto aceita latência inferior a 10 segundos.

## Decisão
Criar uma entry-point de worker separada da entry-point HTTP, com PrismaClient próprio, que consulta eventos pendentes em lotes pequenos a cada 2 segundos, ordenados por criação. Manter um único worker na primeira versão.

## Alternativas Consideradas
- **Trigger de banco para notificar o processo**: rejeitado porque trigger executa SQL, mas não notifica um processo externo sem uma integração improvisada.
- **Worker dentro de `src/server.ts`**: rejeitado porque reinício da API também interromperia o processamento.

## Consequências
### Positivas
- Falhas/reinícios da API não eliminam o worker.
- A implementação usa o mesmo banco e a mesma stack, com isolamento de ciclo de vida.
- Polling de 2 s atende o limite de menos de 10 s.

### Negativas e trade-offs
- A latência mínima inclui o intervalo de polling.
- Um único worker limita throughput e ordering global não é prometido; a ordenação é por `order_id` enquanto essa topologia permanecer.

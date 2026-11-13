# ADR-003 — Retry exponencial e DLQ persistida

## Status
Aceita

## Contexto
Endpoints de clientes podem ficar indisponíveis por manutenção ou falha transitória. O evento não deve desaparecer na primeira falha, mas também não pode ficar retentando indefinidamente.

## Decisão
Tentar no máximo cinco vezes, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Após a última falha, mover o evento e evidências para `webhook_dead_letter`. Disponibilizar replay manual por endpoint administrativo.

## Alternativas Consideradas
- **Três tentativas**: rejeitada por ser insuficiente para manutenções de aproximadamente duas horas.
- **Retry indefinido**: rejeitado porque um endpoint abandonado manteria eventos pendentes para sempre.
- **Marcar `failed` na outbox sem tabela separada**: rejeitado porque mistura operação corrente com evidência de falhas e dificulta reprocessamento.

## Consequências
### Positivas
- Cobre indisponibilidades transitórias e mantém evidência operacional.
- O replay é deliberado e auditável, sem loop automático infinito.

### Negativas e trade-offs
- A última tentativa pode ocorrer quase 15 horas após a primeira falha.
- DLQ exige retenção e operação futura; arquivamento de entregues por 30 dias não faz parte desta entrega.

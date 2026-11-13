# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio
Este trabalho transforma a transcrição de uma reunião técnica em um pacote acionável para a feature de webhooks outbound de mudanças de status de pedidos. O código do OMS foi usado como contexto para que os documentos apontassem integrações reais, sem alterar a aplicação.

A entrega separa produto, arquitetura, decisões e implementação: PRD responde por quê/o quê, RFC propõe a solução, ADRs registram decisões fechadas, FDD descreve como construir e TRACKER comprova a origem de cada afirmação.

## Ferramentas de IA utilizadas
- **Hermes Agent / modelo OpenAI**: exploração do repositório, extração estruturada da transcrição, redação e revisão crítica.
- **Terminal, Git e scripts de validação**: inspeção determinística de caminhos, status do repositório e checklist; não são geradores de conteúdo.

## Workflow adotado
1. Clonei o repositório base em uma branch de documentação e confirmei que o escopo não permitiria alterações em `src/`, `prisma/`, `tests/` ou configurações.
2. Li a transcrição inteira e cataloguei decisões fechadas, requisitos, alternativas rejeitadas, itens adiados e timestamps.
3. Inspecionei o schema Prisma e os módulos de orders, rotas, autenticação, erros, logger e composição da aplicação.
4. Produzi seis ADRs primeiro, consolidando as decisões: outbox, worker, retry/DLQ, HMAC, at-least-once e reuso de padrões.
5. Escrevi RFC, FDD e PRD em alturas diferentes, evitando usar o RFC como cópia do detalhe do FDD.
6. Montei o tracker com origem primária para requisitos e decisões e caminhos reais para integrações de código.
7. Fiz revisão final dos critérios de aceite e mantive `TRANSCRICAO.md` e o código intactos.

## Prompts customizados
```text
Leia TRANSCRICAO.md como fonte normativa. Classifique cada afirmação em:
DECISÃO FECHADA, REQUISITO, RESTRIÇÃO, ALTERNATIVA DESCARTADA,
QUESTÃO EM ABERTO ou FUTURO/FORA DE ESCOPO. Para cada item informe
timestamp, falante e não crie detalhes que não estejam na fonte.
```

```text
Inspecione apenas os caminhos reais do repositório e mapeie a integração
proposta do módulo de webhooks. Para cada integração informe arquivo,
classe/função existente e extensão prevista. Se o arquivo não existir,
marque como inexistente e não o use no documento.
```

```text
Revise o pacote como avaliador: conte requisitos funcionais, endpoints,
ADRs e linhas do tracker; procure contradições com a transcrição,
referências a arquivos inexistentes, secrets em logs e detalhes inventados.
Liste falhas por critério e sugira apenas correções rastreáveis.
```

## Iterações e ajustes
Foram conduzidos três ciclos principais:

1. **Extração inicial**: separei o que foi decidido do que apenas foi discutido. Isso evitou transformar rate limiting, arquivamento e dashboard em requisitos.
2. **Reconciliação com o código**: ajustei as referências para os caminhos reais (`src/modules/orders/order.service.ts`, `src/routes/index.ts`, middleware e Prisma), em vez de assumir uma estrutura genérica.
3. **Revisão de aceitação**: refinei contratos, códigos `WEBHOOK_*`, exemplos de payload, cobertura do tracker e a distinção entre RFC conciso e FDD detalhado.

## Como navegar a entrega
Ordem sugerida:

1. [`docs/PRD.md`](docs/PRD.md) — problema, público, escopo e sucesso.
2. [`docs/RFC.md`](docs/RFC.md) — proposta arquitetural, alternativas e pendências.
3. [`docs/adrs/`](docs/adrs/) — decisões isoladas e trade-offs.
4. [`docs/FDD.md`](docs/FDD.md) — fluxos, contratos, erros e integração no código.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — rastreabilidade.
6. `TRANSCRICAO.md` e `src/` — fontes primárias que não foram alteradas.

## Integridade da entrega
A entrega é documental. `TRANSCRICAO.md`, `src/`, `prisma/`, `tests/` e configurações permanecem fora das alterações desta branch.

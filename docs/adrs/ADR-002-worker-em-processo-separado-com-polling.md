# ADR-002 — Worker em processo separado, com polling a cada 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a latência
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Status

Aceito

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta decidir **como** e **onde** os eventos pendentes são lidos e enviados. O requisito de produto é entrega abaixo de 10 segundos ([09:02] Marcos). O projeto tem hoje uma única entry point, `src/server.ts`, que sobe a API HTTP.

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego). A latência mínima no pior caso fica em 2 s, o que foi aceito ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API**: se a API reinicia, o worker não pode morrer junto ([09:11] Diego). Nova entry point `src/worker.ts` (novo) e script `npm run worker`, no molde de `src/server.ts` ([09:11] Larissa; [09:28] Bruno).
3. **Mesmo banco e mesma stack, instância própria do `PrismaClient`**: mesma `DATABASE_URL`, mas um client novo, porque `PrismaClient` é por processo ([09:11] Bruno e Diego, [09:30] Bruno). A função `createPrismaClient()` de `src/config/database.ts` já permite isso.
4. **Um único worker**: ordenação garantida só por `order_id` e só enquanto houver um worker ([09:12] Diego, [09:13] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Trigger do banco para reagir na hora** ([09:09] Bruno) | MySQL não tem `NOTIFY/LISTEN` como o Postgres; trigger só executa SQL e não avisa processo externo. Avisar o worker exigiria improvisos (arquivo, chamada HTTP) ([09:09] Diego). |
| **Worker dentro do processo da API** | Reinício ou deploy da API derrubaria o worker ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Perde a ordem de entrega por pedido. Particionar por `order_id` ou usar lock pessimista ficou para o futuro ([09:13] Diego). |

## Consequências

**Positivas**
- Simples de operar e de entender; nenhum componente novo além de um processo Node.
- 2 s de intervalo dão folga grande para a meta de 10 s.
- API e worker têm ciclos de vida independentes.

**Negativas / trade-offs**
- Consulta ao banco a cada 2 s mesmo sem eventos (custo baixo graças aos índices de ADR-001).
- Throughput limitado a um worker; **ordering global não é garantido**, só por pedido — limitação conhecida e documentada ([09:13] Larissa). Os clientes não pediram ordering global ([09:14] Marcos).
- Um novo processo para implantar e monitorar.

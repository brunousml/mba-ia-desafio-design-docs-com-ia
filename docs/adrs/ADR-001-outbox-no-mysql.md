# ADR-001 — Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)

## Contexto

Três clientes B2B querem ser avisados quando o status dos pedidos muda, em menos de 10 segundos ([09:00] e [09:02] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e mexe no estoque dos produtos ([09:04] Bruno).

O problema central é registrar o evento de notificação **sem** acoplar a transação de pedidos à disponibilidade do sistema do cliente, e sem perder evento quando o status muda.

## Decisão

Usar o **padrão Outbox** sobre o MySQL que já existe ([09:06] Diego, fechado por Larissa em [09:08]):

- Na mesma transação que muda o status, inserir uma linha na tabela `webhook_outbox` com o evento.
- Se a transação principal fizer commit, o evento existe; se fizer rollback, o evento some junto.
- Se a inserção na outbox falhar, a mudança de status também é desfeita ([09:40] Bruno, [09:41] Diego).
- A tabela tem índice em `status` e `created_at`, para o worker ler só os pendentes em lotes pequenos ([09:08] Diego).

O envio HTTP em si fica a cargo de um worker separado (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo HTTP síncrono dentro do `changeStatus`** ([09:04] Bruno) | A transação já é pesada; um cliente lento travaria mudanças de status de outros pedidos. E se o cliente estiver fora do ar não há resposta aceitável: dar rollback na mudança de status não faz sentido ([09:04] Bruno, [09:06] Diego). |
| **Redis Streams (ou fila equivalente)** ([09:07] Larissa) | Exige subir infraestrutura nova (ex.: Redis Cluster). Para um time pequeno foi considerado overengineering ([09:07] Diego). Também não seria atômico com a transação do MySQL. |

## Consequências

**Positivas**
- Consistência entre status do pedido e evento: não existe "status mudou e evento não saiu" nem o contrário.
- Zero infraestrutura nova: mesmo banco, mesmo Prisma, mesma operação.
- A tabela vira trilha de auditoria natural do que foi (ou não) enviado.

**Negativas / trade-offs**
- A tabela cresce continuamente; arquivar linhas entregues (~30 dias) ficou **fora do escopo** desta feature ([09:08] Diego).
- A transação de `changeStatus` ganha mais uma escrita, e uma falha nela passa a impedir a mudança de status.
- Latência depende do intervalo de leitura do worker (2 s, ver ADR-002), em vez de ser imediata.

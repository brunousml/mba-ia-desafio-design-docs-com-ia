# ADR-007 — Payload gravado como snapshot na inserção da outbox

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, ~09:51 (após a saída de Marcos e Sofia)
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

O evento da outbox pode guardar o payload já montado ou só o `order_id`, montando o JSON no momento do envio ([09:51] Bruno). Entre a mudança de status e o envio podem passar segundos ou, com retry, até ~15 horas (ver [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)).

## Decisão

Gravar o **payload renderizado na inserção** (snapshot), dentro da transação do `changeStatus` ([09:52] Larissa, Diego e Bruno). O evento reflete o estado do pedido no momento em que o status mudou.

O payload é enxuto: `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`, **sem os itens do pedido** ([09:43] Diego, [09:44] Bruno). Limite de 64 KB, com erro se ultrapassar ([09:23] Sofia, [09:24] Diego e Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Guardar só `order_id` e renderizar no envio** | Se o pedido mudar depois, o evento mostraria um estado diferente do momento da transição — "caso esquisito" ([09:52] Larissa). |

## Consequências

**Positivas**
- Retries e replays enviam exatamente o mesmo corpo, o que mantém a assinatura HMAC e a deduplicação coerentes.
- O worker não precisa consultar `orders` para enviar.

**Negativas / trade-offs**
- Ocupa mais espaço por linha na outbox (mitigado pelo payload enxuto).
- Quem precisar de detalhes (itens) faz `GET /orders/:id` ([09:43] Diego).

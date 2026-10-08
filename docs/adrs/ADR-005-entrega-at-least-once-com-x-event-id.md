# ADR-005 — Entrega at-least-once com `X-Event-Id` para deduplicação no cliente

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Diego, Larissa; Sofia e Marcos de acordo
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Status

Aceito

## Contexto

Com outbox e retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), existem cenários em que o cliente recebe o mesmo evento mais de uma vez — por exemplo, o cliente processa a chamada, mas a resposta não chega antes do timeout e o worker tenta de novo. É preciso definir qual garantia a plataforma oferece.

## Decisão

- A plataforma garante **at-least-once**: todo evento é entregue pelo menos uma vez, podendo repetir ([09:24] Diego).
- Cada evento leva um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`**. O cliente deduplica por esse id ([09:25] Diego, fechado por Larissa em [09:26]).
- Marcos documenta esse comportamento em destaque no portal de desenvolvedor ([09:26] Marcos).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Exactly-once** | Exigiria coordenação dos dois lados e fica muito mais complexo ([09:25] Diego). |

## Consequências

**Positivas**
- Mesmo padrão de Stripe e GitHub ([09:25] Diego), familiar para integradores.
- Mantém o worker simples: na dúvida, reenvia.

**Negativas / trade-offs**
- Joga responsabilidade de idempotência para o cliente ([09:25] Sofia). A mitigação é documentação clara no portal.
- O `X-Event-Id` precisa ser estável entre tentativas e entre replays da DLQ, senão a deduplicação quebra.

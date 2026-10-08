# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno; Marcos validou a janela
- **Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Status

Aceito

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). Precisamos decidir quantas vezes tentar, com que intervalo e o que fazer com o evento quando desistimos.

## Decisão

- **5 tentativas** de reenvio com backoff exponencial nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h** — quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego, fechado por Larissa em [09:17]). Como são 5 intervalos e a soma (~14,6 h) só bate com "quase 15 horas" se a última espera de 12 h for cumprida, as 5 tentativas são contadas **após** o envio inicial.
- Falha = erro de rede, resposta não-2xx ou ausência de resposta em **10 segundos** de timeout ([09:42] Diego).
- Esgotadas as tentativas, o evento vai para a **tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
- Reprocessamento **manual**, via endpoint administrativo que recoloca o evento na outbox como pendente ([09:18] Diego), restrito à role `ADMIN` e com log de quem executou ([09:36] Sofia e Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **3 tentativas** ([09:16] Bruno) | Cobriria ~30 minutos; uma indisponibilidade de manhã mataria o evento. Clientes já ficaram 2 h fora ([09:16] Diego). |
| **Retry indefinido com backoff** ([09:15] Diego) | Evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **Marcar `failed` na própria outbox** ([09:17] Larissa) | Tabela separada deixa a leitura da outbox mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego). |

## Consequências

**Positivas**
- Cobre indisponibilidades longas (até ~15 h), aceitas pelo produto: "se cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- A outbox só contém trabalho vivo; a DLQ concentra os casos que precisam de humano.

**Negativas / trade-offs**
- Um evento pode chegar ao cliente até ~15 h depois do fato.
- Replay é manual: alguém precisa olhar a DLQ. Avisar o cliente por e-mail sobre falhas repetidas ficou para uma **próxima fase** ([09:37] Larissa).
- Retries aumentam a chance de duplicidade, tratada por [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).

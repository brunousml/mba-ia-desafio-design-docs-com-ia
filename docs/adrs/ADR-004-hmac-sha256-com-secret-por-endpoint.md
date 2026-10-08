# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com 24 h de carência

- **Status:** Aceito (sujeito à revisão de segurança antes do deploy)
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Sofia (Segurança), com Larissa, Diego e Bruno
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Status

Aceito (sujeito à revisão de segurança antes do deploy)

## Contexto

Os webhooks enviam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir provar que a requisição veio da plataforma e que o corpo não foi adulterado no caminho ([09:19] Sofia). Já houve cliente que vazou secret em log de aplicação ([09:22] Diego).

## Decisão

- Assinar **o corpo do request** com **HMAC-SHA256** e enviar no header `X-Signature` ([09:20] Sofia, fechado em [09:22]).
- **Uma secret única por endpoint de webhook**, nunca uma secret global ([09:21] Sofia). A configuração guarda url, secret, `customer_id` e estado ativo ([09:21] Bruno e Sofia).
- A secret é **gerada pela plataforma** e devolvida ao cliente na criação ([09:31] Marcos).
- **Rotação pela API**: ao rotacionar, a secret antiga continua válida por **24 horas** em paralelo; depois disso, morre ([09:21] Sofia).
- URL do webhook obrigatoriamente `https` (TLS), recusada na validação do schema Zod ([09:23] Sofia).
- A geração de secret e o HMAC passam por **revisão de segurança de pelo menos 2 dias úteis** antes do deploy ([09:46] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Secret global da plataforma** | Se vaza uma, vaza tudo ([09:21] Sofia). |
| **Rotação sem período de carência** (troca imediata) | O cliente precisa de tempo para migrar os sistemas dele; por isso a carência de 24 h ([09:21] Sofia). |

## Consequências

**Positivas**
- Padrão de mercado; qualquer cliente sério tem biblioteca de HMAC-SHA256 ([09:20] Sofia).
- Vazamento de uma secret afeta um só endpoint e pode ser corrigido por rotação.

**Negativas / trade-offs**
- A secret precisa ficar recuperável no banco para assinar (não dá para guardar só o hash, como se faz com senha).
- Durante as 24 h de carência a plataforma precisa lidar com duas secrets válidas para o mesmo endpoint — a mecânica exata está proposta no FDD e aberta no RFC.
- Mais um endpoint e mais estado (secret anterior e prazo de expiração) para manter.

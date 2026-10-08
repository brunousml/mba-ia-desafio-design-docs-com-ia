# ADR-006 — Reuso dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)

## Contexto

O Order Management System já tem convenções claras de organização e infraestrutura. A dúvida era se o webhook seguiria essas convenções ou traria peças novas (logger, formato de erro, estrutura de pastas).

O que existe hoje no código:

| Padrão | Onde está |
| --- | --- |
| Um módulo por domínio, com controller, service, repository, routes e schemas | `src/modules/orders/`, `src/modules/customers/` etc. |
| Classe base de erro com `statusCode`, `errorCode` e `details` | `src/shared/errors/app-error.ts` |
| Subclasses com códigos em caixa alta (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/http-errors.ts` |
| Middleware de erro que trata `AppError`, `ZodError` e erros do Prisma | `src/middlewares/error.middleware.ts` |
| Validação de body/query/params com Zod | `src/middlewares/validate.middleware.ts` |
| Autorização por role (`requireRole`) | `src/middlewares/auth.middleware.ts` |
| Logger Pino com redaction | `src/shared/logger/index.ts` |
| Registro dos routers por módulo | `src/routes/index.ts` |
| IDs UUID em todos os modelos | `prisma/schema.prisma` |

## Decisão

Reuso máximo do que já existe ([09:30] Larissa):

- Novo módulo **`src/modules/webhooks`** com a mesma estrutura dos demais ([09:27] Bruno). A lógica do worker fica dentro do módulo (ex.: `webhook.processor.ts`) e a entry point em `src/worker.ts` (novo) ([09:28] Bruno).
- Erros como subclasses de `AppError`, com **prefixo `WEBHOOK_`** nos códigos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc. ([09:28] Bruno, [09:29] Larissa).
- **Pino** como logger, sem biblioteca nova; o **error middleware** centralizado trata os erros novos sem alteração ([09:29] Bruno).
- Schemas **Zod** para validação, incluindo a regra de URL `https` ([09:23] Sofia).
- Replay da DLQ protegido pelo **`requireRole`** existente, com `ADMIN` ([09:36] Larissa).
- IDs **UUID** nas novas tabelas, como no resto do projeto ([09:51] Larissa).
- Integração com pedidos por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno e Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Infraestrutura própria para o módulo** (logger, formato de erro ou tratamento de erro dedicados) | Duplicaria o que já funciona e quebraria a consistência das respostas de erro. "Não vamos botar nada novo" ([09:29] Bruno). |
| **Injetar o repository de webhooks no `OrderService`** | Acoplamento maior; a função pura que recebe o `tx` resolve com menos dependências ([09:41] Diego). |
| **ID auto incremental na outbox** ([09:51] Diego) | Fugiria do padrão do projeto, em que tudo é UUID ([09:51] Larissa). |

## Consequências

**Positivas**
- Curva de aprendizado zero para quem já conhece o código; revisão mais fácil.
- Respostas de erro com o mesmo formato `{ error: { code, message, details } }` em toda a API.

**Negativas / trade-offs**
- O módulo herda as limitações dos padrões atuais. Por exemplo, o projeto não tem biblioteca de métricas nem de tracing; a observabilidade do webhook se apoia em logs estruturados do Pino (ver FDD).
- O redaction do logger hoje não cobre campos de secret; isso precisa ser estendido para não repetir o caso de vazamento citado em [09:22].

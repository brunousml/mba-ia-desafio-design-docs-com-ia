# Desafio 3 — plano detalhado

A entrega são 6 tipos de documento em Markdown, e o que mais reprova é **registrar algo que a reunião não decidiu**. Por isso o plano começa pelo mapa da transcrição com timestamps, que vira a fonte de todos os documentos e do TRACKER.

## O que precisa passar

| Documento | Números mínimos que a correção confere |
| --- | --- |
| `docs/PRD.md` | 12 seções; ≥ 8 requisitos funcionais; ≥ 1 objetivo com métrica e meta; ≥ 2 itens fora de escopo; ≥ 2 riscos com probabilidade, impacto e mitigação |
| `docs/RFC.md` | 2 a 4 páginas; ≥ 2 alternativas descartadas com trade-off; ≥ 2 questões em aberto; links para ≥ 2 ADRs; participantes como revisores |
| `docs/FDD.md` | 11 seções + "Integração com o sistema existente" com ≥ 4 caminhos reais; ≥ 4 endpoints com request, response e status; erros `WEBHOOK_*`; métricas, logs e tracing |
| `docs/adrs/` | 5 a 8 arquivos `ADR-NNN-titulo-em-kebab-case.md`; cobrir ≥ 5 das 6 decisões principais; ≥ 1 ADR citando arquivos do código |
| `docs/TRACKER.md` | ≥ 80% dos itens dos documentos; ≥ 70% das linhas com fonte TRANSCRICAO no formato `[hh:mm] Nome`; ≥ 5 linhas com fonte CODIGO |
| `README.md` | 6 seções; ≥ 2 prompts em blocos de código; ≥ 2 iterações concretas |

Duas regras valem para tudo: nenhum documento pode contradizer a transcrição ou o código, e nenhum arquivo citado pode ser inexistente. `src/`, `prisma/`, `tests/` e `TRANSCRICAO.md` não podem ser alterados.

## Mapa da transcrição

A reunião fecha 6 decisões principais, que são exatamente as 6 que o enunciado lista para os ADRs. Os timestamps abaixo vêm da minha leitura da `TRANSCRICAO.md` e já servem de coluna "Localização" do TRACKER.

**Decisões principais (candidatas a ADR)**

| # | Decisão | Onde foi fechada | Alternativa descartada |
| --- | --- | --- | --- |
| 1 | Outbox no MySQL, na mesma transação da mudança de status | \[09:06\] Diego, \[09:08\] Larissa | Disparo síncrono no service (\[09:04\] Bruno); Redis Streams (\[09:07\] Larissa e Diego) |
| 2 | Worker em processo separado, polling a cada 2 s, single-worker | \[09:09\] Diego, \[09:10\] Larissa, \[09:11\] Diego | Trigger do banco (\[09:09\] Bruno e Diego); worker dentro da API (\[09:11\] Diego) |
| 3 | 5 tentativas com backoff 1m/5m/30m/2h/12h, depois DLQ em tabela separada | \[09:17\] Larissa, \[09:18\] Diego | 3 tentativas (\[09:16\] Bruno); retry indefinido (\[09:15\] Diego); marcar "failed" na própria outbox (\[09:17\] Larissa) |
| 4 | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h de carência | \[09:20\] a \[09:22\] Sofia | Secret global da plataforma (\[09:21\] Sofia) |
| 5 | Entrega at-least-once, com `X-Event-Id` para deduplicação no cliente | \[09:24\] e \[09:25\] Diego, \[09:26\] Larissa | Exactly-once (\[09:25\] Diego) |
| 6 | Reuso dos padrões do projeto: módulo em `src/modules/webhooks`, `AppError`, Pino, error middleware, Zod, prefixo `WEBHOOK_` | \[09:27\] a \[09:30\] Bruno e Larissa | — |

**Decisões secundárias (ADR extra ou só FDD)**

| Item | Onde |
| --- | --- |
| Payload gravado como snapshot na inserção, não renderizado no envio | \[09:52\] Larissa e Diego |
| IDs UUID na outbox, seguindo o padrão do projeto | \[09:51\] Larissa |
| Filtro de eventos aplicado na inserção da outbox | \[09:34\] Bruno |
| Timeout de 10 s na chamada HTTP do worker | \[09:42\] Diego |
| Limite de 64 KB de payload, com erro se ultrapassar | \[09:24\] Diego e Larissa |
| URL obrigatoriamente `https`, validada no schema Zod | \[09:23\] Sofia |
| Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | \[09:44\] Diego e Sofia, \[09:45\] Diego |
| Formato do payload (`order.status_changed`, sem itens do pedido) | \[09:43\] Diego |
| Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada no `changeStatus` | \[09:41\] Bruno |
| Replay de DLQ só para role ADMIN, com log de quem executou | \[09:36\] Sofia e Larissa |

**Requisitos funcionais (o PRD pede ≥ 8; a reunião dá 10)**

cadastrar webhook com secret gerada e devolvida na criação (\[09:31\] Marcos); editar, remover e listar por customer (\[09:33\] Bruno); escolher os status por endpoint (\[09:33\] Marcos); histórico das últimas 100 entregas (\[09:34\] Marcos); replay manual de DLQ (\[09:18\] Diego); rotação de secret pela API (\[09:21\] Sofia); notificação a cada mudança de status (\[09:00\] Marcos); assinatura de cada envio (\[09:20\] Sofia).

**Fora de escopo e questões em aberto**

| Item | Situação | Onde |
| --- | --- | --- |
| E-mail quando o webhook falha repetidamente | Adiado para próxima fase | \[09:37\] Larissa |
| Dashboard visual para o cliente | Fora de escopo, projeto do frontend | \[09:40\] Larissa |
| Webhooks de entrada (cliente → plataforma) | Fora de escopo | \[09:02\] Marcos |
| Arquivamento de linhas entregues após \~30 dias | Fora de escopo desta feature | \[09:08\] Diego |
| Rate limiting de saída | Em aberto: "observar e decidir depois" | \[09:39\] Larissa |
| Escala para múltiplos workers e ordering global | Em aberto, limitação conhecida | \[09:13\] Diego e Larissa |
| Restringir o CRUD de configuração por role | Em aberto: "mais pra frente" | \[09:37\] Sofia |

**Métrica e prazo:** entrega em menos de 10 segundos é o que os clientes chamam de tempo real (\[09:02\] Marcos); prazo de 3 sprints, com 2 dias úteis de revisão de segurança (\[09:46\] Larissa e Sofia); a Atlas quer para o fim de novembro (\[09:45\] Marcos).

## Armadilhas e ganchos no código

**Sete pontos em que a IA tende a errar**

- **`customer_id` não vem do JWT.** Marcos diz que sim em \[09:31\], mas Larissa corrige em \[09:32\]: vai no body ou no path. A reunião não escolhe entre os dois, então o FDD precisa escolher um e o RFC registrar a pendência.
- **São 5 tentativas, não 3.** As 3 foram proposta do Bruno, descartada em \[09:16\].
- **A janela total de retry é "quase 15 horas"** (\[09:17\] Diego). O "12 ou 24 horas" de \[09:15\] foi uma estimativa anterior.
- **Ordering só por pedido e só com um worker.** Não existe garantia global (\[09:13\] Larissa).
- **O endpoint de rotação de secret não tem caminho definido na reunião.** O FDD pode propor um, mas o TRACKER deve apontar o requisito (\[09:21\] Sofia), não um caminho "decidido".
- **A latência mínima no pior caso é 2 s** pelo polling (\[09:10\] Larissa); a meta de produto é abaixo de 10 s. São números diferentes e os dois aparecem.
- **Nome dos ADRs.** O `docs/adrs/README.md` do repositório sugere `0001-titulo.md`, mas o enunciado exige `ADR-NNN-titulo-em-kebab-case.md`. Vale o enunciado.

**Caminhos reais para a seção "Integração com o sistema existente" (o mínimo é 4)**

| Arquivo | Como o módulo de webhooks se integra |
| --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` (linha 126) já roda dentro de `prisma.$transaction`; é ali que entra a chamada a `publishWebhookEvent(tx, …)`, usando o mesmo `TxClient` |
| `src/modules/orders/order.status.ts` | Fonte dos status e das transições válidas (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED), que definem os valores do filtro de eventos |
| `src/shared/errors/app-error.ts` e `http-errors.ts` | `AppError` e as subclasses (`NotFoundError`, `ConflictError`, `ValidationError` etc.) são a base dos erros `WEBHOOK_*` |
| `src/middlewares/error.middleware.ts` | Já trata `AppError`, Zod e Prisma; os erros novos passam por ele sem alteração |
| `src/middlewares/auth.middleware.ts` | `authenticate` protege o CRUD; `requireRole` (linha 49) restringe o replay a ADMIN |
| `src/middlewares/validate.middleware.ts` | `validate` com schemas Zod, incluindo a regra de URL `https` |
| `src/shared/logger/index.ts` | Logger Pino reutilizado pela API e pelo worker |
| `src/config/database.ts` | `createPrismaClient()` permite ao worker abrir a própria instância, no mesmo banco |
| `src/server.ts` e `package.json` | Modelo para a nova entry `src/worker.ts` e para o script `npm run worker` |
| `src/routes/index.ts` | `buildApiRouter` é onde o router de webhooks é registrado, como os outros módulos |
| `prisma/schema.prisma` | Recebe os novos modelos (configuração, outbox, dead letter, entregas), com IDs UUID |

Os documentos descrevem essas mudanças, mas você não as aplica: a entrega é só documental.

## O que vai em cada documento

Cada documento responde a uma pergunta diferente, e conteúdo repetido entre eles conta contra. A ordem de produção abaixo é a sugerida pelo enunciado.

| Ordem | Documento | Pergunta | O que entra | O que **não** entra |
| --- | --- | --- | --- | --- |
| 1 | ADRs | Por que decidimos assim? | Uma decisão por arquivo: contexto, decisão, alternativas, consequências | Contratos de API, fluxos passo a passo |
| 2 | RFC | O que propomos e o que está em aberto? | Visão geral, alternativas descartadas, questões em aberto, links para os ADRs | Payloads, tabelas de erro, detalhes de implementação |
| 3 | FDD | Como construir? | Fluxos, endpoints com exemplos, matriz `WEBHOOK_*`, resiliência, observabilidade, integração com o código | Justificativa de negócio, histórico da discussão |
| 4 | PRD | Por que e o quê? | Problema dos 3 clientes B2B, requisitos, métrica, escopo, riscos | Nomes de tabelas, código |
| 5 | TRACKER | De onde veio cada item? | Uma linha por requisito, decisão, restrição e trade-off | — |
| 6 | README | Como foi o processo? | Ferramentas, workflow, prompts, iterações | O enunciado original (pode virar link) |

**ADRs propostos (7, dentro do limite de 5 a 8)**

1. `ADR-001-outbox-no-mysql.md`
2. `ADR-002-worker-em-processo-separado-com-polling.md`
3. `ADR-003-retry-com-backoff-e-dlq.md`
4. `ADR-004-hmac-sha256-com-secret-por-endpoint.md`
5. `ADR-005-entrega-at-least-once-com-x-event-id.md`
6. `ADR-006-reuso-dos-padroes-do-projeto.md` (é o que cita arquivos reais do código)
7. `ADR-007-snapshot-do-payload-na-insercao.md` (opcional; pode ficar só no FDD)

**Endpoints para o FDD (o mínimo é 4; a reunião dá 7)**

criar, editar, remover e listar webhooks de um customer; `GET /webhooks/:id/deliveries`; `POST /admin/webhooks/dead-letter/:id/replay`; e a rotação de secret. Só os caminhos de deliveries e de replay foram ditos literalmente na reunião (\[09:34\] Marcos, \[09:35\] Diego). Os demais você define no FDD seguindo o padrão das rotas existentes.

**Códigos de erro citados na reunião:** `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` (\[09:28\] Bruno). Outros códigos da matriz, como o de payload acima de 64 KB, derivam de regras decididas e devem apontar para a fala correspondente no TRACKER.

## Workflow, prazo e entrega

**Workflow com IA (3 a 5 ciclos esperados)**

1. **Contexto.** Fork, clone e uma sessão de exploração: a IA lê o código e a transcrição e devolve um inventário em tabela (decisões, requisitos, descartes, adiamentos), cada linha com timestamp e falante. Confira contra o mapa acima.
2. **Geração por documento**, na ordem da tabela, sempre passando o inventário e os documentos já prontos como contexto.
3. **Revisão crítica a cada documento:** procurar as 7 armadilhas, conteúdo duplicado entre documentos e frases sem origem.
4. **TRACKER como auditoria.** Qualquer item sem "Localização" preenchível sai do documento ou é reescrito.
5. **Verificação final:** conferir que todo caminho de arquivo citado existe e que cada timestamp do TRACKER bate com a fala.

Anote as correções conforme acontecem, porque o README exige ≥ 2 iterações concretas e ≥ 2 prompts. Dois prompts que valem guardar: o de inventário da transcrição com timestamps e o de auditoria ("liste toda afirmação deste documento que não tem origem na transcrição ou no código").

**Prazo:** o cronograma macro reserva 21/10 a 03/11. Como não há código para escrever nem ambiente para subir, estimo de 5 a 7 dias de trabalho: 1 para contexto e inventário, 1 para os ADRs, 1 para o RFC, 2 para o FDD, 1 para PRD e TRACKER, 1 para README e revisão.

**Checklist de entrega**

- [ ] Fork público de `mba-ia-desafio-design-docs-com-ia`
- [ ] 5 a 8 ADRs em `docs/adrs/` no formato `ADR-NNN-titulo-em-kebab-case.md`, com Status, Contexto, Decisão, Alternativas e Consequências
- [ ] `docs/RFC.md` com 2 a 4 páginas, ≥ 2 alternativas, ≥ 2 questões em aberto e links para ≥ 2 ADRs
- [ ] `docs/FDD.md` com ≥ 4 endpoints, matriz `WEBHOOK_*`, observabilidade e "Integração com o sistema existente" (≥ 4 caminhos reais)
- [ ] `docs/PRD.md` com as 12 seções, ≥ 8 requisitos funcionais, métrica quantitativa, ≥ 2 itens fora de escopo e ≥ 2 riscos
- [ ] `docs/TRACKER.md` com ≥ 80% de cobertura, ≥ 70% das linhas da transcrição e ≥ 5 do código
- [ ] `README.md` substituído, com as 6 seções
- [ ] Nenhum arquivo de `src/`, `prisma/`, `tests/` ou a `TRANSCRICAO.md` alterado
- [ ] URL do fork enviada na área de entrega da plataforma

# Tracker de Rastreabilidade

Este tracker liga cada requisito, decisão, alternativa descartada, contrato, erro e integração dos documentos de design (PRD, RFC, FDD e ADRs) à origem dele. Quando a origem é a reunião técnica, a coluna Localização aponta a fala exata em `TRANSCRICAO.md` (`[hh:mm] Nome`). Quando a origem é o código existente, aponta o arquivo real que comprova o padrão ou o ponto de integração. Itens que são proposta do FDD, sem fala direta, apontam a fala que os motiva e o resumo começa com "proposta do FDD para".

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | Avisar o cliente em menos de 10 s após a mudança, para ser "tempo real" (meta 100% com cliente disponível) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Não piorar a operação de pedidos: mudança de status sem falha causada pela feature | TRANSCRICAO | [09:04] Bruno |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Não perder aviso: toda mudança de status confirmada gera aviso registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo | Entregar a feature para a Atlas até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo | Entregar com segurança: 3 sprints, com 2 dias úteis de revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Larissa |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar endereço de envio por cliente; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Editar um cadastro de endereço de envio | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Remover um cadastro de endereço de envio | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Listar os cadastros de endereço de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Escolher, por endereço, os status de pedido que geram aviso | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas por endereço (sucesso ou falha, conteúdo, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Reenvio manual de falha definitiva, só para ADMIN, com registro de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Rotacionar a secret pela API; a antiga continua válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Gerar aviso a cada mudança de status que interesse ao cliente | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Assinar cada envio para o cliente verificar origem e integridade | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Desempenho: aviso começa em menos de 10 s com cliente disponível; pior caso de polling de 2 s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Isolamento: mudança de status não espera nem falha por causa da resposta do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Consistência: status só fica confirmado junto com o registro do aviso | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Segurança: endereço só em https; http recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64 KB por aviso; acima disso, erro registrado (sem truncar) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Cliente que não responde em 10 s é tratado como falha de envio | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Resiliência: até 5 tentativas (1m, 5m, 30m, 2h, 12h), cobrindo cerca de 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Entrega pelo menos uma vez; duplicatas identificáveis pelo ID do evento | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Ordem garantida só por pedido e só com um único processo de envio | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Manutenibilidade: reusar padrões do projeto, sem bibliotecas novas | TRANSCRICAO | [09:30] Larissa |
| PRD-ESC-01 | `docs/PRD.md` | Fora de escopo | E-mail ao cliente quando as falhas se repetem (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | `docs/PRD.md` | Fora de escopo | Painel visual para o cliente acompanhar webhooks (projeto do time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-03 | `docs/PRD.md` | Fora de escopo | Webhooks de entrada (cliente enviando para a plataforma) | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-04 | `docs/PRD.md` | Fora de escopo | Arquivamento de entregas antigas (cerca de 30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-05 | `docs/PRD.md` | Fora de escopo | Ordem global e múltiplos processos de envio (limitação conhecida) | TRANSCRICAO | [09:13] Diego |
| PRD-ESC-06 | `docs/PRD.md` | Questão em aberto | Rate limiting de saída para o cliente: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-ESC-07 | `docs/PRD.md` | Questão em aberto | Restringir o cadastro por perfil de usuário; por enquanto, qualquer perfil autenticado | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Cliente não trata avisos repetidos e processa o mesmo evento duas vezes; mitigado por doc e ID do evento | TRANSCRICAO | [09:25] Diego |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Vazamento de secret do lado do cliente, como já ocorreu; chave por endpoint e rotação com 24 h | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Não cumprir o prazo da Atlas e perder o cliente; mitigado com 3 sprints e confirmação com o cliente | TRANSCRICAO | [09:47] Marcos |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Rajada de avisos para o cliente sem limite de envio; observar em produção | TRANSCRICAO | [09:38] Diego |
| PRD-RISCO-05 | `docs/PRD.md` | Risco | Cliente fora do ar por mais de 15 h; aviso vai para falhas definitivas e exige reenvio manual | TRANSCRICAO | [09:17] Marcos |
| PRD-RISCO-06 | `docs/PRD.md` | Risco | Mudança de status lenta ou com falha por causa da gravação do aviso; mitigado por transação única | TRANSCRICAO | [09:04] Bruno |
| PRD-RISCO-07 | `docs/PRD.md` | Risco | Ordem dos avisos se perde ao escalar para mais de um processo; limitação documentada | TRANSCRICAO | [09:13] Larissa |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Disparo HTTP síncrono dentro de changeStatus: cliente lento trava outras mudanças e não cabe rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams ou outra fila: mais infraestrutura para operar | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger do banco para acordar o worker: MySQL não tem LISTEN/NOTIFY | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Worker dentro do processo da API: um reinício derruba o worker | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Múltiplos workers em paralelo: perde a ordem por pedido | TRANSCRICAO | [09:12] Diego |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Retry indefinido com backoff: evento fica pendurado se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Três tentativas: não cobrem indisponibilidade de duas horas | TRANSCRICAO | [09:16] Bruno |
| RFC-ALT-08 | `docs/RFC.md` | Alternativa descartada | Secret global da plataforma: um vazamento compromete todos os clientes | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-09 | `docs/RFC.md` | Alternativa descartada | Exactly-once: coordenação complexa dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-QA-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída: 50 mudanças em um minuto geram 50 chamadas? Decisão: observar | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | `docs/RFC.md` | Questão em aberto | Escala e ordem global: particionamento por order_id ou lock fica para depois | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | `docs/RFC.md` | Questão em aberto | Restrição de role do CRUD: hoje qualquer role autenticada; endurecer depois | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-04 | `docs/RFC.md` | Questão em aberto | customer_id no body ou no path: reunião não escolheu; proposta do FDD para path /customers/:customerId/webhooks | TRANSCRICAO | [09:32] Larissa |
| RFC-QA-05 | `docs/RFC.md` | Questão em aberto | Caminho da rotação de secret: proposta do FDD para POST /webhooks/:id/rotate-secret, motivada pela carência de 24 h | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-06 | `docs/RFC.md` | Questão em aberto | Assinatura durante a carência: proposta do FDD para enviar duas assinaturas em X-Signature, pendente de revisão da Sofia | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-07 | `docs/RFC.md` | Questão em aberto | Proteção da secret em repouso: HMAC exige a secret em claro; não discutido na reunião, entra na revisão de segurança de 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox no MySQL existente, com registro na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker em processo separado, com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | 5 tentativas com backoff 1m, 5m, 30m, 2h e 12h; depois DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint e rotação com 24 h de carência | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | Entrega at-least-once com X-Event-Id para deduplicação do lado do cliente | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Filtro de status aplicado na inserção da outbox; sem linha se nenhum endpoint quer o status | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-01 | `docs/FDD.md` | Decisão | publishWebhookEvent(tx, ...) grava uma linha por endpoint na mesma transação; erro acima de 64 KB desfaz a mudança | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Decisão | Worker lê PENDING em ordem de created_at, com um processo; proposta do FDD para voltar PROCESSING a PENDING no start | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Decisão | Retry: envio inicial + 5 retentativas (1m, 5m, 30m, 2h, 12h, ~15 h); DLQ após a última falhar | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Decisão | Esgotado o retry, linha vai para webhook_dead_letter; proposta do FDD para replay reaproveitar o mesmo id | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | `docs/FDD.md` | Decisão | Rotação com nova e antiga secret por 24 h; proposta do FDD para enviar duas assinaturas durante a carência | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | E1 POST /api/v1/customers/:customerId/webhooks, 201; secret só na criação; proposta do FDD para o path | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | E2 GET listar webhooks do cliente, 200, sem secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | E3 PATCH editar url, statuses e active; proposta do FDD para bloquear secret no PATCH | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | E4 DELETE remover, 204; proposta do FDD para pendentes irem à DLQ | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | E5 GET deliveries das últimas 100 tentativas, mais recentes primeiro | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | E6 POST rotate-secret, 200, 409 durante a carência; proposta do FDD para o caminho e o 409 | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | E7 POST replay de DLQ, só ADMIN, 202 | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Envio ao cliente: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e corpo enxuto sem itens | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | `docs/FDD.md` | Erro | WEBHOOK_NOT_FOUND, 404, webhook inexistente ou de outro cliente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_URL, 400, URL malformada ou sem https | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_REQUIRED, 500, endpoint sem secret no envio; evento não sai sem assinatura | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | `docs/FDD.md` | Erro | WEBHOOK_PAYLOAD_TOO_LARGE, 422, payload acima de 64 KB; proposta do FDD para o código | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-05 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_ROTATION_IN_PROGRESS, 409, nova rotação durante a carência; proposta do FDD para o código | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-06 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND, 404, id inexistente na DLQ; proposta do FDD para o código | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-07 | `docs/FDD.md` | Erro | WEBHOOK_ALREADY_REPLAYED, 409, item da DLQ já reenviado; proposta do FDD para o código | TRANSCRICAO | [09:35] Diego |
| FDD-ERRO-08 | `docs/FDD.md` | Erro | FORBIDDEN, 403, replay por usuário que não é ADMIN (erro já existente) | TRANSCRICAO | [09:36] Sofia |
| FDD-ERRO-09 | `docs/FDD.md` | Erro | Falhas de envio (timeout, HTTP não 2xx, rede) só em log e lastError; proposta do FDD para os códigos | TRANSCRICAO | [09:42] Diego |
| FDD-RES-01 | `docs/FDD.md` | Decisão | Timeout de 10 s por chamada HTTP; estouro conta como falha e entra no retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Decisão | Polling de 2 s no worker, que define o atraso máximo de envio | TRANSCRICAO | [09:09] Diego |
| FDD-RES-03 | `docs/FDD.md` | Decisão | Worker em processo próprio, isolado do ciclo de vida da API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-04 | `docs/FDD.md` | Decisão | Duplicidade aceita; cliente deduplica pelo X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Requisito Não Funcional | Logs estruturados com Pino (webhook_delivery_*, webhook_dlq_replayed) e redaction de secret; proposta do FDD | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Requisito Não Funcional | Métricas derivadas de logs (latência p95, outbox pendente) e rastro por eventId; proposta do FDD, sem lib de métricas | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-01 | `docs/FDD.md` | Integração | changeStatus chama publishWebhookEvent no mesmo tx, antes do findUnique final | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | OrderStatus e canTransition definem o domínio válido de statuses; sem mudança | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | AppError é a base das classes WEBHOOK_* | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Subclasses de erro (InsufficientStockError etc.) servem de molde para os erros WEBHOOK_* | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Error middleware já trata AppError; sem mudança para o módulo | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | requireRole reaproveitado para restringir o replay a ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | validate com schemas Zod para params e body (URL https, statuses) | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | buildApiRouter ganha webhooks e três montagens (customers, webhooks, admin) | CODIGO | `src/routes/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | buildControllers instancia repository, service e controller de webhooks | CODIGO | `src/app.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Worker chama createPrismaClient para ter instância própria no mesmo banco | CODIGO | `src/config/database.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | server.ts é o molde de bootstrap e shutdown do futuro src/worker.ts | CODIGO | `src/server.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | redactPaths do logger recebe os paths de secret e previousSecret | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-13 | `docs/FDD.md` | Integração | Novos modelos Prisma (outbox, endpoints, DLQ, deliveries) e relação em Customer | CODIGO | `prisma/schema.prisma` |
| FDD-INT-14 | `docs/FDD.md` | Integração | Factories de teste ganham createWebhookEndpoint | CODIGO | `tests/helpers/factories.ts` |
| ADR-001-DEC | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Padrão outbox no MySQL existente; linha gravada na mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-1 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | HTTP síncrono dentro do changeStatus: cliente lento trava a transação | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-2 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Streams: infraestrutura nova, overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-002-DEC | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão | Polling a cada 2 s em processo separado da API, com um único worker | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-1 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Alternativa descartada | Trigger do banco: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-2 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Alternativa descartada | Worker dentro da API: reinício da API derruba o worker | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-3 | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Alternativa descartada | Múltiplos workers: perde a ordem por pedido; escala futura por particionamento | TRANSCRICAO | [09:13] Diego |
| ADR-003-DEC | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas com backoff 1m, 5m, 30m, 2h e 12h; timeout de 10 s; DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-1 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | 3 tentativas: não cobrem indisponibilidade de duas horas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-2 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | Retry indefinido: evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-3 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | Marcar failed na própria outbox: leitura pior e menos evidência para debug | TRANSCRICAO | [09:18] Diego |
| ADR-004-DEC | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h de carência | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-1 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret global da plataforma: um vazamento afeta todos | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-2 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Rotação sem carência: cliente sem tempo para migrar os sistemas | TRANSCRICAO | [09:21] Sofia |
| ADR-005-DEC | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | Entrega at-least-once com X-Event-Id (UUID gerado na outbox) para deduplicação | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-1 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once: exige coordenação complexa dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo: módulo webhooks, AppError, Pino, Zod, requireRole e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-1 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Alternativa descartada | Infraestrutura própria (logger, formato de erro): duplicaria o que já existe | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-2 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Alternativa descartada | Injetar o repository de webhooks no OrderService: acoplamento; função com tx basta | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-3 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Alternativa descartada | ID auto incremental na outbox: foge do padrão UUID do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-006-PAD-1 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Módulo por domínio com controller, service, repository, routes e schemas | CODIGO | `src/modules/orders/order.controller.ts` |
| ADR-006-PAD-2 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Router de pedidos com authenticate aplicado no próprio router | CODIGO | `src/modules/orders/order.routes.ts` |
| ADR-006-PAD-3 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Schemas Zod por domínio, reutilizados no módulo de webhooks | CODIGO | `src/modules/orders/order.schemas.ts` |
| ADR-006-PAD-4 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Serviço de cliente segue o mesmo molde de módulo | CODIGO | `src/modules/customers/customer.service.ts` |
| ADR-006-PAD-5 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | requestLogger com X-Request-Id, usado para correlação de logs | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| ADR-006-PAD-6 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | IDs CHAR(36) UUID nas tabelas da migração inicial | CODIGO | `prisma/migrations/20260519182739_init/migration.sql` |
| ADR-007-DEC | `docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md` | Decisão | Payload renderizado na inserção (snapshot), sem itens do pedido | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-1 | `docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md` | Alternativa descartada | Guardar só order_id e renderizar no envio: evento mostraria estado diferente do da transição | TRANSCRICAO | [09:52] Larissa |
| FDD-DADOS-01 | `docs/FDD.md` | Decisão | eventId é um UUID por evento, repetido nas linhas de cada endpoint (X-Event-Id único por evento) | TRANSCRICAO | [09:25] Diego |
| FDD-ERRO-INV-URL | `docs/FDD.md` | Restrição | Regra https checada no service para emitir WEBHOOK_INVALID_URL, pois validate() converte ZodError em VALIDATION_ERROR | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-ERRO-ORIGEM | `docs/FDD.md` | Decisão | Códigos citados na reunião: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |

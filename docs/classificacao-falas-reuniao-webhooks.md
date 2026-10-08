# Classificação das falas — Reunião de Webhooks

Data de referência: 2026-10-08. Fonte: `TRANSCRICAO.md`.

Critérios:
- **Definido:** requisito, fato ou decisão aceito pelo grupo.
- **Negado:** alternativa ou proposta rejeitada, excluída do escopo ou corrigida depois.
- **Adiado:** pergunta em aberto, item para fase futura ou fala de transição/cortesia.

Os trechos estão na íntegra. Cada linha traz horário e falante.

---

## Atividades da feature

Decisões da classificação "Definido" que viram trabalho de implementação. Cada item aponta o horário da decisão na transcrição.

| # | Atividade | Decisão de origem | Onde está | Validação |
|---|-----------|-------------------|-----------|-----------|
| 1 | Criar tabela `webhook_outbox` e inserir o evento na mesma transação de `changeStatus` | 09:06, 09:40, 09:48 | [ADR-001](adrs/ADR-001-outbox-no-mysql.md):18; [FDD](FDD.md):140-159; [FDD](FDD.md):483 | Contemplado |
| 2 | Criar função `publishWebhookEvent(tx, order, fromStatus, toStatus)` e chamá-la em `order.service` | 09:41 | [FDD](FDD.md):166; [FDD](FDD.md):449; [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md):36 | Contemplado |
| 3 | Filtrar por status na inserção da outbox; não inserir se nenhum webhook do cliente quer o status | 09:34 | [FDD](FDD.md):167; [FDD](FDD.md):484; [RFC](RFC.md):35 | Contemplado |
| 4 | Gravar o payload renderizado (snapshot) no momento da inserção | 09:52 | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md):14; [FDD](FDD.md):168; [PRD](PRD.md):120 | Contemplado |
| 5 | Criar entry-point `src/worker.ts` e script `npm run worker`, em processo separado da API | 09:11, 09:28 | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md):15; [FDD](FDD.md):173, 442, 460, 477 | Contemplado |
| 6 | Implementar polling de 2 segundos com batch pequeno e marcação de status | 09:09, 09:10 | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md):14; [ADR-001](adrs/ADR-001-outbox-no-mysql.md):21; [FDD](FDD.md):176 | Contemplado |
| 7 | Usar instância própria de `PrismaClient` no worker | 09:30 | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md):16; [FDD](FDD.md):459 | Contemplado |
| 8 | Implementar retry com 5 tentativas e backoff 1m/5m/30m/2h/12h | 09:17 | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md):14; [FDD](FDD.md):194-205, 399; [PRD](PRD.md):107 | Contemplado |
| 9 | Timeout de 10 segundos no HTTP do worker; falha vai para retry | 09:42 | [FDD](FDD.md):184, 398; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md):15; [PRD](PRD.md):106 | Contemplado |
| 10 | Criar tabela `webhook_dead_letter` com payload, motivo e timestamp | 09:18 | [FDD](FDD.md):97-110; [FDD](FDD.md):211; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md):16 | Contemplado |
| 11 | Criar endpoint `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role ADMIN, com log de auditoria | 09:18, 09:35, 09:36 | [FDD](FDD.md):209-214, 242 (E7), 332-340, 421; [PRD](PRD.md):92 | Contemplado |
| 12 | Gerar secret única por endpoint e assinar o body com HMAC-SHA256 | 09:21, 09:22 | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md):14-15; [FDD](FDD.md):366; [PRD](PRD.md):117 | Contemplado |
| 13 | Criar endpoint de rotação de secret, com a antiga válida por 24 horas | 09:21, 09:22 | [FDD](FDD.md):218-222, 241, 320-330; [PRD](PRD.md):93; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md):17 | Contemplado com ressalva: caminho do endpoint e mecânica da carência são propostas do FDD; [RFC](RFC.md):65-66 os marca como aberto |
| 14 | Validar URL com `https` no schema Zod e retornar erro de validação para `http` | 09:23 | [FDD](FDD.md):270, 377; [PRD](PRD.md):104; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md):18 | Contemplado |
| 15 | Recusar eventos com payload acima de 64KB, com erro | 09:24 | [FDD](FDD.md):169, 379; [PRD](PRD.md):105 | Contemplado |
| 16 | Enviar `X-Event-Id` (UUID único por evento), `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type` | 09:25, 09:44, 09:45 | [FDD](FDD.md):345-350, 365; [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md):15 | Contemplado |
| 17 | Montar payload com `event_id`, `event_type`, timestamp ISO 8601, ids e campos básicos da order, sem `items` | 09:43, 09:34 | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md):16; [FDD](FDD.md):357-361 | Contemplado |
| 18 | Implementar CRUD de configuração: `POST`, `PATCH`, `DELETE` e `GET` por customer, com autenticação normal | 09:31, 09:33, 09:32 | [FDD](FDD.md):236-239; [PRD](PRD.md):86 (RF-01) | Contemplado |
| 19 | Implementar `GET /webhooks/:id/deliveries` com histórico de entregas | 09:34 | [FDD](FDD.md):240, 300, 492 | Contemplado |
| 20 | Criar módulo `src/modules/webhooks` seguindo o padrão dos demais módulos | 09:27, 09:30 | [FDD](FDD.md):467-477; [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md):30; [RFC](RFC.md):41 | Contemplado |
| 21 | Reusar `AppError`, Pino e error middleware, com códigos prefixados por `WEBHOOK_` | 09:28, 09:29, 09:30 | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md):31-32; [FDD](FDD.md):377-392; [RFC](RFC.md):41 | Contemplado |
| 22 | Documentar a garantia at-least-once e a deduplicação por `X-Event-Id` no portal de desenvolvedor | 09:26, 09:40 | [PRD](PRD.md):68, 126, 157; [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md):16 | Contemplado |
| 23 | Reservar dois dias úteis para revisão de segurança de Sofia antes do deploy | 09:46, 09:49 | [RFC](RFC.md):77; [FDD](FDD.md):443; [PRD](PRD.md):53; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md):19 | Contemplado |
| 24 | Entregar em três sprints, com prazo para o fim de novembro (exigido pela Atlas) | 09:45, 09:47 | [RFC](RFC.md):21, 78; [PRD](PRD.md):52-53; [TRACKER](TRACKER.md):10-11 | Contemplado |

---

## Definido

Requisitos, fatos e decisões aceitos pelo grupo.

- [09:00] Larissa: "Vou começar mesmo faltando o Diego, ele avisou que tá saindo de uma call do time de infra, deve cair aqui em 5 minutos."
- [09:00] Marcos: "Então, a gente recebeu na semana passada um pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo. Os três querem ser notificados em tempo real quando o status dos pedidos deles muda na nossa plataforma. Hoje eles ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles. A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente."
- [09:02] Marcos: "Pra eles, qualquer coisa abaixo de 10 segundos já é "tempo real". O importante é que não fique pendurado e eles tenham que ficar atualizando manualmente."
- [09:02] Marcos: "Só saindo da gente pra eles."
- [09:03] Sofia: "Beleza. Então é outbound webhook, fica mais simples."
- [09:04] Bruno: "A transação de mudança de status hoje já é pesada — atualiza orders, insere na order_status_history, decrementa stock_quantity dos produtos do pedido."
- [09:06] Diego: "Aliás, eu nem chamaria de "fila" — o que a gente quer aqui é um padrão outbox."
- [09:06] Diego: "Outbox é o seguinte: quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. Um worker separado fica lendo essa tabela e disparando as chamadas HTTP. Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível."
- [09:07] Larissa: "Faz sentido."
- [09:07] Diego: "Exato, e a gente é um time pequeno."
- [09:07] Diego: "Outbox no MySQL existente resolve."
- [09:08] Diego: "A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em created_at. Worker lê só os pendentes em batch pequeno, processa, marca como entregue."
- [09:08] Larissa: "Tá decidido então: outbox em MySQL."
- [09:09] Diego: "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca."
- [09:09] Diego: "Polling de 2 segundos atende o requisito de "abaixo de 10 segundos" tranquilo."
- [09:10] Marcos: "2 segundos serve, perfeito."
- [09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos."
- [09:11] Diego: "Uma coisa importante: o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker."
- [09:11] Larissa: "Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script "npm run worker"."
- [09:11] Bruno: "Pode ser, mas vai precisar conectar no mesmo banco"
- [09:11] Diego: "Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo."
- [09:12] Diego: "Se a gente tem um único worker rodando, ele processa em ordem de created_at do outbox. Aí o cliente recebe em ordem. Por enquanto, single-worker e ordering implícita por order_id."
- [09:13] Larissa: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker."
- [09:15] Diego: "Backoff exponencial. Tenta de novo depois de algum tempo, vai aumentando o intervalo, e depois de um teto de tentativas considera falha permanente e move pra DLQ."
- [09:15] Diego: "Eu sugiro 5."
- [09:15] Diego: "Cinco já dá pra cobrir uma janela de até 12 ou 24 horas."
- [09:16] Larissa: "Cinco fica bom."
- [09:17] Diego: "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa."
- [09:17] Marcos: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."
- [09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."
- [09:18] Diego: "Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."
- [09:18] Bruno: "Faz sentido."
- [09:18] Diego: "Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente."
- [09:19] Larissa: "Anota também. Endpoint admin pra replay manual de DLQ."
- [09:19] Sofia: "Primeira coisa: a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio."
- [09:20] Sofia: "Padrão é HMAC. A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele."
- [09:20] Sofia: "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."
- [09:21] Sofia: "Outra coisa importante: cada endpoint de webhook do cliente tem que ter uma secret única."
- [09:21] Sofia: "Sim. E a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre."
- [09:22] Diego: "Isso é importante. A gente já teve cliente que vazou secret em log de aplicação dele uma vez."
- [09:22] Sofia: "Pois é. Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."
- [09:23] Sofia: "TLS obrigatório. URL do webhook tem que ser https. Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod."
- [09:23] Sofia: "Se por algum motivo o evento tiver 500KB, a gente não envia."
- [09:23] Sofia: "Eu sou a favor de erra. Se chegou nesse tamanho, tem algo errado."
- [09:24] Diego: "Acho que 64KB já é um teto generoso. Nenhum evento nosso vai chegar perto disso."
- [09:24] Larissa: "64KB de limite, erro caso ultrapasse. Anotado, mas não vejo como decisão arquitetural separada, é só requisito não funcional."
- [09:24] Diego: "Voltando à entrega: a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."
- [09:25] Diego: "A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele."
- [09:25] Sofia: "Isso joga responsabilidade pro cliente."
- [09:25] Diego: "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim."
- [09:25] Diego: "At-least-once com event_id resolve 99% dos casos."
- [09:26] Marcos: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema."
- [09:26] Larissa: "Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."
- [09:27] Bruno: "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual. Vou propor uma pasta src/modules/webhooks com toda a estrutura."
- [09:28] Diego: "Faz."
- [09:28] Bruno: "Eu colocaria src/worker.ts como entry separada, e a lógica de processamento fica num arquivo dentro do módulo, tipo src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts."
- [09:28] Diego: "Beleza."
- [09:28] Bruno: "Sobre erros: a gente já tem um padrão. Tem classe AppError, classes específicas tipo InsufficientStockError, InvalidStatusTransitionError. Todas usam código tipo INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION. Quero seguir igual pra webhook. Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc."
- [09:29] Larissa: "Prefixo WEBHOOK_ pra tudo do módulo."
- [09:29] Bruno: "E o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo. O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada."
- [09:29] Diego: "Sobre infraestrutura compartilhada: o pool de conexão do Prisma já tá lá."
- [09:30] Bruno: "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node."
- [09:30] Larissa: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."
- [09:31] Marcos: "O cliente precisa cadastrar webhook. Endpoint POST. Campos: url, secret é gerada pela gente e devolvida na criação. Lista de status que ele quer receber."
- [09:32] Marcos: "Pela nossa API direto, autenticado com JWT do nosso sistema. A gente tem usuários que representam o cliente."
- [09:32] Larissa: "Então é endpoint autenticado normal, e o customer_id é passado no body ou no path."
- [09:33] Bruno: "Anotado. Continuando: PATCH pra editar, DELETE pra remover, GET pra listar os webhooks de um customer. Por endpoint a gente pode escolher quais eventos receber."
- [09:33] Marcos: "Sim. Filtro de eventos é uma lista dos status que o webhook quer ouvir. Tipo "só quero saber quando vira SHIPPED e DELIVERED", e a gente filtra na hora de inserir na outbox."
- [09:34] Bruno: "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela."
- [09:34] Diego: "Concordo."
- [09:34] Marcos: "Mais um: o cliente precisa conseguir ver o histórico de entregas. Tipo "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta". GET /webhooks/:id/deliveries."
- [09:35] Larissa: "Anotado. Esse endpoint admin pra reprocessar DLQ que o Diego falou."
- [09:35] Diego: "Sim. POST /admin/webhooks/dead-letter/:id/replay."
- [09:36] Sofia: "Tem que ser ADMIN sim. Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria."
- [09:36] Larissa: "Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe."
- [09:37] Sofia: "Por enquanto sim."
- [09:40] Marcos: "Ok. Eu documento no portal pros clientes saberem como integrar via API."
- [09:40] Bruno: "Sobre integração com o código atual: a alteração crítica é dentro do service de orders, no método changeStatus. Hoje a transação faz update na order, insere no history e atualiza estoque. A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair."
- [09:41] Diego: "Essencial."
- [09:41] Bruno: "Vou propor uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual. Aí o order.service chama isso."
- [09:41] Diego: "Boa, função pura recebendo o tx."
- [09:42] Diego: "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry."
- [09:42] Sofia: "Bom."
- [09:43] Diego: "JSON com event_id, event_type tipo "order.status_changed", timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, e os campos básicos da order tipo total_cents. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois."
- [09:44] Bruno: "Bom, mantém payload enxuto."
- [09:44] Diego: "X-Event-Id com o UUID, X-Signature com o HMAC, X-Timestamp com o timestamp do envio (pra cliente conseguir detectar replay attack se quiser), Content-Type application/json."
- [09:44] Sofia: "Adiciona um X-Webhook-Id também, com o id do endpoint webhook, pra cliente que tem vários conseguir saber qual cadastro caiu naquele envio."
- [09:45] Diego: "Boa, X-Webhook-Id também."
- [09:45] Marcos: "A Atlas quer pra fim de novembro."
- [09:46] Larissa: "Modelagem de outbox e DLQ é uma sprint. Worker e retry é uma sprint. CRUD de configuração e deliveries é meio sprint. Integração no order.service e testes ponta a ponta é mais meio. HMAC, schemas, validações, mais um pouco. Eu chuto três sprints incluindo revisão da Sofia."
- [09:46] Sofia: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma."
- [09:47] Larissa: "Combinado. Três sprints com a revisão da Sofia incluída no fim."
- [09:47] Marcos: "Atlas vai gostar. Eu confirmo prazo com eles."
- [09:48] Larissa: "Padrão outbox no MySQL, transação atômica com mudança de status. Worker separado em polling de 2 segundos. Retry com backoff exponencial 1m/5m/30m/2h/12h, total 5 tentativas, depois DLQ persistida em tabela separada. HMAC-SHA256 sobre payload, secret por endpoint, rotação com grace period de 24h. Idempotência por X-Event-Id, garantia at-least-once. Padrões do projeto reaproveitados: AppError, Pino, error middleware, módulo em src/modules/webhooks, prefixo WEBHOOK_ nos códigos de erro. Endpoints CRUD de configuração autenticados normal, endpoint de replay de DLQ exige role ADMIN. Prazo três sprints."
- [09:49] Diego: "Tá fechado."
- [09:49] Bruno: "Pra mim ok."
- [09:49] Sofia: "Ok, só não esqueçam de me agendar pra revisão de segurança antes de subir."
- [09:49] Marcos: "Tá bom. Eu atualizo os clientes hoje à tarde."
- [09:50] Larissa: "Beleza. Eu vou abrir o doc de design da feature e marcar uma sessão pro Bruno e o Diego revisarem comigo antes da gente começar a codar."
- [09:51] Larissa: "UUID, segue o padrão do resto do projeto. Tudo é uuid."
- [09:51] Diego: "Beleza, só queria confirmar."
- [09:52] Larissa: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou."
- [09:52] Diego: "Concordo, snapshot na inserção."
- [09:52] Bruno: "Beleza, snapshot. Decidido."
- [09:53] Diego: "Por mim tá bom."
- [09:53] Bruno: "Bom."

---

## Negado

Alternativas rejeitadas, excluídas do escopo ou corrigidas depois.

- [09:04] Bruno: "Síncrono não rola."
- [09:04] Bruno: "Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos."
- [09:04] Bruno: "Sem falar que se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá."
- [09:04] Larissa: "Concordo. Pensei a mesma coisa."
- [09:06] Diego: "Síncrono está fora de questão."
- [09:07] Larissa: "A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra."
- [09:07] Diego: "Subir Redis Cluster pra isso é overengineering."
- [09:09] Diego: "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres."
- [09:09] Diego: "Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL."
- [09:09] Diego: "Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito."
- [09:11] Bruno: "usar o mesmo Prisma client." *(revisto em 09:30, quando o Bruno defendeu PrismaClient separado)*
- [09:13] Marcos: "Os clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou."
- [09:15] Diego: "Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu."
- [09:16] Bruno: "3 não é melhor? Mais agressivo."
- [09:16] Diego: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada."
- [09:21] Sofia: "Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."
- [09:26] Diego: "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo."
- [09:40] Diego: "Se ficar fora da transação, perde a garantia toda."
- [09:41] Bruno: "Vai me obrigar a passar um repository do webhook pro OrderService ou uma função de "enqueue event"."
- [09:41] Diego: "Não precisa injetar repository inteiro."
- [09:43] Diego: "Não manda items pra não inflar."
- [09:31] Marcos: "Customer_id implícito do JWT." *(corrigido em 09:32)*
- [09:32] Bruno: "Espera, mas o JWT atual é do usuário operador, não do cliente."
- [09:32] Larissa: "Não vem do JWT."
- [09:39] Diego: "Eu acho que não." *(rate limiting de envio fora do escopo)*
- [09:40] Larissa: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend." *(dashboard visual)*
- [09:48] Larissa: "Dashboard visual fora de escopo." *(resumo final)*
- [09:51] Larissa: "Senão tem caso esquisito." *(justifica não renderizar o payload no envio)*

---

## Adiado

Perguntas em aberto, itens para fase futura e falas de transição ou cortesia.

- [09:00] Larissa: "Bom dia, gente."
- [09:00] Larissa: "Marcos, abre pra gente o contexto."
- [09:02] Marcos: "Eu perguntei especificamente isso."
- [09:02] Sofia: "Antes de a gente entrar em arquitetura, posso fazer uma pergunta de escopo? Os webhooks vão sair só do nosso sistema pra eles, ou eles também enviam pra gente?"
- [09:03] Larissa: "Acho que a primeira pergunta é: a gente dispara isso sincronamente no service de orders quando o status muda, ou faz algum tipo de fila/outbox?"
- [09:04] Bruno: "Pra quem não tá familiarizado, explica rápido?"
- [09:05] Diego entrou na call.
- [09:05] Diego: "E aí, desculpa o atraso, a call de infra encaixou em cima."
- [09:05] Larissa: "Tranquilo, a gente tá no começo. Estamos definindo se webhook vai ser síncrono dentro do service de orders ou se vai pra alguma fila. O Bruno tava argumentando contra síncrono."
- [09:08] Bruno: "E performance? Se acumular muito evento na tabela, o worker não fica lento?"
- [09:08] Diego: "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature."
- [09:08] Larissa: "Próximo ponto: como o worker lê isso?"
- [09:09] Bruno: "Não dá pra usar trigger do banco pra ser mais reativo?"
- [09:12] Larissa: "Agora, ordering. Se o pedido X muda PAID, depois PROCESSING, depois SHIPPED em sequência rápida, o cliente recebe na ordem certa?"
- [09:12] Diego: "Depende."
- [09:12] Diego: "Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia."
- [09:13] Bruno: "E se algum dia a gente quiser escalar?"
- [09:13] Diego: "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora."
- [09:14] Larissa: "Vamos pra retry. Se o cliente tá offline, o que a gente faz?"
- [09:15] Bruno: "Quantas tentativas?"
- [09:16] Larissa: "Qual a progressão do backoff?"
- [09:17] Larissa: "Próximo: DLQ. Faz numa tabela separada ou marca como "failed" na própria outbox?"
- [09:18] Bruno: "E quem reprocessa? Tem endpoint?"
- [09:19] Larissa: "Sofia, daqui pra frente vai entrar tua parte. Segurança."
- [09:20] Bruno: "HMAC com qual algoritmo?"
- [09:21] Bruno: "Então a tabela de configuração de webhook armazena url + secret + customer_id + estado ativo?"
- [09:23] Larissa: "Mais alguma coisa de segurança?"
- [09:23] Sofia: "Trunca? Erra?"
- [09:25] Bruno: "E como ele diferencia?"
- [09:36] Larissa: "Quem é admin? Tem que ser role ADMIN do JWT?"
- [09:37] Marcos: "O resto do CRUD de configuração de webhook pode ser qualquer role autenticada?"
- [09:37] Sofia: "Mais pra frente a gente pode endurecer."
- [09:37] Marcos: "Última pergunta de requisito. Tem como avisar o cliente quando o webhook dele tá com problema? Tipo se ele falhou 3 vezes seguidas, mandar email pra ele."
- [09:37] Larissa: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto."
- [09:38] Marcos: "Beleza, anotado como "futuro"."
- [09:38] Diego: "Outra coisa que ficou na minha cabeça: rate limiting de envio pra cliente. Se o cliente tem 50 pedidos mudando de status em um minuto, a gente bombardeia ele com 50 chamadas?"
- [09:39] Larissa: "Boa pergunta. Faz parte do escopo?"
- [09:39] Diego: "A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto."
- [09:39] Larissa: "Tá. Fica como "observar e decidir depois"."
- [09:39] Marcos: "Dashboard visual? Tipo painel pro cliente ver os webhooks dele?"
- [09:43] Marcos: "E qual o formato do payload?"
- [09:44] Marcos: "Os headers que vão no request?"
- [09:44] Diego: "Mais alguma sugestão?"
- [09:45] Larissa: "Tá fechando bem. Marcos, sobre prazo?"
- [09:45] Marcos: "Larissa, dá em quantos sprints?"
- [09:46] Larissa: "Vou estimar."
- [09:47] Larissa: "Resumo rápido pra todo mundo confirmar antes da gente se despedir:"
- [09:48] Larissa: "Email como fallback fica pra próxima fase. Rate limiting de saída a gente observa. Algo errado ou faltando?"
- [09:50] Larissa: "Valeu, gente."
- [09:50] Marcos saiu.
- [09:50] Sofia saiu.
- [09:50] Diego: "Larissa, posso só te falar uma coisa rapidinho?"
- [09:50] Larissa: "Pode."
- [09:51] Diego: "Quando a gente for modelar a outbox, prefere id auto incremental ou UUID?"
- [09:51] Bruno: "Eu também tenho uma dúvida última: o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?"
- [09:52] Larissa: "Boa pergunta."
- [09:53] Larissa: "Mais alguma? Senão a gente fecha."
- [09:53] Larissa: "Falou. Até."
- [09:53] Fim da call.

---

## Observações

- Cada fala foi classificada pelo desfecho na reunião, não pelo conteúdo isolado. Por isso, um mesmo falante pode aparecer em mais de uma categoria.
- Falas de cortesia e transição ("Bom dia", "Até", entradas e saídas) ficaram em **Adiado**, porque não resolvem nenhum ponto.
- "Não, agora não" (dashboard, 09:40) ficou em **Negado**, porque a fala diz que o painel é projeto separado. O email (09:37) ficou em **Adiado**, porque a própria fala aponta para a próxima fase.
- A linha "usar o mesmo Prisma client" (09:11) está em **Negado** e referenciada a 09:30, quando a decisão de PrismaClient separado foi fechada.
- Excluídos da classificação: o cabeçalho (data, duração, local, participantes), que não é fala.

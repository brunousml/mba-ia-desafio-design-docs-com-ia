# RFC — Notificação de mudança de status de pedidos por webhook

## Metadados

- **Autor:** Larissa (Tech Lead)
- **Status:** Em revisão
- **Data:** quinta-feira da reunião técnica
- **Revisores:** Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança)
- **Documentos relacionados:** [FDD](FDD.md) (detalhes de implementação), [ADRs](#decisões-relacionadas) (decisões individuais)

## Resumo executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pedem para ser notificados quando o status dos seus pedidos muda, em vez de consultar `GET /orders` em loop ([09:00] Marcos). Propomos um sistema de webhooks **somente de saída**, com:

- registro do evento numa **outbox no MySQL existente**, na mesma transação da mudança de status ([09:06] Diego, [09:08] Larissa);
- um **worker em processo separado**, com polling a cada 2 segundos ([09:09] Diego, [09:10] Larissa);
- **5 tentativas** de entrega com backoff de 1m/5m/30m/2h/12h e, depois disso, uma **dead letter** para reprocessamento manual ([09:17] Larissa);
- assinatura **HMAC-SHA256** por endpoint, com rotação de secret e 24 h de carência ([09:22] Sofia);
- entrega **at-least-once**, com `X-Event-Id` para deduplicação do lado do cliente ([09:26] Larissa).

Prazo estimado: três sprints, incluindo revisão de segurança da Sofia ([09:46] Larissa). A Atlas quer a entrega até o fim de novembro ([09:45] Marcos).

## Contexto e problema

Hoje os clientes consultam `GET /orders` periodicamente para saber se algo mudou. Isso deixa a integração lenta e cara para eles, e a Atlas sugeriu migrar para um concorrente se a funcionalidade não sair até o fim do trimestre ([09:00] Marcos).

Para o time, o desafio técnico é registrar o evento **sem** pesar a transação de pedidos. A mudança de status já atualiza `orders`, insere em `order_status_history` e decrementa o estoque. Um HTTP call nesse trecho deixaria qualquer cliente lento travar mudanças de status de outros pedidos ([09:04] Bruno). A mudança acontece em `changeStatus`, no arquivo `src/modules/orders/order.service.ts`.

O requisito de latência do produto é "tempo real", que para os clientes significa menos de 10 segundos ([09:02] Marcos).

## Proposta técnica

**Visão geral.** Quando o status de um pedido muda, a própria transação registra o evento na outbox. Essa transação já é a que muda o status, então o evento existe se e somente se a mudança aconteceu ([09:40] Bruno, [09:41] Diego). Um processo separado lê os eventos pendentes, envia para os endpoints cadastrados pelo cliente e controla as tentativas.

**Cadastro e configuração.** O cliente cadastra endpoints pela nossa API, autenticado como usuário que representa o cliente. Ele escolhe os status que quer receber, e o filtro é aplicado na inserção: se nenhum endpoint do cliente quer um status, nem gravamos o evento ([09:33] Marcos, [09:34] Bruno). A secret é gerada pela plataforma e devolvida na criação ([09:31] Marcos).

**Entrega.** Cliente que não responde em 10 segundos é tratado como falha e vai para retry ([09:42] Diego). O evento guarda o estado do pedido no momento da mudança, e não o estado no envio ([09:52] Larissa). O payload é enxuto e não inclui itens do pedido ([09:43] Diego).

**Segurança.** Cada envio é assinado com HMAC-SHA256 sobre o corpo, com uma secret por endpoint ([09:21] Sofia). URLs precisam ser `https` ([09:23] Sofia). A rotação pela API mantém a secret antiga válida por 24 horas ([09:21] Sofia). Replay manual da dead letter exige role `ADMIN` e registra quem executou ([09:36] Sofia).

**Reuso do projeto.** O módulo segue a estrutura dos demais em `src/modules/` e reaproveita `AppError`, o logger Pino, o middleware de erro, os schemas Zod e o `requireRole` existente. Os códigos de erro usam o prefixo `WEBHOOK_` ([09:30] Larissa). O worker abre o próprio cliente do Prisma, no mesmo banco ([09:30] Bruno).

**Escopo.** Só webhooks de saída. Dashboard, e-mail de aviso por falhas repetidas e webhooks de entrada ficam fora ([09:02] Marcos, [09:37] Larissa, [09:40] Larissa). O arquivamento de linhas entregues (cerca de 30 dias) também fica fora desta feature ([09:08] Diego).

## Alternativas consideradas

Todas foram discutidas na reunião e descartadas.

- **Disparo HTTP síncrono dentro de `changeStatus`** ([09:04] Bruno). Trade-off descartado: simplicidade contra acoplamento. Um cliente lento travaria outras mudanças de status, e não faz sentido dar rollback na mudança por causa de um cliente fora do ar ([09:04] Bruno).
- **Redis Streams ou outra fila** ([09:07] Larissa). Trade-off descartado: ferramenta dedicada contra mais infraestrutura para operar. Diego considerou Redis Cluster overengineering para um time pequeno ([09:07] Diego).
- **Trigger do banco para acordar o worker** ([09:09] Bruno). Trade-off descartado: reatividade contra improviso. MySQL não tem LISTEN/NOTIFY, e a trigger não avisa processo externo ([09:09] Diego).
- **Worker dentro do processo da API** ([09:11] Diego). Trade-off descartado: menos um processo para operar contra acoplamento de ciclo de vida. Um reinício da API derrubaria o worker.
- **Múltiplos workers em paralelo** ([09:12] Diego, [09:13] Diego). Trade-off descartado: vazão contra ordering por pedido. Fica como evolução futura, com particionamento por `order_id` ou lock pessimista.
- **Retry indefinido com backoff** ([09:15] Diego). Trade-off descartado: insistência contra eventos pendurados para sempre se o cliente sumiu.
- **Três tentativas** ([09:16] Bruno). Trade-off descartado: falha mais rápida contra cobertura. Três tentativas em 30 minutos não cobrem uma indisponibilidade de duas horas, como a de um cliente nosso em manutenção planejada ([09:16] Diego).
- **Secret global da plataforma** ([09:21] Sofia). Trade-off descartado: simplicidade de gestão contra isolamento. Se uma secret vaza, vaza tudo.
- **Exactly-once** ([09:25] Diego). Trade-off descartado: garantia mais forte contra coordenação dos dois lados, que é muito mais complexa. O at-least-once com `X-Event-Id` cobre a maioria dos casos ([09:25] Diego).

## Questões em aberto

1. **Rate limiting de saída.** Se um cliente tem 50 pedidos mudando de status em um minuto, recebe 50 chamadas? A decisão foi "observar e decidir depois" ([09:38] Diego, [09:39] Larissa).
2. **Escala e ordering global.** A ordem por pedido só vale com um worker. Não há garantia global, e os clientes não a pediram ([09:12] Diego, [09:13] Larissa, [09:14] Marcos). Quando for preciso escalar, a estratégia (particionamento ou lock) fica para depois ([09:13] Diego).
3. **Restrição de role do CRUD de configuração.** Hoje qualquer role autenticada pode gerenciar os endpoints. Sofia disse que dá para endurecer "mais pra frente" ([09:36] Marcos, [09:37] Sofia).
4. **`customer_id`: body ou path.** O `customer_id` não vem do JWT ([09:32] Larissa corrigiu [09:31] Marcos), mas a reunião não escolheu entre body e path ([09:32] Larissa). O FDD propõe o path `/customers/:customerId/webhooks`, **pendente de confirmação**.
5. **Caminho do endpoint de rotação de secret.** A reunião definiu o requisito de rotação com carência de 24 h ([09:21] Sofia), mas não definiu o caminho. O FDD propõe `POST /webhooks/:id/rotate-secret`, **pendente de confirmação**.
6. **Mecânica de assinatura durante a carência.** Enquanto as duas secrets estão válidas, não está definido como o cabeçalho `X-Signature` é montado. O FDD propõe enviar as duas assinaturas no mesmo cabeçalho, **pendente da revisão da Sofia**.
7. **Proteção da secret em repouso.** Para assinar com HMAC a plataforma precisa da secret em claro, então ela não pode ser guardada só como hash. Como protegê-la no banco não foi discutido; entra na revisão de segurança de 2 dias úteis ([09:46] Sofia).

## Impacto e riscos

- **Transação de pedidos.** `changeStatus` ganha uma escrita a mais. Se a inserção na outbox falhar, a mudança de status também falha ([09:40] Bruno). Isso é intencional, mas aumenta o peso da transação que Bruno já considerava pesada ([09:04] Bruno).
- **Latência.** O pior caso é de 2 segundos só pelo polling ([09:10] Larissa). A meta de produto é abaixo de 10 segundos ([09:02] Marcos). São números diferentes. Diego considerou o polling de 2 s suficiente para a meta ([09:09] Diego).
- **Duplicidade.** O cliente pode receber o mesmo evento mais de uma vez e precisa deduplicar por `X-Event-Id`. Isso joga responsabilidade para o cliente ([09:25] Sofia). Marcos vai destacar isso no portal de desenvolvedor ([09:26] Marcos).
- **Janela de retry.** Um evento pode chegar até cerca de 15 horas depois da mudança de status ([09:17] Diego).
- **Crescimento da outbox.** Sem arquivamento nesta feature, a tabela cresce continuamente ([09:08] Diego).
- **Carga de saída.** Sem rate limiting, um pico de mudanças de status pode gerar um pico de chamadas para o mesmo cliente ([09:38] Diego).
- **Segurança.** HMAC e geração de secret precisam de revisão da Sofia, com pelo menos dois dias úteis antes do deploy ([09:46] Sofia).
- **Prazo.** Três sprints, com a revisão de segurança incluída. O prazo da Atlas é o fim de novembro ([09:45] Marcos, [09:46] Larissa).

## Decisões relacionadas

- [ADR-001](adrs/ADR-001-outbox-no-mysql.md): padrão outbox no MySQL existente
- [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md): worker em processo separado, com polling de 2 s
- [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md): 5 tentativas, backoff e DLQ
- [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md): HMAC-SHA256 com secret por endpoint e rotação com 24 h
- [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md): entrega at-least-once com `X-Event-Id`
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md): reuso dos padrões do projeto
- [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md): snapshot do payload na inserção

Detalhes de endpoints, erros, observabilidade e integração com o código estão no [FDD](FDD.md).

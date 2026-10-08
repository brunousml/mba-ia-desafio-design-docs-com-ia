# PRD — Notificações de mudança de status de pedidos por webhook

**Status:** Rascunho para revisão
**Origem:** reunião técnica de quinta-feira, 09:00 (Larissa, Marcos, Bruno, Diego, Sofia)
**Documentos relacionados:** [RFC](RFC.md), [FDD](FDD.md), [TRACKER](TRACKER.md), [ADRs](adrs/)

---

## 1. Resumo e contexto da feature

Hoje, clientes B2B da plataforma descobrem mudanças nos seus pedidos consultando a API de tempos em tempos. Isso deixa a integração lenta e cara para os dois lados. Esta feature faz a plataforma avisar o cliente, por chamada HTTP para um endereço que ele mesmo cadastra, sempre que o status de um pedido dele muda.

O cliente cadastra e gerencia esses endereços pela nossa API, com a ajuda de operadores que o representam. Cada cadastro tem a sua própria chave de assinatura, que permite ao cliente conferir que o aviso veio de nós e não foi alterado no caminho. Se o cliente estiver fora do ar, a plataforma tenta de novo por cerca de 15 horas antes de desistir, e as falhas definitivas ficam guardadas para reenvio manual.

Os primeiros clientes são **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo**. A Atlas pediu a entrega até o fim de novembro e avisou que pode migrar para um concorrente caso isso não aconteça.

## 2. Problema e motivação

- **Consulta repetida é cara para o cliente.** Os três clientes consultam a lista de pedidos periodicamente só para saber se algo mudou. Isso deixa a integração lenta e aumenta o custo de operação deles ([09:00] Marcos).
- **Falta de aviso gera atrito e risco comercial.** A Atlas sugeriu que, sem a entrega, poderia migrar para um concorrente ([09:00] Marcos).
- **O que o cliente considera "tempo real".** Para os clientes, qualquer aviso em menos de 10 segundos já serve. O que não serve é o cliente ter que atualizar manualmente ([09:02] Marcos).
- **Não podemos piorar a operação de pedidos para resolver isso.** Mudança de status já é uma operação pesada e não pode ficar presa à disponibilidade de um sistema externo. Também não podemos mudar o status de um pedido sem que o aviso seja registrado ([09:04] Bruno; [09:40] Bruno).

## 3. Público-alvo e cenários de uso

**Público externo (clientes B2B)**

- **Atlas Comercial, MaxDistribuição e Nova Cargo.** Recebem os avisos em um endereço próprio, cadastrado por eles ([09:00] Marcos).
- **Operadores dos clientes.** Usuários da plataforma que representam o cliente e gerenciam os cadastros pela API autenticada ([09:32] Marcos).

**Público interno**

- **Administradores da plataforma (role ADMIN).** Podem reenviar manualmente os avisos que falharam definitivamente. Essa ação é restrita e registrada ([09:36] Sofia).
- **Time de Pedidos e Plataforma.** Mantêm a feature e acompanham as entregas.

**Cenários principais**

1. Um operador do cliente cadastra um endereço para receber avisos de pedidos que chegarem a "enviado" e "entregue", e recebe a chave de assinatura na hora do cadastro ([09:31] Marcos; [09:33] Marcos).
2. Um pedido muda de status, e o cliente recebe o aviso em menos de 10 segundos, com os dados básicos do pedido ([09:02] Marcos; [09:43] Diego).
3. O endereço do cliente está fora do ar por algumas horas. A plataforma tenta de novo nos intervalos definidos, e o aviso chega quando o cliente volta ([09:17] Diego).
4. Após a última tentativa falhar, o aviso fica registrado. Um administrador pode reenviá-lo manualmente, com registro de quem fez isso ([09:18] Diego; [09:36] Sofia).
5. O cliente troca a chave de assinatura pela API e tem 24 horas com as duas chaves válidas para migrar os sistemas dele ([09:21] Sofia).
6. O cliente consulta o histórico das últimas 100 entregas, com resultado, conteúdo enviado, resposta recebida e tempo de resposta, para investigar problemas ([09:34] Marcos).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| O-01 | Avisar o cliente rápido o suficiente para ser considerado "tempo real" | Percentual de avisos com envio iniciado em menos de 10 s após a mudança de status, com o cliente disponível | 100% | [09:02] Marcos; [09:10] Larissa |
| O-02 | Não piorar a operação de pedidos | Percentual de mudanças de status que concluem sem falha causada pela feature | 100% | [09:04] Bruno; [09:40] Bruno |
| O-03 | Não perder aviso de mudança de status | Percentual de mudanças de status confirmadas que geram um aviso registrado | 100% | [09:40] Bruno; [09:41] Diego |
| O-04 | Entregar a feature para a Atlas | Data de entrega em produção | Até o fim de novembro | [09:45] Marcos |
| O-05 | Entregar com segurança | Prazo de desenvolvimento e revisão | 3 sprints, incluindo pelo menos 2 dias úteis de revisão de segurança antes do deploy | [09:46] Larissa; [09:46] Sofia |

Nota: não há meta quantitativa de redução de consultas à API. A reunião não definiu esse número, então ele não entra como meta.

## 5. Escopo

### Incluso

- Cadastro, edição, remoção e listagem de endereços de envio por cliente, pela API autenticada ([09:31] Marcos; [09:33] Bruno).
- Escolha, por endereço, dos status de pedido que devem gerar aviso ([09:33] Marcos).
- Aviso a cada mudança de status de pedido que o cliente escolheu receber ([09:00] Marcos).
- Chave de assinatura gerada pela plataforma, devolvida na criação e rotacionável pela API com 24 horas de carência ([09:21] Sofia; [09:31] Marcos).
- Assinatura de cada envio ([09:20] Sofia).
- Histórico das últimas 100 entregas por endereço ([09:34] Marcos).
- Reenvio manual de avisos que falharam definitivamente, restrito a ADMIN, com registro de auditoria ([09:18] Diego; [09:36] Sofia).
- Documentação para o cliente no portal de desenvolvedor, com destaque para a possibilidade de avisos repetidos ([09:26] Marcos; [09:40] Marcos).

### Fora de escopo

| Item | Situação | Origem |
| --- | --- | --- |
| E-mail ao cliente quando o envio falha repetidamente | Adiado para a próxima fase, depois de medir o impacto | [09:37] Larissa |
| Painel visual para o cliente acompanhar os webhooks | Projeto separado do time de frontend; agora só endpoints | [09:40] Larissa |
| Webhooks de entrada (cliente enviando para a plataforma) | Fora de escopo; a feature é só de saída | [09:02] Marcos |
| Arquivamento das entregas antigas (cerca de 30 dias) | Fora de escopo desta feature | [09:08] Diego |
| Garantia de ordem global e múltiplos processos de envio | Limitação conhecida; não é garantia de ordem global | [09:13] Diego; [09:13] Larissa |
| Limitação de volume de envios para o cliente (rate limiting) | Em aberto: observar e decidir depois | [09:39] Larissa |
| Restringir o cadastro de endereços por perfil de usuário | Em aberto: por enquanto qualquer perfil autenticado pode | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | Cadastrar um endereço de envio para um cliente, informando o endereço e os status de interesse. A chave de assinatura é gerada pela plataforma e devolvida na criação. O cliente é identificado na própria requisição, e não pelo login do operador (ver questões 4 e 5 da [RFC](RFC.md)). | [09:31] Marcos |
| RF-02 | Editar um cadastro existente de endereço de envio. | [09:33] Bruno |
| RF-03 | Remover um cadastro de endereço de envio. | [09:33] Bruno |
| RF-04 | Listar os cadastros de endereço de envio de um cliente. | [09:33] Bruno |
| RF-05 | Escolher, por endereço, quais status de pedido geram aviso. Mudanças em status não escolhidos não geram aviso para aquele endereço. | [09:33] Marcos |
| RF-06 | Consultar o histórico das últimas 100 entregas de um endereço, com resultado (sucesso ou falha), conteúdo enviado, resposta recebida e tempo de resposta. | [09:34] Marcos |
| RF-07 | Reenviar manualmente um aviso que falhou definitivamente. Só usuários com perfil ADMIN podem fazer isso, e cada reenvio fica registrado com quem o executou. | [09:18] Diego; [09:36] Sofia |
| RF-08 | Rotacionar a chave de assinatura de um endereço pela API. A chave anterior continua válida por 24 horas, em paralelo com a nova, e depois deixa de valer. | [09:21] Sofia |
| RF-09 | Gerar um aviso a cada mudança de status de pedido que interesse ao cliente. | [09:00] Marcos |
| RF-10 | Assinar cada envio, para que o cliente consiga verificar a origem e a integridade do conteúdo. | [09:20] Sofia |

## 7. Requisitos não funcionais

| ID | Categoria | Requisito | Origem |
| --- | --- | --- | --- |
| RNF-01 | Desempenho | Avisos devem começar a ser enviados em menos de 10 segundos após a mudança de status, quando o cliente está disponível. O pior caso da verificação periódica é de 2 segundos. | [09:02] Marcos; [09:10] Larissa |
| RNF-02 | Isolamento | Mudança de status de pedido não pode esperar nem falhar por causa da resposta de um cliente. | [09:04] Bruno |
| RNF-03 | Consistência | Uma mudança de status só fica confirmada junto com o registro do aviso. Se a mudança for desfeita, nenhum aviso é gerado. | [09:40] Bruno; [09:41] Diego |
| RNF-04 | Segurança | Endereços de envio só podem usar conexão segura (https). Endereços em http são recusados com erro de validação. | [09:23] Sofia |
| RNF-05 | Limites | Avisos com mais de 64 KB não são enviados; a plataforma registra erro em vez de truncar o conteúdo. | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |
| RNF-06 | Desempenho | Cliente que não responde em 10 segundos é tratado como falha de envio. | [09:42] Diego |
| RNF-07 | Resiliência | Após o envio inicial, até 5 novas tentativas, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, cobrindo cerca de 15 horas entre a primeira falha e a última tentativa. | [09:17] Diego; [09:17] Larissa |
| RNF-08 | Garantia de entrega | Entrega "pelo menos uma vez": o cliente pode receber o mesmo aviso mais de uma vez e precisa conseguir identificar duplicatas por um identificador único do evento. | [09:24] Diego; [09:25] Diego |
| RNF-09 | Ordem | A ordem dos avisos é garantida apenas dentro de um mesmo pedido, e só enquanto houver um único processo de envio. Não há garantia de ordem global. | [09:12] Diego; [09:13] Larissa |
| RNF-10 | Manutenibilidade | A feature reaproveita os padrões já existentes no projeto (organização por módulo, formato de erros, registro de logs, validação e controle de perfis), sem bibliotecas novas para isso. | [09:30] Larissa |

## 8. Decisões e trade-offs principais

- **[ADR-001](adrs/ADR-001-outbox-no-mysql.md): registrar o aviso na mesma transação da mudança de status, no banco já existente.** Evita infraestrutura nova e garante consistência, ao custo de uma escrita a mais na transação.
- **[ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md): envio feito por processo separado, com verificação a cada 2 segundos.** Um reinício da API não derruba o envio. O custo é uma latência de até 2 segundos e um único processo de envio.
- **[ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md): 5 tentativas com intervalos crescentes e reenvio manual das falhas definitivas.** Cobre quedas de até cerca de 15 horas. Com isso, um aviso pode chegar até 15 horas depois do fato.
- **[ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md): chave de assinatura única por endpoint, com rotação e 24 horas de carência.** Um vazamento afeta só um cadastro. A chave precisa ficar disponível para a plataforma assinar os envios.
- **[ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md): entrega pelo menos uma vez, com identificador único por evento para deduplicação.** Mais simples e padrão de mercado. O trabalho de deduplicar passa para o cliente.
- **[ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md): reutilizar os padrões do projeto, sem componentes novos.** O módulo herda as limitações atuais, como a falta de métricas e tracing dedicados.
- **[ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md): o conteúdo do aviso é fixado no momento da mudança de status.** Reenvios mostram o estado que existia na mudança. Quem precisar de detalhes do pedido consulta a API.

## 9. Dependências

- **Revisão de segurança (Sofia).** Pelo menos 2 dias úteis antes do deploy, com atenção especial à geração e à assinatura das chaves ([09:46] Sofia).
- **Prazo da Atlas (Marcos).** Marcos confirma a data com o cliente ([09:47] Marcos).
- **Documentação no portal de desenvolvedor (Marcos).** Precisa existir antes do lançamento, com destaque para a deduplicação ([09:26] Marcos).
- **Clientes deduplicarem avisos.** A garantia de entrega depende de o cliente tratar repetições ([09:25] Diego).
- **Perfis de usuário existentes.** O reenvio manual usa o controle de perfis já existente, com o perfil ADMIN ([09:36] Larissa).
- **Infraestrutura atual.** Usa o banco de dados e a stack já em produção. Não há previsão de nova infraestrutura ([09:07] Diego).
- **Decisões de cadastro em aberto.** Como o cliente é identificado na requisição (corpo ou caminho) e o formato do caminho de rotação da chave ainda não foram fechados (ver questões 4 e 5 da [RFC](RFC.md)).

## 10. Riscos e mitigação

As probabilidades e impactos abaixo são uma avaliação de produto feita a partir da reunião, não um dado medido.

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| O cliente não trata avisos repetidos e processa o mesmo evento duas vezes | Média | Alto | Documentação em destaque no portal de desenvolvedor ([09:26] Marcos) e identificador único por evento ([09:25] Diego) |
| Vazamento da chave de assinatura de um cliente, como já aconteceu em log de aplicação de outro cliente | Média | Alto | Chave única por endpoint, rotação pela API com 24 horas de carência ([09:21] Sofia) e revisão de segurança antes do deploy ([09:46] Sofia). |
| Não cumprir o prazo da Atlas e perder o cliente para o concorrente | Média | Alto | Prazo de 3 sprints com revisão de segurança incluída ([09:47] Larissa) e confirmação com o cliente ([09:47] Marcos) |
| Cliente recebe avisos em volume alto, em rajada, sem limite de envio | Média | Médio | Observar o comportamento em produção e decidir sobre limite de envio depois ([09:38] Diego; [09:39] Larissa) |
| Cliente fica fora do ar por mais de 15 horas e o aviso vai para a lista de falhas definitivas | Baixa | Médio | Reenvio manual pelo ADMIN com registro de auditoria ([09:18] Diego; [09:36] Sofia) |
| Mudança de status fica mais lenta ou falha por causa da gravação do aviso | Baixa | Alto | Gravação dentro da mesma transação, sem chamada HTTP ao cliente ([09:04] Bruno; [09:40] Bruno) |
| Ordem dos avisos se perde quando a plataforma passar a usar mais de um processo de envio | Baixa | Médio | Limitação documentada; a escala futura precisa de decisão específica ([09:13] Larissa) |

## 11. Critérios de aceitação

- **RF-01:** cadastrar um endereço https devolve a chave de assinatura na resposta de criação. Endereço http é recusado com erro de validação.
- **RF-05 e RF-09:** um pedido que muda para um status escolhido gera aviso para aquele endereço. Mudança para status não escolhido não gera aviso para ele.
- **RNF-01:** com o cliente disponível, 100% dos avisos começam a ser enviados em menos de 10 segundos após a mudança de status.
- **RNF-03:** uma mudança de status desfeita não deixa aviso registrado.
- **RNF-06 e RNF-07:** um cliente que não responde em 10 segundos é tratado como falha, e o aviso é tentado de novo nos intervalos definidos. Se a quinta nova tentativa também falhar, o aviso vai para a lista de falhas definitivas.
- **RF-07:** só ADMIN consegue reenviar um aviso que falhou. Cada reenvio fica registrado com quem o executou. Outros perfis recebem negativa de acesso.
- **RF-08:** após a rotação, a chave antiga continua válida por 24 horas e deixa de valer depois disso.
- **RF-06:** o histórico mostra as últimas 100 entregas, com sucesso ou falha, conteúdo, resposta e tempo de resposta.
- **RNF-05:** um aviso com mais de 64 KB não é enviado e gera erro registrado.
- **RNF-08:** a documentação de deduplicação está publicada no portal antes do lançamento.
- **Segurança:** a revisão de segurança foi feita com pelo menos 2 dias úteis antes do deploy.

## 12. Estratégia de testes e validação

- **Testes de regra de negócio.** Cobrir cadastro, edição, remoção, listagem e escolha de status, incluindo os casos de não gerar aviso para status não escolhido (RF-01 a RF-05).
- **Testes de consistência.** Confirmar que uma mudança de status com falha ao registrar o aviso é desfeita, e que nenhuma mudança fica sem aviso quando confirmada (RNF-03).
- **Testes de falha do cliente.** Simular cliente fora do ar, lento (acima de 10 segundos) e respondendo com erro, e verificar as 5 novas tentativas nos intervalos definidos e a ida para a lista de falhas definitivas (RNF-06 e RNF-07).
- **Testes de latência.** Medir o tempo entre a mudança de status e o início do envio, com cliente disponível, e comparar com a meta de 10 segundos (RNF-01).
- **Testes de segurança.** Verificar a assinatura dos envios, a recusa de endereços http, a rotação com 24 horas de carência e o controle de acesso ao reenvio manual (RF-07, RF-08, RF-10, RNF-04). Essa parte passa pela revisão da Sofia, com pelo menos 2 dias úteis antes do deploy.
- **Testes de limites.** Confirmar a recusa de avisos acima de 64 KB com erro (RNF-05).
- **Validação com o cliente piloto.** Antes do lançamento para a Atlas, validar o fluxo completo com a própria Atlas, incluindo deduplicação e histórico de entregas. A reunião não definiu essa etapa; é uma sugestão para confirmar com Marcos.

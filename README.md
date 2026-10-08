# Webhooks de Notificação de Pedidos — Design Docs gerados com IA

> Enunciado original: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, Express, Prisma e MySQL) decidiu, numa reunião de ~55 minutos, construir webhooks de saída para avisar três clientes B2B quando o status de um pedido muda. Nada foi registrado além da transcrição literal da call (`TRANSCRICAO.md`). A tarefa era transformar essa conversa, junto com o código existente, num pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker que liga cada item à fala ou ao arquivo de onde veio.

O risco principal não era escrever pouco, e sim escrever **o que a reunião não decidiu**: a conversa tem propostas descartadas (3 tentativas, Redis Streams, trigger no banco), correções no meio do caminho (`customer_id` do JWT) e itens adiados (e-mail, rate limiting). O trabalho foi, em boa parte, de filtragem.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (Claude Opus)** | Sessão principal: leitura do código e da transcrição, plano detalhado, escrita dos ADRs e do FDD, revisão crítica dos documentos gerados pelos subagentes, script de verificação. |
| **Subagentes Claude Haiku** (via Claude Code) | Rascunho do RFC, do PRD e do TRACKER em paralelo, cada um com prompt dirigido (armadilhas listadas, formato e contagens mínimas). Mais baratos; a revisão ficava com a sessão principal. |
| **Script Python de verificação** (escrito pela IA, fora do repositório) | Confere que todo `[hh:mm] Nome` citado existe na transcrição com aquele falante, que todo caminho `src/…`, `prisma/…`, `tests/…` existe e que os links entre documentos resolvem; calcula as proporções do TRACKER. |

## Workflow adotado

1. **Contexto e inventário.** Antes de qualquer documento, a transcrição e o código foram mapeados num plano (`docs/plans/Desafio 3 — plano detalhado.md`) com: as 6 decisões principais com timestamp e alternativa descartada, as decisões secundárias, requisitos funcionais, fora de escopo/questões em aberto, **7 armadilhas** conhecidas e a tabela de arquivos reais para a seção de integração.
2. **ADRs primeiro** (sessão principal), um por decisão, mais um ADR extra para o snapshot do payload. Esses ADRs viraram o "esqueleto" passado como contexto para tudo o que veio depois.
3. **RFC e PRD em paralelo** com subagentes Haiku, enquanto a sessão principal escrevia o **FDD** — o documento mais técnico, onde ficam as escolhas de contrato (path do `customer_id`, rota de rotação, mecânica de dupla assinatura) que RFC e PRD apenas referenciam como pendentes.
4. **Revisão crítica** de cada documento contra as armadilhas (grep por "JWT", "3 tentativas", "12 ou 24", "global", nomes de tabela no PRD).
5. **TRACKER** gerado por subagente varrendo os documentos prontos, depois **auditado pelo script**.
6. **Verificação final** com o script e a checklist de critérios de aceite.
7. **Classificação das falas.** Cada fala da transcrição foi classificada em *Definido*, *Negado* ou *Adiado*, com o trecho na íntegra e o horário. As decisões *Definido* que viram trabalho de implementação foram convertidas em 24 atividades, cada uma com a fonte na reunião e o local onde aparece nos documentos (`docs/classificacao-falas-reuniao-webhooks.md`).
8. **Conferência final contra os critérios de aceite** (`docs/plans/checklist.md`), item a item, com as correções listadas na iteração 9.

Separação de altura adotada: ADR = por quê; RFC = o que propomos e o que está aberto (sem payloads); FDD = como construir; PRD = problema, escopo e métrica, sem nomes de tabela.

## Prompts customizados

**1. Inventário da transcrição** (base do mapa de decisões do plano):

```text
Leia TRANSCRICAO.md e o código em src/ e prisma/. Monte um inventário em tabelas,
cada linha com timestamp e falante no formato [hh:mm] Nome:
- decisões FECHADAS (quem fechou e em que minuto) e a alternativa descartada de cada uma;
- requisitos funcionais explícitos;
- itens descartados, adiados para "próxima fase" e deixados em aberto, separadamente;
- pontos em que alguém foi CORRIGIDO depois (o valor final é o que vale).
Não registre nada que não tenha fala correspondente. Depois, liste os arquivos reais
do código onde a feature vai se integrar, com a linha relevante.
```

**2. Geração dirigida de documento** (versão usada para o RFC; o do PRD seguiu o mesmo molde):

```text
Escreva docs/RFC.md em pt-BR. Leia TRANSCRICAO.md, o plano e os 7 ADRs.
Seções obrigatórias: Metadados (revisores = participantes), TL;DR, Contexto, Proposta
técnica SEM payloads, tabelas de erro ou detalhes de implementação (isso é do FDD),
Alternativas consideradas (cada uma com trade-off e [hh:mm] Nome), Questões em aberto,
Impacto e riscos, Decisões relacionadas com links para os ADRs. 2 a 4 páginas.
Armadilhas a NÃO cometer: customer_id NÃO vem do JWT ([09:32] Larissa corrigiu Marcos);
são 5 tentativas, não 3; janela de "quase 15 horas"; ordering só por pedido e só com
um worker; latência de 2 s no pior caso ≠ meta de produto de 10 s.
Não invente nada sem origem na transcrição ou no código.
```

**3. Auditoria de origem** (rodado por um subagente Haiku sobre os documentos prontos):

```text
Liste toda afirmação deste documento que não tem origem na transcrição ou no código.
Para cada uma: cite o trecho, diga se é (a) invenção, (b) proposta legítima do FDD
que precisa estar marcada como proposta, ou (c) tem origem e você não achou.
Confira também se algum caminho de arquivo citado não existe no repositório.
```

## Iterações e ajustes

Foram **7 etapas principais** (plano → ADRs → RFC/PRD/FDD → TRACKER + verificação → auditoria de origem e correções → classificação das falas → conferência final). Ajustes concretos:

1. **Arquivos "inexistentes" que eram arquivos a criar.** A primeira passada do script de verificação acusou `src/worker.ts` (novo) e `src/modules/webhooks/webhook.processor.ts` (novo) como caminhos inexistentes citados no FDD e nos ADR-002 e ADR-006. Eles foram ditos na reunião ([09:11] Larissa, [09:28] Bruno), mas um leitor (ou corretor) não tem como distinguir "a criar" de "alucinado". Todas as ocorrências passaram a ter a marca `(novo)`, e o script passou a aceitar só caminhos que existem ou que estão explicitamente marcados assim.
2. **Status "falhou" na outbox.** A transcrição lista os status da outbox como "pendente, processando, falhou, entregue" ([09:08] Diego), mas dez minutos depois a reunião decide que falha definitiva vai para tabela separada ([09:18] Diego), descartando o `failed` na própria outbox. Copiar a lista de [09:08] contradiria o ADR-003. O FDD ficou com `PENDING`/`PROCESSING`/`DELIVERED` e explica, em nota, por que não há `FAILED`.
3. **RFC escrito antes do FDD existir.** O subagente do RFC terminou antes do FDD e avisou que não pôde conferir as propostas atribuídas ao FDD (path com `customer_id`, `POST /webhooks/:id/rotate-secret`, dupla assinatura na carência). Na revisão, conferi as três contra o FDD final; batem, e no RFC estão como "pendente de confirmação".
4. **Payload acima de 64 KB × rollback.** Sofia quer "errar" quando o payload passa do limite ([09:23]), e Bruno quer rollback se a outbox falhar ([09:40]). Juntas, as duas regras significam que um payload grande impediria a mudança de status. O FDD deixa isso explícito em vez de esconder, e registra por que o caso não deve ocorrer na prática (payload de tamanho fixo, sem itens — [09:24] Diego).
5. **`NotFoundError` não serve para `WEBHOOK_NOT_FOUND`.** Ao mapear as classes de erro, a leitura de `src/shared/errors/http-errors.ts` mostrou que `NotFoundError` fixa o código `NOT_FOUND`. A matriz de erros do FDD passou a dizer que `WEBHOOK_NOT_FOUND` estende `AppError` direto com status 404.

6. **"5 tentativas": contando ou não o envio inicial?** O subagente do TRACKER apontou que o FDD mandava para a DLQ após a 6ª falha e o PRD dizia "após a quinta falha". A ambiguidade vem da própria reunião: "5 tentativas" com 5 intervalos (1m/5m/30m/2h/12h) e "quase 15 horas" ([09:17] Diego). Só a leitura "envio inicial + 5 retentativas" fecha a soma (~14,6 h), então PRD, ADR-003 e TRACKER foram alinhados a ela, com a justificativa escrita no ADR.

7. **Auditoria de origem (prompt 3) sobre FDD, RFC e PRD.** Um subagente Haiku listou 15 achados; os principais, todos corrigidos:
   - O FDD usava o id de cada linha da outbox como `X-Event-Id`. Como há uma linha por endpoint, um cliente com dois endpoints receberia ids diferentes para o mesmo fato, contra "único por evento" ([09:25] Diego). Passou a existir um `eventId` por evento, compartilhado entre as linhas.
   - `WEBHOOK_INVALID_URL` nunca seria emitido se a regra `https` ficasse só no schema: `validate()` transforma qualquer `ZodError` em `VALIDATION_ERROR`. O FDD agora move a checagem para o service e registra o desvio para revisão.
   - Atribuições erradas (ex.: recuperação de crash creditada a [09:24] Diego, ordenação do histórico creditada a Marcos) e propostas do FDD sem marcação (dupla assinatura no contrato, códigos de erro extras, limiares de métricas, matriz de riscos) foram reescritas como "proposta deste FDD".
   - Duas frases do PRD sem origem foram removidas.
8. **Classificação das falas e atividades.** Ao transformar as decisões *Definido* em atividades, a rotação de secret ficou com ressalva: o caminho do endpoint e a mecânica da carência de 24 h são propostas do FDD, abertas no RFC. Mantive essa ressalva explícita na tabela, em vez de tratar a rotação como decidida. A tabela de 24 atividades foi cruzada com os documentos e cada item ganhou a localização (com link) e a situação.
9. **Conferência final contra o checklist.** Falhas encontradas e corrigidas:
   - O PRD apontava "seção 10" e "seção 9" para as decisões de cadastro em aberto. Essas decisões estão nas questões 4 e 5 da RFC; a referência foi corrigida.
   - O modelo `WebhookDelivery` não guardava o payload enviado, mas o endpoint E5 e o RF-06 exigem esse conteúdo. A linha da outbox é apagada quando o evento é entregue, então o conteúdo não seria recuperável. O campo `payload` foi adicionado.
   - Só o E1 tinha exemplo completo de request e response. Ganharam exemplo os endpoints E3, E5 e E6.
   - O TRACKER não tinha a questão 7 da RFC (proteção da secret em repouso). A linha `RFC-QA-07` foi adicionada.
   - Os ADRs tinham o status apenas como metadado. Cada um ganhou a seção `## Status`, exigida pelo checklist.
   - No ADR-006, `webhook.processor.ts` não tinha a marcação `(novo)`, que a regra da iteração 1 exige.
   - O `DELETE` (E4) conflitava com a chave estrangeira de `webhook_outbox`: o `onDelete` padrão do Prisma impede apagar um endpoint com eventos pendentes, e o FDD dizia que esses eventos iriam para a DLQ. O FDD agora define que a remoção move os pendentes para a DLQ na mesma transação, e que o replay desses itens responde 404. A reunião só definiu que existe um `DELETE` ([09:33] Bruno); o comportamento está marcado como proposta do FDD.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md) — problema dos três clientes, escopo, requisitos e métricas.
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — uma decisão por arquivo:
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado com polling](docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md)
   - [ADR-003 Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
   - [ADR-004 HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
   - [ADR-005 At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 Snapshot do payload](docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md)
4. [`docs/FDD.md`](docs/FDD.md) — modelo de dados, fluxos, contratos, erros, observabilidade e integração com o código.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — de onde veio cada item.
6. [`docs/classificacao-falas-reuniao-webhooks.md`](docs/classificacao-falas-reuniao-webhooks.md) — cada fala classificada em *Definido*, *Negado* ou *Adiado*, e as 24 atividades da feature com a localização de cada uma nos documentos.

**Pontos em aberto após a conferência:**

- **Validação com o cliente piloto.** A reunião não definiu essa etapa; o PRD (seção 12) a sugere como item a confirmar com o Marcos.
- **Questões 1 a 7 da RFC.** Continuam abertas, como registrado no próprio documento.

Material de apoio: [`TRANSCRICAO.md`](TRANSCRICAO.md) (fonte, não alterada) e o plano de trabalho em [`docs/plans/`](docs/plans/).

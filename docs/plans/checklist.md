# Checklist de critérios de aceite — relatório de avaliação

Data da avaliação: 2026-10-08.
Legenda: ☑ atende · ☐ não atende ou fora do alcance da avaliação.

Os critérios abaixo são os do enunciado do desafio (página "Da Reunião ao Documento: Design Docs Gerados por IA", na plataforma). Cada um traz a evidência encontrada.

## Resumo

| Bloco | Critérios | Atendidos |
|---|---|---|
| PRD | 6 | 6 |
| RFC | 5 | 5 |
| FDD | 6 | 6 |
| ADRs | 4 | 4 |
| Tracker | 4 | 4 |
| README | 4 | 4 |
| Consistência geral | 2 | 2 (um com ressalva) |
| **Total** | **31** | **31** |

## Método

- **Contagens e estrutura:** script Python sobre os arquivos (títulos `##`, linhas de tabela, blocos de código, arquivos da pasta `docs/adrs/`).
- **Tracker e citações:** cada `[hh:mm] Nome` do tracker e dos documentos foi conferido contra as falas de `TRANSCRICAO.md` (horário e falante). Cada caminho de fonte `CODIGO` foi conferido no sistema de arquivos.
- **Caminhos de código:** todo caminho `src/`, `prisma/` e `tests/` citado nos documentos foi conferido no repositório.
- **Reverificação final (2026-10-08):** os números deste relatório foram recalculados por script. 7 ADRs com as 5 seções `##`; PRD com 10 RF, 10 RNF e 12 seções; RFC com 8 seções e 7 links de ADR; FDD com 13 seções, 7 endpoints e 11 códigos `WEBHOOK_*`; Tracker com 134 linhas de 6 colunas, 113 com `[hh:mm]` (84,3%) e 21 `CODIGO`. Os únicos caminhos inexistentes são os 4 arquivos novos já citados. `git diff` em `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md`, `package.json` e `package-lock.json` está vazio.
- **Contradições com a transcrição:** leitura manual dos pontos de risco (veja a seção "Consistência geral").

## PRD (docs/PRD.md)

- ☑ Arquivo existe e está em Markdown. `docs/PRD.md`, 168 linhas.
- ☑ Contém todas as seções obrigatórias listadas no requisito 1. 12 seções numeradas, na ordem do enunciado: resumo, problema, público-alvo, objetivos, escopo, requisitos funcionais, requisitos não funcionais, decisões, dependências, riscos, critérios de aceitação e estratégia de testes.
- ☑ Identifica no mínimo 8 requisitos funcionais discutidos na reunião. 10 (RF-01 a RF-10), cada um com a fala de origem.
- ☑ Inclui pelo menos 1 objetivo com métrica e meta quantitativa. 5 objetivos; O-01 a O-03 têm meta de 100% e O-01 mede o envio em menos de 10 s.
- ☑ Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião. 7 itens, com a situação e a fala. Exemplos: e-mail (adiado, 09:37), dashboard (09:40), webhooks de entrada (09:02) e rate limiting (observar, 09:39).
- ☑ Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação. 7 riscos, todos com as três colunas.

## RFC (docs/RFC.md)

- ☑ Arquivo existe e está em Markdown. `docs/RFC.md`, cerca de 1.600 palavras, dentro de 2 a 4 páginas.
- ☑ Contém todas as seções obrigatórias listadas no requisito 2. Metadados (autor, status, data e revisores com os participantes da reunião), resumo executivo, contexto e problema, proposta técnica, alternativas consideradas, questões em aberto, impacto e riscos e decisões relacionadas.
- ☑ Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte. 9 alternativas, cada uma com trade-off e fala de origem. Exemplos: disparo síncrono, Redis Streams, trigger do banco e 3 tentativas.
- ☑ Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião. 7 questões. As duas primeiras são rate limiting e ordering global; outras incluem `customer_id` em body ou path e proteção da secret em repouso.
- ☑ Referencia, com link, pelo menos 2 ADRs do pacote. Links para os 7 ADRs, e todos resolvem.

## FDD (docs/FDD.md)

- ☑ Arquivo existe e está em Markdown. `docs/FDD.md`.
- ☑ Contém todas as seções obrigatórias listadas no requisito 3. 13 seções: contexto, objetivos técnicos, escopo e exclusões, modelo de dados, fluxos detalhados (outbox, worker, retry e DLQ com replay e rotação de secret), contratos públicos, matriz de erros, resiliência, observabilidade, dependências e compatibilidade, integração com o sistema existente, critérios de aceite técnicos e riscos.
- ☑ Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes. 7 endpoints (E1 a E7) com status de sucesso. E1, E3, E5 e E6 têm request e response de exemplo; o envio ao cliente também tem exemplo completo.
- ☑ Matriz de erros usa códigos com prefixo WEBHOOK_. 11 códigos `WEBHOOK_*`, com status HTTP e classe base.
- ☑ Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base. 14 arquivos reais na tabela, mais o modelo de pastas dos arquivos novos, marcados como `(novo)`.
- ☑ Seção "Observabilidade" cita métricas, logs e tracing. Tabela de 7 eventos de log, 5 métricas derivadas dos logs e tracing por `eventId` e `requestId`.

## ADRs (docs/adrs/ADR-NNN-*.md)

- ☑ Pasta docs/adrs/ contém entre 5 e 8 arquivos no formato ADR-NNN-titulo-em-kebab-case.md. 7 ADRs no formato (ADR-001 a ADR-007). A pasta tem também um `README.md`, que não segue o formato ADR, e o total de arquivos é 8.
- ☑ Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências. Os 7 ADRs têm as cinco seções como títulos `##`. A seção `## Status` foi adicionada na conferência final.
- ☑ O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4. Cobre as 6: outbox (ADR-001), worker com polling (ADR-002), retry e DLQ (ADR-003), HMAC (ADR-004), at-least-once (ADR-005) e reuso dos padrões (ADR-006). O ADR-007 trata do snapshot do payload.
- ☑ Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base. O ADR-006 tem uma tabela de padrões com arquivos reais; os ADRs 001 e 002 citam `OrderService.changeStatus` e `src/server.ts`.

## Tracker (docs/TRACKER.md)

- ☑ Arquivo existe e segue o formato de tabela definido no requisito 5. Colunas ID, Documento, Tipo, Conteúdo (resumo), Fonte e Localização. 134 linhas, todas com 6 colunas.
- ☑ Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente. Cobertura por tipo de item: 10 RF, 10 RNF, 5 objetivos, 7 itens fora de escopo e 7 riscos do PRD; 9 alternativas e 7 questões do RFC; 6 decisões da proposta do RFC; os 7 ADRs; 7 endpoints e os erros do FDD. O denominador de "itens identificáveis" não é fechado no enunciado, então o percentual exato não foi medido.
- ☑ Pelo menos 70% das linhas têm Fonte = TRANSCRICAO com timestamp válido no formato [hh:mm] Nome. 113 de 134 linhas (84,3%). Os 113 timestamps conferem com horário e falante na transcrição.
- ☑ Pelo menos 5 linhas têm Fonte = CODIGO com caminho de arquivo real. 21 linhas, todas com caminho existente.

## README (README.md)

- ☑ Contém todas as seções obrigatórias listadas no requisito 6. As 6 seções estão presentes: sobre o desafio, ferramentas de IA, workflow, prompts customizados, iterações e ajustes e como navegar a entrega.
- ☑ Lista pelo menos 1 ferramenta de IA utilizada. 3 ferramentas, cada uma com o papel: Claude Code, subagentes Haiku e um script de verificação.
- ☑ Mostra pelo menos 2 prompts customizados em blocos de código. 3 prompts em blocos de código.
- ☑ Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção. 9 iterações, informando também o número de etapas principais (7).

## Consistência geral

- ☑ Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código. Pontos de risco conferidos:
  - O `customer_id` não vem do JWT ([09:32] Larissa).
  - São 5 tentativas com backoff de 1m/5m/30m/2h/12h. A leitura "envio inicial + 5 retentativas" está justificada no ADR-003.
  - Ordering garantido só por pedido e só com um worker.
  - Latência de pior caso de 2 s separada da meta de produto de 10 s.
  - Os itens adiados ou descartados (e-mail, dashboard, rate limiting, entrada, arquivamento) estão como fora de escopo ou em aberto, e não como requisito.
  - As escolhas do FDD que a reunião não fez (caminho do `customer_id`, rota de rotação, dupla assinatura, remoção de endpoint, códigos extras) estão marcadas como propostas do FDD.
  - As falhas encontradas na conferência (referências cruzadas do PRD, `payload` em `webhook_deliveries`, remoção de endpoint) foram corrigidas. Estão descritas na iteração 9 do README.
- ☑ Nenhum arquivo de código mencionado nos documentos é inexistente no repositório. **Com ressalva.** Todos os caminhos de código existentes citados existem. Quatro caminhos que o FDD manda criar não existem ainda: `src/worker.ts`, `src/modules/webhooks/`, `src/modules/webhooks/webhook.processor.ts` e `src/modules/webhooks/webhook.worker.ts` (este último só aparece em citação literal da transcrição). Estão marcados como `(novo)`, ou aparecem sob "Estrutura do módulo novo" no FDD. Um corretor que checa literalmente a existência de cada caminho pode apontá-los.

## Regras do enunciado fora da checklist

- ☑ `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md` e arquivos de configuração (`package.json`, `package-lock.json`) sem nenhuma alteração. O `package-lock.json`, que tinha uma linha inserida por um `npm install`, foi revertido.
- ☑ Repositório no GitHub a partir de fork. O `origin` é `brunousml/mba-ia-desafio-design-docs-com-ia`. Que seja público e fork de `devfullcycle/…` não foi verificado daqui.
- ☐ Correções finais na `main` do GitHub. O PR #1 foi mergeado só com o commit `2273435`, sem as correções da conferência (ADR `## Status`, RFC-QA-07, E4, referência do PRD). Elas seguem em um novo PR.
- ☐ URL do repositório enviada na área de entrega da plataforma. Depende de você.

## Pendências que não são dos documentos

1. Abrir e mergear o PR com as correções finais.
2. Confirmar que o repositório é público e é fork de `devfullcycle/mba-ia-desafio-design-docs-com-ia`.
3. Enviar a URL na plataforma.

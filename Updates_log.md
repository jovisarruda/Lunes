# Updates Log — Lunes

> Registro cronológico de mudanças. Entrada nova sempre no topo.
> Formato: `AAAA-MM-DD — resumo da mudança`.

## 2026-07-30 (correcoes locais do Gantt, aguardando autorizacao para publicar)
- Escopo executado somente em `LunesGantt_prototype.html`; nenhum push, commit ou deploy realizado.
- **Effort:** numero sem sufixo passa a representar dias; parser aceita `h`, `d`, `w`, `m` e `y`; unidade persistida em `_effortUnit`, preservando itens legados sem unidade como horas.
- **Edicao de datas:** removido o commit durante o evento intermediario do seletor de data. A alteracao agora termina somente com Enter ou ao sair do campo, eliminando o fechamento ao digitar `0`, o popup prematuro e o bloqueio da segunda tentativa. Aplicado na lista e no Details.
- **Today:** removida a data fixa; a linha usa a data local do navegador.
- **Barras cinza de resumo:** intervalo centralizado em `summarySpan()`, calculado apenas com datas descendentes validas; datas manuais do grupo permanecem separadas.
- **Push activities:** removido o limite incorreto baseado na posicao original. O solver fixa a atividade movida, propaga para sucessoras a direita e predecessoras a esquerda, limita a alteracao as cadeias conectadas, respeita lag e ignora atividades sem datas validas.
- **Compatibilidade:** FS/SS/FF/SF mantidos no armazenamento e na UI (equivalentes TI/II/TT/IT).
- **Teste local concluido no Chrome:** 17/17 verificacoes aprovadas, cobrindo render do prototipo, todas as unidades de effort, rejeicao de unidade invalida, Today local, roll-up da barra de resumo, formulas FS/SS/FF/SF com lag, edicao de data sem `change` prematuro e dois cenarios completos do solver (push apenas da cadeia sucessora e ajuste apenas da cadeia predecessora). Evidencia: `gantt-local-test.html` e `gantt-local-test-result.png`.
- Publicacao continua bloqueada ate autorizacao explicita do Jovis.

## 2026-07-30 (retoques de UX)
- **Limpeza de UX + FAQ para a Assistente Lunes.** PUBLICADO + VALIDADO AO VIVO (protótipo e index.html).
  - Removido o rótulo de dev **"prototype · A0–A9"** do título (agora só "LunesGantt").
  - **Removido o chatbot próprio do board Gantt** (FAB ✨ + painel + wiring `botFab/botClose/botSend/botInput`), já que a **Assistente Lunes** global (widget ✨ do index.html + Edge Function gemini) atende esse board. As funções `botAnswer/botAdd/botSendMsg` ficaram como código morto inofensivo (não chamadas). Validado: sem `botFab`/`botPanel`, board carrega e renderiza normal (5 tasks).
  - **FAQ escondido para a Assistente Lunes:** const `PROJECT_FAQ` (2.7k chars, pt-BR, texto plano) no `index.html`, injetado no `system` do `aiChatSend` (seção "FAQ — COMO USAR A FERRAMENTA") + instrução para usá-lo quando o usuário perguntar "como faço X". Cobre Gantt/Workload/Kanban/Calendar, add/editar atividades, dependências (4 tipos), marcos, reagendamento, caminho crítico, cores, ⚙ settings, editor de colunas, IDs, export PDF. Não aparece na UI — só a assistente lê. Validado: `PROJECT_FAQ` definido, `aiChatSend` intacto, app sem erro de parse.
  - Nota: "limpar comentários" foi tratado como **texto de dev visível na UI** (rótulo prototype + subtítulo do bot). Comentários de código (invisíveis) foram mantidos para não arriscar o arquivo sem `node --check`.

- **LunesGantt "Rework" (R) — R4 concluído + esclarecimento de arquitetura.** PUBLICADO + VALIDADO AO VIVO.
  - **Arquitetura real (estava fora dos docs):** board `template==='project'` é renderizado por `renderProjectApp()` no `index.html` = **iframe do próprio protótipo** (`LunesGantt_prototype.html?board=<id>`). O protótipo tem seu **próprio adapter Supabase** (`loadFromDB`/`syncToDB`/`saveSettingsDB`) = o **R1**. `renderProjectTable`/módulo `G` no `index.html` viraram **código morto** (nunca executam p/ project). Isso corrige a confusão que fazia parecer que o protótipo tinha sido "jogado fora".
  - **R2/R3/R5 já estavam implementados e rodando** via o adapter: itens+hierarquia+posições, barras/datas/dependências/marcos/reagendamento/CPM, settings/dedicação/Kanban/Workload — tudo persiste por `syncToDB`/`saveSettingsDB`. Confirmado ao vivo: board carregou via `loadFromDB`, e mudança de campo persistiu por `syncToDB` (status "Done" sobreviveu ao reload).
  - **R4 (o gap real) — FEITO no protótipo:** (1) `logAct` agora também insere em `activity_logs` (`dbLog`); antes só ia p/ memória. (2) Editor de colunas do Details agora persiste em `board_columns`: add/rename/delete coluna (`dbInsertColumn`/`dbRenameColumn`/`dbDeleteColumn`), labels de status (`dbSaveStatusLabels`) e choices de label (`dbSaveColOptions`). Comentários já persistiam (tabela `comments`). Novas colunas recebem o **id real** do `board_columns` (p/ os valores em `col_values` baterem no sync).
  - **Validação E2E ao vivo** (board de teste `ZZ_R4_TEST`, deletado ao final): parse OK no deploy (helpers definidos, 18 linhas no demo); mudar Status gravou em `activity_logs` (field=Status, author=jovis); editor de colunas gravou em `board_columns` (label "In Review", coluna Notes criada/renomeada/excluída); **reload confirmou** que label e status persistiram. 4 boards reais intactos (todos `default`).
  - **Sem `node --check`** nesta sessão (VM do sandbox fora) — sintaxe validada carregando o arquivo publicado no navegador (funções definidas + render sem erro). Edições no `index.html` feitas por engano (antes de perceber o iframe) foram **revertidas**; arquivo idêntico ao `main`.
  - **Fix Kanban/Calendar (R5):** as views Kanban (`renderKb`, arrastar card muda status → persiste por `syncToDB`) e Calendar (`renderCal`, grade mensal por data de início, clicar abre Details) já existiam e estavam nas abas, mas **quebravam no render** por usar `esc()` sem essa função existir (só havia `escapeHtml`/`escAttr`) — por isso apareciam vazias. Corrigido com `const esc=escapeHtml;`. Validado ao vivo (board ZZ_PORT): Kanban 3 cards/4 colunas, Calendar 3 eventos, sem erros. PUBLICADO.
  - **Pendências menores (não-bloqueantes):** drag-reorder com aninhamento-por-dwell na List do protótipo já existe (`startRowDrag`); pessoa adicionada no Workload sem tarefa nem dedicação some no reload (peopleList é derivado das tarefas); board de teste antigo `ZZ_PORT (project)` ainda existe no banco.

## 2026-07-29
- **LunesGantt Fase B — B5 (caminho crítico + motor de reagendamento)** no `index.html`. PUBLICADO + VALIDADO AO VIVO. **Fecha a Fase B do plano original (B0–B5).**
  - Adicionado ao módulo `G`: **motor de reagendamento em cascata** (`computeSchedule` forward+backward pinando a tarefa movida, `applySchedule`, `proposeChange`) — ao arrastar/redimensionar uma barra, se afetar sucessoras/predecessoras abre um **preview** (overlay) com lista de afetados (de→para, → later/← earlier), **avisos de Deadline/Kickoff**, e botões **Push these / Move only this / Cancel**. **CPM** (`computeCritical` forward/backward, folga total=0) + botão **Critical path** (destaque vermelho em barras/setas + popup com a cadeia). Marcos (◆) via flag `col_values._milestone`.
  - As datas resultantes do push são gravadas em lote em `col_values` (Start/Finish). Dependências vêm da tabela `links`; kickoff/deadline de `boards.settings`.
  - Validado E2E ao vivo (board `ZZ_B5`: cadeia A→B→C→E crítica + paralela D): **arrastar B Design disparou o preview** com "affects 2 activities", aviso de Deadline (E Launch), afetados C Build/E Launch → later; **Push aplicou e persistiu** (B 24/08–04/09, C empurrada 05/09–23/09, E 24/09–28/09; A e D **não** empurradas — confirmado relendo o DB). **Critical path** destacou A→B→C→E (D excluída) no popup. Do arquivo publicado: `typeof G==='object'`, `G._apply/_moveOnly` OK; critical path renderizou. Board de teste deletado; 4 boards reais intactos.

- **LunesGantt Fase B — B4 (abas Gantt + Workload)** no `index.html`. PUBLICADO + VALIDADO AO VIVO.
  - Módulo `window.G` (IIFE namespaced, ~1 bloco) com **aba Gantt** (árvore recursiva + timeline + barras por Start/Finish, barras de resumo com roll-up, marcos ◆ via flag `col_values._milestone` com toggle por duplo-clique, zoom Day/Week/Month, cor por Status/Person, linhas Kickoff/Deadline/Today de `boards.settings`, **setas de dependência lendo da tabela `links`**) e **aba Workload** (pessoas×meses, alocado/disponível, sobrecarga em vermelho, a partir do esforço distribuído por dias úteis + dedicação/calendário de `settings`).
  - **Interações que persistem**: arrastar barra move as datas → grava em `col_values` (Start/Finish); redimensionar pontas; arrastar ○ cria dependência → `addDep` na tabela `links` (com guarda de ciclo); clicar seta → menu troca tipo/exclui; duplo-clique → marco. Painel ⚙ Project grava kickoff/deadline/horas-dia/dias-semana/fins-de-semana em `boards.settings`.
  - **Abas** Gantt/Workload aparecem só em boards `template==='project'` (ocultas em default). `setView`/`renderView` despacham para `G.renderGantt()`/`G.renderWorkload()`.
  - Desenvolvido e testado 100% no navegador ao vivo antes de tocar no arquivo (VM do sandbox fora → sem `node --check`; parse confirmado por `typeof G==='object'` no deploy). Teste E2E: board `ZZ_GANTT` (6 itens datados, 3 deps) → Gantt renderizou com barras/setas/cores; **arrastar Backend gravou 31/08→04/09 ⇒ 07/09→11/09 (confirmado relendo do DB)**; Workload calculou Ana 32/168, Bruno 96/168, Elton 8/168+32/176; abas ocultas em board default. Board de teste deletado; 4 boards reais intactos.
  - **Pendente (B5)**: caminho crítico (CPM) e o motor de reagendamento em cascata com preview (empurrar/afetados/aviso kickoff-deadline). Coluna Predecessoras editável na lista e o chatbot só-leitura para project também ficam para depois.

- **LunesGantt Fase B — B2 (persistência do schema Gantt)** no `index.html`.
  - **Migração SQL** (rodar antes de publicar): tabela `links` (dependências `from_item`/`to_item`/`type`/`lag` + RLS `auth_all`), `boards.code_seq` (contador de ID por board), `boards.settings` (jsonb de config do projeto), `items.code` (código do item). SQL completo em `Premissas_setup.md` (seção B2).
  - **Código**: global `links`; `selectBoard` carrega links de boards project; `createItem` atribui código sequencial imutável (`code_seq++`) em boards project; `deleteItem` limpa links locais; helpers de dependência (`addDep/removeDep/setDepType/depsOf/succOf/predText/setPredecessors`), `itemByCode`, e settings (`projSettings/saveProjSettings`). Coluna **ID** (read-only) adicionada à lista dos boards project. Datas/Effort/Person/Status já persistem via colunas do board (criadas no B0); larguras de coluna seguem em localStorage.
  - **SQL aplicado** (Jovis, "Success. No rows returned") e **PUBLICADO + VALIDADO AO VIVO (2026-07-29)**: board default (FIN Purchase) segue intacto; criado board Project `ZZ_TEST_B2` → itens Alpha/Beta/Gamma receberam **códigos 1/2/3** (code_seq=4); `addDep` + `setPredecessors('1SS; 2')` gravaram 3 links; após **reler do banco** os códigos e os 3 links persistiram (predText Gamma = "1SS; 2FS"); coluna **ID** renderiza na lista. Board de teste deletado; 4 boards reais intactos. (VM do sandbox estava fora — sem `node --check`, mas o app carregou sem erros de console e todas as funções B2 inicializaram, confirmando a sintaxe.)

- **LunesGantt Fase B iniciada — B0 (modelo de página) + B1 (hierarquia recursiva)** no `index.html`.
  - **B0**: novo campo `boards.template` (`'default' | 'project'`, default `'default'`). `addBoard()` agora pergunta o tipo na criação (confirm: OK = Project, Cancel = Default) e o envia no insert; **imutável** (sem UI para trocar). Helper `isProjectBoard()`. Boards **project** recebem um conjunto único de colunas de escopo `item`: Person, Status, Start (date), Finish (date), Effort (h). Boards default seguem com o setup de sempre (colunas item + subitem).
  - **B1**: `renderTable()` desvia para `renderProjectTable()` quando o board é project — tabela única **indentada e recursiva (N níveis)** reusando `subsOf`/`parent_id`, `cellHtml`, filtros e somas. Cada linha tem menu ⋯ com "Add sub-element", "Add element (same level)", "Open details" e "Delete (with sub-elements)". `deleteItem` passou a apagar a **subárvore inteira** via novo `descendantIds()` (seguro também p/ boards default). Boards default **inalterados**.
  - **Migração SQL aplicada no Supabase** (rodada pelo Jovis, "Success. No rows returned"): `ALTER TABLE boards ADD COLUMN IF NOT EXISTS template text NOT NULL DEFAULT 'default';`
  - **PUBLICADO e VALIDADO AO VIVO (2026-07-29)**: `index.html` commitado no `main` via Chrome; Pages servindo. Teste end-to-end: criado board Project `ZZ_TEST_B1` (template=project confirmado; colunas Person/Status/Start/Finish/Effort; grupos To Do/Doing/Done) → criados 4 níveis aninhados (Phase 1 → Task A → Subtask A1 → Micro A1a) → `renderProjectTable` mostrou a árvore recursiva indentada com "+ Add sub-element" por nível. Board de teste **deletado** ao final; os 4 boards reais (FIN Purchase/Income, Daily/Weekly, MKT Calendar) confirmados intactos e marcados `template='default'`.
  - Validação prévia: `node --check` OK, arquivo íntegro (`</script></body></html>`), recursão de `descendantIds` testada (4 níveis).
  - Protótipo desktop de referência: `LunesGantt_prototype.html` (no ar em https://jovisarruda.github.io/Lunes/LunesGantt_prototype.html). Plano completo em `LunesGantt.md` (seções 13–14).

## 2026-07-16
- **v3.2 — Fase 5C (IA propõe ALTERAÇÕES em itens existentes)**: nova ação `set_values` no chat; a mudança proposta fica em `col_values._proposed` (chave reservada, sem schema change) — o valor atual NÃO muda até aprovação e o item segue normal em todas as views (diferente de `_pending`, não sai de somas/Kanban/Calendar). AI Input ganhou a seção "✎ Alterações propostas pela IA" com diff "de → para" e botões Aprovar (`aiApproveChanges`, aplica e remove `_proposed`) / Rejeitar (`aiRejectChanges`, só remove). Badge do AI Input conta pendentes + propostas. Alvo identificado por board + nome de item de topo (+ grupo opcional; 0 achados ou ambíguo → erro pedindo grupo). System prompt atualizado (set_values, "não crie outro item para alterar", mantido texto plano sem asteriscos). Subitens fora do escopo (stretch futuro).
- Processo: baixado o `main` (v3.1, sha `6e79634`), 7 blocos aplicados com substituições verificadas (`assert count==1`), `node --check` OK, fim de arquivo íntegro. Commit `4736e19` via Chrome (o botão "Commit changes" precisou de 2º clique — layout deslocou após o ProTip). Pages confirmado servindo v3.2.
- Testado end-to-end ao vivo com item `ZZ_TESTE_5C` no MKT | Calendar (criado só p/ teste): chat → `_proposed` gravado (valor atual intacto) → diff "Status: Backlog → Fazendo" no AI Input, badge (1), resposta sem asteriscos → aprovar aplicou o valor e removeu `_proposed` → rejeitar (reproposta) removeu sem alterar → item de teste deletado. Verificação final: nenhum `_proposed` nem item ZZ_ residual; os pendentes reais do Jovis (FIN | Purchase, 7 itens `_pending`) intocados.

## 2026-07-12 (parte 5)
- **v2.4**: notificações de automação melhoradas no `index.html` (local):
  - Mensagem agora descreve o que aconteceu: `⚡ <pai / >item: <gatilho em texto> — <msg da regra> (<board>)`. Ex.: `⚡ Subscriptions / ↳ NDM: "Due Pay" date is today — pagar boleto (FIN | Purchase)`. Helpers `trigText()`/`notifMsg()`; aplicado nas ações notify do motor imediato (execAction) e do time-based (execTimeAction).
  - Clicar na notificação marca como lida, troca para o board e abre o painel do item/subitem (`openNotif()`); avisa se o board/item não existe mais.
- Obs.: mount do sandbox ficou dessincronizado durante a edição (mostrava versão truncada antiga); o arquivo real no Windows foi verificado completo via Read (termina em `</html>`). Funções novas validadas no navegador.

## 2026-07-12 (parte 4)
- **FIN | Purchase configurado a partir do export do Monday** (arquivos `FIN_*.xlsx` na pasta), via JS no app autenticado:
  - Criados os 6 grupos que faltavam (Doing/Done Monthly, Doing/Done Installment, Doing/Done FAPESP PIPE FASE 1 PJ0012) — total 10 grupos na ordem do Monday.
  - Colunas de subitem preparadas para seleção: Category (21), Paid By (10), Payment (8, do FIN Pay Mode), Supplier (190) e Project (18) — Supplier e Project convertidas de text→label. Labels de status atualizadas com os valores do Monday (1. Autorization, 2. Payment, 3. Processing, 4. Reimbursement).
  - Itens do board zerados (removidos os testes) e preenchidos **apenas**: grupo **Mensal** (9 itens/41 subitens, cópia exata do Monday, com Ref alterada para **2026-08-01** a pedido do Jovis) e **Doing | Monthly** (9 itens/41 subitens, cópia exata, Ref 2026-07-15). Demais grupos vazios de propósito.
  - **Automação AutoDateDuplication corrigida**: destino mudou de Doing | Special Purchases → **Doing | Monthly**. Origem Mensal, data Ref, habilitada. Vai disparar em/após 2026-08-01.
  - Verificado no banco: contagens Mensal 9/41 e Doing | Monthly 9/41, trigger novo confirmado.

## 2026-07-12 (parte 3)
- **v2.3**: ajustes de UX no `index.html` (local, ainda não publicado):
  - Subitens agora renderizam em tabela aninhada própria → colunas de subitem redimensionáveis independentemente das colunas de item (chaves `s_name`/`s_<colId>` no localStorage de larguras) e o texto acompanha a largura (`.cellv max-width:100%`).
  - Clique no título de item/subitem abre o painel de detalhes (com atraso de 260ms p/ não conflitar); duplo clique ou botão ✎ (aparece no hover) renomeia inline. "Open details" e "Rename" removidos dos menus.
  - Subitem: menu ⋯ substituído por ícone 🗑 (delete com confirm). Item mantém o menu ⋯ (Move/Duplicate/Delete).
  - Calendar: seletor de coluna de data ganhou opção **Both (item + subitem)** — item usa a 1ª coluna date de escopo item, subitem a de escopo subitem; criação e drag-and-drop respeitam o escopo.
- Validado com `node --check`.
- **Fix v2.3.1**: `table-layout:fixed` era ignorado porque a tabela tinha `width:auto` → colunas cresciam com o texto e o resize não "pegava". Agora a largura da tabela (item e subitem) é a soma das larguras das colunas em px; durante o arraste a tabela acompanha o delta; nomes truncam com ellipsis. Validado com `node --check`.

## 2026-07-15 (parte 7)
- **v3.1 — Fase 5B (IA escreve via aprovação)**: o chat agora propõe ações num bloco ```json (`add_item`/`add_subitem`); `aiExecuteActions` mapeia board/grupo/coluna por nome→id e insere como PENDENTE (`_pending`); a IA responde "✅ Feito... vá aprovar no AI Input do board X". System prompt reforça: texto plano, SEM asteriscos/negrito. Bug de shadow do `sb` já corrigido no v2.9 herdado.
- Testado end-to-end: pedido "crie item ZZ_TESTE_5B no MKT/Planned com status/date" → criado pendente, apareceu no AI Input do MKT, resposta sem asteriscos, depois limpo. Tudo OK.

## 2026-07-15 (parte 6)
- **v3.0 — Fase 5 (chat de consulta, só leitura)**: FAB ✨ + painel de chat no #app; `aiBoardsDigest()` monta um resumo de todos os boards (exclui pendentes) e envia junto com a pergunta ao Edge Function `gemini` via `sb.functions.invoke`. System prompt pt-BR, anti prompt-injection, deixa claro que é só leitura. Testado no app: pergunta sobre contagem de itens → resposta correta dos 4 boards. No ar.
- Boundary: chave do Gemini nunca tocada pelo assistente (fica no segredo do Supabase).

## 2026-07-15 (parte 5)
- **Fase 3 — fundação serverless**: duas Edge Functions deployadas via editor do dashboard Supabase (conduzido pelo Chrome, setando o código pela API do Monaco):
  - `ping` (health, sem segredo) — testado via curl: HTTP 200 `{ok:true,service:lunes-ping}`.
  - `gemini` (proxy do Gemini; `GEMINI_API_KEY` só no servidor; CORS restrito a jovisarruda.github.io; modelo default gemini-2.5-flash) — testado: retorna `{"error":"GEMINI_API_KEY secret not set"}` (código OK, falta o segredo).
  - Código versionado em `supabase/functions/{ping,gemini}/index.ts`; guia `SETUP_FASE3_EDGE_FUNCTIONS.md`.
  - Boundary respeitado: **não digitei a chave**. Deixei a tela de Secrets aberta com o Nome `GEMINI_API_KEY` pré-preenchido; o Jovis cola o valor e salva.
  - Jovis salvou o segredo. No teste, `gemini-2.5-flash` deu 404 (descontinuado p/ novos usuários). Testei candidatos; **`gemini-flash-latest`** funciona (alias que acompanha o Flash mais recente). Troquei o default e redeployei. Teste OK: `{"text":"Olá! Confirmo que estou funcionando..."}` HTTP 200. Fundação serverless 100% operacional.

## 2026-07-15 (parte 4)
- **v2.8/v2.9 — Fase 2 (AI Input)**: camada de aprovação. Flag `col_values._pending` (sem schema change). Nova aba "AI Input" por board (lista/edita/aprova/rejeita pendentes), render fantasma na Table, badge de contagem, exclusão de pendentes de somas/Kanban/Calendar/automações. Aprovar item aprova subitens pendentes juntos.
- **Bug pego no teste ao vivo**: em `aiApprove`, `for(const sb of ...)` sombreava o cliente Supabase → `sb.from is not a function` ao aprovar subitens. Corrigido (renomeado p/ `su`) no v2.9. Testado end-to-end com itens ZZ_TEST (criar/aprovar/rejeitar/limpar), board restaurado.
- Nota: cache do navegador pode servir versão antiga após deploy — hard-refresh (Ctrl+F5) resolve.

## 2026-07-15 (parte 3)
- **v2.7**: corrigido o drag reorder — a Table agora ordena por `position` (itens e subitens); antes desenhava na ordem do array, então arrastar não movia. Testado ao vivo (troca + desfaz, ordem final intacta). No ar.
- **Keep-alive — email de falha explicado/resolvido**: a falha era do 1º commit do workflow (`b432668`, YAML quebrado). Já corrigido no `2334e79` (`run: |`). Disparei manualmente (workflow_dispatch) e rodou **verde** (success). Nada pendente; passa a rodar a cada 5 dias sem falhar.

## 2026-07-15 (parte 2)
- **v2.6 — Fase 1** (base v2.5): arrastar itens/subitens (alça ⠿, reordena dentro do grupo/pai e salva `position`) + seleção múltipla por checkbox com barra de lote (Change value / Delete, com confirmação). Validado (node --check + teste da lógica de reorder) e verificado no app (22 checkboxes/alças, barra de lote aparece, funções OK). No ar.
- Observação: o board FIN | Purchase foi bastante expandido pelo Jovis (grupo Mensal com vários templates 00/0000 e grupos Doing | Monthly / Done | Monthly com cópias 07/2026).

## 2026-07-15
- Jovis editou o app em paralelo: commits **v2.3** (colunas de subitem redimensionáveis, rename inline, lixeira no subitem, opção "Both" no calendário) e **v2.4** (notificações de automação descrevem o gatilho e clique abre o item). Fonte de verdade passou a ser o `main` do GitHub; minha cópia local foi ressincronizada a partir dele.
- **v2.5** (`74e1ca2`): botão **Download CSV** no menu do board (exporta board inteiro: grupos/itens/subitens; BOM + escape). Validado e no ar.
- Decisões: manter infra no Supabase (não migrar p/ HostGator); aceitar Gemini free (dados podem ser usados p/ treino); aprovação da IA unificada numa view **AI Input**; keep-alive via GitHub Actions (a executar quando aprovado); backup diário CSV no Drive é viável e sem limite (fase futura).
- Criado `Roadmap.md` com a lista de ajustes ordenada (simples → IA) para revisão.
- **Keep-alive implementado**: `.github/workflows/keepalive.yml` (cron `0 12 */5 * *` = a cada 5 dias + workflow_dispatch) pinga o REST do Supabase pra evitar pausa por inatividade. Correção no meio: `run:` precisou virar bloco `run: |` porque o `apikey:` quebrava o YAML. YAML validado via API.
- **Processo:** sempre puxar `main` antes de editar o `index.html` e validar antes do push (evitar sobrescrever/truncar).

## 2026-07-12 (parte 2)
- **v2.2** (`e095c9a`): programado o tipo de automação configurável **AutoDateDuplication** no `index.html` (gatilho + UI no painel Automate + motor em runTimeAutomations). Helpers `addMonth`/`stampMonthName`/`monthStamp`. Validado com `node --check` e testes unitários das helpers.
- Durante a edição, o `index.html` truncou; recuperado a partir da base limpa do GitHub raw e reaplicadas as 8 edições de forma verificada antes do push.
- Push do v2.2 via Chrome; Pages redesdobrou; confirmado no ar.
- Verificado o estado real do Supabase (app logado no Chrome): já existiam FIN | Purchase, FIN | Income e Daily | Weekly. Daily | Weekly já batia com o Excel (22 itens/122 subitens) — não reimportado.
- **MKT | Calendar criado/importado** do Excel via `sb` autenticado: 4 grupos, 15 itens, 64 subitens. Status de rollup do Monday nos itens-pai foram limpos.
- **Automação AutoDateDuplication configurada no FIN | Purchase** (Ref → Mensal → Doing | Special Purchases), habilitada (id `dd75c1ea...`). Dispara em/após 2026-08-15. Renderização da regra confirmada no painel.

## 2026-07-12
- Projeto reconstruído após perda do histórico (notebook). Lidos `SPEC.md`, `supabase-config.md` e `index.html` para recuperar contexto.
- Criados `Premissas_setup.md` (documento vivo de decisões) e este `Updates_log.md`.
- Criado texto de instruções padrão do projeto.
- Verificação do estado real: Supabase confirmado no ar com 10 tabelas + bucket `files` (via API REST). Dados dos boards não verificáveis (RLS exige login).
- Localizado o repo GitHub https://github.com/jovisarruda/Lunes e o app no ar em https://jovisarruda.github.io/Lunes/ (Pages ativo).
- **Achado**: o `index.html` LOCAL está à frente do publicado. A versão no ar (commit `Lunes v2`, 60.690 bytes) não tem: total = soma de subitens, redimensionar colunas, sidebar recolhível, addChoice, tema verde. Última coisa que faltava = push do index.html local para o `main`.
- **RESOLVIDO**: push feito via GitHub (upload/commit no `main`, commit `9ef7718` — "v2.1 - sum-of-subitems total, column resize, collapsible sidebar, person/label add-choice, green theme"). GitHub Pages redesdobrou automaticamente; app no ar confirmado com 66.329 bytes e todos os recursos novos. Repo agora com 3 commits.

## 2026-07-11
- Definida a spec do produto (`SPEC.md`): substituto do Monday, frontend estático + Supabase, UI em inglês.
- Supabase configurado (`supabase-config.md`): projeto ca-central-1, auth email+senha sem confirmação, RLS `auth_all` em todas as tabelas.
- Schema criado via SQL Editor: boards, groups, board_columns, items, automations, automation_logs, notifications, comments, activity_logs.
- Desenvolvido `index.html` (v2): tabela/kanban/calendário, filtros, edição inline, total = soma de subitens, automações (edição + tempo), notificações in-app, comentários, activity log, anexos (bucket `files`, 15 MB).

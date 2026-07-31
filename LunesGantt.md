# LunesGantt — Requisitos do modelo de página "Project"

> Documento de requisitos. Criado em 2026-07-28 · Última atualização: 2026-07-29.
> Fonte: pedido do Jovis + base atual do Lunes (`SPEC.md`, `Premissas_setup.md`, `index.html`).
> Status: **Fase A (protótipo) CONCLUÍDA** — protótipo em `LunesGantt_prototype.html`, no ar em https://jovisarruda.github.io/Lunes/LunesGantt_prototype.html. Próximo: **Fase B (integração)**. Inventário e mapeamento na seção 14.

---

## 1. Objetivo

Criar um **novo tipo de página ("modelo Project")** dentro do Lunes, no estilo de um MS Project simplificado. O coração do modelo é a **visualização Gantt**. O modelo Project reaproveita a base atual (views de lista e kanban, com itens e subitens), mas com regras próprias e novas abas de visualização.

Meta de qualidade: **visual limpo e minimalista, fácil de enxergar**. Tudo que for configuração ou detalhe avançado deve morar em um painel de contexto acessível por um botão de engrenagem (⚙), não poluindo a tela principal.

---

## 2. Conceito de "modelos de página" (templates)

Hoje todos os boards do Lunes são do mesmo tipo. A partir daqui passamos a ter **modelos de página**:

- Ao criar uma página, o usuário escolhe um **modelo** (ex.: modelo padrão atual, modelo **Project**).
- **Depois de criada, a página não pode trocar de modelo.** O modelo é imutável para aquela página.
- O modelo Project habilita features exclusivas (abaixo) que **não** se aplicam às páginas do modelo padrão — em especial **subitem de subitem** (hierarquia de 3+ níveis).

### Impacto no modelo de dados atual
- A tabela `boards` ganha um campo de modelo, ex.: `template` = `'default' | 'project'` (default para os boards existentes).
- Hoje `items` já tem `parent_id` (subitem = item com pai) e só existem 2 níveis. Para o modelo Project precisamos de **profundidade arbitrária (recursiva)**: item → subitem → subsubitem → … A lógica atual (`topItems()`, `subsOf(id)`) precisa virar recursiva, mas **só quando `board.template === 'project'`**.
- Colunas hoje têm `scope` = `item | subitem`. No Project a hierarquia é recursiva; provável que todas as "linhas" compartilhem o mesmo conjunto de colunas de atividade (ver dúvida 12.3).

---

## 3. Abas de visualização do modelo Project

O modelo Project tem as seguintes abas (a barra de tabs já existe: `setView(...)`):

1. **List** — a tabela hierárquica atual, agora aceitando subitem de subitem (3+ níveis).
2. **Kanban** — como hoje.
3. **Gantt** — nova (seção 4).
4. **Workload** — nova (seção 5).

Cada aba renderiza a mesma base de atividades; muda só a forma de visualizar/editar.

---

## 4. Aba Gantt

### Convencao de esforco aprovada (2026-07-30)
- Numero sem sufixo representa dias.
- Sufixos aceitos: `h` horas, `d` dias, `w` semanas, `m` meses e `y` anos.
- Exemplos validos: `5`, `8h`, `2w`, `1m`, `1y`.
- Itens antigos sem unidade persistida sao lidos como horas para nao reinterpretar dados existentes.

Gantt **minimalista e legível**. Requisitos:

### 4.1 Estrutura visual
- Painel esquerdo: **lista/árvore de atividades** (nome, hierarquia recolhível).
- Painel direito: **linha do tempo** com as barras.
- **Barras de resumo** (atividades-pai) desenhadas de forma distinta das **barras de tarefa** (folhas). Por padrão a barra do pai é **calculada** a partir dos filhos (menor início / maior fim). Se o usuário definir **datas próprias** para um pai, mostrar **mensagem de atenção** (deixa de ser resumo automático) — ver decisão 12.3.

### 4.2 Edição direta
- **Editar o nome** das atividades inline (na árvore à esquerda).
- **Mover as datas** de uma atividade arrastando a barra (mudar início e fim juntos) e **redimensionar** as pontas (mudar só início ou só fim = mudar duração).
- **Alterar o zoom / escala do cronograma**: dia / semana / mês / trimestre (largura da coluna de tempo).

### 4.3 Dependências (setas)
- **Conectar setas entre atividades** para indicar dependência, arrastando de uma barra para outra.
- Suporte aos **4 tipos** de dependência:
  - **TI — Término-Início (FS)**: a sucessora começa quando a predecessora termina. (padrão)
  - **II — Início-Início (SS)**: começam juntas.
  - **TT — Término-Término (FF)**: terminam juntas.
  - **IT — Início-Término (SF)**: a sucessora termina quando a predecessora começa. (raro)
- Opcional (a decidir): **lag/lead** (folga +/- dias) na dependência. Deixar como config avançada no ⚙.

### 4.4 Marcos (milestones)
- Atividade com **esforço/duração zero** vira **marco**, desenhado como **losango (◆)** na linha do tempo, sem barra.

### 4.5 Cor das barras
- O usuário escolhe **por qual variável colorir** as barras: por **Status**, por **Pessoa**, (e outras colunas categóricas — ex.: label/categoria).
- Seletor fica no cabeçalho do Gantt ou no ⚙. Legenda de cores visível/compacta.

### 4.6 Caminho crítico
- Botão que **calcula e mostra o caminho crítico atualizado**.
- Exibição: destacar as atividades/setas críticas **no próprio Gantt** (ex.: barras em cor/contorno de destaque) — e/ou resumo em **popup**. (o que for mais simples de implementar; provável: destaque no Gantt + botão que abre popup com a lista ordenada.)
- Caminho crítico recalcula a partir de durações + dependências (algoritmo CPM: forward/backward pass, folga total = 0).

### 4.7 Recalculo automático de datas (engine de agendamento)
Quando o usuário muda a data de uma atividade e isso **quebra o encadeamento** (sobrepõe/conflita com dependências), o sistema **não altera nada silenciosamente**. Em vez disso:

1. **Avisa o usuário** e mostra uma **lista de todas as atividades que precisariam ser empurradas** (para frente ou para trás) para acomodar a mudança.
2. Avisa **explicitamente** se a mudança forçaria empurrar a **data zero (kickoff)** ou a **deadline (data final)** — datas que normalmente **não podem** ser alteradas.
3. Deixa o usuário **decidir**:
   - (a) deixar o sistema **empurrar** as atividades afetadas (para frente ou para trás), ou
   - (b) **não empurrar** e, manualmente, **reduzir a duração** das atividades que ele escolher para encaixar no kickoff/deadline.
- Ou seja: proposta de mudança com preview + confirmação, nunca cascata automática cega.

### 4.8 Exportar PDF
- Botão para **exportar o Gantt em PDF** (cronograma legível, com a árvore de atividades e a linha do tempo).

### 4.9 Painel de configuração (⚙)
Para manter a tela limpa, mandar para o painel de engrenagem tudo que for secundário, ex.:
- Definição de kickoff (data zero) e deadline do projeto.
- Escolha da variável de cor das barras.
- Mostrar/ocultar: fins de semana, grade, folgas (lag), % concluído, nomes nas barras.
- Calendário de trabalho (dias/horas úteis — ligado à aba Workload).
- Opções de exportação PDF.

---

## 5. Aba Workload

Objetivo: enxergar a **carga de trabalho por pessoa ao longo do tempo**.

### 5.1 Entradas (inputs) da aba
- **Lista de pessoas** = a **mesma lista** usada para preencher a pessoa responsável de uma atividade (coluna person do board).
- Por pessoa: **input de horas dedicadas ao projeto** por **dia / semana / mês** (quanto daquela pessoa está alocado a ESTE projeto).
- Variáveis globais do projeto:
  - **Horas de trabalho por dia** (ex.: 8h).
  - **Dias de trabalho por semana** (ex.: 5).

### 5.2 Esforço por atividade (afeta todas as abas)
- Toda atividade passa a ter uma variável de **esforço/custo**: quantidade de **horas / dias / semanas / meses** que a atividade vai custar.
- Esse input aparece como **coluna** disponível também em List e Gantt (é o mesmo dado). **Esforço é independente das datas** — não recalcula início/fim (decisão 12.2); serve para o cálculo de workload.

### 5.3 Tabela de workload
- **Pessoas na vertical**, **meses na horizontal**.
- Cada célula mostra o **somatório de horas de trabalho** daquela pessoa naquele mês, no formato **`15/40`**:
  - `15` = horas de trabalho **alocadas** a este projeto naquele mês (soma do esforço das atividades da pessoa naquele mês);
  - `40` = horas **disponíveis/dedicadas** ao projeto naquele mês (derivadas do input de horas por dia/semana/mês da pessoa × dias úteis do mês).
- Sinalização visual de **sobrecarga** (quando alocado > disponível), mantendo o visual limpo.

---

## 6. Dependências — resumo técnico

| Sigla PT | Sigla EN | Nome | Regra |
|---|---|---|---|
| TI | FS | Término-Início | sucessora inicia após término da predecessora (padrão) |
| II | SS | Início-Início | sucessora inicia junto com o início da predecessora |
| TT | FF | Término-Término | sucessora termina junto com o término da predecessora |
| IT | SF | Início-Término | sucessora termina quando a predecessora inicia (raro) |

Todas devem ser suportadas na criação da seta e respeitadas pelo engine de agendamento (4.7) e pelo CPM (4.6).

---

## 7. Marcos (milestones)
- Definição: atividade com **duração/esforço = 0**.
- Render: **losango ◆** no Gantt (sem barra).
- Podem ser predecessora/sucessora em dependências normalmente.

---

## 8. Exportação
- **PDF** do Gantt (mínimo obrigatório).
- (Já existe export CSV do board no app atual — manter.)

---

## 9. Princípios de UI/UX
- **Minimalismo acima de tudo**: tela principal mostra só o essencial (árvore + timeline + barras).
- Botão **⚙ de configuração** concentra o avançado (seção 4.9).
- Ações destrutivas ou em cascata (empurrar datas) sempre com **preview + confirmação**.
- Reaproveitar o estilo visual atual do Lunes (tema verde, CSS enxuto no `<style>`).

---

## 10. Restrições da plataforma atual (a respeitar na integração)
- App é **arquivo único `index.html`**, **vanilla JS, sem build**, Supabase como backend.
- Modelo de dados: `boards → groups → items(+parent_id) → subitems`, colunas por board com `scope`, valores em `col_values` (JSONB) com chaves reservadas (`_pending`, `_proposed`).
- UI em **inglês**; conversamos em **português**.
- Regras do projeto (ver `INSTRUCOES_DO_PROJETO.md`): manter `Premissas_setup.md` e `Updates_log.md` atualizados; validar `node --check` e final `</script></body></html>` antes de qualquer deploy.

---

## 11. Escopo desta iniciativa (faseamento macro)
1. **Fase A — Protótipo desktop standalone** ✅ **CONCLUÍDA (2026-07-29)**: todas as funcionalidades construídas num protótipo isolado (`LunesGantt_prototype.html`), sem backend, com dados fictícios. Publicado para revisão em **https://jovisarruda.github.io/Lunes/LunesGantt_prototype.html** (arquivo separado do `index.html` de produção). Inventário completo na **seção 14**.
2. **Fase B — Integração** (a iniciar): portar para a plataforma atual (modelo de página, schema, recursão, persistência no Supabase, deploy no `index.html`), **reaproveitando o que a plataforma já tem** (chatbot Gemini, comentários, activity log, status, colunas). Plano na seção 13 (Fase B) e mapeamento na seção 14.

Detalhe da proposta de etapas na seção 13.

---

## 12. Decisões (confirmadas com o Jovis em 2026-07-28)
1. **Stack do protótipo**: **HTML single-file no mesmo padrão do Lunes** (vanilla JS, sem build). Facilita a fusão na Fase B.
2. **O que dirige a duração**: **as datas início/fim mandam.** O esforço em horas é um dado à parte, usado **só na aba Workload** — não recalcula datas.
3. **Comportamento dos níveis pai**: por padrão, **estilo MS Project** — a barra do pai é **resumo automático** dos filhos (menor início / maior fim) e dependências ficam entre as tarefas-folha. **PORÉM** o modelo é **flexível**: o usuário pode dar **datas próprias a um nível pai**; nesse momento o sistema exibe uma **mensagem de atenção** avisando que aquele pai deixou de ser resumo automático (as datas dele não seguem mais os filhos).
4. **Calendário**: a escolha de **contar ou não fins de semana e feriados** fica no **painel de configuração (⚙)** da página. Ou seja, dias úteis vs. dias corridos é uma opção configurável, não fixa.

---

## 13. Proposta de desenvolvimento em etapas

### FASE A — Protótipo desktop (isolado, dados fictícios) — ✅ CONCLUÍDA
- **A0 ✅ Fundação**: árvore + timeline, dados mock, zoom dia/semana/mês, cabeçalho de datas em 2 níveis, linhas Kickoff/Today/Deadline, pan horizontal (arrastar / Shift+roda).
- **A1 ✅ Árvore hierárquica infinita**: List recursiva (N níveis), editar nome inline, recolher/expandir, coluna de esforço.
- **A2 ✅ Barras**: barras por início/fim, marcos (losango), barras de resumo (roll-up), arrastar e redimensionar.
- **A2+ ✅ Operações**: menu ⋮ por linha (Details, add elemento/subelemento, duplicar, subir/descer hierarquia, recalcular datas, excluir); drag-reorder com **aninhar por dwell de 2s**.
- **A3 ✅ Dependências**: criar setas arrastando ○ (lado do drop define o tipo), 4 tipos TI/II/TT/IT, clicar seta p/ trocar tipo/excluir, guarda de ciclo.
- **A4 ✅ Engine de agendamento**: cascata com **preview + lista de afetados (→/←) + aviso de kickoff/deadline** e opções (empurrar / mover só este / cancelar p/ ajuste manual).
- **A5 ✅ Caminho crítico**: CPM (forward/backward, folga=0), destaque no Gantt + popup.
- **A6 ✅ Cor das barras**: por status/pessoa/categoria + legenda.
- **A7 ✅ Workload**: dedicação por pessoa (h/dia/semana/mês), variáveis globais, add/remover pessoas, tabela pessoas×períodos no formato `alocado/disponível`, **granularidade dia/semana/mês**, sobrecarga em vermelho.
- **A8 ✅ Export PDF** (via print, CSS de impressão A3 paisagem).
- **A9 ✅ Painel ⚙**: kickoff/deadline, horas-dia, dias-semana, contar/mostrar fins de semana, cor das barras, Collapse/Expand e Export PDF.

**Extras adicionados na Fase A (além do plano original):**
- **Colunas de data** (Início/Término) editáveis na lista + **acoplamento esforço↔datas** (diálogo: manter esforço/mover data vs. recalcular; ou manter datas).
- **Coluna Person** editável na lista (datalist das pessoas).
- **ID sequencial imutável** por elemento (código de referência para predecessoras; nunca muda; sem reuso).
- **Coluna Predecessoras** (texto `1FS; 10FS`, default FS) 100% sincronizada com as setas; ignora IDs inválidos e reverte ciclos.
- **Painel "Details" por item** (qualquer nível) estilo card dos outros boards: campos nativos (ID, Person, Start, Finish, Effort, Milestone, Predecessors) + colunas configuráveis (Status, Category, Link, Notes) + **Comments** + **Activity log** + botão ⚙ **editor de colunas** (add/remover/renomear colunas; tipos text/number/date/status/person/label/link/checkbox; editar labels de status e choices).
- **ChatBot somente-leitura** (FAB ✨): responde dúvidas sobre o board (contagens, datas, caminho crítico, dependências, responsável, esforço, workload). **Não insere/edita/remove nada** e por isso **não tem aba AI Input**.
- **Largura da lista redimensionável**: painel inteiro + **cada coluna** individualmente (salvo no navegador via localStorage).

### FASE B — Integração à plataforma Lunes (a iniciar)
> Princípio-chave: **reaproveitar ao máximo o que o `index.html` já tem** (ver seção 14). Só criar o que é realmente novo (Gantt, Workload, hierarquia recursiva, engine de datas, dependências).
- **B0 — Modelo de página**: campo `template` (`'default' | 'project'`) em `boards`; seleção no create; **bloqueio de troca** após criado; gate das features Project (subitem de subitem, abas Gantt/Workload, etc.).
- **B1 — Hierarquia recursiva**: hoje `items.parent_id` só tem 2 níveis; generalizar `topItems()/subsOf()` para recursão (N níveis) **apenas** quando `template==='project'`.
- **B2 — Persistência do schema Gantt**: onde guardar cada dado novo:
  - datas início/fim, esforço, milestone, código sequencial → colunas dedicadas ou `col_values` (JSONB) do item;
  - **dependências** → nova tabela `links` (`from_item`, `to_item`, `type`, `lag`) ou array em `col_values`; decidir na B2;
  - kickoff/deadline, calendário (horas-dia, dias-semana, fins de semana), cor-por, dedicação por pessoa, larguras de coluna → config do board;
  - **ID sequencial** precisa de contador por board (nunca reusar; imutável).
- **B3 — Reuso do que já existe** (não reimplementar):
  - **Comentários e Activity log** já existem por item/subitem → o painel Details do Project usa os mesmos.
  - **Status / Person / Label / Link / Number / Date** já são tipos de coluna do board → o editor de colunas do Project reusa o `openColumns()` atual.
  - **ChatBot já existe na plataforma** (widget ✨ + Edge Function `gemini`, leitura de digest dos boards). No board Project ele deve rodar **em modo somente-leitura** (sem propor `add_item`/`set_values`, sem aba AI Input) — basta um system prompt/flag "read-only" para páginas `template==='project'`. **Não** criar um bot novo.
  - **Export CSV** do board já existe → manter, além do **PDF** do Gantt (novo).
- **B4 — Portar as views**: Gantt e Workload do protótipo para dentro do `index.html` (vanilla, tema verde), como novas abas ao lado de Table/Kanban/Calendar, exibidas só no template project.
- **B5 — Engine de datas & CPM no app**: portar o solver (forward/backward + preview modal) e o CPM; decidir client-side (como no protótipo) — sem servidor.
- **B6 — Automações/regras** existentes convivendo com o novo modelo; **RLS `auth_all`**; testes e deploy (validar `node --check` e fim `</script></body></html>`).
- **B7 — Docs**: atualizar `Premissas_setup.md`, `Updates_log.md`, `Roadmap.md`.

---

## 14. Estado do protótipo & mapeamento para a plataforma (para a Fase B)

### 14.1 O que a plataforma Lunes **já tem** (reusar, não reimplementar)
- `boards → groups → items(+parent_id) → subitems` (2 níveis hoje); colunas por board com `scope`; valores em `col_values` (JSONB, chaves reservadas `_pending`/`_proposed`).
- Tipos de coluna: text, number/currency, date, **status** (labels coloridos), **person**, **label**, **link**, checkbox.
- **Comentários** e **Activity log** por item e subitem.
- **Notificações** in-app; **automações** por board; **export CSV**.
- **ChatBot de IA já implementado**: widget ✨ no app + Edge Function `gemini` (chave só no servidor). Hoje faz leitura (digest dos boards) e também escrita via AI Input em outros boards. → No Project: **reusar em modo só-leitura**.

### 14.2 O que é **novo** e precisa ser construído na integração
- Abas **Gantt** e **Workload**; **hierarquia recursiva** (subitem de subitem); **ID sequencial imutável**; **dependências** (4 tipos) + storage; **engine de reagendamento** com preview; **CPM**; **acoplamento esforço↔datas**; **kickoff/deadline** + calendário de trabalho; **cor das barras**; **export PDF**; **larguras de coluna** persistidas.

### 14.3 Regras que persistem do protótipo
- Datas mandam a duração; esforço é só para Workload (decisão 12.2).
- Pai = resumo automático, com opção de datas manuais + aviso (decisão 12.3).
- Contar/mostrar fins de semana no ⚙ (decisão 12.4).
- ID nunca muda; predecessora referencia o ID; ciclos bloqueados.
- Bot do Project é **somente-leitura** (sem AI Input).

---

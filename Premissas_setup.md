# Premissas & Setup — Lunes

> Documento vivo. **Sempre que uma configuração for decidida, ela deve ser registrada aqui.**
> Última atualização: 2026-07-29

## O que é o Lunes
Ferramenta interna de gestão de trabalho da Liconic, um substituto simples e minimalista do Monday.com. Frontend estático (SPA em HTML/JS puro, arquivo único `index.html`) + Supabase (Postgres + Auth). **UI em inglês.**

## Regras locais do LunesGantt (2026-07-30)
- Effort aceita numero sem unidade como **dias**; unidades: `h` (horas), `d` (dias), `w` (semanas), `m` (meses) e `y` (anos). Exemplos: `5`, `8h`, `2w`, `1m`, `1y`.
- A unidade e persistida em `items.col_values._effortUnit`. Itens antigos sem essa chave continuam interpretados como horas para manter compatibilidade.
- Datas na lista e no Details somente sao confirmadas no `Enter` ou ao sair do campo; a digitacao intermediaria nao abre o reagendamento.
- A linha Today usa a data local real do navegador.
- Barras de resumo automaticas usam somente descendentes com inicio e termino validos; pais com `_manualDates` usam seu intervalo proprio.
- O reagendamento mantem a atividade alterada fixa: apenas sucessoras conectadas podem ser empurradas para a direita e apenas predecessoras conectadas podem ser puxadas para a esquerda, respeitando FS/SS/FF/SF e lag.

## Arquitetura
- **Frontend**: single-page app em vanilla JS, tudo em um único `index.html`. Sem framework, sem build. Usa o SDK `@supabase/supabase-js@2` via CDN.
- **Hospedagem**: GitHub Pages (site estático).
- **Backend**: Supabase (free tier) — Postgres + Auth. Sem servidor próprio.
- **Autenticação**: login individual por email+senha (Supabase Auth). Sem papéis/permissões — todo usuário logado vê tudo.

## Configuração do Supabase
- **Project URL**: https://ybzrpqgofxkzaqasfsna.supabase.co
- **Publishable (anon) key**: sb_publishable_0I8ZhiGBgfjWKBwTHV3TSA_11Kj3Jid
- **Região**: ca-central-1
- **Auth**: email+senha, confirmação de email desativada, signups habilitados
- **RLS**: todas as tabelas com política `auth_all` (qualquer usuário autenticado, acesso total)
- **Schema criado em**: 2026-07-11 via SQL Editor

### Tabelas
`boards`, `groups`, `board_columns`, `items` (subitens = `items` com `parent_id`), `automations`, `automation_logs`, `notifications`, `comments`, `activity_logs`, `attachments`.

### Modelo de página (template) — LunesGantt, iniciado em 2026-07-29 (Fase B)
- **Novo campo `boards.template`** (`text`, default `'default'`). Migração:
  ```sql
  ALTER TABLE boards ADD COLUMN IF NOT EXISTS template text NOT NULL DEFAULT 'default';
  ```
  (rodar no SQL Editor do Supabase **antes** de publicar o `index.html` novo — a criação de board passa a enviar `template`.)
- Valores: `'default'` (Table/Kanban/Calendar, como sempre) e `'project'` (modelo LunesGantt).
- **Imutável**: o template é escolhido só na criação (`addBoard()` pergunta via confirm) e não há UI para trocá-lo depois.
- **Gate**: `isProjectBoard()` = `curBoard.template==='project'`. Boards default seguem 100% inalterados.
- **B1 — hierarquia recursiva** (só project): `renderTable()` desvia para `renderProjectTable()` (tabela única indentada, N níveis, recursiva via `subsOf`/`parent_id`). `deleteItem` agora apaga a subárvore inteira (`descendantIds`, seguro p/ todos os boards). Boards project usam **um único conjunto de colunas** (escopo `item`): Person, Status, Start (date), Finish (date), Effort (h) — reaproveitado pela Gantt depois. `board_columns` de escopo `subitem` não são usados em boards project.
- **B4 CONCLUÍDO (2026-07-29)**: abas **Gantt** e **Workload** para boards project (módulo `window.G` no `index.html`). Gantt lê datas de `col_values` (colunas Start/Finish), esforço/pessoa/status idem, dependências da tabela `links`, config de `boards.settings`. Arrastar barra grava datas; arrastar ○ cria dependência; ⚙ Project grava settings. Marco via flag `col_values._milestone` (duplo-clique). Abas só aparecem em `template==='project'`.
- **B5 CONCLUÍDO (2026-07-29)** — fecha a Fase B (B0–B5). No módulo `G`: **motor de reagendamento** (`proposeChange`/`computeSchedule`/`applySchedule`) com preview em overlay (afetados, avisos kickoff/deadline, Push/Move only/Cancel); **CPM** (`computeCritical`) + botão Critical path (destaque vermelho + popup). Arrastar barra dispara o reagendamento; datas gravam em `col_values`.
- **Refinamentos futuros (fora da Fase B)**: coluna Predecessoras editável na lista do Project; chatbot só-leitura para boards project (reusar widget existente); Kanban/Calendar em boards project ainda mostram só 2 níveis; duplicação profunda de subárvore; export PDF do Gantt na plataforma.

### B2 — Persistência do schema Gantt (2026-07-29)
- **Nova tabela `links`** (dependências): `id uuid pk`, `board_id`, `from_item`, `to_item` (FKs → items, `on delete cascade`), `type` (`FS|SS|FF|SF`, default `FS`), `lag int default 0`, `created_at`. RLS `auth_all` (authenticated, full).
- **`boards.code_seq int default 1`** — contador de ID sequencial por board (imutável, nunca reusa). **`boards.settings jsonb default '{}'`** — config do projeto (kickoff, deadline, horas-dia, dias-semana, fins de semana, cor-por, dedicação por pessoa). **`items.code int`** — código do item (referência de predecessora; só preenchido em boards project).
- **SQL da migração** (rodar no SQL Editor antes de publicar):
  ```sql
  create table if not exists links (id uuid primary key default gen_random_uuid(), board_id uuid not null references boards(id) on delete cascade, from_item uuid not null references items(id) on delete cascade, to_item uuid not null references items(id) on delete cascade, type text not null default 'FS', lag int not null default 0, created_at timestamptz not null default now());
  alter table links enable row level security;
  drop policy if exists auth_all on links; create policy auth_all on links for all to authenticated using (true) with check (true);
  alter table boards add column if not exists code_seq int not null default 1;
  alter table boards add column if not exists settings jsonb not null default '{}'::jsonb;
  alter table items add column if not exists code int;
  ```
- **Código (`index.html`)**: global `links`; `selectBoard` carrega `links` do board (só project); `createItem` atribui `it.code = board.code_seq` e incrementa o contador (só project); `deleteItem` limpa links locais órfãos (DB faz cascade). Helpers: `addDep/removeDep/setDepType/depsOf/succOf/predText/setPredecessors` (parse `1FS; 10SS`, default FS, ignora ID inválido), `itemByCode`, `projSettings/saveProjSettings`. A lista de boards project ganhou a **coluna ID** (read-only). **Datas/Effort/Person/Status já persistem** via `board_columns`+`col_values` (colunas Start/Finish/Effort criadas no B0). Milestone/manual-dates (flags) → chaves reservadas em `col_values` quando forem implementadas no Gantt.
- Larguras de coluna seguem em **localStorage** (client), não no DB.

### Storage
- Bucket `files` (upload de anexos por item/subitem, público). Limite de **15 MB por arquivo**.

## Modelo de dados
- **Boards** → **Groups** (ordenados, coloridos, colapsáveis) → **Items** → **Subitems**.
- Colunas totalmente configuráveis **por board**, separadamente para itens e subitens (`scope` = `item` | `subitem`).
- **Tipos de coluna**: text, number/currency (R$), date, status (labels coloridos customizados por coluna), person (texto/select puro — sem conta, sem notificação), label/categoria, link (ex.: Google Drive), checkbox.
- Múltiplas colunas de status por board permitidas (ex.: Autorização → Pagamento → Processamento → Reembolso).
- **Rodapé de grupo**: soma automática para colunas number/currency.
- **Regra do total = soma de subitens**: para o primeiro par de colunas number (item + subitem), o total do item vira a soma dos subitens automaticamente. Digitar um total manual apaga os valores dos subitens (com confirmação), e vice-versa.

## Views (por board)
1. **Table** (principal): grupos com itens/subitens, edição inline, adicionar item/subitem, redimensionar colunas (largura salva em localStorage por board).
2. **Kanban**: colunas = valores de uma coluna de status escolhida; arrastar card muda o status.
3. **Calendar**: visão mensal por uma coluna de data escolhida; arrastar card muda a data.
- **Filtros** em todas as views: busca livre + por coluna (person, data range, status, número min/max, checkbox, texto).

## Automações
Módulo "Automate" por board. Regras montadas por frases com dropdowns (sem código): "When [gatilho], then [ação]".
- **Gatilhos**: status muda para X; uma data chega (hoje); todo dia N do mês; item criado.
- **Ações**: mover item para grupo X; setar coluna para valor; duplicar item (com subitens) para grupo X; arquivar item; notificar usuário X (in-app).
- Recorrência mensal (folha de pagamento etc.) = "todo dia 1 do mês → duplicar item".
- **Execução**: gatilhos de edição disparam na hora; gatilhos de tempo são checados e executados quando alguém abre o app (sem servidor 24/7 — aceitável dado o uso diário).
- **Log de automação**: toda execução é registrada (regra, item afetado, timestamp); histórico dentro do painel Automate de cada board.

## Notificações
- Ícone de sino na barra superior com contador de não lidas; **apenas in-app** (sem email/push).
- Criadas pela ação de automação "notify user X".
- Clicar para ver e marcar como lida.

## Atividade & comentários por item
- **Activity log** por item e subitem: criado por/quando, toda mudança de coluna registrada (valor antigo → novo, quem, quando).
- **Comentários** por item e subitem: autor + timestamp, thread estilo Monday.
- **Files**: aba de anexos por item/subitem (upload para o bucket `files`).

## Fora de escopo (explícito)
- Dashboards, docs, notificações por email/push, acesso por responsável, integrações, features de IA.
  (Armazenamento de arquivo era originalmente fora de escopo — usar coluna de link para Google Drive — mas depois foi adicionado upload de anexos via Supabase Storage.)

## Boards existentes no Supabase (verificado 2026-07-12)
Todos os 4 boards existem e estão populados:
- **FIN | Purchase** (reconfigurado 2026-07-12 a partir do export do Monday) — 10 grupos, na ordem do Monday: Input | Special Purchases / Doing | Special Purchases / Done | Special Purchases / Mensal / Doing | Monthly / Done | Monthly / Doing | Installment / Done | Installment / Doing | FAPESP PIPE FASE 1 PJ0012 / Done | FAPESP PIPE FASE 1 PJ0012. Colunas de item: Total (R$) [number], Ref [date], Status [status: Input/Working on it/Done/Monthly ON]. Colunas de subitem: Price [number], Due Pay [date], Paid In [date], 1. Autorization / 2. Payment / 3. Processing / 4. Reimbursement [status, labels do Monday], Category (21), Paid By (10), Payment (8 = Pay Mode), Supplier (190), Project (18) [todas label com choices importadas dos boards auxiliares do Monday]. **Dados**: só Mensal (9 itens, 41 subitens, Ref = 2026-08-01 — decisão do Jovis, era 2026-08-15 no Monday) e Doing | Monthly (9 itens, 41 subitens, Ref 2026-07-15, cópia exata do Monday). Todos os demais grupos vazios de propósito.
- **FIN | Income** — grupos: [Fixed List] Active Monthly Entries / To Be Processed / Done.
- **Daily | Weekly** — grupos: DOING / To Do | Paused | Blocked. 22 itens, 122 subitens (importado do Monday). Colunas item: Owner [person], Status, Priority [status], Deadline [date], Department [label].
- **MKT | Calendar** — grupos: Backlog / Doing / Planned / Done. 15 itens, 64 subitens (importado do Monday em 2026-07-12). Colunas item: Doc [link], Status [status], Date [date]. Subitem: Status [status], Channels [label], Date [date], People [person].

## Automação especial AutoDateDuplication (v2.2)
Tipo de automação configurável (dropdowns no painel Automate): "quando a data <Ref> chega, duplicar todos os itens do grupo <origem>, carimbar o nome com o MM/AAAA do mês atual (substitui um padrão MM/AAAA existente no nome, senão anexa), mover as cópias para o grupo <destino>, e empurrar as datas (item + subitens) dos originais +1 mês". Roda no máximo 1x por mês (guarda `trigger.last_run='AAAA-MM'`), disparada por qualquer item do grupo origem cuja data já chegou; é time-based (executa quando alguém abre o app).
- **Configurada no FIN | Purchase**: Ref → Mensal → **Doing | Monthly** (corrigida em 2026-07-12; antes apontava errado para Doing | Special Purchases). Regra habilitada, id `dd75c1ea...`. Dispara em/após 2026-08-01 (Ref dos itens do Mensal).
- Decisões: "código" = nome do item; mês = mês atual (hoje).

## Repositório & deploy
- **Repo GitHub**: https://github.com/jovisarruda/Lunes (público, só `index.html`).
- **App no ar (GitHub Pages)**: https://jovisarruda.github.io/Lunes/ — publica automaticamente a partir do branch `main`.
- **A pasta local do projeto NÃO é um repo git** (sem `.git`). O versionamento acontece direto no GitHub; a pasta local é o working copy.
- Commits: `Lunes v1` (11/07 16:45), `Lunes v2` (11/07 20:50), `v2.1` (12/07, `9ef7718`), `v2.2` (12/07, `e095c9a` — AutoDateDuplication).
- **Deploy via Chrome logado**: upload/commit do `index.html` pela interface do GitHub (Add file → upload). O Pages redesdobra sozinho a partir do `main`.
- **Importação de dados**: feita rodando JS no console do app já autenticado (usa o client `sb` da sessão), respeitando o RLS. Não dá pra inserir via chave anon.
- **ATENÇÃO — bug de edição observado**: uma edição grande no `index.html` truncou o arquivo. Se reeditar, sempre validar `node --check` no script e conferir que termina com `</script></body></html>` antes do push. Base limpa recuperável do GitHub raw.

## Estágio atual (em 2026-07-12) — VERIFICADO
- App funcional em arquivo único `index.html` (~1070 linhas), rotulado internamente como **v2**.
- Implementado: auth, boards/grupos/colunas, tabela + kanban + calendário, filtros, edição inline, total = soma de subitens, redimensionar colunas, sidebar recolhível, automações (edição e tempo), notificações in-app, comentários, activity log, anexos.
- Supabase configurado com schema completo (10 tabelas, incl. `attachments`) e RLS — confirmado via API REST em 2026-07-12.
- **SINCRONIZADO em 2026-07-12**: o `index.html` local (66.329 bytes) foi commitado/publicado no `main` (commit `9ef7718`). O app no ar agora está igual ao local, com: total = soma de subitens, redimensionar colunas, sidebar recolhível, addChoice em person/label, tema verde. Confirmado via GitHub Pages (66.329 bytes).
- **v2.2 (2026-07-12)**: adicionada a automação AutoDateDuplication (ver seção acima). App no ar confirmado.
- **Boards CONFIRMADOS no Supabase**: FIN Purchase, FIN Income, Daily/Weekly e MKT Calendar todos existem e populados (Daily já estava; MKT criado em 12/07). Ver seção "Boards existentes".

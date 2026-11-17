# Plano — Team Pulse

## Contexto

App web simples, só para uso pessoal em localhost (sem login), para acompanhar o time:
lista de pessoas, perfil de cada uma e registro de 1:1. O repositório `team-pulse` está
vazio (só `NOTAS.md`), então tudo é criado do zero.

Decisões já tomadas:

- **Stack:** TypeScript, Node 22, SQLite, sem framework pesado.
- **Interface:** HTML gerado no servidor com template strings e formulários HTML comuns
  (`<form method="post">`), sem build de frontend.
- **Pessoa:** nome, cargo, e-mail.
- **1:1:** data, notas, pulso (1–5).
- **Etapa 1:** listar + buscar por nome. Como você escolheu "formulário de cadastro" para
  ter dados, a etapa 1 inclui também um cadastro mínimo de pessoa. Sem script de seed.

## Escolhas técnicas

- **Servidor:** `node:http` puro com um roteador pequeno feito à mão (método + caminho).
  Sem Express.
- **SQLite:** `node:sqlite` (`DatabaseSync`), que já vem no Node: síncrono e sem dependência
  nativa. Exige Node ≥ 22.13 (primeira versão sem a flag `--experimental-sqlite`);
  `package.json` declara `"engines": { "node": ">=22.13" }` e o repositório ganha um `.nvmrc` com `22`.
- **TypeScript:** `tsx` para rodar em desenvolvimento e `tsc --noEmit` para checar tipos.
  Sem etapa de build.
- **Testes:** `node:test` (já vem no Node), rodando com `node --import tsx --test`.
- **Banco:** arquivo `data/team-pulse.db` (no `.gitignore`); os testes usam `:memory:`.
- **Migrações:** um array de SQL em `src/db.ts`, controlado por `PRAGMA user_version`.

## Etapas

### Etapa 1 — Lista do time e busca por nome (+ cadastro mínimo)

- `GET /` → redireciona para `/people`.
- `GET /people?q=texto` → lista em ordem alfabética, com uma caixa de busca (form GET).
  A busca ignora maiúsculas e acentos ("joao" encontra "João") e casa qualquer parte do nome.
- `GET /people/new` + `POST /people` → formulário com nome (obrigatório), cargo e e-mail;
  depois de salvar, redireciona para `/people`. Se houver erro de validação, mostra o
  formulário de novo com os valores já preenchidos.
- Tabela `people(id, name, name_search, role, email, created_at)`. `name_search` guarda
  o nome em minúsculas e sem acentos, e a busca é um `LIKE` nessa coluna.
- Lista vazia mostra "Nenhuma pessoa ainda" com um link para cadastrar.

Arquivos criados:

- `package.json` (scripts `dev`, `start`, `test`, `typecheck`; `engines`), `tsconfig.json`, `.gitignore`, `.nvmrc`
- `src/server.ts` — cria o servidor HTTP, lê o corpo dos formulários (`URLSearchParams`), roteia
- `src/db.ts` — abre o banco e roda as migrações; recebe o caminho do banco por parâmetro (para os testes)
- `src/people.ts` — `listPeople(db, q?)`, `createPerson(db, input)`, `normalizeName()`
- `src/html.ts` — tag `html\`\``que escapa valores automaticamente, e`layout()`
- `src/views/people.ts` — páginas da lista e do formulário
- `test/people.test.ts` — busca (acentos, maiúsculas, parte do nome, `%`/`_` literais) e validação
- `test/server.test.ts` — sobe o servidor numa porta aleatória e testa `GET /people?q=` e `POST /people`

### Etapa 2 — Perfil da pessoa

- `GET /people/:id` → nome, cargo, e-mail; responde 404 se o id não existir.
- Os nomes na lista passam a ser links para o perfil.
- Arquivos: `src/views/profile.ts`; `getPerson()` em `src/people.ts`; testes.

### Etapa 3 — Registrar novo 1:1

- Migração 2: `one_on_ones(id, person_id FK, date, pulse 1–5 CHECK, notes, created_at)`.
- `GET /people/:id/one-on-ones/new` + `POST /people/:id/one-on-ones` → data (padrão: hoje),
  pulso (botões de 1 a 5) e notas; depois de salvar, volta para o perfil.
- O perfil mostra o histórico de 1:1, do mais recente para o mais antigo.
- Arquivos: `src/oneOnOnes.ts`, `src/views/oneOnOne.ts`, `test/oneOnOnes.test.ts`.

### Etapa 4 (opcional) — Pulso do time

- A lista mostra o último pulso e a data do último 1:1 de cada pessoa, para ver quem está
  sem conversa há muito tempo. Editar e excluir pessoa entram aqui ou depois.

Cada etapa termina com testes passando, `typecheck` limpo e um commit.

## O que pode dar errado

- **`node:sqlite` ainda é experimental:** mostra um `ExperimentalWarning` ao iniciar e a API
  pode mudar entre versões do Node. Em Node < 22.13 o import falha sem a flag. Mitigação:
  `engines` + `.nvmrc`, todo acesso ao banco concentrado em `src/db.ts` (fácil trocar por
  `better-sqlite3` se precisar), e uma checagem da versão do Node na inicialização com
  mensagem clara.
- **XSS:** nomes e notas vão para o HTML. Mitigação: sempre gerar HTML pela tag `html`,
  que escapa os valores; nunca concatenar strings direto.
- **Busca com acentos:** o `LOWER`/`LIKE` do SQLite só trata ASCII ("Á" não vira "á").
  Mitigação: coluna `name_search` normalizada no TypeScript (`NFD` + remoção dos acentos).
  Ela precisa ser atualizada sempre que o nome mudar.
- **Curingas do LIKE:** buscar `%` ou `_` traria resultados errados. Mitigação: escapar
  esses caracteres e usar `ESCAPE '\'`; sempre usar parâmetros `?`, nunca interpolação (evita SQL injection).
- **Corpo do POST:** sem framework, é preciso ler o stream, limitar o tamanho (ex.: 100 KB)
  e tratar a codificação de caracteres. Testar com acentos.
- **Reenvio de formulário:** após cada POST, redirecionar com 303 (padrão Post/Redirect/Get),
  para que atualizar a página não duplique o cadastro.
- **Datas e fuso horário:** gravar a data do 1:1 como `YYYY-MM-DD` (data local, sem hora),
  para evitar o 1:1 aparecer no dia anterior por causa do UTC.
- **CSRF / exposição na rede:** sem login, qualquer pessoa na rede local poderia acessar.
  Mitigação: escutar só em `127.0.0.1`.
- **Backup:** os dados ficam num único arquivo `.db` fora do git; se ele for apagado, tudo se perde.

## Verificação

- `npm test`: testes unitários e de servidor passam.
- `npm run typecheck`: sem erros de tipo.
- Teste manual na etapa 1: `npm run dev`, abrir `http://127.0.0.1:3000`, cadastrar "João Silva"
  e "Maria", buscar "joao", "SIL" e "%", e conferir que um nome com `<script>` aparece como texto.

## Depois da aprovação

Salvar este plano como `PLAN.md` na raiz do repositório, substituindo a versão anterior
(Node 20) que está lá sem commit. O código da etapa 1 só começa quando você pedir.

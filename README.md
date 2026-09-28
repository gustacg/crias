# Crias

CRIAS é um aplicativo de hábitos e metas em grupo com gamificação no estilo de um jogo de RPG. A pessoa cria rotinas, entra em grupos, faz o check in do que cumpriu no dia e ganha ouro, vida e ofensiva (dias seguidos). Tem loja de personagens, baú de prêmios, ranking semanal do grupo e feed com foto de cada check in.

Ele roda no navegador do celular e pode ser instalado na tela de início como um aplicativo comum (PWA). As notificações push são o coração do produto: lembram a pessoa na hora certa e o check in acontece em dois toques, inclusive direto pela notificação.

Este guia leva você do zero até o seu próprio Crias no ar. Siga as seções na ordem.

---

## Stack

| Camada | Tecnologia |
| --- | --- |
| Front | React 19, Vite, TypeScript, Tailwind, shadcn/ui, lucide-react |
| Backend | Supabase: Postgres, Auth, Storage, RLS, `pg_cron`, `pg_net` e Edge Functions em Deno |
| Push | Web Push nativo com VAPID, cifragem escrita à mão em `supabase/functions/_shared/webpush.ts` |
| Deploy | Vercel |
| Testes | Vitest (unidade) e Playwright (ponta a ponta contra um Supabase real) |

### Estrutura de pastas

| Pasta | O que tem |
| --- | --- |
| `src/` | O aplicativo React: páginas, componentes, hooks e regras de tela |
| `public/` | Service Worker (`sw.js`), manifesto do PWA, ícones e os sprites já processados em `public/sprites/` |
| `supabase/migrations/` | Todo o banco: tabelas, RLS, funções, gatilhos, jobs do cron e o catálogo da loja. Numeradas de `0001` em diante |
| `supabase/functions/` | As três Edge Functions: `push-dispatch`, `quick-check-in` e `limpar-fotos` |
| `arte-origem/` | Arte original versionada: a logo e os personagens da família "Crias da casa" |
| `scripts/` | Pipeline de arte (sprites, catálogo, ícones) e o executor de SQL |
| `testes/` e `src/**/*.test.ts` | Testes de unidade |
| `e2e/` | Testes de ponta a ponta, inclusive os de segurança que tentam acesso cruzado entre usuários |

---

## 1. O que instalar

1. **Node.js 20 ou mais novo**: `https://nodejs.org`. Confira com `node --version`.
2. **Git**.
3. Uma conta gratuita no **Supabase** (`https://supabase.com`) e outra na **Vercel** (`https://vercel.com`).
4. Opcional: **Docker**, só se você quiser rodar o Supabase inteiro no seu computador.

A CLI do Supabase não precisa ser instalada: todos os comandos abaixo usam `npx supabase`.

---

## 2. Baixar e instalar

```bash
git clone https://github.com/gustacg/crias.git
```

```bash
cd crias
```

```bash
npm install
```

```bash
cp .env.example .env
```

O `.env` é ignorado pelo git e **nunca** deve ser commitado. O `.env.example` é a lista em branco dos campos.

---

## 3. Criar o projeto no Supabase

1. No painel do Supabase, crie um projeto novo. Anote a **senha do banco** que você escolher.
2. O **ref** do projeto é o código que aparece na URL do painel e no endereço da API: `https://<ref>.supabase.co`.
3. Em **Project Settings → API**, copie a **URL**, a chave **anon** e a chave **service_role**.
4. Em **Account → Access Tokens**, gere um **Personal Access Token** (começa com `sbp_`).
5. Abra `supabase/config.toml` e troque o valor de `project_id` pelo seu ref. Os scripts do projeto recusam rodar quando o ref do `.env` e o do `config.toml` não batem, e isso protege você de aplicar SQL no projeto errado quando a sua conta tem mais de um.

### Gerar as chaves de notificação (VAPID)

```bash
npx -y web-push generate-vapid-keys
```

O comando imprime uma chave pública e uma privada.

### Gerar o segredo interno

É a senha que o cron do banco usa para chamar as Edge Functions. Qualquer texto aleatório longo serve:

```bash
openssl rand -hex 32
```

### Preencher o `.env`

| Variável | O que é | Vai para o navegador? |
| --- | --- | --- |
| `VITE_SUPABASE_URL` | `https://<ref>.supabase.co` | Sim, é pública |
| `VITE_SUPABASE_ANON_KEY` | Chave anon. Sozinha não lê dado de ninguém: quem decide é a RLS | Sim, é pública |
| `VITE_VAPID_PUBLIC_KEY` | Chave pública VAPID | Sim, é pública |
| `VITE_APP_TIMEZONE` e `TZ` | Sempre `America/Sao_Paulo` | |
| `SUPABASE_PROJECT_REF` | O ref do projeto, igual ao `project_id` do `config.toml` | Nunca |
| `SUPABASE_ACCESS_TOKEN` | O token `sbp_`. Abre **todos** os projetos da sua conta | Nunca |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave mestra do banco, ignora a RLS | Nunca |
| `SUPABASE_DB_PASSWORD` | Senha do banco | Nunca |
| `SUPABASE_ORG_ID` | Id da organização no Supabase (opcional) | Nunca |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` | O par VAPID. A privada assina as notificações | Nunca |
| `VAPID_SUBJECT` | `mailto:seu@email.com`, contato exigido pelo padrão Web Push | Nunca |
| `CRIAS_INTERNAL_SECRET` | O segredo interno que você gerou | Nunca |
| `VERCEL_TOKEN`, `VERCEL_APP_URL` | Opcionais, só para operar a Vercel pela API | Nunca |

**Regra de ouro:** tudo que começa com `VITE_` é embutido no JavaScript que qualquer visitante baixa. Segredo nunca leva o prefixo `VITE_`.

---

## 4. Criar o banco: as migrations

Uma **migration** é um arquivo SQL numerado em `supabase/migrations/`. Juntas, as migrations constroem o banco inteiro do zero: tabelas, regras de segurança (RLS), funções, gatilhos, os jobs do `pg_cron`, os buckets de foto e o catálogo de itens da loja. Elas são aplicadas **em ordem crescente**, e cada uma só roda uma vez.

### 4.1 Aplicar todas de uma vez (projeto novo)

Carregue o `.env` no terminal:

```bash
set -a; source .env; set +a
```

Ligue a pasta ao seu projeto:

```bash
npx -y supabase@latest link --project-ref "$SUPABASE_PROJECT_REF" --password "$SUPABASE_DB_PASSWORD"
```

Aplique tudo:

```bash
npx -y supabase@latest db push --password "$SUPABASE_DB_PASSWORD"
```

A CLI mostra a lista do que vai aplicar e pede confirmação. Ela registra cada migration aplicada no próprio banco, então rodar `db push` de novo só aplica as que faltam.

### 4.2 Três ajustes obrigatórios depois da primeira aplicação

As migrations foram escritas para o projeto original, e três coisas dependem do **seu** projeto. Rode no **SQL Editor** do painel do Supabase, trocando os valores entre `< >`.

**a) Apontar o cron para as suas Edge Functions.** As migrations `0012` e `0028` agendam jobs que chamam a URL do projeto original. Este comando reescreve os jobs para o seu ref:

```sql
select cron.alter_job(jobid, command := replace(command, 'oeaftenwsmbkdxqseqrb', '<SEU_REF>'))
from cron.job
where command like '%oeaftenwsmbkdxqseqrb%';
```

**b) Guardar o segredo interno no banco.** O cron lê o segredo de uma tabela no schema `privado`, que é invisível para a API:

```sql
insert into privado.config (chave, valor)
values ('internal_secret', '<SEU_CRIAS_INTERNAL_SECRET>')
on conflict (chave) do update set valor = excluded.valor, atualizado_em = now();
```

**c) Conferir os jobs.** Devem aparecer sete: `expurgar-intencoes`, `gerar-ocorrencias`, `limpar-fotos` (nasce **desligado** de propósito, veja abaixo), `marcar-atrasadas`, `premiar-semana`, `push-dispatch` e `resolver-validacoes`:

```sql
select jobname, schedule, active from cron.job order by jobname;
```

### 4.3 Sobre as migrations de dados

Algumas migrations (`0033`, `0039`, `0040`, `0044`) ajustaram rotinas específicas do grupo que usava o app original, identificadas por id. Em um banco novo esses ids não existem, então os comandos não encontram nada e passam sem efeito. Elas continuam na sequência porque o histórico de migrations nunca é reescrito.

O job `limpar-fotos` apaga do Storage fotos sem check-in há mais de 30 dias. Ele nasce desligado. Antes de ligar, confira o que ele apagaria:

```sql
select * from public.fotos_orfas();
```

### 4.4 Criar uma migration nova

1. Olhe o maior número em `supabase/migrations/` e use o próximo. Exemplo: se o último é `0045_...`, o seu é `0046_descricao_em_snake_case.sql`.
2. Escreva SQL **idempotente**, que pode rodar duas vezes sem estrago: `create ... if not exists`, `drop ... if exists`, `insert ... on conflict do nothing`.
3. **Nunca edite uma migration que já foi aplicada.** Se algo precisa mudar, crie outra.
4. Função nova precisa de `revoke execute on function ... from public, anon, authenticated;` seguido de `grant` só para quem deve chamar. O Supabase concede `execute` ao papel `authenticated` por padrão.
5. Dentro de policy, use `(select auth.uid())` entre parênteses, que o Postgres avalia uma vez só em vez de uma vez por linha.
6. Aplique com `npx -y supabase@latest db push`, ou, para um arquivo só, com o executor do projeto:

```bash
node scripts/sql.mjs supabase/migrations/0046_descricao_em_snake_case.sql
```

O mesmo executor roda uma consulta avulsa:

```bash
node scripts/sql.mjs -e "select public.hoje_sp()"
```

Se ele responder `ABORTADO`, o `SUPABASE_PROJECT_REF` do `.env` e o `project_id` do `supabase/config.toml` estão diferentes. Corrija antes de tentar de novo.

---

## 5. Configurar a autenticação

No painel, em **Authentication**:

1. **URL Configuration**: `Site URL` com o endereço do seu site (ex.: `https://seu-app.vercel.app`) e o mesmo endereço em **Redirect URLs**, junto com `http://localhost:8080` para desenvolvimento.
2. **Providers → Email**: o app espera cadastro sem confirmação de e-mail (`Confirm email` desligado). Se você ligar, a tela de cadastro já trata o caso.
3. **Recuperação de senha por e-mail** está desligada no código (`RECUPERACAO_POR_EMAIL` em `src/pages/Entrar.tsx`), porque o SMTP embutido do Supabase só entrega para o dono da organização. Configure um SMTP próprio no painel e vire a constante para `true` para ligar o fluxo.
4. Recomendado: ligue **Secure password change** (exigir a senha atual para trocar a senha). A tela já pede a senha atual, mas só o servidor garante.

---

## 6. Publicar as Edge Functions

Cadastre os segredos das funções:

```bash
npx -y supabase@latest secrets set --project-ref "$SUPABASE_PROJECT_REF" VAPID_SUBJECT="$VAPID_SUBJECT" VAPID_PUBLIC_KEY="$VAPID_PUBLIC_KEY" VAPID_PRIVATE_KEY="$VAPID_PRIVATE_KEY" CRIAS_INTERNAL_SECRET="$CRIAS_INTERNAL_SECRET"
```

`SUPABASE_URL` e `SUPABASE_SERVICE_ROLE_KEY` o Supabase injeta sozinho.

Publique as três funções, sempre com `--no-verify-jwt`:

```bash
npx -y supabase@latest functions deploy push-dispatch --no-verify-jwt --project-ref "$SUPABASE_PROJECT_REF"
```

```bash
npx -y supabase@latest functions deploy quick-check-in --no-verify-jwt --project-ref "$SUPABASE_PROJECT_REF"
```

```bash
npx -y supabase@latest functions deploy limpar-fotos --no-verify-jwt --project-ref "$SUPABASE_PROJECT_REF"
```

Desligar o `verify_jwt` **não** deixa as funções abertas: `push-dispatch` e `limpar-fotos` exigem o segredo interno no header `x-internal-secret`, comparado em tempo constante, e `quick-check-in` exige um token de uso único que só existe dentro da notificação cifrada. O `verify_jwt` já se religou sozinho em redeploys de outros projetos, então confira no painel, em **Edge Functions**, depois de cada deploy.

Trocar o valor de um segredo exige atualizar **três lugares**: `.env`, `secrets set` e `privado.config`, e publicar as funções de novo.

---

## 7. Rodar no seu computador

```bash
npm run dev
```

Abra `http://localhost:8080`. Para desligar, `Control` + `C`.

| Comando | Para que serve |
| --- | --- |
| `npm run dev` | Servidor de desenvolvimento na porta 8080 |
| `npm run build` | Monta a versão de produção. É o portão antes de publicar |
| `npm run typecheck` | Confere os tipos do TypeScript |
| `npm run test` | Testes de unidade (Vitest) |
| `npx playwright test` | Testes de ponta a ponta. **Eles criam e apagam usuários de verdade no Supabase do `.env`**, então rode contra um projeto de teste, nunca contra produção com gente usando |

---

## 8. Publicar na Vercel

1. Na Vercel, **Add New → Project** e importe o seu fork do repositório. O framework é detectado como Vite.
2. Em **Settings → Environment Variables**, cadastre só estas três, nos ambientes Production, Preview e Development:

| Variável | Valor |
| --- | --- |
| `VITE_SUPABASE_URL` | O mesmo do `.env` |
| `VITE_SUPABASE_ANON_KEY` | O mesmo do `.env` |
| `VITE_VAPID_PUBLIC_KEY` | O mesmo do `.env` |

3. Faça o deploy. A partir daí, todo push na `main` publica sozinho.

**Nunca** cadastre na Vercel `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ACCESS_TOKEN`, `SUPABASE_DB_PASSWORD`, `VAPID_PRIVATE_KEY` ou `CRIAS_INTERNAL_SECRET`. O front não usa nenhuma delas.

**Variável nova exige deploy novo.** O Vite grava o valor no JavaScript na hora do build, então cadastrar a variável sem publicar de novo não muda nada (o sintoma clássico é tela branca).

O `vercel.json` já cuida dos cabeçalhos de segurança e do cache: o `sw.js` é revalidado a cada visita, senão o celular ficaria preso numa versão velha do app.

Depois de publicar, volte ao passo 5 e confira que o `Site URL` do Supabase é o endereço da Vercel.

---

## 9. Arte: personagens, acessórios, cenários e fundos

A loja vende quatro tipos de peça, todos na tabela `avatar_items`, separados pela coluna `slot`: `personagem`, `acessorio`, `cenario` e `fundo`. Cada peça existe em **três lugares que precisam concordar**:

1. O PNG em `public/sprites/<pasta>/`.
2. A entrada no catálogo `src/lib/catalogo.ts`, que é **gerado por script**, nunca editado à mão.
3. A linha em `avatar_items` no banco, criada por migration. É ela que dá preço e permite comprar.

O teste `testes/catalogo.test.ts` falha se o catálogo e os arquivos divergirem.

### 9.1 Como a arte deve chegar

- Imagem PNG ou JPEG em pixel art, com **fundo liso de uma cor só** (magenta puro `#FF00FF` é o ideal). O script remove o fundo por inundação a partir da borda.
- Personagem **sem chapéu e de mãos vazias**, de corpo inteiro e em pé, senão os acessórios da loja não encaixam.
- Um personagem por imagem.

### 9.2 Adicionar um personagem novo, passo a passo

Exemplo: um personagem chamado "Dragão Azul", custando 800 de ouro, com o id `cri-8`.

**Passo 1. Coloque a arte na pasta versionada.** Salve o arquivo em `arte-origem/personagens/`, com o nome e o preço no próprio nome do arquivo, para ficar fácil de achar depois:

```
arte-origem/personagens/Dragao Azul - 800.png
```

**Passo 2. Registre o personagem no pipeline.** Em `scripts/processar-sprites.mjs`, dentro de `MAPA.cria`, acrescente uma linha. A chave é um pedaço único do nome do arquivo, e o valor é `[id, nome visível]`:

```js
'Dragao Azul': ['cri-8', 'Dragão Azul'],
```

O id segue o padrão `<família>-<número>`. As famílias existentes estão em `FAMILIA`, no `scripts/gerar-catalogo.mjs`, e `cri` é a de arte própria.

**Passo 3. Processe a arte.** O script recorta, descobre o tamanho do pixel, reduz para 128 por 128, mede onde fica a cabeça e a mão (para os acessórios encaixarem) e grava o PNG indexado em `public/sprites/personagens/`:

```bash
node scripts/processar-sprites.mjs
```

Se sobrar um bloco de fundo preso entre as pernas ou entre o braço e o corpo, acrescente o id em `FUNDO_PRESO` no mesmo script e rode de novo.

**Passo 4. Dê o preço e gere o catálogo.** Em `scripts/gerar-catalogo.mjs`, acrescente o id em `PRECO`:

```js
'cri-8':800,
```

Depois gere o catálogo:

```bash
node scripts/gerar-catalogo.mjs
```

**Passo 5. Crie a migration que coloca a peça à venda.** Um arquivo novo com o próximo número, por exemplo `supabase/migrations/0046_personagem_dragao_azul.sql`:

```sql
insert into avatar_items (id, nome, slot, custo_ouro, sprite_path, ativo) values
  ('cri-8', 'Dragão Azul', 'personagem', 800, '/sprites/personagens/cri-8-dragao-azul.png', true)
on conflict (id) do nothing;
```

O `sprite_path` é o caminho do arquivo que o passo 3 gerou, sem o `?v=` que aparece no catálogo. Aplique com `npx -y supabase@latest db push`.

**Passo 6. Confira.**

```bash
npm run test
```

```bash
npm run dev
```

Abra a Loja e veja o personagem, com e sem acessório.

### 9.3 Acessórios, cenários e fundos

Seguem o mesmo caminho, trocando a entrada do `MAPA` (`item`, `cenario` ou `fundo`) e o prefixo do id (`ace-`, `cen-`, `fun-`). A arte bruta desses tipos é lida da pasta apontada pela variável `ARTE_GERADA`, com as subpastas `Personagens`, `Fundos` e `Itens`, e por padrão `arte-origem/gerada/`.

### 9.4 Regras que não podem ser quebradas

- **Nunca renomeie o arquivo nem troque o id de uma peça que já está à venda.** O id é o mesmo de `avatar_items`, e trocá-lo apaga a posse de quem já comprou. Para refazer a arte, substitua o PNG mantendo o nome e rode `node scripts/gerar-catalogo.mjs`: o catálogo carimba `?v=<hash do conteúdo>` na URL, e é isso que faz o celular baixar o desenho novo em vez de usar o cache.
- **Peça antiga não se apaga.** Para tirar da loja, marque `ativo = false` em `avatar_items` numa migration. Quem já tem continua vendo.
- **Ícones e logo** saem de `arte-origem/logo.png` por `node scripts/gerar-icones.mjs`, que lê a cor direto do token `--primary` em `src/index.css`. Troque a arte e rode o script; nunca edite os ícones à mão.

### 9.5 Cores e tema

As cores vivem **só** no bloco de tokens de `src/index.css`. Para mudar a identidade visual, troque os tokens e rode `node scripts/gerar-icones.mjs` para regenerar ícones e `theme-color`.

---

## 10. Segurança

- **RLS é a fonte da verdade.** A tela só esconde; quem nega é o banco. Toda tabela sensível tem policy, e os testes em `e2e/rls.spec.ts` e `e2e/seguranca.spec.ts` tentam ler e escrever dados de outro usuário esperando zero linha.
- **Ouro, vida, XP, ofensiva e prêmios só mudam por RPC `security definer`** no servidor. O navegador diz "marquei o hábito X"; quem calcula a recompensa é o banco.
- **Segredos só no `.env` (ignorado pelo git) e nos secrets das Edge Functions.** Nunca em arquivo versionado, nunca em variável `VITE_`.
- Encontrou uma falha? Abra uma issue sem detalhes de exploração e peça um canal privado.

---

## 11. Se algo der errado

| Problema | O que fazer |
| --- | --- |
| `command not found: npm` | Instale o Node.js (seção 1) |
| Página em branco | Confira `VITE_SUPABASE_URL` e `VITE_SUPABASE_ANON_KEY` no `.env` e reinicie o `npm run dev`. Na Vercel, cadastre as variáveis e publique de novo |
| `ABORTADO` no `scripts/sql.mjs` | O `SUPABASE_PROJECT_REF` do `.env` difere do `project_id` do `supabase/config.toml` |
| Notificação não chega | No iPhone, o app precisa estar **instalado na tela de início**. Confira também se o job `push-dispatch` aponta para o seu ref (seção 4.2) e se os secrets VAPID estão cadastrados |
| O cron diz `succeeded` mas nada acontece | O `pg_net` só enfileira a chamada. O resultado real fica em `select * from net._http_response order by created desc limit 20;` |
| Função responde 401 | O `verify_jwt` religou. Publique de novo com `--no-verify-jwt` |
| Site publicado desatualizado | Publique de novo na Vercel e recarregue segurando `Shift` |

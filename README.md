# Mouse

App pessoal de organização financeira + supermercado. HTML/CSS/JS puro
(módulos ES, sem framework de build), Supabase para os dados, deploy na
Vercel, instalável como PWA a partir do Safari no iPhone.

Feito para uso pessoal de um único usuário — sem multi-tenant, sem checkout,
sem termos de uso.

## Stack

- HTML/CSS/JS puro com módulos ES nativos — sem bundler, sem framework
- [Supabase](https://supabase.com) (Postgres + Auth) para os dados
- Deploy na [Vercel](https://vercel.com) (plano gratuito)
- PWA: instalável no iPhone via "Adicionar à Tela de Início", cache de shell offline

## Estrutura do projeto

```
index.html            Shell da SPA: gate de autenticação + 5 telas + navegação inferior
sw.js                 Service worker (cache do app shell)
public/
  manifest.json        Manifest do PWA
  icons/                Ícones 180x180, 512x512, maskable 512x512
src/
  styles/
    tokens.css          Tokens de design (cores, espaçamento, escala tipográfica)
    base.css             Reset + layout do app shell
    components.css       Card, pill, botão, nav, sheet, formulário, etc.
    gate.css              Tela de PIN + tela de conexão
    screens/               Estilos por tela (today.css, ...)
  js/
    app.js                Bootstrap: service worker, gate de autenticação, fluxo de PIN
    supabaseClient.js       Cliente Supabase (lê src/js/config.js)
    auth.js                  Sessão + hash/verificação de PIN
    router.js                 Roteador por hash entre as 5 telas
    nav.js                     Lógica da navegação inferior
    config.example.js          Modelo para credenciais locais do Supabase
    lib/
      format.js               Formatação de moeda/data (pt-BR / BRL)
      charts.js                 Gráficos de linha/barra/donut em canvas
      dom.js                     Funções utilitárias de DOM
    screens/
      today.js                  Resumo de gastos, gráficos, transações
      cards.js                   Carrossel de cartões, barra de limite, tendência de fatura
      basket.js                  Lista de compras, catálogo rápido, gráfico por corredor
      bills.js                   Resumo do mês, donut de status, contas recorrentes
      profile.js                 Preferências, categorias, fluxo de troca de PIN
scripts/
  build-config.js        Gera src/js/config.js a partir das variáveis de ambiente no deploy
supabase/
  schema.sql              Schema completo, políticas de RLS e dados de exemplo do catálogo
```

## Como funcionam a autenticação e a trava por PIN

Este é um app de usuário único, então não existe fluxo de cadastro:

1. **Supabase Auth** — você cria manualmente um usuário no painel do
   Supabase (e-mail + senha). O app faz login com essas credenciais na
   primeira vez que roda em um aparelho e mantém a sessão salva no
   `localStorage` depois disso (o Supabase renova o token automaticamente,
   então você não precisa digitar e-mail/senha com frequência).
2. **RLS** — toda tabela tem uma coluna `owner_id` com valor padrão
   `auth.uid()`, com políticas de row-level security que só permitem esse
   usuário autenticado ler/escrever suas próprias linhas. É isso que
   realmente protege seus dados, já que a anon key do Supabase fica exposta
   no bundle do navegador por design.
3. **Trava por PIN** — por cima disso, um PIN de 4 dígitos bloqueia a
   interface toda vez que o app é aberto (como uma tela de bloqueio). O PIN
   é hasheado (SHA-256 + salt aleatório, via Web Crypto API) e guardado em
   `app_settings`; o PIN em texto puro nunca sai do dispositivo.

## Configuração

### 1. Criar o projeto no Supabase

1. Crie um projeto em [supabase.com](https://supabase.com).
2. Em **SQL Editor**, rode o conteúdo de [`supabase/schema.sql`](supabase/schema.sql).
   Isso cria todas as tabelas, ativa o RLS e popula `catalog_items` com ~75
   itens comuns de supermercado brasileiro.
3. Em **Authentication → Users**, adicione um usuário (seu e-mail + uma
   senha). Essa é a única conta que o app vai usar.
4. Em **Project Settings → API**, copie a **Project URL** e a **anon public
   key** — você vai precisar das duas a seguir.

### 2. Configurar as variáveis de ambiente

O app não tem build, mas ainda assim precisa da URL/chave do Supabase
injetadas em `src/js/config.js` (esse arquivo está no `.gitignore`, nunca é
commitado) antes de rodar. Um script Node bem simples
(`scripts/build-config.js`, sem dependências) gera esse arquivo a partir de
variáveis de ambiente.

**Desenvolvimento local:**

```bash
# Opção A — rodar o gerador uma vez
SUPABASE_URL="https://SEU-PROJETO.supabase.co" \
SUPABASE_ANON_KEY="SUA-ANON-KEY" \
node scripts/build-config.js

# Opção B — copiar o exemplo e preencher manualmente
cp src/js/config.example.js src/js/config.js
```

Depois sirva a pasta com qualquer servidor estático, por exemplo:

```bash
npx serve .
# ou: python -m http.server 5500
```

Abra em `http://localhost:<porta>` — os recursos de PWA/service worker
precisam de uma origem HTTP de verdade (não funcionam em `file://`).

### 3. Deploy na Vercel

Os dois caminhos usam o [`vercel.json`](vercel.json) incluído no projeto,
que roda `node scripts/build-config.js` como comando de build para gerar o
`src/js/config.js` antes de servir os arquivos estáticos — sem nenhum passo
de build de framework envolvido.

**Opção A — Vercel CLI (sem precisar de repositório Git):**

```bash
npx vercel login                              # login único pelo navegador
npx vercel link --yes                         # cria/vincula o projeto na Vercel
npx vercel env add SUPABASE_URL production
npx vercel env add SUPABASE_ANON_KEY production
npx vercel deploy --prod --yes                # publica
```

Rode `vercel deploy --prod --yes` de novo sempre que quiser publicar
mudanças novas — sem repositório Git conectado não existe deploy automático.

**Opção B — Projeto conectado ao Git (deploy automático a cada push):**

1. Suba este projeto para um repositório Git e importe na Vercel.
2. Em **Project Settings → Environment Variables**, adicione `SUPABASE_URL`
   e `SUPABASE_ANON_KEY`.
3. Publique. Todo push na branch conectada gera um novo deploy
   automaticamente.

O plano gratuito da Vercel é suficiente para uso pessoal nos dois casos.

### 4. Instalar no iPhone

1. Abra a URL publicada no **Safari** do seu iPhone.
2. Toque no ícone de Compartilhar → **Adicionar à Tela de Início**.
3. Abra o Mouse pelo ícone na tela de início — ele abre em modo standalone
   (sem a barra do navegador), respeita as áreas seguras do notch/indicador
   de início, e usa o cache do shell pra carregar rápido mesmo com conexão
   ruim.
4. Primeiro uso: faça login uma vez com o usuário do Supabase que você
   criou, depois defina seu PIN de 4 dígitos. A partir daí, abrir o app só
   pede o PIN.

## Modelo de dados

Veja [`supabase/schema.sql`](supabase/schema.sql) para as definições completas.

| Tabela | Finalidade |
|---|---|
| `app_settings` | Uma linha por usuário: hash/salt do PIN, nome, média diária, orçamento padrão de compras, preferências de notificação |
| `categories` | Categorias de gasto gerenciadas no Perfil |
| `transactions` | Lançamentos de receita/despesa, opcionalmente vinculados a um cartão |
| `cards` | Cartões de crédito: banco, últimos 4 dígitos, limite, fatura, dia de vencimento/fechamento |
| `bills` | Contas fixas/recorrentes com status (pendente/paga/atrasada) |
| `shopping_lists` | Viagens ao mercado (ou listas salvas como modelo) |
| `shopping_items` | Itens dentro de uma lista de compras, agrupados por corredor |
| `catalog_items` | Catálogo de referência compartilhado (~75 itens comuns) usado nos chips de adição rápida da tela Cesta |

Todas as tabelas vinculadas a um usuário têm políticas de RLS restringindo
o acesso a `owner_id = auth.uid()`. `catalog_items` é dado de referência
somente leitura, selecionável por qualquer usuário autenticado.

## Status

As 5 telas estão prontas. Cada uma é um módulo independente em
`src/js/screens/` com uma função `mount(container)` conectada ao roteador
por hash em [`src/js/router.js`](src/js/router.js).

- ✅ Estrutura de pastas, schema SQL, design system, shell do PWA, trava por PIN
- ✅ **Hoje**: saudação, card de "quanto pode gastar hoje", gráfico de linha
  acumulado dos últimos 30 dias, gráfico de barras dos últimos 7 dias, donut
  de categorias do dia, últimas transações
- ✅ **Cartões**: carrossel de cartões (gradiente escuro, número mascarado),
  barra de limite usado, tendência da fatura nos últimos 6 meses, lista de
  transações por cartão, cadastro de novo cartão
- ✅ **Cesta**: cabeçalho com orçamento/total do carrinho, catálogo rápido por
  corredor, adição por texto livre, checklist agrupado por corredor, gráfico
  de gastos por corredor, duplicar lista anterior
- ✅ **Contas**: resumo do mês (previsto x pago), donut de status, listas
  agrupadas por atrasada/a vencer/paga, marcar como paga, contas recorrentes
  avançam automaticamente pro mês seguinte quando pagas
- ✅ **Perfil**: preferências financeiras, gerenciamento de categorias
  (vem com 11 categorias padrão), preferências de notificação, aparência
  (só modo claro por enquanto), fluxo de troca de PIN

Tudo foi testado de ponta a ponta contra um backend Supabase simulado
(autenticação, criação/troca de PIN, CRUD em todas as tabelas), sem erros
no console.

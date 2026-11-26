# Planejamento de Desenvolvimento — The Ultimate SaaS

Roteiro incremental de desenvolvimento do **The Ultimate SaaS — Enterprise-Grade Micro-Actions as a Service**, utilizando exclusivamente a stack especificada no `readme.md`:

- **Frontend & Backend:** Next.js (App Router), React, TypeScript e Tailwind CSS
- **Autenticação & Banco de Dados:** Supabase (Auth e PostgreSQL com RLS)
- **Pagamentos & Webhooks:** Lemon Squeezy (API REST v1 + validação HMAC-SHA256 com `node:crypto`)
- **Infraestrutura & Deploy:** Vercel

---

## Árvore de Arquivos

```text
theultimatesaas/
├── .env.example
├── .env.local                                   # Variáveis locais (ignorado pelo git)
├── next.config.ts
├── package.json
├── postcss.config.mjs
├── readme.md
├── tsconfig.json
├── supabase/
│   └── schema.sql                               # Tabela profiles, políticas RLS e trigger de signup
└── src/
    ├── middleware.ts                            # Refresh de sessão Supabase e proteção de /dashboard
    ├── types/
    │   └── saas.ts                              # Tipos de plano, perfil e contratos das APIs
    ├── lib/
    │   ├── supabase/
    │   │   ├── client.ts                        # Cliente Supabase (Client Components)
    │   │   ├── server.ts                        # Cliente Supabase (Server Components / Route Handlers)
    │   │   └── admin.ts                         # Cliente Supabase Service Role (Webhooks e updates privilegiados)
    │   ├── plans.ts                             # Catálogo Starter/Pro/Enterprise, preços (0% OFF anual) e RBAC
    │   └── lemonsqueezy.ts                      # Criação de checkout REST e verificação HMAC-SHA256
    ├── components/
    │   ├── PaywallModal.tsx                     # Modal satírico de upgrade para recursos bloqueados
    │   ├── DarkModeToggle.tsx                   # Switch de Dark Mode as a Service (DMaaS)
    │   ├── Navbar.tsx                           # Cabeçalho global, navegação, badge do plano e DMaaS
    │   ├── PricingTable.tsx                     # Tabela de planos + toggle anual "0% OFF" + checkout
    │   ├── QuantumCalculator.tsx                # Calculadora Quântica (adição com delay + blur no free)
    │   ├── FridayChecker.tsx                    # Verificador de Sexta-feira ("Maybe" vs resposta real)
    │   └── PasswordGenerator.tsx                # Gerador de Senhas ("123456" vs senha aleatória segura)
    └── app/
        ├── globals.css                          # Tailwind CSS + variante dark baseada na classe `.dark`
        ├── layout.tsx                           # RootLayout com SSR de tema (DMaaS) e Navbar global
        ├── page.tsx                             # Landing Page satírica Enterprise
        ├── login/
        │   └── page.tsx                         # Login e Cadastro via Supabase Auth
        ├── pricing/
        │   └── page.tsx                         # Enterprise Pricing Page
        ├── dashboard/
        │   └── page.tsx                         # Painel autenticado com status da assinatura e micro-ações
        ├── auth/
        │   └── callback/
        │       └── route.ts                     # GET /auth/callback (troca code por sessão)
        └── api/
            ├── auth/
            │   └── signout/
            │       └── route.ts                 # POST /api/auth/signout (encerra sessão)
            ├── tools/
            │   ├── calculator/
            │   │   └── route.ts                 # POST /api/tools/calculator
            │   ├── friday/
            │   │   └── route.ts                 # GET /api/tools/friday
            │   └── password/
            │       └── route.ts                 # POST /api/tools/password
            ├── preferences/
            │   └── theme/
            │       └── route.ts                 # PATCH /api/preferences/theme (DMaaS)
            ├── checkout/
            │   └── route.ts                     # POST /api/checkout (gera checkout Lemon Squeezy)
            └── webhooks/
                └── lemonsqueezy/
                    └── route.ts                 # POST /api/webhooks/lemonsqueezy (sincroniza assinatura)
```

---

## Fase 1 — Base (Setup, Dados, Tipos e Autenticação)

- [ ] Etapa 1 - [SETUP] Inicialização do projeto Next.js, Tailwind CSS e variáveis de ambiente
  - **Arquivos afetados:** *(Exceção inicial permitida)* Inicializar projeto Next.js (App Router, TypeScript, Tailwind CSS, pasta `src/`) gerando `package.json`, `tsconfig.json`, `next.config.ts`, `postcss.config.mjs`, `src/app/globals.css`, `src/app/layout.tsx` e `src/app/page.tsx`; instalar `@supabase/supabase-js` e `@supabase/ssr`; configurar `@custom-variant dark (&:where(.dark, .dark *));` em `src/app/globals.css`; criar `.env.example` com todas as chaves do projeto e criar `.env.local` inicialmente vazio/placeholder (garantindo que `.gitignore` ignore `.env*` exceto `.env.example`).
  - **Contratos / Reutilizáveis:** Variáveis de ambiente padronizadas em `.env.example`:
    - `NEXT_PUBLIC_APP_URL`
    - `NEXT_PUBLIC_SUPABASE_URL`
    - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
    - `SUPABASE_SERVICE_ROLE_KEY`
    - `LEMONSQUEEZY_API_KEY`
    - `LEMONSQUEEZY_STORE_ID`
    - `LEMONSQUEEZY_WEBHOOK_SECRET`
    - `LEMONSQUEEZY_VARIANT_PRO_MONTHLY`
    - `LEMONSQUEEZY_VARIANT_PRO_ANNUAL`
    - `LEMONSQUEEZY_VARIANT_ENTERPRISE_MONTHLY`
    - `LEMONSQUEEZY_VARIANT_ENTERPRISE_ANNUAL`
  - **Validação e Testes:**
    - **Objetivo:** Confirmar que a base Next.js + React + TypeScript + Tailwind CSS + pacotes Supabase inicializa e compila sem erros.
    - **Passo a passo:** Execute `npm run dev` no terminal e acesse `http://localhost:3000` no navegador; em seguida execute `npx tsc --noEmit`.
    - **Critério de Sucesso:** O servidor local responde HTTP 200 em `http://localhost:3000` sem erros no terminal e `npx tsc --noEmit` encerra com código `0`.

- [ ] Etapa 2 - [DADOS] Schema PostgreSQL, políticas RLS e trigger de perfis no Supabase
  - **Arquivos afetados:**
    - Criar `supabase/schema.sql` definindo a tabela `public.profiles` (`id`, `email`, `plan`, `subscription_status`, `lemon_squeezy_customer_id`, `lemon_squeezy_subscription_id`, `dark_mode_enabled`, `created_at`, `updated_at`), constraints de valores permitidos, trigger `on_auth_user_created` para criar automaticamente um perfil `starter` no signup, trigger `set_updated_at`, e políticas **Row Level Security (RLS)** permitindo apenas `SELECT` pelo próprio usuário (`auth.uid() = id`) e bloqueando `INSERT`/`UPDATE`/`DELETE` diretos pelo cliente (impedindo auto-upgrade via DevTools).
  - **Ação Manual Externa:**
    1. Acesse o **Painel do Supabase** (`https://supabase.com/dashboard`) → Clique em **New project**.
    2. Escolha a organização, dê o nome `the-ultimate-saas`, defina uma senha forte de banco, selecione a região mais próxima e clique em **Create new project**.
    3. Aguarde o provisionamento → No menu lateral esquerdo, clique em **SQL Editor** → **+ New query**.
    4. Cole todo o conteúdo de `supabase/schema.sql` e clique em **Run**.
    5. No menu lateral esquerdo, clique em **Project Settings** (ícone de engrenagem) → **API** (ou **Data API / API Keys**).
    6. Copie e cole os valores no arquivo `.env.local`:
       - **Project URL** → `NEXT_PUBLIC_SUPABASE_URL`
       - **anon / public key** → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
       - **service_role / secret key** → `SUPABASE_SERVICE_ROLE_KEY`
  - **Validação e Testes:**
    - **Objetivo:** Garantir que a tabela `public.profiles`, as restrições de plano, o trigger em `auth.users` e o RLS foram aplicados corretamente no PostgreSQL.
    - **Passo a passo:** No **Supabase → Table Editor**, abra a tabela `profiles` e confirme as colunas; em **Authentication → Policies**, verifique que `profiles` está com **RLS Enabled** e possui apenas a política de `SELECT`. No **SQL Editor**, rode `select count(*) from public.profiles;`.
    - **Critério de Sucesso:** A consulta retorna `0` sem erros, a tabela lista todas as 9 colunas esperadas e o RLS está ativo bloqueando escritas públicas.

- [ ] Etapa 3 - [DADOS] Tipos de domínio, catálogo de planos e regras de RBAC
  - **Arquivos afetados:**
    - Criar `src/types/saas.ts` com os tipos `PlanId` (`'starter' | 'pro' | 'enterprise'`), `BillingInterval` (`'monthly' | 'annual'`), `SubscriptionStatus`, `UserProfile`, `FeatureKey`, `CalculatorResponse`, `FridayResponse`, `PasswordResponse` e `ThemePreferenceResponse`.
    - Criar `src/lib/plans.ts` com o catálogo `PLANS` (*Starter* grátis, *Pro* \$29/mês ou \$348/ano — **0% OFF**, *Enterprise* \$999/mês ou \$11.988/ano — **0% OFF** e lista de benefícios satíricos), mais os helpers puros `isPaidPlan(plan, status)`, `hasFeatureAccess(profile, feature)`, `getVariantIdForPlan(planId, interval)` e `getPlanByVariantId(variantId)`.
  - **Contratos / Reutilizáveis:**
    - `isPaidPlan(plan: PlanId, status: SubscriptionStatus): boolean`
    - `hasFeatureAccess(profile: Pick<UserProfile, 'plan' | 'subscription_status'> | null, feature: FeatureKey): boolean`
    - `getVariantIdForPlan(planId: Exclude<PlanId, 'starter'>, interval: BillingInterval): string | null`
    - `getPlanByVariantId(variantId: string): PlanId`
    - Constante `PLANS` consumida pela Pricing Page e Dashboard.
  - **Validação e Testes:**
    - **Objetivo:** Validar a tipagem estrita e a lógica de controle de acesso por plano (RBAC) e cálculo anual de 0% de desconto.
    - **Passo a passo:** Execute `npx tsc --noEmit` e valide as regras com um teste rápido via Node/TypeScript (ex: verificar que `PLANS.pro.annualPrice === PLANS.pro.monthlyPrice * 12`, `isPaidPlan('starter', 'free') === false`, `isPaidPlan('pro', 'active') === true` e `isPaidPlan('pro', 'expired') === false`).
    - **Critério de Sucesso:** Zero erros de compilação TypeScript e todas as asserções de RBAC e preço anual (0% OFF) retornam `true`.

- [ ] Etapa 4 - [CORE] Clientes Supabase (Browser, Server e Admin Service Role)
  - **Arquivos afetados:**
    - Criar `src/lib/supabase/client.ts` exportando `createBrowserSupabaseClient()` com `@supabase/ssr`.
    - Criar `src/lib/supabase/server.ts` exportando `createServerSupabaseClient()` (integrado a `cookies()` do `next/headers`) e `getAuthenticatedProfile()` (retorna `{ user, profile }` usando `supabase.auth.getUser()`).
    - Criar `src/lib/supabase/admin.ts` exportando `createAdminSupabaseClient()` com `SUPABASE_SERVICE_ROLE_KEY` e `persistSession: false` (exclusivo server-side).
  - **Contratos / Reutilizáveis:**
    - `createBrowserSupabaseClient()`
    - `createServerSupabaseClient(): Promise<SupabaseClient>`
    - `getAuthenticatedProfile(): Promise<{ user: User | null; profile: UserProfile | null }>`
    - `createAdminSupabaseClient(): SupabaseClient`
  - **Validação e Testes:**
    - **Objetivo:** Garantir que os 3 clientes Supabase compilam corretamente sob App Router (Next.js) e respeitam a separação entre cliente público e `service_role`.
    - **Passo a passo:** Execute `npx tsc --noEmit` e confirme que `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` e `SUPABASE_SERVICE_ROLE_KEY` estão preenchidas em `.env.local`.
    - **Critério de Sucesso:** `npx tsc --noEmit` conclui com código `0` e sem avisos de tipagem envolvendo `cookies()` ou `@supabase/ssr`.

- [ ] Etapa 5 - [AUTH] Middleware de sessão, callback de autenticação e endpoint de logout
  - **Arquivos afetados:**
    - Criar `src/middleware.ts` para renovar cookies de sessão do Supabase em cada requisição, redirecionar usuários não autenticados de `/dashboard` para `/login?next=/dashboard` e redirecionar usuários já logados de `/login` para `/dashboard`.
    - Criar `src/app/auth/callback/route.ts` implementando `GET /auth/callback` para trocar `code` via `exchangeCodeForSession` e redirecionar com segurança para `/dashboard`.
    - Criar `src/app/api/auth/signout/route.ts` implementando `POST /api/auth/signout` para encerrar a sessão no servidor e retornar `{ ok: true }`.
  - **Contratos / Reutilizáveis:**
    - `GET /auth/callback?code=...&next=/dashboard` → HTTP `307` Redirect
    - `POST /api/auth/signout` → HTTP `200` `{ "ok": true }`
  - **Ação Manual Externa:**
    1. Acesse o **Painel do Supabase** → **Authentication** → **URL Configuration**.
    2. Em **Site URL**, informe `http://localhost:3000`.
    3. Em **Redirect URLs**, adicione `http://localhost:3000/auth/callback` e clique em **Save**.
    4. *(Recomendado para testes locais rápidos)* Acesse **Authentication** → **Providers** → **Email**, desative temporariamente **Confirm email** e clique em **Save** (assim novas contas criadas no `/login` entram direto sem aguardar e-mail).
  - **Validação e Testes:**
    - **Objetivo:** Garantir que o middleware protege `/dashboard` contra acessos anônimos e que a rota de logout responde corretamente.
    - **Passo a passo:** Com `npm run dev` rodando, execute no PowerShell:
      ```powershell
      curl.exe -i http://localhost:3000/dashboard
      curl.exe -i -X POST http://localhost:3000/api/auth/signout
      ```
    - **Critério de Sucesso:** `GET /dashboard` retorna `HTTP/1.1 307 Temporary Redirect` com header `Location: /login?next=%2Fdashboard`; `POST /api/auth/signout` retorna `HTTP/1.1 200 OK` com `{"ok":true}`.

---

## Fase 2 — Core e Integrações (APIs das Micro-Ações & Lemon Squeezy)

- [ ] Etapa 6 - [API] Endpoint da Quantum Calculator (adição com delay artificial e paywall server-side)
  - **Arquivos afetados:**
    - Criar `src/app/api/tools/calculator/route.ts` implementando `POST /api/tools/calculator`: exige usuário autenticado (`401`), valida que `a` e `b` são números finitos (`400`), aplica delay artificial de `1200ms` simulando computação quântica e aplica RBAC no servidor: usuários `starter` recebem `locked: true` e `result: null` (nunca vazando a soma no payload), enquanto usuários pagos recebem `locked: false` e `result: a + b`.
  - **Contratos / Reutilizáveis:**
    - `POST /api/tools/calculator`
    - Request JSON: `{ "a": 2, "b": 2 }`
    - Response JSON (Free): `{ "locked": true, "result": null, "displayValue": "8888", "computeTimeMs": 1200, "message": "Quantum collapse requires a Pro or Enterprise subscription." }`
    - Response JSON (Paid): `{ "locked": false, "result": 4, "displayValue": "4", "computeTimeMs": 1200, "message": "Verified by 128 virtual qubits." }`
  - **Validação e Testes:**
    - **Objetivo:** Confirmar que a rota rejeita anônimos (`401`), valida payloads inválidos e aplica o bloqueio no servidor sem vazar o resultado real.
    - **Passo a passo:** Com `npm run dev` ativo, execute:
      ```powershell
      curl.exe -i -X POST http://localhost:3000/api/tools/calculator -H "Content-Type: application/json" -d "{\"a\":2,\"b\":2}"
      ```
    - **Critério de Sucesso:** Retorna `HTTP/1.1 401 Unauthorized` com JSON de erro (a validação autenticada de Free vs Paid será repetida ponta a ponta após a tela de Login na Etapa 13/16).

- [ ] Etapa 7 - [API] Endpoints do Friday Checker e Insecure Password Generator
  - **Arquivos afetados:**
    - Criar `src/app/api/tools/friday/route.ts` implementando `GET /api/tools/friday`: exige autenticação (`401`); no plano gratuito retorna `locked: true`, `answer: "Maybe"`, `isFriday: null`; nos planos pagos calcula o dia real da semana e retorna a resposta definitiva (`"Yes, it is Friday."` ou `"No, it is not Friday."`).
    - Criar `src/app/api/tools/password/route.ts` implementando `POST /api/tools/password`: exige autenticação (`401`), valida `length` opcional (8–64, padrão 16); no plano gratuito retorna sempre `password: "123456"` com `locked: true`; nos planos pagos gera senha aleatória criptograficamente segura via `crypto.randomInt` (`node:crypto`).
  - **Contratos / Reutilizáveis:**
    - `GET /api/tools/friday` → `FridayResponse` (`{ locked, answer, isFriday, checkedAt }`)
    - `POST /api/tools/password` com `{ "length"?: number }` → `PasswordResponse` (`{ locked, password, entropyBits, message }`)
  - **Validação e Testes:**
    - **Objetivo:** Garantir que ambos os endpoints estão ativos, tipados e exigem sessão autenticada antes de expor qualquer resposta.
    - **Passo a passo:** Execute:
      ```powershell
      curl.exe -i http://localhost:3000/api/tools/friday
      curl.exe -i -X POST http://localhost:3000/api/tools/password -H "Content-Type: application/json" -d "{\"length\":16}"
      ```
    - **Critério de Sucesso:** Ambas as chamadas retornam `HTTP/1.1 401 Unauthorized` com JSON `{ "error": "Unauthorized" }` e `npx tsc --noEmit` passa sem erros.

- [ ] Etapa 8 - [API] Endpoint de Dark Mode as a Service (DMaaS)
  - **Arquivos afetados:**
    - Criar `src/app/api/preferences/theme/route.ts` implementando `PATCH /api/preferences/theme`: valida sessão (`401`), valida `{ darkMode: boolean }` (`400`), verifica `isPaidPlan(profile.plan, profile.subscription_status)`: retorna `403 Forbidden` para plano gratuito; para assinantes pagos, persiste `dark_mode_enabled` no Supabase usando `createAdminSupabaseClient()` filtrado por `user.id`.
  - **Contratos / Reutilizáveis:**
    - `PATCH /api/preferences/theme`
    - Request JSON: `{ "darkMode": true }`
    - Response JSON (Free): HTTP `403` `{ "locked": true, "darkModeEnabled": false, "error": "Dark Mode as a Service (DMaaS) requires Pro or Enterprise." }`
    - Response JSON (Paid): HTTP `200` `{ "locked": false, "darkModeEnabled": true }`
  - **Validação e Testes:**
    - **Objetivo:** Assegurar que a alteração do tema é protegida no backend (usuário não pode forçar Dark Mode sem assinatura ativa).
    - **Passo a passo:** Execute:
      ```powershell
      curl.exe -i -X PATCH http://localhost:3000/api/preferences/theme -H "Content-Type: application/json" -d "{\"darkMode\":true}"
      ```
    - **Critério de Sucesso:** Retorna `HTTP/1.1 401 Unauthorized` sem sessão (e `403` quando chamado por usuário autenticado no plano `starter`, validado na Etapa 16).

- [ ] Etapa 9 - [PAGAMENTOS] Integração Lemon Squeezy e API de Checkout
  - **Arquivos afetados:**
    - Criar `src/lib/lemonsqueezy.ts` com `createCheckoutUrl({ variantId, userId, userEmail, redirectUrl })` (chamada `fetch` para `POST https://api.lemonsqueezy.com/v1/checkouts` enviando `checkout_data.custom.user_id`) e `verifyLemonSqueezySignature(rawBody, signature)` usando `crypto.createHmac('sha256', ...)` + `crypto.timingSafeEqual`.
    - Criar `src/app/api/checkout/route.ts` implementando `POST /api/checkout`: exige sessão (`401`), valida `{ planId: 'pro' | 'enterprise', interval: 'monthly' | 'annual' }`, resolve o Variant ID via `getVariantIdForPlan`, cria o checkout na Lemon Squeezy com `redirect_url = ${NEXT_PUBLIC_APP_URL}/dashboard?checkout=success` e retorna `{ checkoutUrl }`.
  - **Contratos / Reutilizáveis:**
    - `createCheckoutUrl(params): Promise<string>`
    - `verifyLemonSqueezySignature(rawBody: string, signature: string): boolean`
    - `POST /api/checkout` → Request `{ planId, interval }` → Response `{ checkoutUrl: string }`
  - **Ação Manual Externa:**
    1. Crie/acesse sua conta na **Lemon Squeezy** (`https://app.lemonsqueezy.com`) e ative o interruptor **Test Mode** no canto inferior esquerdo do menu.
    2. Acesse **Settings → Stores** e copie o **Store ID** → cole em `LEMONSQUEEZY_STORE_ID` no `.env.local`.
    3. Acesse **Store → Products → + New Product** e crie dois produtos de assinatura recorrente:
       - **The Ultimate SaaS — Pro**: crie 2 variantes: *Monthly* (\$29/mês) e *Annual* (\$348/ano — 0% OFF).
       - **The Ultimate SaaS — Enterprise**: crie 2 variantes: *Monthly* (\$999/mês) e *Annual* (\$11.988/ano — 0% OFF).
    4. Em cada variante (ícone `...` → **Copy ID**), copie os IDs e cole no `.env.local`:
       - `LEMONSQUEEZY_VARIANT_PRO_MONTHLY`
       - `LEMONSQUEEZY_VARIANT_PRO_ANNUAL`
       - `LEMONSQUEEZY_VARIANT_ENTERPRISE_MONTHLY`
       - `LEMONSQUEEZY_VARIANT_ENTERPRISE_ANNUAL`
    5. Acesse **Settings → API** → clique em **+ (Create API key)**, nomeie `the-ultimate-saas-dev`, copie o token gerado e cole em `LEMONSQUEEZY_API_KEY` no `.env.local`.
  - **Validação e Testes:**
    - **Objetivo:** Verificar que a rota de checkout protege contra chamadas anônimas e que a configuração da Lemon Squeezy compila e resolve corretamente os Variant IDs.
    - **Passo a passo:** Com `npm run dev` rodando, execute:
      ```powershell
      curl.exe -i -X POST http://localhost:3000/api/checkout -H "Content-Type: application/json" -d "{\"planId\":\"pro\",\"interval\":\"monthly\"}"
      ```
    - **Critério de Sucesso:** Retorna `HTTP/1.1 401 Unauthorized` sem sessão e `npx tsc --noEmit` conclui com código `0` (o checkout autenticado real será testado na Etapa 14).

- [ ] Etapa 10 - [PAGAMENTOS] Webhook da Lemon Squeezy com verificação HMAC e sincronização no Supabase
  - **Arquivos afetados:**
    - Criar `src/app/api/webhooks/lemonsqueezy/route.ts` implementando `POST /api/webhooks/lemonsqueezy`: lê o corpo bruto (`await request.text()`), verifica o header `x-signature` com `verifyLemonSqueezySignature` (retorna `401` se ausente/inválido), extrai `meta.event_name`, `meta.custom_data.user_id`, `data.id`, `data.attributes.customer_id`, `data.attributes.variant_id` e `data.attributes.status`, e atualiza `public.profiles` via `createAdminSupabaseClient()`:
      - `subscription_created` / `subscription_updated`: grava `plan` (mapeado pelo `variant_id`), `subscription_status`, `lemon_squeezy_customer_id` e `lemon_squeezy_subscription_id`; se status for `expired`, faz downgrade para `starter` e reseta `dark_mode_enabled = false`.
      - `subscription_cancelled`: atualiza `subscription_status` para `cancelled`.
      - `subscription_expired`: faz downgrade para `plan = 'starter'`, `subscription_status = 'expired'` e `dark_mode_enabled = false`.
  - **Contratos / Reutilizáveis:**
    - `POST /api/webhooks/lemonsqueezy` (Header obrigatório: `X-Signature: <hex-hmac-sha256>`).
  - **Ação Manual Externa:**
    1. Defina um segredo forte (ex: 32 caracteres aleatórios) e salve em `LEMONSQUEEZY_WEBHOOK_SECRET` no `.env.local`.
    2. *(Para testar webhooks reais da Lemon Squeezy ainda em localhost)* Inicie um túnel HTTPS apontando para a porta 3000 (ex: `npx cloudflared tunnel --url http://localhost:3000` ou `ngrok http 3000`).
    3. Na **Lemon Squeezy (Test Mode)** → **Settings → Webhooks** → clique em **+ (New Webhook)**:
       - **Callback URL:** `https://<sua-url-tunel>/api/webhooks/lemonsqueezy`
       - **Signing secret:** cole o mesmo valor de `LEMONSQUEEZY_WEBHOOK_SECRET`
       - **Events:** marque `subscription_created`, `subscription_updated`, `subscription_cancelled` e `subscription_expired`.
       - Clique em **Save Webhook**.
  - **Validação e Testes:**
    - **Objetivo:** Comprovar que o webhook rejeita requisições falsificadas (HMAC inválido) com `401` e aceita requisições com assinatura HMAC-SHA256 válida com `200`.
    - **Passo a passo:** Com `npm run dev` rodando, envie uma requisição com assinatura falsa:
      ```powershell
      curl.exe -i -X POST http://localhost:3000/api/webhooks/lemonsqueezy -H "Content-Type: application/json" -H "X-Signature: assinatura_invalida" -d "{\"meta\":{\"event_name\":\"subscription_created\"}}"
      ```
      Depois gere uma assinatura válida localmente com Node usando seu `LEMONSQUEEZY_WEBHOOK_SECRET` e envie para o endpoint.
    - **Critério de Sucesso:** A primeira requisição retorna `HTTP/1.1 401 Unauthorized`; a requisição assinada corretamente retorna `HTTP/1.1 200 OK` com `{"received":true}` e atualiza o usuário correspondente no Supabase.

---

## Fase 3 — Interface e Experiência do Usuário

- [ ] Etapa 11 - [UI] Modal de Paywall satírico, Toggle DMaaS e Navbar global
  - **Arquivos afetados:**
    - Criar `src/components/PaywallModal.tsx`: modal acessível (fechar com `Esc` ou clique fora) exibindo título corporativo satírico, nome do recurso bloqueado e botão para `/pricing`.
    - Criar `src/components/DarkModeToggle.tsx`: botão/switch interativo de tema Light/Dark; se `isPaid` for `false`, abre `PaywallModal`; se `isPaid` for `true`, chama `PATCH /api/preferences/theme`, alterna a classe `dark` em `document.documentElement` e dispara `router.refresh()`.
    - Criar `src/components/Navbar.tsx`: barra de navegação fixa com marca **The Ultimate SaaS**, links (`Home`, `Pricing`, `Dashboard`), `<DarkModeToggle />`, badge do plano atual (`Starter`, `Pro`, `Enterprise`) e botão `Sign In` ou `Sign Out` (`POST /api/auth/signout`).
  - **Contratos / Reutilizáveis:**
    - `<PaywallModal isOpen={boolean} onClose={() => void} featureName={string} />`
    - `<DarkModeToggle isPaid={boolean} initialDarkMode={boolean} />`
    - `<Navbar userEmail={string | null} plan={PlanId | null} isPaid={boolean} initialDarkMode={boolean} />`
  - **Validação e Testes:**
    - **Objetivo:** Garantir que os componentes globais compilam sem erros de tipos e respeitam a diretiva `'use client'`.
    - **Passo a passo:** Execute `npx tsc --noEmit`.
    - **Critério de Sucesso:** Compilação TypeScript limpa com código `0`.

- [ ] Etapa 12 - [UI] RootLayout com persistência SSR de tema (DMaaS) e Landing Page satírica
  - **Arquivos afetados:**
    - Atualizar `src/app/layout.tsx`: Server Component que consulta `getAuthenticatedProfile()`, define `<html lang="en" className={shouldUseDark ? 'dark' : ''}>` (evitando flash de tema em usuários pagos) e renderiza `<Navbar />` + `{children}` com classes Tailwind adaptadas para light e dark mode.
    - Atualizar `src/app/page.tsx`: Landing Page parodiando startups enterprise com Hero ("Enterprise-Grade Micro-Actions as a Service"), métricas absurdas (SLA 99.99999%, Zero Value Delivered), prévia das 4 micro-ações com etiquetas de paywall e CTAs para `/login` e `/pricing`.
  - **Validação e Testes:**
    - **Objetivo:** Confirmar que a Landing Page renderiza corretamente com a Navbar global e que clicar no botão de Dark Mode sem assinatura abre o `PaywallModal`.
    - **Passo a passo:** Rode `npm run dev`, acesse `http://localhost:3000`, inspecione o layout em desktop e mobile e clique no botão de Dark Mode na Navbar.
    - **Critério de Sucesso:** Página carrega sem erros no Console do navegador; clicar em Dark Mode abre o modal satírico oferecendo upgrade para `/pricing`, sem alterar o tema da página.

- [ ] Etapa 13 - [UI] Tela de Login e Cadastro com Supabase Auth
  - **Arquivos afetados:**
    - Criar `src/app/login/page.tsx`: interface limpa com abas/alternância **Sign In** e **Create Account**, usando `createBrowserSupabaseClient()` (`signInWithPassword` e `signUp`), feedback de erro/sucesso, estado de loading e redirecionamento para o parâmetro `?next=` (padrão `/dashboard`) seguido de `router.refresh()`.
  - **Validação e Testes:**
    - **Objetivo:** Validar o fluxo completo de cadastro, criação automática do perfil `starter` pelo trigger SQL, login e logout.
    - **Passo a passo:** Acesse `http://localhost:3000/login`, alterne para **Create Account**, crie um usuário de teste (ex: `ceo@startup.dev` / senha com 6+ caracteres), verifique o redirecionamento para `/dashboard` (ou mensagem de confirmação se o e-mail estiver ativo), clique em **Sign Out** na Navbar e faça login novamente.
    - **Critério de Sucesso:** O usuário aparece em **Supabase → Authentication → Users** e uma linha correspondente surge automaticamente em **Table Editor → profiles** com `plan = 'starter'` e `subscription_status = 'free'`.

- [ ] Etapa 14 - [UI] Enterprise Pricing Page com toggle anual "0% OFF" e Checkout Lemon Squeezy
  - **Arquivos afetados:**
    - Criar `src/components/PricingTable.tsx`: renderiza os 3 cards (*Starter*, *Pro* destacado como "Most Aggressive", *Enterprise*), toggle Mensal/Anual com selo **"Save 0%!"** (exibindo exatamente `preço mensal × 12`), lista de perks satíricos, estado de "Current Plan" para o plano ativo e botão de upgrade que dispara `POST /api/checkout` e redireciona via `window.location.href = checkoutUrl` (ou redireciona para `/login?next=/pricing` se deslogado).
    - Criar `src/app/pricing/page.tsx`: Server Component que lê `getAuthenticatedProfile()`, renderiza cabeçalho satírico, `<PricingTable />` e FAQ corporativo sobre faturamento.
  - **Validação e Testes:**
    - **Objetivo:** Confirmar o funcionamento do toggle anual de 0% de desconto e a geração real de sessão de checkout na Lemon Squeezy.
    - **Passo a passo:** Logado no app, acesse `http://localhost:3000/pricing`, alterne o toggle **Monthly / Annual** verificando os valores (`$29/mo` ↔ `$348/yr`; `$999/mo` ↔ `$11,988/yr`) e clique em **Upgrade to Pro**.
    - **Critério de Sucesso:** O navegador é redirecionado para a URL de checkout da Lemon Squeezy em Test Mode com o produto Pro selecionado e o e-mail do usuário pré-preenchido.

- [ ] Etapa 15 - [UI] Componentes interativos: Quantum Calculator e Friday Checker
  - **Arquivos afetados:**
    - Criar `src/components/QuantumCalculator.tsx`: inputs numéricos `A` e `B`, botão "Run Quantum Addition" chamando `POST /api/tools/calculator`, indicador de processamento quântico durante o delay; quando `locked === true`, renderiza resultado desfocado (`blur-md select-none`) com botão sobreposto que abre `<PaywallModal />`; quando `locked === false`, exibe a soma real e o tempo computacional.
    - Criar `src/components/FridayChecker.tsx`: botão "Check if Today is Friday" chamando `GET /api/tools/friday`; exibe `"Maybe"` com badge `PRO` e botão para abrir `<PaywallModal />` quando `locked === true`, ou a resposta definitiva destacada quando `locked === false`.
  - **Contratos / Reutilizáveis:**
    - `<QuantumCalculator isPaid={boolean} />`
    - `<FridayChecker isPaid={boolean} />`
  - **Validação e Testes:**
    - **Objetivo:** Garantir compilação limpa e contratos compatíveis com as APIs das Etapas 6 e 7.
    - **Passo a passo:** Execute `npx tsc --noEmit`.
    - **Critério de Sucesso:** Zero erros de compilação TypeScript.

- [ ] Etapa 16 - [UI] Insecure Password Generator e Dashboard completo de Micro-Ações
  - **Arquivos afetados:**
    - Criar `src/components/PasswordGenerator.tsx`: controle de comprimento da senha, botão "Generate Password" chamando `POST /api/tools/password` e botão de copiar; exibe `123456` com alerta de segurança satírico e botão de upgrade (`PaywallModal`) para plano free, ou senha aleatória segura com bits de entropia para assinantes pagos.
    - Criar `src/app/dashboard/page.tsx`: Server Component autenticado que consulta `getAuthenticatedProfile()`, exibe banner de sucesso quando `?checkout=success` está presente na URL, resumo da assinatura atual (plano + status), card dedicado de **Dark Mode as a Service (DMaaS)** reutilizando `<DarkModeToggle />`, e o grid com `<QuantumCalculator />`, `<FridayChecker />` e `<PasswordGenerator />`.
  - **Validação e Testes:**
    - **Objetivo:** Validar ponta a ponta todas as 4 micro-ações e a segurança de paywall tanto no plano **Starter (Free)** quanto após o upgrade via **Lemon Squeezy Webhook**.
    - **Passo a passo:**
      1. Faça login com um usuário `starter` e acesse `http://localhost:3000/dashboard`.
      2. Teste **Quantum Calculator** (`7 + 5`): verifique o delay, o resultado borrado e confirme na aba **Network** do DevTools que o JSON da resposta trouxe `"result": null`.
      3. Teste **Friday Checker**: confirme resposta `"Maybe"`.
      4. Teste **Password Generator**: confirme resposta `"123456"`.
      5. Clique no toggle **DMaaS**: confirme abertura do `PaywallModal` e resposta HTTP `403` se tentar chamar `/api/preferences/theme` diretamente.
      6. Complete um pagamento de teste em `/pricing` (cartão de teste da Lemon Squeezy `4242 4242 4242 4242`) com o webhook ativo (ou dispare um webhook HMAC válido) e recarregue `/dashboard`.
      7. Repita os testes 2, 3, 4 e 5 como assinante **Pro** e recarregue a página (`F5`) com o Dark Mode ativado.
    - **Critério de Sucesso:** No plano Free todas as 4 funcionalidades ficam bloqueadas tanto visualmente quanto na API; após o webhook, o badge muda para **Pro**, a calculadora exibe `12` nítido, o Friday Checker retorna a resposta real, o gerador cria senhas aleatórias fortes e o Dark Mode permanece ativo após recarregar a página.

---

## Fase 4 — Infraestrutura e Entrega (Deploy na Vercel)

- [ ] Etapa 17 - [DEPLOY] Build de produção, deploy na Vercel e configuração final de produção
  - **Arquivos afetados:**
    - Revisar `next.config.ts` para garantir compatibilidade de build de produção na Vercel.
    - Atualizar `readme.md` adicionando a seção enxuta `## Continuidade e Execução` com instruções de instalação, variáveis de ambiente e URL/status do projeto.
  - **Ação Manual Externa:**
    1. Faça commit e push do repositório para o **GitHub**.
    2. Acesse a **Vercel** (`https://vercel.com/new`) → clique em **Import** no repositório `theultimatesaas`.
    3. Em **Environment Variables**, cadastre todas as chaves listadas no `.env.example`, ajustando `NEXT_PUBLIC_APP_URL` para a URL gerada pela Vercel (ex: `https://theultimatesaas.vercel.app`) e clique em **Deploy**.
    4. Acesse o **Painel do Supabase** → **Authentication → URL Configuration**: atualize **Site URL** para `https://theultimatesaas.vercel.app` e adicione `https://theultimatesaas.vercel.app/auth/callback` em **Redirect URLs** → **Save**.
    5. Acesse a **Lemon Squeezy** → **Settings → Webhooks**: cadastre o webhook apontando para `https://theultimatesaas.vercel.app/api/webhooks/lemonsqueezy` com o mesmo `LEMONSQUEEZY_WEBHOOK_SECRET` configurado na Vercel e os 4 eventos de assinatura marcados.
  - **Validação e Testes:**
    - **Objetivo:** Confirmar que o build de produção compila sem erros e que o fluxo SaaS completo (Auth + Banco + Checkout + Webhook + Paywalls) funciona no ambiente publicado da Vercel.
    - **Passo a passo:** Execute `npm run build` localmente; após o deploy na Vercel, acesse a URL de produção, crie uma conta, teste as 4 micro-ações no plano Free, conclua um checkout de teste na Lemon Squeezy e verifique no painel de **Webhooks** da Lemon Squeezy o entrega HTTP `200` e o desbloqueio instantâneo no `/dashboard`.
    - **Critério de Sucesso:** `npm run build` finaliza sem erros/warnings bloqueantes; o deploy na Vercel fica **Ready**; o webhook da Lemon Squeezy registra status `200 OK` e o usuário desbloqueia todos os recursos pagos em produção.

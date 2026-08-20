# ERP OPME — Fase 0: Fundação multiempresa

Núcleo de segurança do sistema. **Nenhum módulo de negócio deve ser criado antes desta fase estar aplicada e validada**, porque toda tabela de negócio vai depender de `company_id` + `app.has_permission()`.

Validado em PostgreSQL 16 com 27 asserções (isolamento de tenant, RBAC, tentativa de escalação de privilégio e integridade da trilha de auditoria).

---

## Como aplicar

**Via Supabase CLI** (recomendado — mantém histórico):

```bash
supabase migration new fase0   # e cole o conteúdo, ou copie os arquivos
supabase db push
```

**Via SQL Editor do dashboard:** execute os 6 arquivos **na ordem numérica**, um por vez. A ordem não é opcional — `0003` cria funções que `0005` referencia nas policies.

| Arquivo | Conteúdo |
|---|---|
| `0001_extensions_and_utils.sql` | schema `app`, `updated_at`, validadores de CPF/CNPJ |
| `0002_companies_and_profiles.sql` | `companies`, `profiles`, `user_companies`, trigger de signup |
| `0003_permissions_and_security.sql` | RBAC + **funções de segurança** |
| `0004_audit.sql` | `audit_logs` + trigger genérica |
| `0005_rls_policies.sql` | policies + grants |
| `0006_seed_and_bootstrap.sql` | catálogo de permissões + `bootstrap_company()` |

### Criando a primeira empresa

`bootstrap_company` é `service_role` apenas — não é chamável pelo frontend por design. Após criar o usuário no Auth:

```sql
select public.bootstrap_company(
    '<uuid-do-usuario>',
    '11222333000181',
    'Alfa Distribuidora OPME LTDA'
);
```

Isso cria a empresa, os 4 papéis padrão, vincula o usuário como Administrador e define a empresa ativa.

---

## Decisões que valem entender antes de mexer

**Helpers vivem no schema `app`, não em `public`.** O PostgREST expõe automaticamente tudo que está em `public` como endpoint. Função de segurança em `public` vira API pública.

**Toda função usada em policy é `SECURITY DEFINER` + `STABLE` + `search_path` fixo.** Sem `SECURITY DEFINER`, a policy da tabela X consulta a tabela X e o Postgres estoura recursão infinita — é o erro nº 1 em projeto Supabase, e a "solução" que costuma aparecer é desligar a RLS. Sem `STABLE`, a função reexecuta por linha e a consulta cai de performance.

**Empresa ativa fica em `profiles.active_company_id`, não no JWT.** Trocar de empresa é um UPDATE, sem relogin. A guarda está em trigger (`0002`), então mesmo um UPDATE direto via PostgREST não consegue apontar para empresa sem vínculo — o teste cobre exatamente esse caminho.

**Permissão crítica está na policy, não no React.** Esconder botão não protege nada: qualquer um com a anon key chama o PostgREST direto. `app.can('stock.adjust', company_id)` é a única barreira que conta.

**`audit_logs` é insert-only.** Não existe policy de UPDATE nem DELETE — nem para admin. A gravação acontece por trigger `SECURITY DEFINER`, que passa por cima da RLS. Se auditoria dependesse do frontend lembrar de gravar, um dia ele esquece exatamente na operação que você precisava rastrear.

**`user_roles.company_id` é desnormalizado de propósito.** Permite policy sem JOIN. A FK composta contra `roles(id, company_id)` impede divergência — o banco não deixa apontar para papel de outra empresa.

### Dois bugs que os testes pegaram

Ficam registrados porque são o tipo de coisa que passa despercebida em revisão:

1. O papel **Consulta** recebia toda permissão `%.view`, o que incluía `audit.view` — usuário somente-leitura enxergaria custo anterior, preço anterior e limite de crédito de toda a empresa pela trilha. Hoje `audit.view` é concessão explícita.
2. A trigger de auditoria resolvia tenant lendo a coluna `company_id`. `companies` não tem essa coluna (o tenant é o `id`), então o log nascia com `company_id = NULL` e ficava **invisível para todos**. `0004` resolve tenant explicitamente para `companies`, `profiles` e `role_permissions`.

---

## Rodando os testes

Requer PostgreSQL local. `test/00_stub_supabase.sql` recria o mínimo do ambiente Supabase (schema `auth`, `auth.uid()`, roles).

```bash
./scripts/test-db.sh
```

O script instala o PostgreSQL se preciso, recria a base do zero, aplica as migrations na ordem e roda a suite.

As asserções rodam sob `set role authenticated`. Rodar como superusuário fura a RLS e faz o teste passar mentindo.

---

## Pendências desta fase

- **Edge Function de convite de usuário** — criar usuário no Auth exige service role. Fica na Fase 1.
- **`app.routine`** — a trigger lê `current_setting('app.routine')` para gravar contexto no log. Só passa a ser preenchido quando as RPCs de negócio existirem.
- Retenção/particionamento de `audit_logs` — só vira problema com volume; `ix_audit_company_date` já suporta a consulta por período.

## Próximo

Fase 1 — cadastros: `products` (com série/lote/ANVISA/TUSS), `customers`, `suppliers`, `product_categories`, `product_manufacturers`, `payment_terms`, `supplier_products` (de-para para XML de NF-e) e `document_sequences`.

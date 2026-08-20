# ERP OPME — contexto do projeto

ERP para distribuidoras de OPME (Órteses, Próteses e Materiais Especiais).
Multiempresa, React + TypeScript + Tailwind no frontend, Supabase/PostgreSQL no backend.

**Leia `README.md` antes de escrever qualquer SQL.** Ele documenta as decisões de
arquitetura da Fase 0 e dois bugs reais que já foram corrigidos — não os reintroduza.

---

## Estado atual

| Fase | Escopo | Status |
|---|---|---|
| 0 | Multiempresa, RBAC, RLS, auditoria | **Concluída e testada** |
| 1 | Cadastros (produtos, clientes, fornecedores) | Próxima |
| 2 | Estoque (saldo por trigger, reserva, consignação) | — |
| 3 | Compras + importação de XML de NF-e | — |
| 4 | Vendas + financeiro | — |

Migrations em `supabase/migrations/`, numeradas e sequenciais. Nunca edite uma
migration já aplicada — crie a próxima.

---

## Invariantes — não negociáveis

Estas regras já custaram bug em produção neste projeto. Violar qualquer uma
significa que a tarefa não está pronta.

1. **Toda tabela de negócio tem `company_id`** com FK para `companies`, RLS
   habilitada e policies usando `app.can('<permissao>', company_id)`.
2. **Função usada em policy é `SECURITY DEFINER` + `STABLE` + `set search_path`.**
   Sem isso: recursão infinita na RLS. A saída *nunca* é desabilitar RLS.
3. **Helper de segurança vive no schema `app`, nunca em `public`.** O PostgREST
   expõe `public` automaticamente como endpoint HTTP.
4. **Permissão crítica é verificada na policy**, não no componente React.
   Esconder botão não protege — a anon key chama o PostgREST direto.
5. **Saldo de estoque nunca é escrito diretamente.** É consequência de
   `stock_movements`, aplicado por trigger com `FOR UPDATE` na linha de saldo.
   Saldo negativo é bloqueado no banco, salvo permissão `stock.negative`.
6. **Documento financeiro movimentado não é deletado.** Cancelamento/estorno
   gera novo registro; o original permanece.
7. **Auditoria é trigger**, nunca chamada da aplicação. `audit_logs` é
   insert-only: sem policy de UPDATE ou DELETE, nem para admin.
8. **Valores monetários são `numeric(15,4)`.** Nunca `float`/`real`/`double`.
   Quantidades: `numeric(15,4)`. Datas com hora: sempre `timestamptz`.
9. **Produto com `controla_lote` exige lote em toda entrada e saída.** Idem
   validade e número de série. Validação no banco, não só no form.

## Convenções

- Nomes de tabela e coluna em `snake_case`, inglês; **conteúdo de domínio e UI em
  português** (`razao_social`, `nome_fantasia` permanecem em português por serem
  termos fiscais).
- Permissões: `<modulo>.<acao>` — ver catálogo em `0006_seed_and_bootstrap.sql`.
  Permissão nova entra no catálogo por migration, não por INSERT manual.
- Status e tipos: `varchar` + `CHECK`, não `enum` (ALTER de enum em produção dói).
- CPF/CNPJ armazenados **sem máscara**, validados por `app.is_valid_cnpj()` /
  `app.is_valid_cpf()` em CHECK constraint. Formatação é responsabilidade da UI.
- Toda tabela: `id uuid default gen_random_uuid()`, `created_at`, `updated_at`
  com trigger `app.set_updated_at()`.
- Soft delete via `is_active boolean`, não `deleted_at`.

---

## Loop de validação — obrigatório

**Migration não validada contra um Postgres real não está pronta.** O ambiente
já foi usado assim; reproduza:

```bash
# Se o apt falhar com 404 em security.ubuntu.com:
sed -i 's|security\.ubuntu\.com|archive.ubuntu.com|g' \
    /etc/apt/sources.list.d/*.sources /etc/apt/sources.list 2>/dev/null
apt-get update && apt-get install -y postgresql postgresql-contrib
pg_ctlcluster 16 main start

su postgres -c "psql -q -c 'drop database if exists opme;' -c 'create database opme;'"
su postgres -c "psql -v ON_ERROR_STOP=1 -d opme -f test/00_stub_supabase.sql"
for f in supabase/migrations/*.sql; do
  su postgres -c "psql -v ON_ERROR_STOP=1 -q -d opme -f $PWD/$f" || break
done
for f in test/*_test_*.sql; do
  su postgres -c "psql -v ON_ERROR_STOP=1 -q -d opme -f $PWD/$f"
done
```

`test/00_stub_supabase.sql` recria o mínimo do ambiente Supabase (schema `auth`,
`auth.uid()`, roles `anon`/`authenticated`/`service_role`).

### Regras de teste

- **Asserções rodam sob `set local role authenticated`.** Rodar como superusuário
  fura a RLS e o teste passa mentindo.
- Todo módulo novo ganha um `test/NN_test_<modulo>.sql` que prova, no mínimo:
  o caminho feliz; que **outro tenant não enxerga o dado**; que a ação crítica é
  **bloqueada** para quem não tem a permissão; e que a regra de negócio central
  do módulo falha quando violada.
- Teste que só verifica que algo funciona é metade do teste. O valor está em
  provar que a coisa errada é **recusada**.
- Suite inteira verde antes de considerar a fase concluída. Se um teste falhar,
  investigue a causa — não relaxe a asserção.

---

## Definition of done (por fase)

- [ ] Migrations aplicam limpo em base zerada, na ordem numérica
- [ ] RLS habilitada em toda tabela nova, com policy por operação
- [ ] GRANTs explícitos para `authenticated`; nada para `anon`
- [ ] Trigger de auditoria anexada às tabelas críticas do módulo
- [ ] Suite de testes verde, incluindo os casos de bloqueio
- [ ] README atualizado com as decisões não óbvias

## Como trabalhar aqui

- Uma fase por vez. Não adiante módulo de fase futura "já que estamos aqui".
- Antes de implementar, apresente o desenho das tabelas e espere confirmação.
- Prefira menos telas com regra de negócio correta a muitas telas rasas.
- Se uma decisão tiver trade-off relevante, exponha o trade-off em vez de
  escolher silenciosamente.
- Achou inconsistência entre este arquivo e o código? Aponte, não contorne.

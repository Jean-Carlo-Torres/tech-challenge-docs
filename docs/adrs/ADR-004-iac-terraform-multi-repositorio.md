# ADR 004: Infraestrutura como código em Terraform, com um repositório e um state por componente e ambiente

* **Status**: Aceito
* **Data**: 2026-10-01
* **Autores**: Time do Tech Challenge (13SOAT)
* **Escopo**: Infra K8s, Database, Function Serverless

---

## Contexto

O desafio pede **4 repositórios separados**, cada um com CI/CD próprio, e a
infraestrutura provisionada por **Terraform**. Três deles têm infraestrutura (cluster,
banco, função) e precisam:

* ser aplicados e destruídos de forma independente;
* ter **homologação** e **produção** isoladas;
* nunca permitir que um `plan` de staging rode contra o state de produção;
* funcionar com as restrições do AWS Academy (sem criar roles, sessão de ~4 h).

---

## Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| Um monorepo de IaC com todos os componentes | Um único `apply` | Contraria o requisito de 4 repositórios; blast radius grande |
| Terraform workspaces (CLI) no mesmo diretório | Menos código | Ambiente invisível no código; fácil aplicar no workspace errado |
| HCP Terraform (Terraform Cloud) | UI, runs remotos | Agente precisaria de credenciais do lab renovadas a cada 4 h; testado e abandonado |
| **Repositório por componente + state S3 por repositório/ambiente** (escolhida) | Independência total; ambiente explícito no path da key; lock nativo do S3 | Ordem de deploy entre repositórios precisa ser documentada |

---

## Decisão

* **Um repositório por componente**, cada um com `ci.yml` (fmt, validate, test, scan) e
  `deploy.yml` (plan → gate → apply).
* **State em um bucket S3 compartilhado** (criado uma vez pelo `bootstrap/` da Infra K8s),
  com versionamento e `use_lockfile` (sem DynamoDB). Keys literais:
  `k8s/<env>/terraform.tfstate`, `db/<env>/terraform.tfstate`,
  `lambda/<env>/terraform.tfstate`.
* **Ambiente derivado da branch**: `develop` → `staging`, `main` → `prod`.
* **Produção protegida**: environment `production` com *required reviewers*; o job de
  apply re-planeja e só aplica se o plano for igual ao aprovado; guardas que abortam se o
  plan destruir cluster, VPC, ECR, segredo JWT ou a instância RDS.
* **Destroy só por `workflow_dispatch`** (`destroy.yml`), digitando o nome do ambiente.
* **Ordem de deploy**: Infra K8s → Database → Function → App.

---

## Consequências

### Positivas

* [✔️] Cada repositório evolui no próprio ritmo, com PR e CI próprios.
* [✔️] Impossível aplicar staging no state de prod (key e tfvars derivam do mesmo valor).
* [✔️] Bucket fora do state: nenhum `terraform destroy` apaga o state de todos.

### Negativas / Riscos

* [⚠️] Dependência de ordem entre repositórios (tags e SSM precisam existir antes).
* [⚠️] Credenciais do lab precisam ser regravadas nos secrets a cada sessão.

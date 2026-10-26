# RFC 001: Escolha do provedor de nuvem — AWS (us-east-1)

* **Status**: Aceita
* **Data**: 2026-09-20
* **Atualizado em**: 2026-10-24
* **Autores**: Time do Tech Challenge (13SOAT)
* **Repositórios afetados**: todos
* **Decisões resultantes**: [ADR-002](../adrs/ADR-002-hpa-e-escalabilidade-no-eks.md), [ADR-004](../adrs/ADR-004-iac-terraform-multi-repositorio.md)

## Problema

A Fase 3 exige, em nuvem: um **cluster Kubernetes com escalabilidade**, uma **Function
Serverless**, um **banco de dados gerenciado**, um **API Gateway**, tudo provisionado por
**Terraform** e com deploy automatizado. Precisamos de um único provedor que ofereça os
quatro serviços gerenciados e que caiba no orçamento acadêmico do time.

## Critérios

| Critério | Peso | Por quê |
|---|---|---|
| Serviços gerenciados para K8s, serverless e Postgres | Alto | Requisito direto do desafio |
| Custo / crédito disponível | Alto | Projeto acadêmico; o time usa o **AWS Academy Learner Lab** |
| Maturidade do provider Terraform e módulos da comunidade | Alto | Todo o IaC é Terraform |
| Conhecimento prévio do time e material das aulas | Médio | Reduz risco de execução |
| Integração com GitHub Actions (OIDC) | Médio | Deploy sem chave estática |

## Alternativas consideradas

### 1. AWS (proposta)

* **Serviços**: EKS, Lambda (container image), RDS PostgreSQL, ECR, SSM Parameter Store,
  CloudWatch, NLB.
* **A favor**: crédito do AWS Academy disponível para todo o time; módulos
  `terraform-aws-modules` maduros (VPC, EKS, RDS); conteúdo das aulas da pós em AWS;
  OIDC nativo com GitHub Actions; continuidade com as Fases 1 e 2.
* **Contra**: o Learner Lab impõe restrições — não permite criar roles IAM (usa-se a
  `LabRole`), a sessão expira a cada ~4 h e uma SCP bloqueia algumas chamadas
  (`iam:GetRole`, `s3:GetBucketObjectLockConfiguration`). Exigiu adaptações documentadas
  (módulo EKS vendored, bucket de state criado via CLI, script de reciclagem de nós).

### 2. Google Cloud (GKE + Cloud Functions + Cloud SQL)

* **A favor**: GKE Autopilot simplifica a operação do cluster; Cloud SQL é bem integrado.
* **Contra**: sem crédito acadêmico disponível para o time; menor familiaridade; material
  da disciplina não cobre.

### 3. Azure (AKS + Azure Functions + Azure Database for PostgreSQL)

* **A favor**: crédito estudantil possível; AKS com control plane gratuito no tier free.
* **Contra**: menor domínio do time; provider Terraform com mais *breaking changes* entre
  versões principais; nenhuma continuidade com as fases anteriores.

### Comparativo

| Critério | AWS | GCP | Azure |
|---|---|---|---|
| Serviços exigidos | ✅ | ✅ | ✅ |
| Crédito disponível ao time | ✅ AWS Academy | ❌ | ⚠️ |
| Módulos Terraform | ✅ muito maduros | ✅ | ⚠️ |
| Conhecimento do time / aulas | ✅ | ❌ | ⚠️ |
| Restrições do ambiente | ⚠️ Learner Lab | — | — |

## Proposta

Adotar **AWS, região `us-east-1`** (região do Learner Lab e com todos os serviços
disponíveis), com o seguinte mapeamento:

| Requisito | Serviço AWS |
|---|---|
| Cluster Kubernetes com escalabilidade | Amazon EKS + managed node group + HPA |
| Function Serverless | AWS Lambda (container image no ECR) |
| Banco gerenciado | Amazon RDS for PostgreSQL 15 |
| API Gateway | Kong no EKS atrás de um NLB (ver [ADR-003](../adrs/ADR-003-kong-como-api-gateway.md)) |
| Segredos e contratos | SSM Parameter Store |
| Monitoramento | CloudWatch (Container Insights, Logs, alarmes) |
| State do Terraform | S3 com `use_lockfile` |

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Sessão do lab expira (~4 h) e invalida credenciais do CI | Script `atualizar-credenciais-lab.sh` regrava os secrets `AWS_LAB_*` nos repositórios; pipelines aceitam OIDC (`AWS_DEPLOY_ROLE`) fora do Academy |
| Nós do EKS não se registram após o lab religar | Script `reciclar-nos.sh` no repositório de Infra K8s |
| Custo com o ambiente de pé (~US$ 7/dia com tudo) | Staging desligável (`STAGING_ENABLED`), `destroy.yml` sob demanda, teardown documentado |
| Sem IRSA (não é possível criar roles) | Kong invoca a Lambda com a `LabRole` do nó; registrado como dívida consciente |

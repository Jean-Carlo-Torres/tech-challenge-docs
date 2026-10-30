# Diagrama de Componentes

Visão completa da solução da Fase 3: **nuvem (AWS)**, **APIs** (Kong, aplicação e Function
Serverless), **banco de dados** (RDS PostgreSQL) e **monitoramento** (New Relic, CloudWatch,
Container Insights e métricas Prometheus do Kong). Cada caixa indica o repositório que a provisiona.

## Visão geral

```mermaid
flowchart TB
    classDef user fill:#fff2cc,stroke:#d6b656,color:#000
    classDef gw fill:#e1d5e7,stroke:#9673a6,color:#000
    classDef app fill:#dae8fc,stroke:#6c8ebf,color:#000
    classDef fn fill:#d5e8d4,stroke:#82b366,color:#000
    classDef data fill:#f8cecc,stroke:#b85450,color:#000
    classDef mon fill:#ffe6cc,stroke:#d79b00,color:#000
    classDef ci fill:#f5f5f5,stroke:#666,color:#000

    CLI["Cliente da oficina<br/>(autentica por CPF)"]:::user
    STAFF["Funcionário<br/>(atendente / mecânico, e-mail e senha)"]:::user

    subgraph GH["GitHub (4 repositórios + docs)"]
        direction LR
        GHA["GitHub Actions<br/>CI + deploy por branch<br/>develop → staging · main → prod"]:::ci
    end

    subgraph AWS["AWS · us-east-1"]
        direction TB

        S3[("S3<br/>state do Terraform<br/>uma key por repositório/ambiente")]:::data
        ECR[("ECR<br/>imagens da API e da Lambda<br/>tag imutável")]:::data

        subgraph VPC["VPC techchallenge-&lt;env&gt; · 3 AZs · repo Infra K8s"]
            direction TB
            NLB["NLB internet-facing<br/>preserva IP do cliente"]:::gw

            subgraph EKS["EKS techchallenge-&lt;env&gt; · managed node group AL2023"]
                direction TB
                subgraph NSKONG["namespace kong"]
                    KIC["Kong Ingress Controller<br/>DB-less"]:::gw
                    KONG["Kong Gateway · HPA 2..5<br/>jwt · aws-lambda · rate-limiting<br/>correlation-id · prometheus"]:::gw
                end
                subgraph NSAPP["namespace techchallenge · repo App"]
                    API["techchallenge-api (NestJS)<br/>HPA 3..5 por CPU · PDB · agente APM<br/>/api/* · Swagger /api/docs"]:::app
                end
                MS["metrics-server<br/>(alimenta o HPA)"]:::mon
                CWA["amazon-cloudwatch-observability<br/>(Container Insights)"]:::mon
                NRI["nri-bundle · namespace newrelic<br/>métricas de nós e pods, eventos,<br/>logs (Fluent Bit) e Prometheus agent"]:::mon
            end

            LAMBDA["Lambda techchallenge-auth-&lt;env&gt;<br/>Node 22 · imagem ECR<br/>valida CPF e emite JWT · repo Function"]:::fn
            RDS[("RDS PostgreSQL 15<br/>db.t4g.micro · cifrado · TLS obrigatório<br/>subnets privadas · repo Database")]:::data
        end

        SSM["SSM Parameter Store<br/>/techchallenge/&lt;env&gt;/auth/jwt-secret<br/>/techchallenge/&lt;env&gt;/db/*<br/>/techchallenge/&lt;env&gt;/eks/node-sg-id<br/>/techchallenge/&lt;env&gt;/newrelic/license-key"]:::data
        CW["CloudWatch<br/>Logs (API, Lambda, Postgres)<br/>4 alarmes + dashboard do RDS<br/>métricas nativas da Lambda"]:::mon
    end

    NR["New Relic (SaaS)<br/>APM e traces · Logs · Dashboards<br/>Alertas · Synthetics"]:::mon
    EXT["Datadog / New Relic<br/>(forwarder opcional)"]:::mon

    CLI -->|"HTTP"| NLB
    STAFF -->|"HTTP"| NLB
    NLB --> KONG
    KIC -->|"config a partir de Ingress,<br/>KongPlugin, KongConsumer"| KONG
    KONG -->|"POST /auth<br/>plugin aws-lambda"| LAMBDA
    KONG -->|"/api/* com JWT válido<br/>(público: login, health, docs)"| API
    LAMBDA -->|"SELECT customers<br/>TLS 5432"| RDS
    API -->|"Prisma · TLS 5432"| RDS
    LAMBDA -.->|"lê segredo e contrato do banco"| SSM
    API -.->|"deploy monta o Secret<br/>a partir do SSM"| SSM

    GHA -->|"docker push"| ECR
    GHA -->|"terraform apply"| AWS
    GHA -->|"kubectl apply -k"| EKS
    GHA -.->|"state + lock"| S3
    ECR -.->|"pull"| API
    ECR -.->|"pull"| LAMBDA

    MS -.-> API
    MS -.-> KONG
    CWA -->|"métricas e logs de pods/nós"| CW
    LAMBDA -->|"logs JSON + correlationId"| CW
    RDS -->|"métricas + logs postgresql"| CW
    CW -.->|"subscription filter"| EXT
    NRI -->|"métricas, eventos e logs"| NR
    API -->|"agente APM: traces e logs<br/>com correlationId e trace.id"| NR
    NR -.->|"Synthetics: ping<br/>/api/health a cada 5 min"| NLB
```

## Componentes e responsabilidades

| Componente | Tecnologia | Responsabilidade | Repositório |
|---|---|---|---|
| Rede | VPC 3 AZ, NAT único, subnets públicas/privadas com tags de descoberta | Isolar os ambientes `staging` e `prod`; subnets privadas para EKS, Lambda e RDS | [Infra K8s](https://github.com/thiagoalmeida88/TechChallenge---Infraestrutura-Kubernetes-com-Terraform) |
| Cluster | Amazon EKS 1.35, managed node group (2..4 nós) | Executar a aplicação e o gateway | Infra K8s |
| API Gateway | Kong Ingress Controller 3.9 (chart `kong/ingress` 0.24.0), DB-less, atrás de um NLB | Roteamento, validação de JWT, rate limit, correlation-id, métricas por rota e invocação da Lambda | Infra K8s (instalação) + App (rotas e plugins) |
| Aplicação | NestJS + Prisma em container, Kustomize | Domínio da oficina: clientes, veículos, catálogo, peças, OS, orçamentos, métricas | [App](https://github.com/DevKaue/tech-challenge-app-) |
| Function Serverless | AWS Lambda (container image, Node 22) | Autenticar o cliente por CPF e emitir JWT | [Function](https://github.com/DevKaue/TechChallange---Function-Serverless) |
| Banco de dados | Amazon RDS PostgreSQL 15 | Persistência transacional de todos os contextos | [Database](https://github.com/DevKaue/tech-challenge-database-infra) |
| Configuração e segredos | SSM Parameter Store (`String` / `SecureString`) | Contrato entre os repositórios: segredo JWT, endpoint/credenciais do banco, SG dos nós | Infra K8s + Database |
| Registro de imagens | Amazon ECR | Imagens versionadas da API e da Lambda | Infra K8s (API) + Function (Lambda) |
| State do IaC | S3 com versionamento e `use_lockfile` | State Terraform isolado por repositório e ambiente | Infra K8s (`bootstrap/`) |
| Monitoramento | New Relic (`nri-bundle` no cluster, agente APM na API, dashboards, alertas e Synthetics), CloudWatch (Container Insights, Logs, alarmes, dashboard), plugin `prometheus` do Kong | Saúde do cluster, traces e logs da API com correlation-id, volume e tempo por status das OS, uptime, saúde do banco | Infra K8s (`nri-bundle`) + App (agente e logs) + Todos |
| CI/CD | GitHub Actions | Testes, build de imagem, `terraform plan/apply` com gate de aprovação em produção, `kubectl apply` com rollback | Todos |

## Observabilidade ponta a ponta

| Camada | O que é coletado | Onde ver |
|---|---|---|
| Gateway (Kong) | Latência, status HTTP e banda por rota/serviço (plugin `prometheus`); `X-Correlation-ID` gerado em toda requisição | Endpoint `/metrics` do Kong (scrape por Prometheus/Datadog/New Relic) |
| Aplicação (EKS) | CPU/memória por pod e nó, reinícios, logs `stdout` dos containers | CloudWatch Container Insights |
| Escalabilidade | Uso de CPU vs. request para o HPA da API e do Kong | `kubectl get hpa -A` (metrics-server) |
| Function Serverless | Uma linha JSON por evento (`auth.success`, `auth.denied`, `db.error`...) com `correlationId`; `Invocations`, `Duration`, `Throttles` | CloudWatch Logs `/aws/lambda/techchallenge-auth-<env>` + métricas da Lambda |
| Aplicação (New Relic) | Agente APM Node (`-r newrelic`) e logs JSON com `correlationId`, `trace.id` e eventos `http.request`, `http.error`, `service_order.created` e `service_order.status_changed` | New Relic: APM e Distributed tracing, Logs e dashboards por NRQL |
| Cluster (New Relic) | `nri-bundle` (infraestrutura, kube-state-metrics, eventos do Kubernetes e logs) | New Relic: Kubernetes explorer, CPU e memória por pod, réplicas do HPA |
| Uptime (New Relic) | Monitor Synthetics `techchallenge-health` (ping em `/api/health`) e alerta de falha | New Relic: Synthetic monitoring |
| Banco (RDS) | Alarmes de CPU, memória livre, conexões e latência de escrita; queries > 1 s; `pg_stat_statements` | CloudWatch alarmes + dashboard `techchallenge-<env>-database` |

O `X-Correlation-ID` gerado pelo Kong viaja até a Lambda e a aplicação, então um mesmo ID
liga o log do gateway, o log da função e o log da API. No New Relic, o mesmo
`correlationId` filtra os logs da API, e o `trace.id` do log abre o trace da requisição.

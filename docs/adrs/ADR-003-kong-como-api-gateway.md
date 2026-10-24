# ADR 003: Kong Ingress Controller (DB-less) como API Gateway

* **Status**: Aceito
* **Data**: 2026-10-06
* **Autores**: Time do Tech Challenge (13SOAT)
* **Escopo**: Infra K8s (instalação via Helm/Terraform), App (rotas e plugins), Function (invocada pelo Kong)
* **Documento de origem**: [ADR-005 da App](https://github.com/DevKaue/tech-challenge-app-/blob/main/docs/ADRs/ADR-005-kong-como-api-gateway.md)

---

## Contexto

A Fase 3 exige um **API Gateway** para controle e roteamento, com rotas sensíveis
protegidas pela autenticação via CPF. Requisitos:

* validar o JWT antes de o tráfego chegar à aplicação;
* rotear o login por CPF para a Function Serverless;
* rate limiting nas rotas de login (enumeração de CPF / força bruta);
* correlation-id e métricas de latência/status por rota;
* configuração versionada no Git e aplicada pelo CI/CD.

---

## Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| AWS API Gateway (HTTP API) + Lambda Authorizer | Gerenciado, sem pods | VPC Link + LB privado; um Authorizer invocado por requisição (cold start e custo); configuração fora do cluster e em outro ciclo de deploy |
| **Kong Ingress Controller DB-less no EKS** (escolhido) | Config como CRDs no Git (Ingress, KongPlugin, KongConsumer); plugins prontos (`jwt`, `aws-lambda`, `rate-limiting`, `correlation-id`, `prometheus`); escala com HPA no mesmo cluster; visto em aula | Mais pods para operar; Kong Manager só leitura em DB-less; evento enviado à Lambda no formato REST v1.0 |
| Kong com banco (Postgres) | Kong Manager editável | Mudanças pela UI geram drift em relação ao Git; gateway passa a depender de banco e migrations |

---

## Decisão

Adotar **Kong Ingress Controller, chart `kong/ingress` 0.24.0 (Kong 3.9), modo DB-less**,
exposto por um **NLB** internet-facing.

* **Instalação** pelo `terraform/kong.tf` do repositório de Infra K8s, no namespace `kong`.
* **Rotas e plugins** no overlay `k8s/overlays/aws/` do repositório da App, aplicados pelo
  mesmo `kubectl apply -k` do deploy da aplicação.

| Rota | Plugins | Destino |
|---|---|---|
| `POST /auth` | `aws-lambda`, `rate-limiting` (10/min/IP), `correlation-id`, `prometheus` | Lambda `techchallenge-auth-<env>` |
| `POST /api/auth/login` | `rate-limiting`, `correlation-id`, `prometheus` | API (login de funcionário) |
| `GET /api/health`, `GET /api/docs` | `correlation-id`, `prometheus` | API (público) |
| `POST /api/service-orders/estimates/:id/external-status` | `correlation-id`, `prometheus` | API (webhook com HMAC) |
| `/api/*` (demais) | `jwt` (consumer pelo `iss`, verifica `exp`), `correlation-id`, `prometheus` | API (protegido) |

---

## Consequências

### Positivas

* [✔️] Token ausente, inválido ou expirado é recusado no gateway (401) e não consome a
  aplicação.
* [✔️] O que está no cluster é o que está no PR (DB-less).
* [✔️] Um único NLB substitui o ALB da aplicação.
* [✔️] Métricas por rota (prometheus) e `X-Correlation-ID` em toda resposta.

### Negativas / Riscos

* [⚠️] Sem TLS até haver domínio e certificado ACM.
* [⚠️] No AWS Academy não há IRSA: o Kong invoca a Lambda com a `LabRole` do nó (IMDSv2).
* [⚠️] Segredo HS256 único compartilhado por Kong, Lambda e App ([RFC-003](../rfcs/RFC-003-estrategia-de-autenticacao.md)).

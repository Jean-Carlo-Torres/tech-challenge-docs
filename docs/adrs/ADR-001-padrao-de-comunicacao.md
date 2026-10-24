# ADR 001: Comunicação síncrona HTTP/REST com entrada única pelo API Gateway

* **Status**: Aceito
* **Data**: 2026-10-06
* **Autores**: Time do Tech Challenge (13SOAT)
* **Escopo**: todos os repositórios

---

## Contexto

A solução da Fase 3 tem três componentes que conversam em tempo de requisição — Kong,
Function Serverless de autenticação e aplicação NestJS — e quatro repositórios que precisam
trocar configuração em tempo de deploy (endpoint do banco, segredo JWT, rede, SG).

Precisávamos decidir:

1. o padrão de comunicação **em runtime** entre cliente, gateway, função e aplicação;
2. o padrão de integração **entre repositórios** (como um descobre o que o outro criou).

Requisitos que pesaram: autenticação devolve token na mesma chamada (é inerentemente
síncrona); a abertura de OS precisa de resposta imediata com o ID criado; os repositórios
devem ter ciclo de vida independente; nenhum segredo pode circular por state do Terraform
ou por variável de pipeline copiada à mão.

---

## Opções consideradas

### Runtime

| Opção | Prós | Contras |
|---|---|---|
| **HTTP/REST síncrono via Kong** (escolhida) | Simples, contrato OpenAPI, cliente recebe a resposta na hora, observável por rota | Acoplamento temporal: se a Lambda ou o banco cair, a chamada falha (tratado com `503` explícito) |
| Mensageria assíncrona (SQS/SNS/EventBridge) | Desacoplamento temporal, retentativa nativa | Login e abertura de OS precisam de resposta síncrona; adicionaria fila, consumidores e consistência eventual sem caso de uso |
| gRPC entre serviços | Contrato forte, performance | Só há um serviço de negócio (monólito modular); clientes HTTP/browser exigiriam proxy |

### Integração entre repositórios

| Opção | Prós | Contras |
|---|---|---|
| `terraform_remote_state` | Nativo do Terraform | Cada repositório precisa ler o state (com segredos) do outro; acopla versões |
| **SSM Parameter Store com paths previsíveis + descoberta de rede por tags** (escolhida) | Ninguém lê o state de ninguém; IAM por path; custo zero; legível por Lambda e pipelines | Contrato implícito, precisa ser documentado e testado |
| Outputs copiados para variáveis do GitHub | Simples | Manual, sujeito a erro, segredos espalhados |

---

## Decisão

**Runtime:** toda comunicação externa é **HTTP/REST síncrona** e entra por um **único
ponto, o Kong** (NLB → Kong → Lambda ou API). O Kong invoca a Lambda diretamente pelo
plugin `aws-lambda` (sem AWS API Gateway) e encaminha `/api/*` ao Service da aplicação. Não
há chamada direta entre Lambda e aplicação: as duas compartilham apenas o banco (a Lambda
só lê `customers`) e o segredo JWT.

**Entre repositórios:** contratos publicados no **SSM** em
`/techchallenge/<env>/...` e rede descoberta por **tags** (`Project`, `Environment`,
`ManagedBy`, `kubernetes.io/role/internal-elb`).

| Parâmetro | Publicado por | Consumido por |
|---|---|---|
| `/techchallenge/<env>/auth/jwt-secret` | Infra K8s | Lambda (runtime), deploy da App (Secret + KongConsumers) |
| `/techchallenge/<env>/eks/node-sg-id` | Infra K8s | Database (libera 5432) |
| `/techchallenge/<env>/db/{endpoint,host,port,name,username,password,client-sg-id}` | Database | Lambda (runtime), deploy da App (`DATABASE_URL`) |

---

## Consequências

### Positivas

* [✔️] Um único endereço público (NLB do Kong) e um único lugar para autenticação, rate
  limit, correlation-id e métricas.
* [✔️] Contrato da API documentado em OpenAPI/Swagger e Postman.
* [✔️] Cada repositório é aplicado e destruído de forma independente; o contrato SSM é
  conferido por teste e após cada deploy.

### Negativas / Riscos

* [⚠️] Acoplamento temporal: indisponibilidade do banco devolve `503` ao cliente (não há
  fila para absorver).
* [⚠️] Ordem de deploy obrigatória: Infra K8s → Database → Function → App.
* [⚠️] Mudar uma tag ou um path no SSM quebra os consumidores — por isso são tratados
  como contrato e documentados nos READMEs.

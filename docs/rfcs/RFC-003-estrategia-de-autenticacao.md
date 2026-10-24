# RFC 003: Estratégia de autenticação — Lambda emite, Kong valida, aplicação revalida

* **Status**: Aceita
* **Data**: 2026-10-06
* **Autores**: Time do Tech Challenge (13SOAT)
* **Repositórios afetados**: App, Function Serverless, Infra K8s
* **Documentos de origem**: [RFC-001 da App](https://github.com/DevKaue/tech-challenge-app-/blob/main/docs/RFCs/RFC-001-estrategia-de-autenticacao.md), [RFC-001 da Function](https://github.com/DevKaue/TechChallange---Function-Serverless/blob/main/docs/RFCs/RFC-001-autenticacao-por-cpf-com-jwt.md)

## Problema

Dois tipos de usuário consomem a API:

* **Funcionários** (atendente e mecânico): login por e-mail e senha, já existente na
  aplicação (`POST /api/auth/login`).
* **Clientes**: a Fase 3 exige autenticação **apenas pelo CPF**, implementada como
  **Function Serverless** separada da aplicação, que consulta a existência e o status do
  cliente e devolve um JWT.

O API Gateway precisa validar os dois tipos de token sem conhecer a regra de negócio de
cada um, e as rotas sensíveis precisam estar protegidas.

## Alternativas consideradas

| Alternativa | A favor | Contra | Resultado |
|---|---|---|---|
| **Amazon Cognito** (user pool + custom auth) | Gerenciado, padrões OIDC | "Login só com CPF" exige fluxo custom com 3 Lambdas (define/create/verify challenge); outro diretório de usuários fora do Postgres; restrições do AWS Academy | Descartada |
| **Autenticação dentro da aplicação** | Nenhuma peça nova | Não atende o requisito de Function Serverless | Descartada |
| **AWS API Gateway + Lambda Authorizer** | Gerenciado | Authorizer por requisição (latência + custo), VPC Link, config fora do cluster | Substituída pelo plugin `jwt` do Kong ([ADR-003](../adrs/ADR-003-kong-como-api-gateway.md)) |
| **JWT RS256 + JWKS** | Só a Lambda teria a chave privada | Publicar/rotacionar JWKS e alterar o `JwtStrategy` da aplicação | Adiada (próximo passo natural) |
| **JWT HS256 com segredo compartilhado no SSM** (proposta) | Sem mudança no validador da aplicação; validação barata no gateway | Rotação exige redeploy dos três consumidores | **Escolhida** |

## Proposta

### Um contrato de token, dois emissores

| Emissor | `iss` | Claims | KongConsumer |
|---|---|---|---|
| Lambda `techchallenge-auth-<env>` (`POST /auth`) | `techchallenge-auth-lambda` | `sub` (id do cliente), `type: customer`, `jti`, `iat`, `exp` (1 h) | `auth-lambda` |
| Aplicação (`POST /api/auth/login`) | `techchallenge-app` | `sub` (id do usuário), `email`, `role`, `exp` | `app-staff` |

* Algoritmo **HS256**, segredo único por ambiente em
  `/techchallenge/<env>/auth/jwt-secret` (SSM), gerado pelo repositório de Infra K8s.
* A Lambda lê o segredo em runtime; o deploy da aplicação monta o Secret do app e as
  credenciais dos KongConsumers a partir do mesmo parâmetro.
* O token do cliente **não** contém CPF, nome, e-mail ou telefone.

### Regras da Function Serverless

1. Valida formato e dígitos verificadores do CPF **antes** de qualquer I/O → `400`.
2. Consulta `customers` por `(document, document_type = 'CPF')` (índice único).
3. Elegível somente se `status = ACTIVE` **e** `deleted_at IS NULL`.
4. Não cadastrado e inelegível retornam o **mesmo `401`** — impede enumeração de CPF.
5. Banco/SSM indisponível → `503`; nunca emite token em caso de dúvida.

### Camadas de proteção

| Camada | O que garante |
|---|---|
| Kong `rate-limiting` (10/min por IP) em `/auth` e `/api/auth/login` | Mitiga força bruta e enumeração |
| Kong `jwt` em `/api/*` (consumer pelo `iss`, verifica assinatura e `exp`) | Token **autêntico**; requisição inválida não chega à aplicação |
| Aplicação (`JwtAuthGuard` + `RolesGuard` + `@CustomerAccess`) | Token **autorizado**: papel do funcionário, cliente ativo, posse da OS. Token de cliente só passa em rotas marcadas, e só para as próprias OS |

A aplicação continua validando o token (defesa em profundidade): sem isso a API ficaria
aberta a quem alcançasse o Service por dentro do cluster.

## Riscos aceitos

* O gateway não consulta o banco: um token de cliente bloqueado continua passando pelo Kong
  até expirar (até 1 h). Quem barra é a aplicação, que a cada requisição confere se o
  cliente do `sub` ainda está ativo (`ValidateAuthenticatedCustomerUseCase`).
* Rotação do segredo HS256 exige redeploy de Lambda, aplicação e Kong.

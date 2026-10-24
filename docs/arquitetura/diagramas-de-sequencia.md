# Diagramas de Sequência

Dois fluxos centrais da Fase 3: **autenticação** (cliente por CPF via Function Serverless e
funcionário por e-mail/senha na aplicação) e **abertura de ordem de serviço**, incluindo o
acompanhamento da OS pelo próprio cliente com o token emitido pela Lambda.

Todo o tráfego externo entra pelo **Kong** (API Gateway). O Kong valida o JWT antes de a
requisição chegar à aplicação, e a aplicação valida de novo (defesa em profundidade) —
decisão registrada na [RFC-003](../rfcs/RFC-003-estrategia-de-autenticacao.md) e no
[ADR-003](../adrs/ADR-003-kong-como-api-gateway.md).

## 1. Autenticação do cliente por CPF

```mermaid
sequenceDiagram
    autonumber
    actor C as Cliente
    participant K as Kong (EKS)
    participant L as Lambda techchallenge-auth-{env}
    participant S as SSM Parameter Store
    participant DB as RDS PostgreSQL

    C->>K: POST /auth {"cpf": "111.444.777-35"}
    K->>K: rate-limiting (10/min por IP) + gera X-Correlation-ID
    K->>L: Invoke (plugin aws-lambda, evento REST v1.0)

    opt primeira invocação do container (cold start)
        L->>S: GetParameter jwt-secret e /db/host, port, name, username, password
        alt falha ao ler SSM ou montar dependências
            L-->>K: 503 service_unavailable + correlationId
            K-->>C: 503
        else dependências montadas
            S-->>L: valores (cache por container)
        end
    end

    L->>L: valida formato e dígitos verificadores do CPF (antes de qualquer I/O)
    alt CPF inválido / JSON malformado
        L-->>K: 400 invalid_request "CPF inválido"
        K-->>C: 400
    else CPF válido
        L->>DB: SELECT id, status, deleted_at FROM customers<br/>WHERE document = $1 AND document_type = 'CPF'
        alt banco indisponível
            L-->>K: 503 service_unavailable
            K-->>C: 503
        else não cadastrado, arquivado, INACTIVE ou BLOCKED
            DB-->>L: nenhuma linha / cliente inelegível
            L-->>K: 401 "CPF não autorizado" (mesma resposta, evita enumeração)
            K-->>C: 401
        else cliente ACTIVE e não arquivado
            DB-->>L: id, status = ACTIVE, deleted_at = null
            L->>L: assina JWT HS256 (sub = id, type = customer,<br/>iss = techchallenge-auth-lambda, jti, exp = 1 h)
            L-->>K: 200 {access_token, token_type: Bearer, expires_in: 3600}
            K-->>C: 200 + x-correlation-id
        end
    end
```

## 2. Autenticação do funcionário e abertura da ordem de serviço

```mermaid
sequenceDiagram
    autonumber
    actor F as Funcionário (atendente)
    participant K as Kong (EKS)
    participant A as API NestJS (EKS)
    participant DB as RDS PostgreSQL

    F->>K: POST /api/auth/login {email, password}
    K->>K: rota pública + rate-limiting (10/min por IP)
    K->>A: encaminha com X-Correlation-ID
    A->>DB: SELECT users WHERE email = $1
    DB-->>A: usuário + password_hash
    A->>A: compara hash (scrypt) e assina JWT HS256<br/>(iss = techchallenge-app, sub, role)
    A-->>K: 200 {access_token}
    K-->>F: 200 token

    F->>K: POST /api/service-orders (Authorization: Bearer token)<br/>{customerId, vehicleId, mileage, notes, services?, parts?}
    K->>K: plugin jwt: escolhe o consumer pelo iss,<br/>verifica assinatura HS256 e exp
    alt token ausente, adulterado ou expirado
        K-->>F: 401 (a requisição não chega à aplicação)
    else token válido
        K->>A: encaminha com X-Correlation-ID
        A->>A: JwtAuthGuard revalida o token<br/>(token de cliente nesta rota → 403)
        A->>DB: busca cliente e veículo (CustomerManagementFacade)
        alt cliente/veículo inexistente ou veículo de outro dono
            A-->>K: 404 / 422
            K-->>F: erro de validação
        else dados consistentes
            A->>DB: BEGIN (unit of work)
            A->>DB: INSERT service_orders (status RECEIVED)
            A->>DB: INSERT service_order_status_history (null → RECEIVED)
            opt serviços ou peças informados
                A->>DB: UPDATE status IN_DIAGNOSIS + histórico
                A->>DB: INSERT estimates + estimate_items (orçamento inicial)
            end
            A->>DB: COMMIT
            A-->>K: 201 {id, status, ...}
            K-->>F: 201 OS aberta
        end
    end
```

## 3. Fluxo completo: cliente acompanha e aprova a própria OS

Depois que a OS existe, o cliente usa o token emitido pela Lambda para acompanhá-la. As
rotas marcadas com `@CustomerAccess()` aceitam o token `type: customer`, desde que a OS
seja do próprio cliente (`sub` = `service_orders.customer_id`).

```mermaid
sequenceDiagram
    autonumber
    actor C as Cliente
    participant K as Kong (EKS)
    participant L as Lambda auth
    participant A as API NestJS
    participant DB as RDS PostgreSQL

    C->>K: POST /auth {cpf}
    K->>L: Invoke (aws-lambda)
    L->>DB: consulta existência e status do cliente
    L-->>K: 200 {access_token (type = customer)}
    K-->>C: 200 token

    C->>K: GET /api/service-orders/{id}/status (Bearer token)
    K->>K: plugin jwt (consumer auth-lambda pelo iss)
    K->>A: encaminha
    A->>A: valida token, carrega o cliente (ACTIVE e não arquivado)
    A->>DB: SELECT service_orders WHERE id = $1 AND customer_id = sub
    alt OS de outro cliente
        A-->>K: 404 (fora do escopo do cliente)
        K-->>C: 404
    else OS do próprio cliente
        A-->>K: 200 {status: WAITING_APPROVAL, ...}
        K-->>C: 200
    end

    C->>K: PATCH /api/service-orders/estimates/{estimateId}/status {status: APPROVED}
    K->>A: encaminha (JWT validado)
    A->>DB: atualiza orçamento + histórico de status da OS
    A-->>K: 200
    K-->>C: 200 orçamento aprovado
```

> Fontes Mermaid relacionadas nos repositórios:
> [`09-sequencia-autenticacao-os.mmd`](https://github.com/DevKaue/tech-challenge-app-/blob/main/docs/diagrams/09-sequencia-autenticacao-os.mmd) (App) e
> [`02-sequencia-autenticacao.mmd`](https://github.com/DevKaue/TechChallange---Function-Serverless/blob/main/docs/diagrams/02-sequencia-autenticacao.mmd) (Function).

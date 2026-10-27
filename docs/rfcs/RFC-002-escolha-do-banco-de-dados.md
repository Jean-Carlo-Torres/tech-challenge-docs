# RFC 002: Escolha do banco de dados gerenciado — Amazon RDS for PostgreSQL

* **Status**: Aceita
* **Data**: 2026-09-27
* **Autores**: Time do Tech Challenge (13SOAT)
* **Repositórios afetados**: Database (provisiona), App (dona do schema), Function (leitura)
* **Decisão resultante**: [ADR-001 do repositório de banco](https://github.com/DevKaue/tech-challenge-database-infra/blob/main/docs/ADRs/ADR-001-rds-postgresql-gerenciado.md)
* **Documento detalhado**: [RFC-001 do repositório de banco](https://github.com/DevKaue/tech-challenge-database-infra/blob/main/docs/RFCs/RFC-001-escolha-do-banco-gerenciado.md)

## Problema

Nas Fases 1 e 2 o PostgreSQL rodava em container (Docker Compose / StatefulSet). A Fase 3
exige um **banco de dados gerenciado**, separado da aplicação, com ciclo de vida e CI/CD
próprios, que atenda:

* a aplicação NestJS + Prisma, com **13 migrations** já existentes que usam enums nativos
  e **índice único parcial** (`vehicles.plate WHERE deleted_at IS NULL`);
* a Function Serverless de autenticação, que faz uma leitura indexada em `customers`;
* integridade referencial forte (OS, orçamentos e itens são documentos financeiros);
* custo compatível com Free Tier / AWS Academy.

## Alternativas consideradas

| Critério | **RDS PostgreSQL** (proposta) | Aurora Serverless v2 | Postgres no EKS (StatefulSet + EBS) | DynamoDB |
|---|---|---|---|---|
| Compatível com Prisma e migrations sem retrabalho | ✅ | ✅ | ✅ | ❌ reescrita total |
| Enums nativos e índice parcial | ✅ | ✅ | ✅ | ❌ |
| ACID + FKs no banco | ✅ | ✅ | ✅ | ⚠️ transações limitadas, sem FK |
| Gerenciado (backup, patch, failover) | ✅ | ✅ | ❌ operação manual | ✅ |
| Custo mínimo | ~US$ 0,47/dia (`db.t4g.micro`, Free Tier) | mínimo 0,5 ACU ~US$ 1,4/dia | barato, mas consome nós do cluster | pay-per-request |
| Disponível no AWS Academy | ✅ | ⚠️ restrições | ✅ | ✅ |
| Atende "banco gerenciado" do desafio | ✅ | ✅ | ❌ | ✅ |

* **Aurora Serverless v2**: mesma compatibilidade, mas o piso de capacidade custa ~3× o
  RDS para uma carga acadêmica, e a elasticidade não é necessária nesta escala.
* **Postgres no EKS**: não é gerenciado; acopla o ciclo de vida do dado ao do cluster —
  destruir o cluster colocaria os dados em risco.
* **DynamoDB**: o domínio é fortemente relacional (cliente → veículo → OS → orçamento →
  itens, com histórico). Modelar em chave-valor exigiria reescrever a camada de persistência
  e mover a integridade para a aplicação.

## Proposta

**Amazon RDS for PostgreSQL 15**, instância `db.t4g.micro`, uma por ambiente (`staging` e
`prod`), provisionada pelo repositório
[tech-challenge-database-infra](https://github.com/DevKaue/tech-challenge-database-infra):

* subnets privadas da VPC do EKS, **sem IP público**, storage cifrado, `rds.force_ssl = 1`;
* acesso somente a partir do SG dos nós do EKS e do SG de cliente usado pela Lambda;
* senha gerada por `random_password` e publicada em SSM `SecureString`;
* `prod` endurecido (`deletion_protection`, snapshot final), `staging` descartável;
* alarmes e dashboard no CloudWatch; log de queries > 1 s e `pg_stat_statements`.

## Ajustes no modelo relacional

O schema continua sendo das migrations do Prisma na aplicação. O ajuste exigido pela Fase 3
é a coluna **`customers.status`** (enum `ACTIVE | INACTIVE | BLOCKED`, `NOT NULL DEFAULT
'ACTIVE'`), consultada pela Lambda junto com `deleted_at`. Detalhes, diagrama ER e
relacionamentos em [banco-de-dados/modelo-relacional.md](../banco-de-dados/modelo-relacional.md).

## Riscos aceitos

| Risco | Por que é aceito | Gatilho para revisão |
|---|---|---|
| Single-AZ | Multi-AZ dobra o custo e sai do Free Tier | Dados reais de clientes / SLA |
| Sem RDS Proxy | Poucas conexões (Lambda com `max: 1`, API com pool fixo) | Alarme `rds-conexoes` (> 80) disparar |
| Usuário master compartilhado | Simplicidade do contrato SSM | Dados reais: usuário só com `SELECT` para a Lambda |

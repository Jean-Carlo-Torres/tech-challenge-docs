# ADR 002: Escalabilidade horizontal com HPA no EKS

* **Status**: Aceito
* **Data**: 2026-10-08
* **Autores**: Time do Tech Challenge (13SOAT)
* **Escopo**: App (manifestos Kustomize) e Infra K8s (cluster, Kong, metrics-server)

---

## Contexto

O desafio exige um **cluster Kubernetes com escalabilidade**. A carga da oficina é
irregular (picos na abertura e no fechamento do expediente) e o orçamento é limitado ao
crédito do AWS Academy. Precisávamos de escala automática para a aplicação e para o
gateway sem pagar capacidade ociosa.

---

## Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| Réplicas fixas | Previsível | Sobra capacidade ociosa ou falta no pico |
| **HPA (`autoscaling/v2`) por CPU** (escolhida) | Nativo, sem componente extra além do metrics-server; reage em ~1 min | Depende de `requests` de CPU bem definidos |
| HPA por CPU **e** memória | Mais sinais | Em Node.js o heap não devolve RSS após o pico: a memória travaria as réplicas no máximo (catraca) |
| KEDA (por eventos/métricas externas) | Escala por fila, latência, Prometheus | Componente a mais para operar; não há fila neste projeto |
| VPA | Ajusta requests automaticamente | Recria pods; conflita com HPA na mesma métrica |

---

## Decisão

1. **HPA da aplicação** (`k8s/base/hpa.yaml`): `minReplicas: 2`, `maxReplicas: 5`, alvo de
   **70% de CPU** sobre o request. Comportamento:
   * scale-up: até +100% a cada 60 s, janela de estabilização de 60 s;
   * scale-down: até −25% a cada 60 s, janela de 300 s (evita *flapping*).
2. **Somente CPU**, de propósito — memória fica protegida pelo `limit` (OOMKill), não pelo
   HPA.
3. **PodDisruptionBudget** com `minAvailable: 1` (sempre ≤ `minReplicas`), para drenagem
   de nó sem indisponibilidade.
4. **Kong** com autoscaling próprio (2..5 réplicas), `topologySpreadConstraints` entre
   zonas e `maxUnavailable: 1` no rolling update.
5. **metrics-server** instalado como addon do EKS pelo Terraform (sem ele o HPA fica
   `<unknown>`).
6. **Node group gerenciado** com `min 2 / desired 2 / max 4` nós em produção, para
   comportar o pico de réplicas.

---

## Consequências

### Positivas

* [✔️] A aplicação e o gateway absorvem picos sem intervenção manual.
* [✔️] Fora do pico, o custo volta ao mínimo de 2 réplicas.
* [✔️] Validado com teste de carga (`k6/stress-test.js` no repositório da App) observando
  `kubectl get hpa -w`.

### Negativas / Riscos

* [⚠️] Não há Cluster Autoscaler/Karpenter: se 5 réplicas da API + 5 do Kong não couberem
  nos nós atuais, pods ficam `Pending` até o node group ser aumentado (por PR no Terraform).
* [⚠️] `rate-limiting` do Kong com `policy: local` conta por pod: com N réplicas o limite
  efetivo é N × 10/min (limite por pod; ver ADR-002).
* [⚠️] Conexões no RDS crescem com as réplicas; o alarme `rds-conexoes` (> 80) é o gatilho
  para avaliar RDS Proxy.

# Guia de instalação — Dashboard e alertas de observabilidade EKS

Guia prático para reproduzir a plataforma descrita no artigo em **qualquer**
ambiente EKS + Grafana. Os exemplos usam placeholders (`SEU-...`, `NOME-DO-CLUSTER`,
`WORKSPACE_ID`) — substitua pelos valores do seu ambiente.

> Companion do dashboard `dashboard/eks-overview.json`.

---

## 0. Pré-requisitos

- `kubectl` e `helm` (>= 3.x) apontando para o cluster-alvo.
- Acesso ao cluster (no EKS via *access entry* ou `aws-auth`).
- Um **Grafana** (on-prem ou gerenciado) que consiga **alcançar** o Prometheus pela rede.
- Para o modo agente com destino gerenciado: **OIDC/IRSA** habilitado no cluster.

---

## 1. Instalar o Prometheus no cluster

Escolha **um** dos dois cenários conforme onde as métricas serão armazenadas.

### Cenário A — Stack completo (Prometheus local no cluster)

Ideal para começar (Fase 1) ou para clusters isolados. Instala Prometheus +
kube-state-metrics + node-exporter via `kube-prometheus-stack`.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring

cat > values-prom.yaml <<'EOF'
grafana:
  enabled: false            # usamos um Grafana externo/central
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests: { cpu: 500m, memory: 1Gi }
      limits:   { memory: 2Gi }
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources: { requests: { storage: 50Gi } }
EOF

helm upgrade --install kube-prom prometheus-community/kube-prometheus-stack \
  -n monitoring -f values-prom.yaml

kubectl -n monitoring get pods
```

### Cenário B — Modo agente (`remote_write` → armazenamento central)

Recomendado para escalar a frota (Fase 2). **Sem banco local** (~algumas centenas de
MB por cluster); só faz scrape e reenvia. O rótulo `externalLabels.cluster` é o que
habilita a variável **Cluster** do dashboard.

```bash
cat > values-agent.yaml <<'EOF'
grafana:      { enabled: false }
alertmanager: { enabled: false }
prometheus:
  agentMode: true                       # scrape + remote_write, sem TSDB
  prometheusSpec:
    externalLabels:
      cluster: "NOME-DO-CLUSTER"        # <-- IMPRESCINDÍVEL: identifica o cluster
    remoteWrite:
      - url: https://SEU-ENDPOINT-CENTRAL/api/v1/remote_write
        # se o destino exigir SigV4 (ex.: serviço gerenciado):
        # sigv4: { region: SUA-REGIAO }
# se usar IRSA para autenticar o remote_write:
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::SUA-CONTA:role/SEU-ROLE-REMOTEWRITE
EOF

helm upgrade --install kube-prom-agent prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f values-agent.yaml
```

> **GitOps:** em clusters com Flux/ArgoCD, versione esse `HelmRelease`/`Application`
> no repositório em vez de rodar `helm` à mão. Rollout controlado, rollback trivial.

### Scrapes adicionais recomendados (boas práticas EKS)

Para popular painéis de DNS e esgotamento de IP, adicione *ServiceMonitors*:

- **CoreDNS:** serviço `kube-dns`, porta `9153` (métricas `coredns_*`).
- **VPC CNI:** DaemonSet `aws-node`, porta `61678` (métricas `awscni_*`).

---

## 2. Conectar o Grafana ao Prometheus

O Grafana precisa **alcançar** o Prometheus pela rede.

- **Endpoint estável (recomendado):** exponha o Prometheus por um NLB/Ingress interno,
  restrito à origem do Grafana. Evite depender do IP do pod (muda a cada restart).
- Garanta **inbound TCP 9090** (ou a porta do seu endpoint) no security group.

### Opção 1 — Pela UI

1. **Connections → Data sources → Add new data source → Prometheus**.
2. **Name:** ex. `EKS-Prometheus`.
3. **URL:** `http://SEU-ENDPOINT-PROMETHEUS:9090`.
4. **Performance:** HTTP method `POST`, scrape interval `30s`.
5. **Save & test** → deve retornar "Successfully queried".

### Opção 2 — Provisioning (arquivo YAML)

```yaml
apiVersion: 1
datasources:
  - name: EKS-Prometheus
    uid: eks-prom
    type: prometheus
    access: proxy
    url: http://SEU-ENDPOINT-PROMETHEUS:9090
    isDefault: true
    jsonData: { httpMethod: POST, timeInterval: "30s" }
```

Monte em `/etc/grafana/provisioning/datasources/` e reinicie o Grafana.

### Destino gerenciado com SigV4 (opcional)

Se o armazenamento central for um serviço gerenciado que exige assinatura SigV4:

```yaml
- name: EKS-Central
  type: prometheus
  access: proxy
  url: https://SEU-WORKSPACE-URL
  jsonData: { httpMethod: POST, sigV4Auth: true, sigV4Region: SUA-REGIAO }
```

O Grafana precisa de credencial com permissão de leitura (`QueryMetrics`/`GetLabels`/`GetSeries`).

---

## 3. Importar o dashboard

1. **Dashboards → New → Import → Upload JSON** → `dashboard/eks-overview.json`.
2. No topo do dashboard, selecione seu datasource na variável **Datasource**.
3. Ajuste as variáveis **Cluster** e **Namespace** conforme necessário.

> Se a lista de datasource vier vazia, o datasource Prometheus ainda não existe —
> volte ao passo 2.

---

## 4. Alertas

O dashboard cobre a visualização; os alertas fecham o ciclo. As 13 regras cobrem:
Node NotReady, Memory/DiskPressure, CPU/memória/disco de nó altos, CrashLoopBackOff,
pods não-Running, deployment degradado, OOMKilled, CPU throttling e comprometimento
de CPU/memória — todas agregadas `by (cluster)`.

**Import no Grafana 12.x (UI):**

1. **Alerting → Alert rules → Import to Grafana-managed rules**.
2. Source = **Prometheus YAML file**; faça upload do seu arquivo de regras.
3. Em **Data source**, escolha o Prometheus do EKS e a pasta de destino.
4. **Roteamento:** defina o *contact point* por regra (ou via notification policy),
   apontando para o canal onde sua equipe já está (Slack, Google Chat, Teams...).

**Boas práticas de alerta:**

- Agregue `by (cluster)` para saber *qual* cluster disparou.
- Use `severity` (`critical`/`warning`) como label para rotear crítico → on-call
  e aviso → canal de observabilidade.
- Defina `for:` (duração) para evitar flapping — ex. `Node NotReady` por 5m.

---

## 5. Troubleshooting rápido

| Sintoma | Causa provável | Ação |
|---|---|---|
| Variável "Datasource" vazia | Nenhum datasource Prometheus criado | Criar o datasource (passo 2) antes de importar |
| "Save & test" falha (timeout) | Sem rota/SG para o Prometheus | Liberar inbound 9090; checar rotas de rede |
| Painéis vazios, datasource OK | Job de scrape ausente (ex.: apiserver) | Conferir `up by (job)` na seção de coleta |
| Filtro `cluster` vazio | 1 cluster sem `externalLabels` | Normal na Fase 1; resolvido no modo agente |
| Seção Control Plane vazia | `apiserver` não está sendo scrapeado | Habilitar o job `apiserver`/`kubernetes-apiservers` |

---

## Jobs de scrape exigidos pelo dashboard

`kube-state-metrics` · `node-exporter` · `kubernetes-cadvisor` + `kubelet` ·
`apiserver`. O `kube-prometheus-stack` já configura todos por padrão.

# grafana-eks — Observabilidade multi-cluster para EKS

Dashboard do Grafana + guia de instalação para observar uma frota de clusters
**Amazon EKS** com **Prometheus e Grafana**, seguindo boas práticas
(USE / RED / Golden Signals) e com um bloco de **indicadores antecipados**
(`predict_linear()`) para agir *antes* do incidente.

Material que acompanha o artigo:
**"Observabilidade de clusters EKS num Grafana on-premises: do voo cego à prevenção"**.

## Conteúdo

| Arquivo | O que é |
|---|---|
| [`dashboard/eks-overview.json`](dashboard/eks-overview.json) | Dashboard multi-cluster (46 painéis, 9 seções). Template genérico para importar. |
| [`guia-instalacao.md`](guia-instalacao.md) | Passo a passo: Prometheus (stack completo *ou* modo agente) + datasource + alertas. |
| [`assets/dashboard-eks.png`](assets/dashboard-eks.png) | Screenshot do dashboard. |

## Como usar

1. Instale o `kube-prometheus-stack` no cluster (ver [`guia-instalacao.md`](guia-instalacao.md)).
2. Aponte seu Grafana para o Prometheus (novo datasource).
3. No Grafana: **Dashboards → New → Import → Upload JSON** e envie
   `dashboard/eks-overview.json`. Selecione seu datasource na variável **Datasource** no topo.

O dashboard é parametrizado por **datasource**, **cluster** (múltipla escolha) e
**namespace** — serve tanto para um único cluster quanto para uma frota multi-cluster.

> Os artefatos são **templates genéricos**: não contêm endpoints, contas ou
> credenciais. Ajuste URLs, rótulos de `cluster` e thresholds à sua realidade.

## Licença

Use e adapte livremente. Contribuições e sugestões são bem-vindas via issues.

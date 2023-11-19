# 使用 ArgoCD 建立 Kubernetes 基礎建設


當一個叢集建立好之後總會需要安裝 Metrics 相關服務，這邊使用 ArgoCD 建立相關應用程式，讓 Cluster 加入之後就可以直接用 ArgoCD 部屬 Metrics-Server / Prometheus-adapter 等服務。

1. CLI Command - 設定 Cluster/加入 Repository
2. Metrics-Server 的 Application 設定 yaml
3. Prometheus-adapter 的 Application 設定 yaml

<!--more-->

# ArgoCli

* 設定 Cluster

```bash
argocd cluster add srvk8s-admin@srvk8s --name Master-Cluster
```

* 加入 Repo

```bash
argocd repo add https://prometheus-community.github.io/helm-charts --type helm --name Prometheus
argocd repo add https://kubernetes.github.io/dashboard --type helm --name kubernetes-dashboard
argocd repo add https://kubernetes-sigs.github.io/metrics-server --type helm --name metrics-server
```

# Metrics-Server

[helm repository](https://artifacthub.io/packages/helm/metrics-server/metrics-server)

* ArgoCD Application yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server-master-cluster
spec:
  destination:
    name: Master-Cluster
    namespace: kube-system
    server: ''
  source:
    path: ''
    repoURL: 'https://kubernetes-sigs.github.io/metrics-server/'
    targetRevision: 3.8.2
    chart: metrics-server
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: metrics.enabled
          value: 'true'
      values: |-
        args:
          - --kubelet-insecure-tls
  project: cluster-infra
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true

```

# Prometheus-adapter

{{<admonition warning >}}
此方法搭配 Istio & Istio 其中佩附的 Prometheus，是 Metrics-Server 的替代作法。

由於 metrics-server 安裝後無法在 [ lens ](https://k8slens.dev/) (kubernetes IDE) 上看到各個 pod 的資源狀態，如果 cluster 的資源夠多，我會建議使用這個做法。
{{</admonition >}}

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-adapter-Master-Cluster
spec:
  destination:
    name: Master-Cluster
    namespace: istio-system
    server: ''
  source:
    path: ''
    repoURL: 'https://prometheus-community.github.io/helm-charts'
    targetRevision: 3.0.1
    chart: prometheus-adapter
    helm:
      valueFiles:
        - values.yaml
      values: |-
        rules:
          default: true
          resource:
            cpu:
              containerQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, container!=""}[5m])) by (<<.GroupBy>>)
              nodeQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, id='/'}[5m])) by (<<.GroupBy>>)
              resources:
                overrides:
                  node:
                    resource: node
                  namespace:
                    resource: namespace
                  pod:
                    resource: pod
              containerLabel: container
            memory:
              containerQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>, container!=""}) by (<<.GroupBy>>)
              nodeQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>,id='/'}) by (<<.GroupBy>>)
              resources:
                overrides:
                  node:
                    resource: node
                  namespace:
                    resource: namespace
                  pod:
                    resource: pod
              containerLabel: container
            window: 5m
          custom:
          - seriesQuery: '{__name__=~"istio_requests_total"}'
            seriesFilters: []
            resources:
              overrides:
                kubernetes_namespace:
                  resource: namespace
                kubernetes_pod_name:
                  resource: pod
                destination_service_name:
                  resource: service
            name:
              matches: "^(.*)_total"
              as: "${1}_per_min"
            metricsQuery: sum(increase(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
      parameters:
        - name: prometheus.url
          value: 'http://prometheus.istio-system.svc.cluster.local'
  project: cluster-infra
  syncPolicy:
    syncOptions: []

```

# Kubernetes Dashboard

{{<admonition danger >}}
這個設定我尚未完成測試使用，僅成功安裝 & 利用 Lens 的 Forward 功能連進去，但是登入的部分碰到困難。

另外我的環境都有 istio，之後等測試 OK 的時候會再補上 istio 的 ingress 相關設定
{{</admonition >}}

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-dashboard-master-cluster
spec:
  destination:
    name: Master-Cluster
    namespace: kube-dashboard
    server: ''
  source:
    path: ''
    repoURL: 'https://kubernetes.github.io/dashboard/'
    targetRevision: 5.3.1
    chart: kubernetes-dashboard
    helm:
      valueFiles:
        - values.yaml
  project: cluster-infra
  syncPolicy:
    syncOptions:
      - PruneLast=true
      - CreateNamespace=true

```


# 使用 k3d 快速建立本機 Kubernetes (k3s) 叢集以用於測試與開發


本文紀錄在本機使用 k3d CLI 快速使用 k3s 建立 kubernetes 叢集的方法

<!--more-->

# k3d

{{<admonition info>}}
k3d 是社群開發的 k3s in docker 的快速建立叢集的工具，方便在 docker 內使用 k3s 建立 kubernetes 叢集
由於是純社群開發的工具，且因為是跑在 docker 內部，我會建議用在 kubernetes 上的服務開發用途，而非正式服務部署
{{</admonition>}}

## 語法紀錄

{{<admonition info>}}
前置作業：安裝 docker
{{</admonition>}}

使用 k3d 建立 k8s 叢集，只有開放 k8s api server port 的命令

{{<admonition warning>}}
這邊的 cluster 建立命令資訊如下
叢集名稱 = service-lab
api server (master node) 數量 = 1
agent server (worker node) 數量 = 6

以下命令有開本機的 8080 port 配對到 k3s 的 loadbalance (80 port)，有需要開其他 port 可以另外設定
{{</admonition>}}

因為我這邊使用的服務都有搭配 istio 做事，跟 k3d 內建的網路工具 traefik 有衝突，所以要關掉
```bash
k3d cluster create service-lab --servers 1 --agents 6 --port 8080:80@loadbalancer --api-port 6443 --k3s-arg '--disable=traefik@server:0'
```

如果沒有要關的話，命令如下
```bash
k3d cluster create service-lab --servers 1 --agents 6 --api-port 6443 --port 8080:80@loadbalancer
```

```bash
# 建立完叢集才想到要暴露 k8s 服務的進入 port 的時候可以這樣加
k3d cluster edit service-lab --port-add 8080:80@loadbalancer

# or when create cluster
k3d cluster create service-lab --servers 1 --agents 6 --port 8080:80@loadbalancer --api-port 6443 --k3s-arg '--disable=traefik@server:0'
```

移除 Cluster
```bash=
k3d cluster delete service-lab
```

停止 Cluster
```bash=
k3d cluster stop service-lab
```

啟動 Cluster
```bash=
k3d cluster start service-lab
```

## 參考資料

* [k3s官網](https://k3s.io/)
* [k3d 官網](https://k3d.io/)
* [在k3d上快速安装Istio，助你在本地灵活使用K8S！](https://dockone.io/article/9917)
* [K3d and Istio (Service Mesh - Governing the data plane)](https://gist.github.com/rafi/ebc50f0b0cd12dc061e4c69b3ee2521c)


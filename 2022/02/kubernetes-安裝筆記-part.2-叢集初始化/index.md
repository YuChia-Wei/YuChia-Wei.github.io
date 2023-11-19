# Kubernetes 安裝筆記 part.2 : 叢集初始化


使用 Kubeadm 初始化 Kubernetes Cluster.

{{<admonition info >}}
|||
|--|--|
|OS:| CentOs 7 (kernel = 3.10.1)|
|K8s:| v1.23.4 (新安裝時預設使用當下的最新版，因此此版本資訊參考用)|
{{</admonition >}}

<!--more-->

# Init Kubernetes Cluster

> 此篇安裝 kubernetes 的方法是使用 kubeadm

## kubeadm init

安裝時建議

1. 增加 `--upload-certs` 參數，這樣未來要加入其他 Master (Control plane) Node 以建立 HA 環境時方便共用憑證。

2. 設定 --control-plane-endpoint ，並使用已註冊於 DNS 的 endpoint，可以讓叢集內的 Node 不用倚賴 Node IP 來與 Master 溝通

```bash
kubeadm init --control-plane-endpoint=<server in dns> --upload-certs
```

其它 Node 要用 Master 形式加入的話就可以用下面這個命令
```bash
kubeadm join internal.k8s.master:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash> \
        --control-plane --certificate-key <cert>
```

其它 Node 要用 worker 的形式加入 cluster 的話要使用以下命令
```bash
kubeadm join internal.k8s.master:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```

重新產生 Token
```bash
kubeadm token create
```

取得 cert
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Single Node Setting

需要在 master node 中部屬服務時需要下這個指令

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 部屬 weave 網路 (CNI)

* 部屬 weave 網路 (CNI)

    * 查看 weave net pods 的部屬設定
        ```bash
        kubectl describe -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
        ```

    * 部屬 weave net pods
        ```bash
        kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
        ```

    * 移除 weave net pods 的部屬
        ```bash
        kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
        ```

* 部屬 weave net 的時候如果無法部屬成功，可能是因為機器上已有路由表
    REF 1. https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#troubleshooting
    REF 2. https://tonybai.com/2016/12/30/install-kubernetes-on-ubuntu-with-kubeadm-2/

    weavenet 預設使用的路由為 10.32.0.0/12 (可用 ip 網段範圍為 10.32.0.0 ~ 10.47.0.0)，須注意要加入 k8s 叢集的主機上是否有路由衝突，若有，會導致新加入的機器上面的 weavenet 服務部屬失敗

    可以使用以下命令確認路由
    ```bash
    netstat -rn
    ```

    修改後的 apply 命令
    ```bash
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=172.30.0.0/16"
    kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=172.30.0.0/16"
    ```

* 安裝時因為 weave net image 的來源不是 docker hub，而是 "ghcr.io"，如果有網路環境問題的話，所以可以在 docker daemon 中調整設定

```json
 "insecure-registries":["ghcr.io"],
```

## 安裝 HELM (建議)

> kubernetes 通常使用 HELM 來安裝套件，建議在 master 上安裝 helm 工具

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```


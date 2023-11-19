# Istio Install - Use istioctl


使用 Istio 官方 CLI (istioctl) 安裝 Istio

<!--more-->

# 官方參考資料

* 用官方工具安裝 : https://istio.io/latest/docs/setup/install/istioctl/
* 用 HELM 安裝 : https://istio.io/latest/docs/setup/install/helm/

# Use istioctl

download
```bash
# Latest
curl -L https://istio.io/downloadIstio | sh -

# specific version
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.0 TARGET_ARCH=x86_64 sh -
```

記得到參數檔案內設定 istioctl 的執行檔位置
```bash
vim ~/.bash_profile

# 把 istio 工具加入執行路徑
PATH=$PATH:$HOME/bin:/root/istio-1.12.0/bin
#PATH=$PATH:$HOME/bin:/root/istio-1.12.1/bin
```

如果是在 Single Node Cluster 的環境下，記得下這個指令才能在 Master (Control plane) 安裝服務

{{<admonition info >}}
istio 除了 core 之外都是屬於一般服務，如果沒有下 taint 的指令會無法安裝成功
{{</admonition >}}

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-

# kubernetes 1.24 要改用這個
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

在正式環境上，建議最小安裝，後續其他東西慢慢個別裝

```bash
istioctl install --set profile=minimal
```

{{<admonition warning >}}
istio cni 只是 k8s cni 的擴充套件，仍需要安裝 weave net cni 等 cni 服務
[官方安裝資料](https://istio.io/latest/docs/setup/additional-setup/cni/)
{{</admonition >}}

使用最小 + istio cni
```bash
istioctl install --set profile=minimal --set components.cni.enabled=true
```

{{<admonition info >}}
建議使用最小 + CNI
Ingress / Egress Gateway 都可以之後再依據實際的網路需求安裝 (例如要控制對外開放的 port 之類的)
只是要注意，Ingress / Egress Gateway 的自訂安裝需要在同一個檔案，不能個別製作檔案再用 istioctl 工具安裝，因為不存在於當次安裝使用的 yaml 檔中的 gateway 設定會被 istioctl 工具清掉
{{</admonition >}}

使用 default + istio cni

```bash
istioctl install --set components.cni.enabled=true
```

--取得 istio ingress gateway 的狀態 (使用官方提供的預設安裝檔時可用)

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

--查 k8s ingress 設定狀況

```bash
kubectl get ingress --all-namespaces
```

--查 k8s gateway 設定狀況

```bash
kubectl get gateway --all-namespaces
```


# Set SiteCar
```bash
kubectl label namespace default istio-injection=enabled --overwrite
kubectl get namespace -L istio-injection
```

{{<admonition warning >}}
如果車掛不上去，或是掛上去之後發現無法部屬，請檢查 kube-apiserver 這個 pod 的環境變數
當 api server 有掛 proxy 的話，會無法掛車

REF (SCH): https://istio.io/latest/zh/docs/ops/common-problems/injection/#automatic-sidecar-injection-fails-if-the-Kubernetes-API-server-has-proxy-settings
REF (ENG): https://istio.io/latest/docs/ops/common-problems/injection/#automatic-sidecar-injection-fails-if-the-kubernetes-api-server-has-proxy-settings

有兩種解法 (建議方法 2) ：

重啟方式的參考 : https://stackoverflow.com/a/51763158

1. 調整環境變數並重啟 (刪除) pods/kube-apiserver，讓 apiserver 套用設定
    * 設定環境變數
    ```bash
    export NO_PROXY=NO_PROXY=istio-sidecar-injector.istio-system.svc,.svc
    ```
    * 刪除 kube-apiserver (因為 api server 是由 k8s 系統管理，刪除後會自動重建)
    ```bash
    kubectl delete po kube-apiserver-<node name> -n kube-system
    ```
    * 如果有定義開機 (登入) 後自動建立 PROXY 的環境變數，將 NO_PROXY 設定加入
    ```bash
    vim ~/.bash_profile

    # 把 NO_PROXY 設定檔加到環境變數
    export NO_PROXY=NO_PROXY=istio-sidecar-injector.istio-system.svc,.svc

    source ~/.bash_profile
    ```
2. 編輯 api server 的設定檔後重啟
    * 開啟設定檔
    ```
    vim /etc/kubernetes/manifests/kube-apiserver.yaml
    ```
    * 找到 env 節點並加入 NO_PROXY 設定
    ```yaml
    env:
    - name: HTTPS_PROXY
      value: <your proxy>
    - name: NO_PROXY
      value: istio-sidecar-injector.istio-system.svc,.svc
    ```
    * 刪除 kube-apiserver (因為 api server 是由 k8s 系統管理，刪除後會自動重建)
    ```
    kubectl delete po kube-apiserver-<node name> -n kube-system
    ```
{{</admonition >}}

# 解除安裝 Istio

```bash
istioctl x uninstall --purge
istioctl x uninstall --set profile=minimal --purge
kubectl delete namespace istio-system
```

# 驗證 Instio 安裝
```bash
istioctl manifest generate > $HOME/generated-manifest.yaml
istioctl manifest generate --set profile=minimal > $HOME/generated-manifest.yaml
istioctl verify-install -f $HOME/generated-manifest.yaml
```

